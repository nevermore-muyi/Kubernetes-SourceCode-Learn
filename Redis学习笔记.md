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


```

