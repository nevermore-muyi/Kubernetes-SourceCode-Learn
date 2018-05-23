

# Kubernetes-apiserver源码分析

1.入口函数

```
入口函数在cmd/kube-apiserver/apiserver.go上，通过 command := app.NewAPIServerCommand() 创建启动参数和配置文件。
在NewAPIServerCommand()函数中，调用options.NewServerRunOptions()方法生成默认配置，通过Run方法启动，具体Run函数如下：
```

```
func Run(completeOptions completedServerRunOptions, stopCh <-chan struct{}) error {
	// To help debugging, immediately log version
	glog.Infof("Version: %+v", version.Get())

	server, err := CreateServerChain(completeOptions, stopCh)
	if err != nil {
		return err
	}
	
	return server.PrepareRun().Run(stopCh)
}
```

```
其中，CreateServerChain()函数是通过进一步生成配置文件，返回GenericAPIServer对象，之后使用该对象进行PrepareRun()和Run()操作。
PrepareRun()主要进行一些API的注册操作，主要有两个Install函数，如下：
```

```
func (s *GenericAPIServer) PrepareRun() preparedGenericAPIServer {
	if s.swaggerConfig != nil {
		routes.Swagger{Config: s.swaggerConfig}.Install(s.Handler.GoRestfulContainer)
	}
	if s.openAPIConfig != nil {
		routes.OpenAPI{
			Config: s.openAPIConfig,
		}.Install(s.Handler.GoRestfulContainer, s.Handler.NonGoRestfulMux)
	}

	s.installHealthz()

	return preparedGenericAPIServer{s}
}
```

```
主要包括SwaggerUI webservice的注册和openapi的注册。以后执行Run()方法，这是最终的启动函数，代码如下：
```

```
func (s preparedGenericAPIServer) Run(stopCh <-chan struct{}) error {
	// Register audit backend preShutdownHook.
	if s.AuditBackend != nil {
		s.AddPreShutdownHook("audit-backend", func() error {
			s.AuditBackend.Shutdown()
			return nil
		})
	}

	err := s.NonBlockingRun(stopCh)
	if err != nil {
		return err
	}

	<-stopCh

	return s.RunPreShutdownHooks()
}
```

```
启动前，先执行AddPreShutdownHook()方法，添加钩子；主要启动在NonBlockingRun()内，调用serveSecurely()方法最终启动apiserver。
```
