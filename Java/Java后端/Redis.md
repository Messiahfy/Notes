## 概要
redis是非关系型数据库，数据为键值对，数据存储在内存中，一般用于缓存、任务队列等。

## 数据结构
redis存储的数据为键值对，值可以是以下类型：
1. 字符串 strings
2. 散列 hashes
3. 列表 lists
4. 集合 sets
5. 有序集合sorted sets

## 命令操作
### 字符串
* 存储：set key value
* 获取：get key
* 删除：del key

### 散列
* 存储：hset key field value
* 获取：hget key field，或者hgetall key 获取所有的field和value 
* 删除：hdel key field

其他类似

### 通用命令
* keys * 查询所有的键
* type key：获取键对应的值的类型
...

### 持久化
redis的数据存储在内存中，所以需要持久化来避免数据丢失，持久化方式分为两种：
1. RDB：默认方式，在指定的时间间隔能对你的数据进行快照存储
2. AOF：记录每次对服务器写的操作，当服务器重启的时候会重新执行这些命令来恢复原始的数据

## Jedis
redis提供各个语言的客户端连接工具，比如Java，有对应的Jedis的Jar包