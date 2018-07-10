### Kubernetes高可用Master搭建

####搭建haproxy

##### 1.下载haproxy

```
# yum install -y haproxy
```

##### 2.配置haproxy

```
vi /etc/haproxy/haproxy.cfg

global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin
        stats timeout 30s
        user haproxy
        group haproxy
        daemon
        nbproc 1

defaults
        log     global
        timeout connect 5000
        timeout client  10m
        timeout server  10m

listen kube-master
        bind 0.0.0.0:8443    #8443表示代理端口
        mode tcp
        option tcplog
        balance source
        # 5.0.0.191:6443  5.0.0.192:6443 分别表示两个apiserver的访问地址
        server s1 5.0.0.191:6443  check inter 10000 fall 2 rise 2 weight 1
        server s2 5.0.0.192:6443  check inter 10000 fall 2 rise 2 weight 1 
```

##### 3.启动

```
haproxy可以部署多个节点，以systemd方式启动

vi /usr/lib/systemd/system/haproxy.service

[Unit]
Description=HAProxy Load Balancer
After=syslog.target network.target

[Service]
EnvironmentFile=/etc/sysconfig/haproxy
ExecStartPre=/usr/bin/mkdir -p /run/haproxy
ExecStart=/usr/sbin/haproxy-systemd-wrapper -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid $OPTIONS
ExecReload=/bin/kill -USR2 $MAINPID
KillMode=mixed

[Install]
WantedBy=multi-user.target

# systemctl start haproxy
```

#### 搭建keepalived

##### 1.下载安装

```
# yum install -y keepalived
```

##### 2.配置

```
vi /etc/keepalived/keepalived.conf

# keepalived 统一id
global_defs {
    router_id lb-backup
}

vrrp_script check-haproxy {
    script "killall -0 haproxy"
    interval 5
    weight -30
}

vrrp_instance VI-kube-master {
    state MASTER             # 表示是主节点，备节点为BACKUP
    priority 120             # 优先级，小于255，备节点要比主节点小
    dont_track_primary
    interface eno1           # 网卡地址
    virtual_router_id 57
    advert_int 3
    track_script {			# haproxy健康检测
        check-haproxy
    }
    virtual_ipaddress {      # 虚拟ip地址为5.0.0.200，也可以是一个网段，自己配置
        5.0.0.200
    }
}

从节点配置
global_defs {
    router_id lb-backup
}

vrrp_instance VI-kube-master {
    state BACKUP
    priority 110
    dont_track_primary
    interface eno1
    virtual_router_id 57
    advert_int 3
    virtual_ipaddress {
        5.0.0.200
    }
}
```

##### 3.启动

```
# systemctl start keepalived
```

#### 使用

```
搭建完成之后，可以通过ip a查看到地址信息

2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 50:65:f3:e6:60:0f brd ff:ff:ff:ff:ff:ff
    inet 5.0.0.191/24 brd 5.0.0.255 scope global eno1
       valid_lft forever preferred_lft forever
    inet 5.0.0.200/32 scope global eno1
       valid_lft forever preferred_lft forever
    inet6 fe80::e1f4:1ef:bd23:7159/64 scope link 
       valid_lft forever preferred_lft forever
       
之后访问apiserver即可通过 https://5.0.0.200:8443去连接。
```

#### 问题

```
1.目前在使用了两个节点的机器后，断开任一台上的keepalived之后，kubelet访问不到https://5.0.0.200:8443这个地址，但是curl是可以通的，等待一段时间之后，Node变成Ready。
```

