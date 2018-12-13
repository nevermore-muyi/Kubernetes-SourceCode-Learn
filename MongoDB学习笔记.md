### MongoDB学习笔记

#### 安装

```
官网找到yum源，使用yum install下载，systemctl start mongod，配置文件在/etc/mongod.conf下
```

#### 集群模式

```
MongoDB有三种集群模式：
1.M/S主从模式：备用节点只是主节点的备份，主节点失效后备用节点不会接管，没什么用，一般用在备份节点特别多的情况下或者只是用来复制单个数据库；
2.副本集模式：备用节点会同步主节点数据，并且主节点失效后备用节点会有选主操作；
3.分片模式：每个分片数据不一样（类似于Redis的槽），可以设置分片为副本集模式（与Redis的集群方式类似）

副本集模式：
备份节点不能执行写操作；
默认情况下，备份节点也不能读，需要设置setSlaveOK之后才可以读;
每个节点上配置，主要保证replSetName名称一致，然后通过config执行rs.initiate(config)生成一个副本集

分片模式有三种节点：
1.mongos：路由节点，接受外部请求，路由到相应的分片，每个节点之间没有关联，指定sharding.configDB；
2.config：配置节点，保存分片信息，供mongos启动读取，只能有一个副本集，指定sharding.clusterRole为configsvr；
3.shard： 分片节点，保存最终的数据，多个副本集对应多个shard，指定sharding.clusterRole为shardsvr。
```

#### 仲裁节点

```
仲裁节点是为了避免重现偶数个节点的情况，而不应该增加这种情况。
```

#### 副本集复制策略

```
节点复制时，不一定都是从主节点复制，可能会从比自己新的备份节点去复制，这样会出现复制链过长，因此也可以手动指定复制源。
```

#### 分片模式

```
启用分片时，必须要先指定，否则只是会默认到某一个副本集上，并且不会迁移，除非启用了分片；
1.集合的数据库启用分片，sh.enableSharding("test")
2.选择片健创建索引，db.users.ensureIndex({"username": 1})
3.对集合分片，sh.shardCollection("test.users", {"username": 1})

查询时，包含片健的话，相当于索引，可以直接将查询发送到目标分片或者某一个子集上，称为定向查询；
不包含片健，则为分散-聚集查询，mongos将查询分散到所有分片，然后收集查询结果。
```

#### 停止MongoDB

```
1.Systemctl stop mongod
2.mongod ... --shutdown
3.kill -2 ...
```

#### 认证搭建

```
1.先无认证正常启动，然后添加用户，赋予相应的权限
# use admin
# db.createUser(
  {
    user: "adminUser",
    pwd: "adminPass",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
  }
 )
2.关闭数据库，生成认证文件
# openssl rand -base64 741 > /data/mongokey
# chmod 600 /data/mongokey
3.认证文件分发到各个节点相同的目录下
4.修改conf文件，增加认证文件
security:
  keyFile: "/data/mongokey"
  clusterAuthMode: "keyFile"
5.保存启动，登录
方式一： mongo --port 27017 -u "adminUser" -p "adminPass" --authenticationDatabase "admin"
方式二： 
# mongo --port 27017；
# use admin
# db.auth('adminUser','adminPass')
```

#### 索引类型
```
1.唯一索引：保证每一个文档中的指定的索引的健是唯一的，索引健不存在的文档会被置为null，下次再插入就出现重复的错误提示
2.稀疏索引：解决唯一索引的键不存在则被置为null的问题，稀疏索引在键存在的时候可以索引，不存在则不用关心。
```
