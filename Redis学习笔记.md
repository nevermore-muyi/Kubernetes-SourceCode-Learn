### Redis学习笔记

#### 基础知识

```
1. Redis采用单线程架构和I/O多路复用模型，纯内存访问；
2. 持久化使用RDB和AOF。
```

#### 数据结构

```
Redis有5种数据类型：string、hash、list、set、zset
1. string
string内部编码有三种：
int: 8个字节长整形
embstr: <=39字节的字符串
raw: >39字节字符串
string类型使用场景：
1、缓存； 2、计数； 3、session； 4、限速（短信验证码）

2.hash （键值本身也是一个键值对，相当于一个结构体的某一条对应了其他的信息）
hash内部编码有两种：
ziplist：列表
hashtable：哈希表
hash类型使用场景： 用户属性等信息，和关系型数据库匹配

3.list（一对多，有顺序的）
list内部编码有两种：
ziplist（个数或者值比较小）
linkedlist
list使用场景： 消息队列、列表等

4.set（一对多，无序的，不能重复）
set内部编码有两种：
intset： 整数集合
hashtable： 哈希表
set使用场景: 微博类似的标签、社交

5.zset
zset内部编码有两种：
ziplist
skiplist： 跳跃表
zset使用场景： 排行榜之类的，有特定排序规律的
```

#### 遍历键

```
Redis遍历键有两种方式：
1、keys：全量遍历键，会阻塞
2、scan：渐进式遍历，需要多次遍历完成，类似的，hgetall、smembers、zrange都会阻塞，相应的有hscan、sscan、zscan等命令替代。
```

#### 慢查询

```
Redis执行命令过程：
发送-->排队-->执行（慢查询统计的地方）-->返回
```

#### Redis-Shell

```
redis-cli --slave命令，可以用来监视其他操作。
```

#### Pipeline

```
将命令打包，不用每次命令花费多余的时间在除了执行之后的其他操作上，主要在网络时延比较高的情况下很明显，pipeline是非原子的。
```

#### Bitmap

```
按位去存储数据，用 0 1 表示，可以节省内存（主要对于数据量大一点的）。
```

#### HyperLogLog

```
一种基数算法，利用极小的内存空间完成独立总数的统计，存在误差率。
```

#### 发布订阅功能、GEO（地理信息定位）...