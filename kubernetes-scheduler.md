

# Kubernetes-scheduler源码分析

主要是获取PodSpec.NodeName为空的Pod，一直watch /v1/pod获取信息

## 1.源码入口

```
import (
	goflag "flag"
	"fmt"
	"math/rand"
	"os"
	"time"

	"github.com/spf13/pflag"
	utilflag "k8s.io/apiserver/pkg/util/flag"
	"k8s.io/apiserver/pkg/util/logs"
	"k8s.io/kubernetes/cmd/kube-scheduler/app"
	_ "k8s.io/kubernetes/pkg/client/metrics/prometheus" // for client metric registration
	_ "k8s.io/kubernetes/pkg/version/prometheus"        // for version metric registration
)
```

```
在cmd/kube-scheduler/scheduler.go下，为kube-scheduler main方法入口，主要使用"github.com/spf13/pflag"命令解析库，调用command.Execute()方法将命令解析，各种命令以及相应代码处理在cmd/kube-scheduler/app/server.go中，main主要代码如下：

func main() {
	rand.Seed(time.Now().UTC().UnixNano())

	command := app.NewSchedulerCommand()

	// TODO: once we switch everything over to Cobra commands, we can go back to calling
	// utilflag.InitFlags() (by removing its pflag.Parse() call). For now, we have to set the
	// normalize func and add the go flag set by hand.
	pflag.CommandLine.SetNormalizeFunc(utilflag.WordSepNormalizeFunc)
	pflag.CommandLine.AddGoFlagSet(goflag.CommandLine)
	// utilflag.InitFlags()
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
		os.Exit(1)
	}
}
```

```
Scheduler结构如下：
```

```
// Config is an implementation of the Scheduler's configured input data.
// TODO over time we should make this struct a hidden implementation detail of the scheduler.
type Config struct {
	// It is expected that changes made via SchedulerCache will be observed
	// by NodeLister and Algorithm.
	SchedulerCache schedulercache.Cache
	// Ecache is used for optimistically invalid affected cache items after
	// successfully binding a pod
	Ecache     *core.EquivalenceCache
	NodeLister algorithm.NodeLister
	Algorithm  algorithm.ScheduleAlgorithm
	GetBinder  func(pod *v1.Pod) Binder
	// PodConditionUpdater is used only in case of scheduling errors. If we succeed
	// with scheduling, PodScheduled condition will be updated in apiserver in /bind
	// handler so that binding and setting PodCondition it is atomic.
	PodConditionUpdater PodConditionUpdater
	// PodPreemptor is used to evict pods and update pod annotations.
	PodPreemptor PodPreemptor

	// NextPod should be a function that blocks until the next pod
	// is available. We don't use a channel for this, because scheduling
	// a pod may take some amount of time and we don't want pods to get
	// stale while they sit in a channel.
	NextPod func() *v1.Pod

	// WaitForCacheSync waits for scheduler cache to populate.
	// It returns true if it was successful, false if the controller should shutdown.
	WaitForCacheSync func() bool

	// Error is called if there is an error. It is passed the pod in
	// question, and the error
	Error func(*v1.Pod, error)

	// Recorder is the EventRecorder to use
	Recorder record.EventRecorder

	// Close this to shut down the scheduler.
	StopEverything chan struct{}

	// VolumeBinder handles PVC/PV binding for the pod.
	VolumeBinder *volumebinder.VolumeBinder

	// Disable pod preemption or not.
	DisablePreemption bool
}
```

```
1、生成config配置文件；
2、创建scheduler对象；
3、Prometheus注册和/metrics http接口提供；
4、内部选主操作。
```



## 2.后端核心代码

```
后端核心代码位于pkg/scheduler包下面，在pkg/scheduler/scheduler.go中，Scheduler.Run()方法开始真正的调度，调用Scheduler.scheduleOne()方法每次负责一个Pod的调度，方法如下：
```

```
// scheduleOne does the entire scheduling workflow for a single pod.  It is serialized on the scheduling algorithm's host fitting.
func (sched *Scheduler) scheduleOne() {
	pod := sched.config.NextPod()	//获取要调度的Pod信息
	if pod.DeletionTimestamp != nil {
		sched.config.Recorder.Eventf(pod, v1.EventTypeWarning, "FailedScheduling", "skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
		glog.V(3).Infof("Skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
		return
	}

	glog.V(3).Infof("Attempting to schedule pod: %v/%v", pod.Namespace, pod.Name)

	// Synchronously attempt to find a fit for the pod.
	start := time.Now()
	suggestedHost, err := sched.schedule(pod)
	if err != nil {
		// schedule() may have failed because the pod would not fit on any host, so we try to
		// preempt, with the expectation that the next time the pod is tried for scheduling it
		// will fit due to the preemption. It is also possible that a different pod will schedule
		// into the resources that were preempted, but this is harmless.
		// 如果失败了，通过抢占式调度
		if fitError, ok := err.(*core.FitError); ok {
			preemptionStartTime := time.Now()
			sched.preempt(pod, fitError)
			metrics.PreemptionAttempts.Inc()
			metrics.SchedulingAlgorithmPremptionEvaluationDuration.Observe(metrics.SinceInMicroseconds(preemptionStartTime))
		}
		return
	}
	metrics.SchedulingAlgorithmLatency.Observe(metrics.SinceInMicroseconds(start))
	// Tell the cache to assume that a pod now is running on a given node, even though it hasn't been bound yet.
	// This allows us to keep scheduling without waiting on binding to occur.
	assumedPod := pod.DeepCopy()

	// Assume volumes first before assuming the pod.
	//
	// If no volumes need binding, then nil is returned, and continue to assume the pod.
	//
	// Otherwise, error is returned and volume binding is started asynchronously for all of the pod's volumes.
	// scheduleOne() returns immediately on error, so that it doesn't continue to assume the pod.
	//
	// After the asynchronous volume binding updates are made, it will send the pod back through the scheduler for
	// subsequent passes until all volumes are fully bound.
	//
	// This function modifies 'assumedPod' if volume binding is required.
	err = sched.assumeAndBindVolumes(assumedPod, suggestedHost)
	if err != nil {
		return
	}

	// assume modifies `assumedPod` by setting NodeName=suggestedHost
	err = sched.assume(assumedPod, suggestedHost)
	if err != nil {
		return
	}
	// bind the pod to its host asynchronously (we can do this b/c of the assumption step above).
	go func() {
		err := sched.bind(assumedPod, &v1.Binding{
			ObjectMeta: metav1.ObjectMeta{Namespace: assumedPod.Namespace, Name: assumedPod.Name, UID: assumedPod.UID},
			Target: v1.ObjectReference{
				Kind: "Node",
				Name: suggestedHost,
			},
		})
		metrics.E2eSchedulingLatency.Observe(metrics.SinceInMicroseconds(start))
		if err != nil {
			glog.Errorf("Internal error binding pod: (%v)", err)
		}
	}()
}
```

```
suggestedHost, err := sched.schedule(pod)方法实现调度，调度接口为：

type ScheduleAlgorithm interface {
	//实现调度的接口
	Schedule(*v1.Pod, NodeLister) (selectedMachine string, err error)
	// Preempt receives scheduling errors for a pod and tries to create room for
	// the pod by preempting lower priority pods if possible.
	// It returns the node where preemption happened, a list of preempted pods, a
	// list of pods whose nominated node name should be removed, and error if any.
	Preempt(*v1.Pod, NodeLister, error) (selectedNode *v1.Node, preemptedPods []*v1.Pod, cleanupNominatedPods []*v1.Pod, err error)
	// Predicates() returns a pointer to a map of predicate functions. This is
	// exposed for testing.
	Predicates() map[string]FitPredicate
	// Prioritizers returns a slice of priority config. This is exposed for
	// testing.
	Prioritizers() []PriorityConfig
}
```

```
默认实现调度接口的调度器是genericScheduler，位于pkg/scheduler/core/generic-scheduler.go中，方法如下：
```

```
func (g *genericScheduler) Schedule(pod *v1.Pod, nodeLister algorithm.NodeLister) (string, error) {
	trace := utiltrace.New(fmt.Sprintf("Scheduling %s/%s", pod.Namespace, pod.Name))
	defer trace.LogIfLong(100 * time.Millisecond)

	if err := podPassesBasicChecks(pod, g.pvcLister); err != nil {
		return "", err
	}

	//从cache中获取可被调度的NodeList
	nodes, err := nodeLister.List()
	if err != nil {
		return "", err
	}
	if len(nodes) == 0 {
		return "", ErrNoNodesAvailable
	}

	// 更新一下NodeName
	// Used for all fit and priority funcs.
	err = g.cache.UpdateNodeNameToInfoMap(g.cachedNodeInfoMap)
	if err != nil {
		return "", err
	}

	trace.Step("Computing predicates")
	startPredicateEvalTime := time.Now()
	// 开始进行预选
	filteredNodes, failedPredicateMap, err := findNodesThatFit(pod, g.cachedNodeInfoMap, nodes, g.predicates, g.extenders, g.predicateMetaProducer, g.equivalenceCache, g.schedulingQueue, g.alwaysCheckAllPredicates)
	if err != nil {
		return "", err
	}

	if len(filteredNodes) == 0 {
		return "", &FitError{
			Pod:              pod,
			NumAllNodes:      len(nodes),
			FailedPredicates: failedPredicateMap,
		}
	}
	metrics.SchedulingAlgorithmPredicateEvaluationDuration.Observe(metrics.SinceInMicroseconds(startPredicateEvalTime))

	trace.Step("Prioritizing")
	startPriorityEvalTime := time.Now()
	// 只获取到一个，没得选择，只有它了
	// When only one node after predicate, just use it.
	if len(filteredNodes) == 1 {
		metrics.SchedulingAlgorithmPriorityEvaluationDuration.Observe(metrics.SinceInMicroseconds(startPriorityEvalTime))
		return filteredNodes[0].Name, nil
	}

	metaPrioritiesInterface := g.priorityMetaProducer(pod, g.cachedNodeInfoMap)
	// 获取到的Node比较多，那就打分了，先出最合适的
	priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)
	if err != nil {
		return "", err
	}
	metrics.SchedulingAlgorithmPriorityEvaluationDuration.Observe(metrics.SinceInMicroseconds(startPriorityEvalTime))

	trace.Step("Selecting host")
	return g.selectHost(priorityList)
}
```

```
预选算法接口
```

```
func findNodesThatFit(
	pod *v1.Pod,
	nodeNameToInfo map[string]*schedulercache.NodeInfo,
	nodes []*v1.Node,
	predicateFuncs map[string]algorithm.FitPredicate,
	extenders []algorithm.SchedulerExtender,
	metadataProducer algorithm.PredicateMetadataProducer,
	ecache *EquivalenceCache,
	schedulingQueue SchedulingQueue,
	alwaysCheckAllPredicates bool,
) ([]*v1.Node, FailedPredicateMap, error) {
	var filtered []*v1.Node
	failedPredicateMap := FailedPredicateMap{}

	if len(predicateFuncs) == 0 {
		filtered = nodes
	} else {
		// Create filtered list with enough space to avoid growing it
		// and allow assigning.
		filtered = make([]*v1.Node, len(nodes))
		errs := errors.MessageCountMap{}
		var predicateResultLock sync.Mutex
		var filteredLen int32

		// We can use the same metadata producer for all nodes.
		meta := metadataProducer(pod, nodeNameToInfo)

		var equivCacheInfo *equivalenceClassInfo
		if ecache != nil {
			// getEquivalenceClassInfo will return immediately if no equivalence pod found
			equivCacheInfo = ecache.getEquivalenceClassInfo(pod)
		}

		// 调用 podFitsOnNode 用配置好的预选策略对Node进行检查
		checkNode := func(i int) {
			nodeName := nodes[i].Name
			fits, failedPredicates, err := podFitsOnNode(
				pod,
				meta,
				nodeNameToInfo[nodeName],
				predicateFuncs,
				ecache,
				schedulingQueue,
				alwaysCheckAllPredicates,
				equivCacheInfo,
			)
			if err != nil {
				predicateResultLock.Lock()
				errs[err.Error()]++
				predicateResultLock.Unlock()
				return
			}
			if fits {
				filtered[atomic.AddInt32(&filteredLen, 1)-1] = nodes[i]
			} else {
				predicateResultLock.Lock()
				failedPredicateMap[nodeName] = failedPredicates
				predicateResultLock.Unlock()
			}
		}
		// 根据Node数量，最多同时启动16个协程完成预选Node
		workqueue.Parallelize(16, len(nodes), checkNode)
		filtered = filtered[:filteredLen]
		if len(errs) > 0 {
			return []*v1.Node{}, FailedPredicateMap{}, errors.CreateAggregateFromMessageCountMap(errs)
		}
	}

	// 如果配置了其他算法, 则执行相应算法再次进行筛选
	if len(filtered) > 0 && len(extenders) != 0 {
		for _, extender := range extenders {
			if !extender.IsInterested(pod) {
				continue
			}
			filteredList, failedMap, err := extender.Filter(pod, filtered, nodeNameToInfo)
			if err != nil {
				if extender.IsIgnorable() {
					glog.Warningf("Skipping extender %v as it returned error %v and has ignorable flag set",
						extender, err)
					continue
				} else {
					return []*v1.Node{}, FailedPredicateMap{}, err
				}
			}

			for failedNodeName, failedMsg := range failedMap {
				if _, found := failedPredicateMap[failedNodeName]; !found {
					failedPredicateMap[failedNodeName] = []algorithm.PredicateFailureReason{}
				}
				failedPredicateMap[failedNodeName] = append(failedPredicateMap[failedNodeName], predicates.NewFailureReason(failedMsg))
			}
			filtered = filteredList
			if len(filtered) == 0 {
				break
			}
		}
	}
	return filtered, failedPredicateMap, nil
}
```

```
默认的预选算法
```

```
func podFitsOnNode(
	pod *v1.Pod,
	meta algorithm.PredicateMetadata,
	info *schedulercache.NodeInfo,
	predicateFuncs map[string]algorithm.FitPredicate,
	ecache *EquivalenceCache,
	queue SchedulingQueue,
	alwaysCheckAllPredicates bool,
	equivCacheInfo *equivalenceClassInfo,
) (bool, []algorithm.PredicateFailureReason, error) {
	var (
		eCacheAvailable  bool
		failedPredicates []algorithm.PredicateFailureReason
	)
	predicateResults := make(map[string]HostPredicate)

	podsAdded := false
	// We run predicates twice in some cases. If the node has greater or equal priority
	// nominated pods, we run them when those pods are added to meta and nodeInfo.
	// If all predicates succeed in this pass, we run them again when these
	// nominated pods are not added. This second pass is necessary because some
	// predicates such as inter-pod affinity may not pass without the nominated pods.
	// If there are no nominated pods for the node or if the first run of the
	// predicates fail, we don't run the second pass.
	// We consider only equal or higher priority pods in the first pass, because
	// those are the current "pod" must yield to them and not take a space opened
	// for running them. It is ok if the current "pod" take resources freed for
	// lower priority pods.
	// Requiring that the new pod is schedulable in both circumstances ensures that
	// we are making a conservative decision: predicates like resources and inter-pod
	// anti-affinity are more likely to fail when the nominated pods are treated
	// as running, while predicates like pod affinity are more likely to fail when
	// the nominated pods are treated as not running. We can't just assume the
	// nominated pods are running because they are not running right now and in fact,
	// they may end up getting scheduled to a different node.
	for i := 0; i < 2; i++ {
		metaToUse := meta
		nodeInfoToUse := info
		if i == 0 {
			podsAdded, metaToUse, nodeInfoToUse = addNominatedPods(util.GetPodPriority(pod), meta, info, queue)
		} else if !podsAdded || len(failedPredicates) != 0 {
			break
		}
		// Bypass eCache if node has any nominated pods.
		// TODO(bsalamat): consider using eCache and adding proper eCache invalidations
		// when pods are nominated or their nominations change.
		eCacheAvailable = equivCacheInfo != nil && !podsAdded
		for _, predicateKey := range predicates.Ordering() {
			var (
				fit     bool
				reasons []algorithm.PredicateFailureReason
				err     error
			)
			//TODO (yastij) : compute average predicate restrictiveness to export it as Prometheus metric
			if predicate, exist := predicateFuncs[predicateKey]; exist {
				// Use an in-line function to guarantee invocation of ecache.Unlock()
				// when the in-line function returns.
				func() {
					var invalid bool
					if eCacheAvailable {
						// Lock ecache here to avoid a race condition against cache invalidation invoked
						// in event handlers. This race has existed despite locks in equivClassCacheimplementation.
						ecache.Lock()
						defer ecache.Unlock()
						// PredicateWithECache will return its cached predicate results.
						fit, reasons, invalid = ecache.PredicateWithECache(
							pod.GetName(), info.Node().GetName(),
							predicateKey, equivCacheInfo.hash, false)
					}

					if !eCacheAvailable || invalid {
						// we need to execute predicate functions since equivalence cache does not work
						fit, reasons, err = predicate(pod, metaToUse, nodeInfoToUse)
						if err != nil {
							return
						}

						if eCacheAvailable {
							// Store data to update equivClassCacheafter this loop.
							if res, exists := predicateResults[predicateKey]; exists {
								res.Fit = res.Fit && fit
								res.FailReasons = append(res.FailReasons, reasons...)
								predicateResults[predicateKey] = res
							} else {
								predicateResults[predicateKey] = HostPredicate{Fit: fit, FailReasons: reasons}
							}
							result := predicateResults[predicateKey]
							ecache.UpdateCachedPredicateItem(
								pod.GetName(), info.Node().GetName(),
								predicateKey, result.Fit, result.FailReasons, equivCacheInfo.hash, false)
						}
					}
				}()

				if err != nil {
					return false, []algorithm.PredicateFailureReason{}, err
				}

				if !fit {
					// eCache is available and valid, and predicates result is unfit, record the fail reasons
					failedPredicates = append(failedPredicates, reasons...)
					// if alwaysCheckAllPredicates is false, short circuit all predicates when one predicate fails.
					if !alwaysCheckAllPredicates {
						glog.V(5).Infoln("since alwaysCheckAllPredicates has not been set, the predicate" +
							"evaluation is short circuited and there are chances" +
							"of other predicates failing as well.")
						break
					}
				}
			}
		}
	}

	return len(failedPredicates) == 0, failedPredicates, nil
}
```

```
默认预选算法在pkg/scheduler/algorithmprovider/defaults/defaults.go中defaultPredicates()接口，主要有NoDiskConflict、GeneralPredicates、CheckNodeMemoryPressurePredicate、CheckNodeDiskPressurePredicate、CheckNodeConditionPredicate以及PodToleratesNodeTaints几种调度检测。
```

```
预选和优选之后，获取到最佳调度Node，调用assumedPod方法，将Node和Pod信息先缓存起来，然后起新的协程将Node和Pod信息Binding，写到etcd中去，当然工作还是apiserver来做。
```



## 参考文档

https://blog.csdn.net/WaltonWang/article/details/54565638