

# Kube-proxy源码分析

### 初始化

```
Kubernetes的初始化过程基本一致，都是通过 command := app.NewProxyCommand()开始启动，调用NewOptions函数，通过调用Run方法最终生成NewProxyServer配置，然后启动。
NewProxyServer中主要有以下操作：
1.配置检查
2.iptables工具生成
3.如果是cleanup模式，直接清除iptables
4.创建kube客户端和event客户端
5.事件广播器，类似于kube-scheduler
6.健康监测
7.确定模式，并清除掉其他模式残留的iptables数据
```

### 启动

```
初始化完成之后，开始进入server.Run函数，有几个步骤：
1.如果配置cleanup为true，则直接清除所有模式的iptables（userspace、iptables、ipvs），直接退出；
2.正常运行的话，首先设置OOMAdjuster参数，其实就是内核自我保护的机制，用于调节score，处于(-1000,1000)之间，值越大进程越有可能被杀死，因为kube-proxy比较重要，所有默认就是-999，一般不去修改；
3.启动ResourceContainer，主要对资源进行设置，通过cgroups控制；
4.事件广播；
5.HealthzServer；
6.MetricsServer
7.Conntrack，设置一些网络连接参数；
8.informer（service、endpoint）
9.service和endpoint监听与回调配置，然后启动
10.无限循环...
```

