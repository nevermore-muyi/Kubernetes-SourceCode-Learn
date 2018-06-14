### Kube-arbitrator使用总结

#### 1. 原因

```
Kubernetes自身对资源的分配都是使用的绝对值，实现起来非常不方便，所以有了kube-arbitrator，实现资源按照比例分配，主要测试的是对不同的namespace的实现。
```

#### 2. 使用

##### 2.1 先决条件

```
Kubernetes对namespace控制主要使用ResourceQuota，所以apiserver启动时必须指定好 --enable-admission-plugins=ResourceQuota；
```

##### 2.2 使用步骤

```
1.创建QuotaAllocator，yaml文件如下：
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: quotaallocators.arbitrator.incubator.k8s.io
spec:
  group: arbitrator.incubator.k8s.io
  names:
    kind: QuotaAllocator
    listKind: QuotaAllocatorList
    plural: quotaallocators
    singular: quotaallocator
  scope: Namespaced
  version: v1

2.创建QuotaAllocator，该资源是用来控制资源分配比例的，参考yaml如下：
apiVersion: "arbitrator.incubator.k8s.io/v1"
kind: QuotaAllocator
metadata:
  name: xxx
  namespace: xxx
spec:
  weight: xxx
  request:
    resources:
      cpu: "xxx"
      memory: xxxGi
对每一个namespace都需要创建响应的QuotaAllocator，主要作用在weight变量，按照比例分配，weight越大，分配的空间越多

3.创建ResourceQuota
不同的namespace下创建好ResourceQuota，参考格式如下：
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ns01
  namespace: ns01
spec:
  hard:
    pods: "4"
    requests.cpu: "0"
    requests.memory: 0
    limits.cpu: "0"
    limits.memory: 0
    
4.开启kube-quotalloc
# mkdir -p $GOPATH/src/github.com/kubernetes-incubator/
# cd $GOPATH/src/github.com/kubernetes-incubator/
# git clone https://github.com/kubernetes-incubator/kube-arbitrator
# cd $GOPATH/src/github.com/kubernetes-incubator/kube-arbitrator
# make
# ./_output/bin/kube-quotalloc --kubeconfig /root/.kube/config
```

#### 3. 观察结果

```
# kubectl get quota ns01 -n ns01 -o yaml
可以看到不同的namespace下，quota按照比例分配了
```

#### 4.参考资料

https://github.com/kubernetes-incubator/kube-arbitrator

https://github.com/kubernetes-incubator/kube-arbitrator/blob/master/doc/usage/quotalloc_tutorial.md
