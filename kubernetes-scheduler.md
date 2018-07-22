# Kubernetes-scheduler源码分析

### 1. 使用背景

```
以kubernetes1.9.2为背景，手动编译调试
# git clone https://github.com/kubernetes/kubernetes
# cd kubernetes
# git checkout v1.9.2
# make kube-scheduler
按照以上步骤即可编译1.9.2版本的kube-scheduler，二进制文件在_output/bin目录下
```

### 2. 入口

```
1.9.2版本的kube-scheduler还是位于plugin目录下，kubernetes/plugin/cmd/kube-scheduler/scheduler.go中，使用github上的"github.com/spf13/pflag"库，使用命令解析的方式启动代码

func main() {
	command := app.NewSchedulerCommand()

	pflag.CommandLine.SetNormalizeFunc(utilflag.WordSepNormalizeFunc)
	pflag.CommandLine.AddGoFlagSet(goflag.CommandLine)
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```

### 3. 配置初始化

```
app.NewSchedulerCommand()用来初始化所有的操作，配置项之类的，scheduler初始化经历了几个过程，主要使用scheme结构体，流程如下：
NewOptions()-->NewScheme()-->AddToScheme()-->init()-->addDefaultingFuncs回调()-->RegisterDefaults()-->SetObjectDefaults_KubeSchedulerConfiguration()-->SetDefaults_KubeSchedulerConfiguration() + SetDefaults_LeaderElectionConfiguration()-->ApplyDefaults()
直到ApplyDefaults函数才生成默认的配置，主要的配置文件位于kubernetes/pkg/apis/componentconfig和v1alpha1包下。
```

### 4. 再次初始化

```
配置初始化之后，只是有了一些初始的配置，启动schduler需要转换成NewSchedulerServer的配置，其中包括：
1.客户端，主要有三类：scheduler、lead选举、event；
2.InformerFactory：事件监听回调，必须Node的状态变化等；
3.健康检查、度量检查
整个初始化的过程在NewSchedulerServer()里面。
```

### 5. 启动

```
其中代码里有一个SchedulerConfig()函数，主要注册了各种监听回调器，使用了factory.NewConfigFactory()函数，例如Node就是监听了Core().V1().Nodes()这个地址，会一直去监听Node的变化，之后依次协程启动：
1.HealthzServer
2.MetricsServer
3.所有的Informer，每一个独立一个协程
4.scheduler controller同步Pod
5.选举过程单独启动
之后调用Scheduler.Run()方法进行调度，在调度之前，有一个VolumeBinder的过程，应该是为了保证Pod的调度和Volume一致的问题，防止冲突，在1.9.2上默认是关闭的。
```

### 6. 具体流程

```
调度的入口在scheduleOne处，从函数名可以看出，每次调度一个Pod到Node上
1.NextPod获取当前未调度的Pod
2.schedule函数通过一系列的算法获取到最佳Host
3.如果schedule失败，会去执行preempt函数通过抢占策略调度
4.调用assume函数将预期的Pod和Node联系起来
5.调用bind函数将信息写入到etcd中(协程实现)
```

#### 6.1 调度

```
默认的调度在generic_scheduler.go文件内，位于kubernetes/plugin/pkg/scheduler/core目录下
findNodesThatFit函数处理预选过程
PrioritizeNodes函数处理优选打分过程
优选完之后最终选取一个合适的节点地址返回
```

##### 6.1.1 预选

```
默认的预选算法初始化在kubernetes/plugin/pkg/scheduler/algorithmprovider/defaults/defaults.go下，默认的算法有：
1.NoVolumeZoneConflict
2.MaxEBSVolumeCount
3.MaxGCEPDVolumeCount
4.MaxAzureDiskVolumeCount
5.MatchInterPodAffinity
6.NoDiskConflict
7.GeneralPredicates
8.CheckNodeMemoryPressure
9.CheckNodeDiskPressure
10.CheckNodeCondition
11.PodToleratesNodeTaints
12.CheckVolumeBinding

预选接口是findNodesThatFit，调用了checkNode函数每一次最多检测16个和node的最大值去匹配Node，分别对每一个Node去做检测，判断该Node是否满足要求；
checkNode内调用podFitsOnNode函数匹配是否满足预选要求。
预选完成之后，如果出错或者没有匹配的Node，会直接返回结果；如果有一个Node匹配也是直接返回，之前的版本上没有对这一点进行修改，还是去执行优选，导致有一定的损耗。
```

##### 6.1.2 优选

```
优选接口是PrioritizeNodes，预选主要是通过打分的形式，最低是0，最高是10，并且还有一个weight的权重，最终得到最高的分数即为优选Node。
与预选一样，优选也是同时进行，最多16个Node上同时执行获取相应的Node信息，然后执行优选算法，默认的优选算法有：
1.SelectorSpreadPriority
2.InterPodAffinityPriority
3.LeastRequestedPriority
4.BalancedResourceAllocation
5.NodePreferAvoidPodsPriority
6.NodeAffinityPriority
7.TaintTolerationPriority
```

##### 6.1.3 最终选择

```
优选完成之后，如果存在多个相同评分的Node，则会有一个简单的robin轮询，找到最终的Node匹配。
```

#### 6.2 抢占

```
应该是在1.8版本开始，如果调度失败了，根据优先级的策略，依旧会有一个抢占的过程，抢占函数在Scheduler.preempt内。
```

#### 6.3 实现

```
通过预优选和抢占之后，如果没有退出，则肯定找到一个合适的Node可供调度，先将Pod缓存到Cache内，再将Pod和Node的关系绑定起来到cache内，调用Scheduler.assume函数，最终通过调用bind函数将Pod和Node绑定，并将信息写回到apiserver中，更新Pod和Node相关信息。
```

### 7. 选举

```
选举的主要过程就是去抢占获取到endpoint为kube-scheduler的实例，哪个kube-scheduler获取到了就是leader，通过加锁的方式进行leader election；当前拿到锁的无法工作时，别的实例会拿到锁，变为新的leader。
成为leader的scheduler会去执行整个调度过程，而没有获取到的会一直在等待锁的释放，直到自己成为leader的时候才去执行调度过程，这样就可以保证虽然有多个实例，但是同时只会有一个生效，只要一个有问题，锁就会被其他的实例拿到成为leader。
```

### 8. 理解

```
1.Node Pod Cache Binding
2.NodeSelector
3.NodeAffinity PodAffinity
4.Taint Toleration
4.Priority Preemption
5.DaemonSet是由DaemonSet Controller控制的，之后会改成Default Scheduler调度
```



## 参考文档

https://blog.csdn.net/WaltonWang/article/details/54565638