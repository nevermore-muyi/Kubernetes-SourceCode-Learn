# Kubernetes NetworkPolicy 

### 背景

Kubernetes集群默认情况下，Pod之间是可以相互通信的。但是特定情况下，需要对Pod之间的流量进行隔离，Kubernetes提供了NetworkPolicy，可以针对Pod之间以及namespace之间的流量进行控制。

### 限制

使用NetworkPolicy对网络插件有要求，依赖cni插件的支持，目前不支持flannel，支持的有calico、weave、kube-router等。

### 策略

- ip段

  可以针对ip段对pod的流量进行限制，下图的policy表示只有在172.17.0.0/16段内，但是不包括172.17.1.0/24内的ip段可以访问到标签为name: busybox的pod。

  ```
  kind: NetworkPolicy
  apiVersion: networking.k8s.io/v1
  metadata:
    name: access-nginx
  spec:
    podSelector:
      matchLabels:
        name: busybox
    ingress:
    - from:
      - ipBlock:
          cidr: 172.17.0.0/16
          except:
          - 172.17.1.0/24
  ```

- namespaceSelector

  使用namespaceSelector对pod流量进行限制，下图的policy表示只有标签为zhulaoshi: shuai的namespace内的pod可以访问标签为name: busybox的pod。

  ```
  kind: NetworkPolicy
  apiVersion: networking.k8s.io/v1
  metadata:
    name: access-nginx
  spec:
    podSelector:
      matchLabels:
        name: busybox
    ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            zhulaoshi: shuai
  ```

- podSelector

  使用podSelector对pod流量进行限制，下图的policy表示只有标签为access: "true"的pod可以访问标签为name: busybox的pod。

  ```
  kind: NetworkPolicy
  apiVersion: networking.k8s.io/v1
  metadata:
    name: access-nginx
  spec:
    podSelector:
      matchLabels:
        name: busybox
    ingress:
    - from:
      - podSelector:
          matchLabels:
            access: "true"
  ```

  以上三种策略是“或”的关系，同时设置的话满足一项即可。另外，还可以对Port以及出口规则设置。

  理解下面规则：

  ```
  
  ```

  ### 原理

  使用calicoctl get policy可以发现创建完成之后calico会生成新的policy

  ```
  - apiVersion: v1
    kind: policy
    metadata:
      name: knp.default.default.access-nginx
    spec:
      ingress:
      - action: allow
        destination: {}
        source:
          nets:
          - 172.34.0.0/24
      - action: allow
        destination: {}
        source:
          selector: pcns.zhulaoshi == 'shuai'
      order: 1000
      selector: calico/k8s_ns == 'default' && name == 'busybox'
      types:
      - ingress
  ```

  calico的网络策略主要依赖的是iptables，policy最终实现也是iptables，最主要的是**cali-pi-[POLICY]@filter** 这条Chain进行策略的过滤，以任一pod为例：

  执行命令  iptables -nxvL cali-tw-cali12d4a061371 -t filter

  流量进入前

  ```
  Chain cali-tw-cali12d4a061371 (1 references)
      pkts      bytes target     prot opt in     out     source               destination         
         6      488 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:QDweCTUR7_Cz-dqI */ ctstate RELATED,ESTABLISHED
         0        0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:tp2WUDDKWGzlC-9I */ ctstate INVALID
        28     2152 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:Qf97Vi5Ejny2VFus */ MARK and 0xfeffffff
        13     1012 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:k_atMv7z9RYiKACH */ /* Start of policies */ MARK and 0xfdffffff
        13     1012 cali-pi-_wrg4pJPiI1zj6SR3Ljt  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:wO1J7CYT_cTmSkXK */ mark match 0x0/0x2000000
         4      312 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:4E2aBxOLp1kGcV-T */ /* Return if policy accepted */ mark match 0x1000000/0x1000000
         9      700 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:8ijBjmAom8t5gg_O */ /* Drop if no policies passed packet */ mark match 0x0/0x2000000
         0        0 cali-pri-kns.default  all  --  *      *       0.0.0.0/0            0.0.0.0/0  
  ```

   流量进入后

  ```
  Chain cali-tw-cali12d4a061371 (1 references)
      pkts      bytes target     prot opt in     out     source               destination         
         6      488 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:QDweCTUR7_Cz-dqI */ ctstate RELATED,ESTABLISHED
         0        0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:tp2WUDDKWGzlC-9I */ ctstate INVALID
        29     2228 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:Qf97Vi5Ejny2VFus */ MARK and 0xfeffffff
        14     1088 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:k_atMv7z9RYiKACH */ /* Start of policies */ MARK and 0xfdffffff
        14     1088 cali-pi-_wrg4pJPiI1zj6SR3Ljt  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:wO1J7CYT_cTmSkXK */ mark match 0x0/0x2000000
         4      312 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:4E2aBxOLp1kGcV-T */ /* Return if policy accepted */ mark match 0x1000000/0x1000000
        10      776 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:8ijBjmAom8t5gg_O */ /* Drop if no policies passed packet */ mark match 0x0/0x2000000
         0        0 cali-pri-kns.default  all  --  *      *       0.0.0.0/0            0.0.0.0/0
  ```

  可以明显发现通过MARK、cali-pi-_wrg4pJPiI1zj6SR3Ljt两个target处理后，数据包直接被DROP掉。