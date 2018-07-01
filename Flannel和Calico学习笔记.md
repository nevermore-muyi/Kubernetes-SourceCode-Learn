## Flannel和Calico学习笔记

```
Flannel和Calico作为网络插件，在kubernetes使用上非常广泛；flannel是overlay网络，而calico是纯路由的网络，都是用在三层网络之上。
```

### Flannel

```
Flannel作为overlay网络，是最早和kubernetes配合使用的。网络模式主要有三种，分别为：UDP、VXLAN和host-gw。其中UDP性能最差，host-gw主要用在二层网络上，以主机作为网关，而VXLAN是普遍使用的一种方式，性能上，host-gw > VXLAN > UDP。
```

### Flannel使用

```
1.传统的使用方式，flannel在宿主机上起一个flanneld进程，连接docker0端和flannel0端，flannel启动之后，需要与etcd进行交互，所以etcd中需要填写flannel的相关信息，并且docker启动时修改docker启动参数，比较局限；
2.新版本的方式，flannel以DaemonSet的方式启动，启动时也可以不用向Etcd注册，而是直接使用每个节点上的信息，用--kube-subnet-mgr参数指定，使用cni插件，所以不用去修改docker0参数，所有的网络通过cni0去起到和docker0类似的作用；
3.多网卡时候，可以指定启动命令--iface去指定，--iface可以拼接多个地址，最先找到的使用，默认使用的是默认网卡；或者使用相应的正则匹配参数。
4.--ip-masp主要用来隐藏源地址，如果不指定的话，src是flannel.1的网卡地址；指定的话，地址为实际的pod地址，可以通过tcpdump -i cni0 icmp -w ./target.cap 去抓cni0的网卡信息包。
```



### Calico

```
Calico是一个纯三层的路由协议，与flannel相比，减少了封包解包，所以速率会快不少。Calico主要的网络模式有BGP和IPIP，原则上，效率为： flannel(host-gw) > calico(bgp) > flannel(vxlan) ~= calico(ipip) > flannel(udp)。
```

### Calico使用

```
Calico主要依赖Etcd去存储数据，类似于Flannel存储网络数据一样；每台机器上需要跑一个agent，专业术语叫Felix，主要负责配置路由等信息；另外还需要一个中间层，相当于控制层，专业术语叫BIRD，主要负责路由的分发，保证节点之间通信（配置各种策略等）。
使用了BGP和自身的路由转发机制，不依赖硬件，相当于在主机上启动了虚拟的路由器，容器之间的通信并不依赖iptables NAT或者Tunnel技术。
Calico可以在Kubernetes上使用Network Policy，流量隔离基于iptables实现，数据存储在Etcd中。
Calico以DaemonSet部署的时候，还是用了Confd去对配置进行管理。
Calicoctl工具配置可以通过etcd，或者通过kubernetes，路径如下：
https://docs.projectcalico.org/v2.6/reference/calicoctl/setup/kubernetes
```

