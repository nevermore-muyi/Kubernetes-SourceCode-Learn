

# Kubernetes-controller-manager源码分析

#### 初始化

```
Kube-controller-manager启动文件与其他组件类似，位于cmd下kube-controller-manager目录，通过调用NewCMServer()函数返回CMServer结构体，通过app.KnownControllers()调用确定需要启动的Controller，通过app.ControllersDisabledByDefault()调用确定默认不需要启动的Controller，最终通过app.Run()启动。
```

#### 启动过程

```
启动主要过程位于cmd/kube-controller-manager/app/controllermanager.go下Run()函数。主要有几个步骤：
1.配置合法性检测；
2.创建相应的客户端，包含kubeClient、leaderElectionClient以及kubeconfig；
3.启动http server，提供Prometheus监控；
4.选举/不选举启动
```

#### Controller类型

```
默认启动的Controller有以下类型：
EndpointController
ReplicationController
PodGCController
ResourceQuotaController
NamespaceController
ServiceAccountController
GarbageCollectorController
DaemonSetController
JobController
DeploymentController
ReplicaSetController
HPAController
DisruptionController
StatefulSetController
CronJobController
CSRSigningController
CSRApprovingController
CSRCleanerController
TTLController
BootstrapSignerController
TokenCleanerController
ServiceController
NodeController
RouteController
PersistentVolumeBinderController
AttachDetachController
VolumeExpandController
ClusterRoleAggregrationController
PVCProtectionController
以上就是所有的默认启动的Controller，其中有经常使用到的资源如Endpoint、Deployment，还有GC相关的如PodGCController，与证书相关的如CSRApprovingController、BootstrapSignerController，与Volume相关的如AttachDetachController、VolumeExpandController。
Controller的原理基本类似，主要就是利用informer与cache，通过对各种资源的List与Watch机制，对相关联的资源进行依赖，如Deployment需要关联Pods、RS以及Deployments，利用informer的回调，OnAdd、OnUpdate以及OnDelete函数对数据进行实时更新，主要依赖与client-go的一些功能。

可以通过samplecontroller的例子理解整体的使用。
```

