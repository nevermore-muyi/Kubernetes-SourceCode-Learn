### Rabbitmq学习笔记

#### 基本概念

```
Exchange：生产者将消息放入Exchange中；
Queue：存储消息，消费者从Queue取走消息；
Binding：Exchange和Queue之间的一种绑定关系；
Broker：RabbitMQ服务端程序，包含Exchange和Queue；
Virtual Host：类似于命名空间，主要对权限控制。
```

#### Exchange

```
发送者发送消息时，消息指定routing_key，queue与exchange绑定通过routing_key，消费者订阅queue即可。

Exchange和Queue绑定有四种策略：
1.Direct Exchange：完全匹配；
2.Fanout Exchange：广播式路由，binding时不需要routing_key，广播到binding的queue中；
3.Topic Exchange：模糊匹配，相当于正则；
4.Headers Exchange：头部带上键值对参数，并且通过设置参数可以控制对多个键值对的匹配。
```

#### Queue

```
Durable：如果为true表示队列持久化，重启之后依旧存在；
Exclusive：只允许被一个客户端使用，客户端断开连接即被删除；
Auto-delete：当所有消费者都取消订阅时，队列自动被删除；
ps：当一个Queue被多个消费者订阅时，消息只会被一个消费者消费。
```

#### Message

```
消息如果想置为持久化，相应的exchange和queue都需要设置为持久化。
```

#### 镜像队列

```
RabbitMQ集群中，Exchange和Binding是在各个节点都是同步存在的，但是Queue只在声明它的节点存在，所以需要镜像队列的存在。
```

#### 安装

```
可以直接去官网下载rpm包安装，预先下载好erlang，并且dashboard界面需要enable plugins
# rabbitmq-plugins enable rabbitmq_management
默认guest用户只能在localhost下访问，所以需要添加配置
loopback_users = none
```

