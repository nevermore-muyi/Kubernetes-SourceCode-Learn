# **Centos搭建Ceph集群教程**

## **一.准备（每台节点）**

1.关闭selinux，vim /etc/selinux/config，修改为disabled，并reboot；

2.创建新的用户（不要使用root用户搭建集群）

```
这里创建的用户为：cent 密码是cent
sudo useradd -d /home/cent -m cent
sudo passwd cent
echo "cent ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cent
sudo chmod 0440 /etc/sudoers.d/cent
su cent
```

3.校时

```
sudo yum install ntp ntpdate ntp-doc
sudo yum install openssh-server
```

## 二.开始安装（选择一台部署节点，一台monitor，两台osd）

1.修改hosts文件，配置节点信息

```
vim /etc/hosts

192.168.88.109 node1
192.168.88.107 node2
192.168.88.108 node3
```

2.配置yum源

```
vim /etc/yum.repos.d/ceph.repo

[Ceph]
name=Ceph packages for $basearch
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/$basearch
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.163.com/ceph/keys/release.asc
priority=1

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/noarch
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.163.com/ceph/keys/release.asc
priority=1

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/SRPMS
enabled=1
gpgcheck=0
type=rpm-md
gpgkey=https://mirrors.163.com/ceph/keys/release.asc
priority=1
```

3.安装ceph-deploy

```
sudo yum install yum-plugin-priorities
sudo yum install ceph-deploy
```

4.配置ssh互信

```
ssh-keygen （一直回车）
ssh-copy-id cent@node1
ssh-copy-id cent@node2
ssh-copy-id cent@node3

vim ~/.ssh/config

Host node1
   Hostname node1
   User cent
Host node2
   Hostname node2
   User cent
Host node3
   Hostname node3
   User cent

sudo chmod 600 ~/.ssh/config
```

5.创建集群

```
mkdir my-cluster 
cd my-cluster
ceph-deploy new node1 (成功后会有ceph.conf)

vim ceph.conf （在global段最后添加，定义最少可以允许有两个osd，默认是3个，如果节点数量足够，可不用修改）
osd pool default size = 2

#其他可选参数
# mon_clock_drift_allowed = 2    #定义多个mon节点之间时间的误差为2s
# public_network=10.5.10.0/24    #定义互相通信的公有网络

#以下三行配置都是为解决数据节点的存储盘为ext4文件系统时，解决“ERROR: osd init failed: (36) File name too long”错误的。ceph官方推荐存储盘使用xfs文件系统，但在有些特定的场合下，我们只能使用ext4文件系统。

# osd_max_object_name_len = 256
# osd_max_object_namespace_len = 64
# filestore_xattr_use_omap = true
```

6.安装cpeh

```
ceph-deploy install node1 node2 node3
ceph-deploy mon create-initial
```

7.添加并激活osd（使用本地磁盘）

```
ssh node2
sudo mkdir /var/local/osd0
sudo chmod -R 777 /var/local/osd0/
exit

ssh node3
sudo mkdir /var/local/osd1
sudo chmod -R 777 /var/local/osd1/
exit

#激活osd-node并启动
ceph-deploy osd prepare node2:/var/local/osd0 node3:/var/local/osd1
ceph-deploy osd activate node2:/var/local/osd0 node3:/var/local/osd1

#往各节点同步配置文件：
ceph-deploy admin node1 node2 node3

sudo chmod +r /etc/ceph/ceph.client.admin.keyring
```

8.查看ceph集群状态

```
ceph -s
ceph osd tree
```

PS:如果之前安装过ceph，可以先执行如下命令以获得一个干净的环境

```
sudo yum remove -y ceph ceph-common

ceph-deploy purgedata node1 node2 node3
ceph-deploy forgetkeys
ceph-deploy purge node1 node2 node3
```



## 三.K8S使用Ceph(RBD)

1.创建rbd块用于kubernetes存储，先创建存储池，ceph提供了一个叫做rbd的默认存储池，我们再创建一个kube的存储专门用来存储kubernetes使用的块设备：

```
ceph osd pool create kube 100 100 #后面两个100分别为pg-num和pgp-num
```

2.在kube存储池创建一个映像文件，就叫mysql-sonar，该映像文件的大小为5GB：

```
rbd create kube/mysql-sonar --size 5120 --image-format 2 --image-feature layering
```

3.将上面创建的映象文件映射为块设备：

```
sudo rbd map kube/mysql-sonar --name client.admin

如果报以下错误：
rbd: sysfs write failed
RBD image feature set mismatch. You can disable features unsupported by the kernel with "rbd feature disable".
In some cases useful info is found in syslog - try "dmesg | tail" or so.
rbd: map failed: (6) No such device or address

查看rbd info foo，
rbd image 'foo':
    size 1024 MB in 256 objects
    order 22 (4096 kB objects)
    block_name_prefix: rbd_data.10612ae7234b
    format: 2    features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
    flags:
layering: 支持分层
striping: 支持条带化 v2
exclusive-lock: 支持独占锁
object-map: 支持对象映射（依赖 exclusive-lock ）
fast-diff: 快速计算差异（依赖 object-map ）
deep-flatten: 支持快照扁平化操作
journaling: 支持记录 IO 操作（依赖独占锁）

通过rbd feature disable foo exclusive-lock, object-map, fast-diff, deep-flatten关闭属性即可

另外一种方式：
在各个cluster node的/etc/ceph/ceph.conf中加上这样一行配置：
rbd_default_features = 1 #仅是layering对应的bit码所对应的整数值
查看变化：
ceph --show-config|grep rbd|grep features  # = 1
```

4.构建K8S Secret

```
cat /etc/ceph/ceph.client.admin.keyring 
[client.admin]
    key = AQDRvL9YvY7vIxAA7RkO5S8OWH6Aidnu22OiFw==
    caps mds = "allow *"
    caps mon = "allow *"
    caps osd = "allow *"
    
做base64处理：
echo "AQDRvL9YvY7vIxAA7RkO5S8OWH6Aidnu22OiFw==" | base64
QVFEUnZMOVl2WTd2SXhBQTdSa081UzhPV0g2QWlkbnUyMk9pRnc9PQo=

创建secret:
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
type: "kubernetes.io/rbd"  
data:
  key: QVFEUnZMOVl2WTd2SXhBQTdSa081UzhPV0g2QWlkbnUyMk9pRnc9PQo=
```

5.创建PV

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-sonar-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
      - 10.5.10.117:6789
    pool: kube
    image: mysql-sonar
    user: cent
    secretRef:
      name: ceph-secret
    fsType: ext4
    readOnly: false
  persistentVolumeReclaimPolicy: Recycle
```

6.创建PVC

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-sonar-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

7.使用PVC

```
apiVersion: extensions/v1beta1
kind: Deployment          
metadata:
  name: mysql-sonar
spec:
  replicas: 1                                          
  template:
    metadata:
      labels:
        app: mysql-sonar
    spec:
      containers:                       
      - name: mysql-sonar
        image: myhub.fdccloud.com/library/mysql-yd:5.6      
        ports:
        - containerPort: 3306           
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "mysoft"
        - name: MYSQL_DATABASE
          value: sonardb
        - name: MYSQL_USER
          value: sonar
        - name: MYSQL_PASSWORD
          value: sonar
        volumeMounts:
        - name: mysql-sonar
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-sonar
        persistentVolumeClaim:
          claimName: mysql-sonar-pvc
```

PS：所有node节点上安装ceph-common包

## 四.安装CephFS

1.步骤三安装完成之后，安装CephFS需要安装MDS：

```
ceph-deploy mds create node1 node2 node3 #安装三台元数据服务器
```

2.创建两个存储池，MDS需要使用两个pool，一个pool用来存储数据，一个pool用来存储元数据：

```
ceph osd pool create fs_data 32
ceph osd pool create fs_metadata 32
rados lspools
```

3.创建Cephfs

```
ceph fs new cephfs fs_metadata fs_data
ceph fs ls
ceph mds stat  #查看MDS状态
```

4.挂载Cephfs

```
modprobe rbd #加载rbd内核模块
cat ceph.client.admin.keyring  #获取admin key 
mount -t ceph node1,node2,node3:6789:/ /mnt -o name=admin,secret=AQDchXhYTtjwHBAAk2/H1Ypa23WxKv4jA1NFWw==  #创建挂载点，尝试本地挂载，secret与														    #ceph.client.admin.keyring一致
df -h 
```

## 五.命令

```
rbd create ceph-rbd-pv-test --size 1024 #创建一个大小为1024M的ceph image
rbd list
sudo rbd map ceph-rbd-pv-test #ceph-rbd-pv-test image 映射到内核
rbd showmapped
ceph osd pool ls #列出所有pool
ceph mds stat #查看MDS状态
```

## 六.参考文档

http://docs.ceph.org.cn/cephfs/

http://blog.csdn.net/ns2250225/article/details/52074222

https://cloud.tencent.com/developer/article/1006283

https://www.cnblogs.com/breezey/p/6558967.html

http://tonybai.com/2016/11/07/integrate-kubernetes-with-ceph-rbd/

http://tonybai.com/2017/05/08/mount-cephfs-acrossing-nodes-in-kubernetes-cluster/

http://tonybai.com/2017/02/17/temp-fix-for-pod-unable-mount-cephrbd-volume/

http://blog.csdn.net/minxihou/article/details/66478268

http://blog.csdn.net/aixiaoyang168/article/details/78999851

https://www.cnblogs.com/keithtt/p/6410288.html

https://github.com/kubernetes-incubator/external-storage

https://www.insoz.com/2017/05/11/integrage-cephfs-to-kubernetes/
