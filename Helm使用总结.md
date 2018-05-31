## Helm使用总结

#### 安装

安装`helm`客户端 

```
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

安装依赖

```
yum install -y socat
```

创建tiller的`serviceaccount`和`clusterrolebinding` 

```
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```

然后安装helm服务端tiller 

```
helm init -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.9.1
PS：由于默认的init会从gcr.io下载镜像，网络会有问题，所以使用-i命令指定镜像地址安装，安装需保证服务端和客户端地址版本相同。可通过helm version查看版本信息,这里使用的是2.9.1的版本和阿里云的镜像地址。
```

为应用程序设置`serviceAccount` 

```
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

