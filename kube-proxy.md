

# Kube-proxy源码分析

1.入口函数

```
入口函数在cmd/kube-proxy/proxy.go上，通过 command := app.NewProxyCommand()创建启动参数和配置文件。
通过Run方法启动，具体Run函数如下：
```

```
Run: func(cmd *cobra.Command, args []string) {
			verflag.PrintAndExitIfRequested()
			utilflag.PrintFlags(cmd.Flags())

			if err := initForOS(opts.WindowsService); err != nil {
				glog.Fatalf("failed OS init: %v", err)
			}

			cmdutil.CheckErr(opts.Complete())
			cmdutil.CheckErr(opts.Validate(args))
			cmdutil.CheckErr(opts.Run())
		}
```

```
主要操作在opts.Run()函数中，具体函数如下：
```

```
func (o *Options) Run() error {
	if len(o.WriteConfigTo) > 0 {
		return o.writeConfigFile()
	}

	proxyServer, err := NewProxyServer(o)
	if err != nil {
		return err
	}

	return proxyServer.Run()
}
```

```
NewProxyServer函数生成了启动的配置文件，结构体为ProxyServer，通过一系列初始化操作生成配置；进入proxyServer.Run()中，
```

```
func (s *ProxyServer) Run() error {
	// To help debugging, immediately log version
	glog.Infof("Version: %+v", version.Get())
	// remove iptables rules and exit
	if s.CleanupAndExit {
		encounteredError := userspace.CleanupLeftovers(s.IptInterface)
		encounteredError = iptables.CleanupLeftovers(s.IptInterface) || encounteredError
		encounteredError = ipvs.CleanupLeftovers(s.IpvsInterface, s.IptInterface, s.IpsetInterface, s.CleanupIPVS) || encounteredError
		if encounteredError {
			return errors.New("encountered an error while tearing down rules.")
		}
		return nil
	}

	// TODO(vmarmol): Use container config for this.
	var oomAdjuster *oom.OOMAdjuster
	if s.OOMScoreAdj != nil {
		oomAdjuster = oom.NewOOMAdjuster()
		if err := oomAdjuster.ApplyOOMScoreAdj(0, int(*s.OOMScoreAdj)); err != nil {
			glog.V(2).Info(err)
		}
	}

	if len(s.ResourceContainer) != 0 {
		// Run in its own container.
		if err := resourcecontainer.RunInResourceContainer(s.ResourceContainer); err != nil {
			glog.Warningf("Failed to start in resource-only container %q: %v", s.ResourceContainer, err)
		} else {
			glog.V(2).Infof("Running in resource-only container %q", s.ResourceContainer)
		}
	}

	if s.Broadcaster != nil && s.EventClient != nil {
		s.Broadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: s.EventClient.Events("")})
	}

	// Start up a healthz server if requested
	if s.HealthzServer != nil {
		s.HealthzServer.Run()
	}

	// Start up a metrics server if requested
	if len(s.MetricsBindAddress) > 0 {
		mux := mux.NewPathRecorderMux("kube-proxy")
		healthz.InstallHandler(mux)
		mux.HandleFunc("/proxyMode", func(w http.ResponseWriter, r *http.Request) {
			fmt.Fprintf(w, "%s", s.ProxyMode)
		})
		mux.Handle("/metrics", prometheus.Handler())
		if s.EnableProfiling {
			routes.Profiling{}.Install(mux)
		}
		configz.InstallHandler(mux)
		go wait.Until(func() {
			err := http.ListenAndServe(s.MetricsBindAddress, mux)
			if err != nil {
				utilruntime.HandleError(fmt.Errorf("starting metrics server failed: %v", err))
			}
		}, 5*time.Second, wait.NeverStop)
	}

	// Tune conntrack, if requested
	// Conntracker is always nil for windows
	if s.Conntracker != nil {
		max, err := getConntrackMax(s.ConntrackConfiguration)
		if err != nil {
			return err
		}
		if max > 0 {
			err := s.Conntracker.SetMax(max)
			if err != nil {
				if err != readOnlySysFSError {
					return err
				}
				// readOnlySysFSError is caused by a known docker issue (https://github.com/docker/docker/issues/24000),
				// the only remediation we know is to restart the docker daemon.
				// Here we'll send an node event with specific reason and message, the
				// administrator should decide whether and how to handle this issue,
				// whether to drain the node and restart docker.
				// TODO(random-liu): Remove this when the docker bug is fixed.
				const message = "DOCKER RESTART NEEDED (docker issue #24000): /sys is read-only: " +
					"cannot modify conntrack limits, problems may arise later."
				s.Recorder.Eventf(s.NodeRef, api.EventTypeWarning, err.Error(), message)
			}
		}

		if s.ConntrackConfiguration.TCPEstablishedTimeout != nil && s.ConntrackConfiguration.TCPEstablishedTimeout.Duration > 0 {
			timeout := int(s.ConntrackConfiguration.TCPEstablishedTimeout.Duration / time.Second)
			if err := s.Conntracker.SetTCPEstablishedTimeout(timeout); err != nil {
				return err
			}
		}

		if s.ConntrackConfiguration.TCPCloseWaitTimeout != nil && s.ConntrackConfiguration.TCPCloseWaitTimeout.Duration > 0 {
			timeout := int(s.ConntrackConfiguration.TCPCloseWaitTimeout.Duration / time.Second)
			if err := s.Conntracker.SetTCPCloseWaitTimeout(timeout); err != nil {
				return err
			}
		}
	}

	informerFactory := informers.NewSharedInformerFactory(s.Client, s.ConfigSyncPeriod)

	// Create configs (i.e. Watches for Services and Endpoints)
	// Note: RegisterHandler() calls need to happen before creation of Sources because sources
	// only notify on changes, and the initial update (on process start) may be lost if no handlers
	// are registered yet.
	serviceConfig := config.NewServiceConfig(informerFactory.Core().InternalVersion().Services(), s.ConfigSyncPeriod)
	serviceConfig.RegisterEventHandler(s.ServiceEventHandler)
	go serviceConfig.Run(wait.NeverStop)

	endpointsConfig := config.NewEndpointsConfig(informerFactory.Core().InternalVersion().Endpoints(), s.ConfigSyncPeriod)
	endpointsConfig.RegisterEventHandler(s.EndpointsEventHandler)
	go endpointsConfig.Run(wait.NeverStop)

	// This has to start after the calls to NewServiceConfig and NewEndpointsConfig because those
	// functions must configure their shared informer event handlers first.
	go informerFactory.Start(wait.NeverStop)

	// Birth Cry after the birth is successful
	s.birthCry()

	// Just loop forever for now...
	s.Proxier.SyncLoop()
	return nil
}
```

```
首先，检查CleanupAndExit变量，如果为true，直接退出，并且清空iptables等各种设定的规则；接着通过NewOOMAdjuster函数设定OOMAdjuster；检查ResourceContainer的值，如果设定了则启动RunInResourceContainer函数；之后设置健康检查和metrics检测服务，最后通过监听模式watch service和endpoints，分别启动三个协程去处理，最终同步一下。
```
