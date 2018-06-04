

# Kubelet源码分析

1.入口函数

```
入口函数在cmd/kubelet/kubelet.go上，通过 command := app.NewKubeletCommand()创建启动参数和配置文件。
通过Run方法启动，具体Run函数如下：
```

```
func Run(s *options.KubeletServer, kubeDeps *kubelet.Dependencies) error {
	// To help debugging, immediately log version
	glog.Infof("Version: %+v", version.Get())
	if err := initForOS(s.KubeletFlags.WindowsService); err != nil {
		return fmt.Errorf("failed OS init: %v", err)
	}
	if err := run(s, kubeDeps); err != nil {
		return fmt.Errorf("failed to run Kubelet: %v", err)
	}
	return nil
}
```

```
可以看到，在1.10版本中，加入了对windows操作系统的特殊处理，之后才开始进入真正的run函数处理。
```

```
在run函数中，前面部分主要都是在对kubeDeps这个变量进行操作，有点类似于Java Spring的依赖注入的意思，将需要的组件作为参数传递进来，包含的组件有许多，主要有：
Cloud： cloudprovider；
KubeClient： 用来和apiserver通信的客户端；
Auth： 认证相关；
CAdvisorInterface： 提供 cAdvisor 接口功能的组件，用来获取监控信息；
ContainerManager： Container资源管理；
...
```

```
进入一系列复杂的注入操作，终于进入了最核心的接口，RunKubelet()，最后提供一个http的healthz的接口。
```

```
进入RunKubelet，首先是一些初始化的创建等操作，真正的运行在下面函数中：
```

```
if runOnce {
		if _, err := k.RunOnce(podCfg.Updates()); err != nil {
			return fmt.Errorf("runonce failed: %v", err)
		}
		glog.Infof("Started kubelet as runonce")
	} else {
		startKubelet(k, podCfg, kubeCfg, kubeDeps, kubeFlags.EnableServer)
		glog.Infof("Started kubelet")
	}
```

```
进入startKubelet函数，如下：
```

```
func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *kubelet.Dependencies, enableServer bool) {
	wg := sync.WaitGroup{}

	// start the kubelet
	wg.Add(1)
	go wait.Until(func() {
		wg.Done()
		k.Run(podCfg.Updates())
	}, 0, wait.NeverStop)

	// start the kubelet server
	if enableServer {
		wg.Add(1)
		go wait.Until(func() {
			wg.Done()
			k.ListenAndServe(net.ParseIP(kubeCfg.Address), uint(kubeCfg.Port), kubeDeps.TLSOptions, kubeDeps.Auth, kubeCfg.EnableDebuggingHandlers, kubeCfg.EnableContentionProfiling)
		}, 0, wait.NeverStop)
	}
	if kubeCfg.ReadOnlyPort > 0 {
		wg.Add(1)
		go wait.Until(func() {
			wg.Done()
			k.ListenAndServeReadOnly(net.ParseIP(kubeCfg.Address), uint(kubeCfg.ReadOnlyPort))
		}, 0, wait.NeverStop)
	}
	wg.Wait()
}
```

```
k.Run(podCfg.Updates())中，podCfg.Updates()是一个channel，会实时获取到Pod返回过来的信息，主要包含定时向apiserver更新Node信息，管理Pod状态，健康检查等操作。
```

#### 参考资料

http://cizixs.com/2017/06/06/kubelet-source-code-analysis-part-1

http://www.dockone.io/article/2100