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

