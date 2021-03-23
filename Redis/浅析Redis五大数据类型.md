浅析Redis五大数据类型



#### 一、redis介绍

**Redis**（remote dictionary server）是一种**NoSql**类型数据库。

什么是NoSql数据库？

现在随着现阶段业务海量用户以及高并发业务，使得关系型数据库遇到很多瓶颈。例如：性能瓶颈，读写大量数据，使得磁盘IO性能低下；扩展性能差，在数据关系更复杂的情况下，关系型数据库扩展性变差，不再满足大规模集群。

若要解决这些问题，第一：减少磁盘访问的次数，可以考虑在内存加内存；第二：去除复杂的数据关系，越简单越好，可以考虑不存储关系，只存储数据

而NoSql（NotOnlySql）是关系型数据库的拓展。其特点是可扩容、可伸缩、大数据量下具有高性能、灵活的数据模型、高可用。Redis是NoSql的一种数据库。

Redis 是一种**高性能键值对**(key-value)数据库，可以用作数据库、缓存、消息中间件等

什么是键值对？

对于mysql关系型数据库的结构是：库-表-主键-信息
键值对是：K-V   Key永远是string类型，而Value可以是第二章介绍的5种数据结构。

值得注意的是，redis只存储热点数据，而不是全部数据，基础数据信息还是存储在mysql使用。所以redis是和mysql配合使用的。

**Redis特点**：数据间没有必要的关联关系

-   内部采用单线程机制进行工作
-   高性能-内部
-   多数据类型支持
-   持久化支持，可以进行数据灾难恢复

**应用场景**：(后续我们详细介绍每种数据类型对应的场景)

- 为热点数据加速查询（主要场景）。热点商品，热点新闻等高访问信息
- 任务队列，如秒杀，抢购
- 即时信息查询，例如排行榜、在线人数等
- 时效性信息控制，如验证码
- 消息队列
- 分布式锁

#### 二、redis五大数据类型

##### 1、String

最简单的数据存储类型。一个存储空间只保存一个数据，若字符串是数字按照数字处理。redis所有的操作是原子性的，采用单线程处理所有的业务，命令是一个一个执行的，因此无需考虑高并发带来的数据影响

最大存储量：512MB

常用操作

```go
set key value //赋值
get key //获取指定key的value
del key //删除指定key
incr key //给指定key的值加一
decr key //给指定的key的值减一
incrby key increment //给指定的key的值加increment
decrby key decrement //给指定的key的值减decrement
```

![1616466802892](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1616466802892.png)



**适用场景**

1）投票的有效时长

可以指定数据的生命周期，来控制数据是什么时候失效，用过数据失效控制对应的业务行为，适用于具有时效性限定控制操作。

```go
setnx key seconds value // 设置key对应value值的有效时长是seconds
```

2）大V用户信息的数据存储情况：粉丝数、博文数等

增加一个关注的粉丝，使用incr直接增加

![1616467349230](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1616467349230.png)

![1616467460976](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1616467460976.png)

redis用于各种数据结构和非结构高热度数据流访问。

这里说明一下数据库中热点数据key命名惯例：使用：分割数据层次

![1616467747538](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1616467747538.png)

3）分布式锁

[分布式锁实践](https://juejin.cn/post/6889268039827619848)

##### 2、Hash

对存储的数据进行编组，典型的应用存储对象信息。一个存储空间可以保存多个键值对数据。底层是hash表的实现。

![1616468697989](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1616468697989.png)

value值只存储字符串，每个hash可以存储2^32-1个键值对

常用操作

```go
hset key field value //为指定的key设置一个含有field、value的值
hmset key field1 value1 field2 value2 //给指定的key设置多个含有field、value的值
hget key field //获取指定key、field的值
hmget key field field2 //获取多个key、field的值
hgetall key //获取key下所有field的值，若内部field过多，遍历整体数据效率会很耗时。
hdel key field1 field2 //删除指定key下指定field的值(可以单个或者多个)
hdel key //删除指定key下所有field的值
hexists key field //查看指定key的field是否存在
hlen key //查看指定key下field的个数
hkeys key //获取指定key下所有的field
hvals key //获取指定key下所有的value
```

**适用场景**

1）电商网站购物车

购物车信息，可以容易增删改查，管理数量等

2）商家信息管理

应用于抢购，发放消费券(发完为止)等数据存储设计

##### 3、List

存储多个数据，对数据进入存储空间的顺序要进行区分。数据主要要体现顺序，底层使用双向链表实现。

list保存的数据都是string类型的，最多有2^32-1个数据。list有索引的概念。

常用操作

```go
lpush key value1 value2 //给指定的key从头部开始添加value，如果key不存在则会自动创建
rpush key value1 value2 //给指定的key从尾部开始添加value，如果key不存在同样自动创建
lrange key 0 -1 //获取指定key从开始位置到结束位置的值，从0开始-1为最后一位
lpop key //将指定key头部的值弹出，如果key不存在返回nil，如果存在返回头部元素
rpop key //将指定key尾部的值弹出，如果key不存在返回nil，如果存在返回尾部元素
llen key //获取指定key链表的长度，如果key不存在返回0
lpushx key value //给指定key从头部开始添加value，只有key存在才能添加，不存在返回0，并且只能添加一个值
rpushx key value //给指定key从尾部开始添加value，同上
lrem key index value //删除指定key的指定值，如果index大于0则删除index位置上与value相等的值，小于则从尾部开始，等于0则删除链表中所有与value相等的值
lset key index value //给指定位置上的value修改为指定值，如果角标不存在则报错((error) ERR index out of range)
linsert key before value newValue //给指定key链表中指定value前添加一个新value，从头开始
linsert key after value newValue //给指定key链表中指定value前添加一个新value，从尾开始
rpoplpush key key2 //将指定key的尾部元素弹出压入key2的头部，如果两个key相同，则在同一个链表中执行尾弹头压的操作
```

**适用场景**

1）朋友圈点赞，按照点赞顺序显示好友

应用于具有操作先后顺序的数据控制。

2）分页操作

通常第一页信息来自list，其他页信息来自数据库进行加载。

##### 4、Set

存储大量数据，在查询方面提供更好的查询效率。与hash的存储结构相同，不同的是，只存储fileds值，不存储value值。Set不允许数据重复，也不能启动value功能



![1616470102514](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1616470102514.png)

常用操作

```go
sadd key value1 value2 //给指定key中添加数据，元素不重复
smembers key //获取指定key中的元素
srem key members value1 value2 //删除指定key中的指定value
sismember key value //判断指定key中的指定value值是否存在，存在返回1，不存在返回0或者该key本身
sdiff key key2 //获取key自身元素中与key2不相同的元素(差集)
sinter key key2 //获取key与key2中相同的元素(交集)
sunion key key2 //将key与key2元素合并返回，不包括相同元素(并集)
scard key //获取指定key中元素的个数
srandmember key //随机获取指定key中的一个元素
sdiffstore key key1 key2 //将key1与key2的差集存储到key中
sinterstore key key1 key2 //将key1与key2的交集存储到key中
sunionstore key key1 key2 //将key1与key2的并集存储到key中
spop key // 获取某个key,将该数据移除
```

**适用场景**

1）应用于同类消息的关联搜索

​	显示共同好友/关注（一度）
​    由用户A出发，获取共同好友B的好友列表（一度）
​	由用户A出发，获取共同好友B的购物清单/游戏充值列表(二度)   

2）随机推荐类信息检索，例如推荐热点音乐、新闻等

3）不同类型不重复数据的合并操作，如权限配置

4）应用于同类型数据的快速去重，如访问量统计

5）基于黑名单与白名单设定服务控制（利用set的去重性）

##### 5、Sorted_set

需求：数据排序有利于数据的展示，需要提供一种根据自身特征进行排序。在set的基础上添加score排序字段。score不存储数据，只用于排序。

![1616479884856](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1616479884856.png)

常用操作

```go
zadd key score name score2 name //将 score name 存储到指定的key中，如果有相同的name则覆盖之前的，返回值为新加入的元素，之前存在的不算
zscore key name //获取指定key中name的score
zcard key //获取指定key中成员数量
zrem key name name2 //删除指定key中指定name、name2的值，可以指定多个
zrange key start end withscores //获取指定key中起始位置到结束位置的元素，不加withscores则只返回name，加上将score一并返回，从小到大
zrevrange key start end withscores//获取指定key中起始位置到结束位置元素以从大到小的顺序返回，包含两端
zremrangebyrank key start end //按照指定的排名范围删除元素，排名为分数从小到大，排名从0开始
zremrangebyscore key min max //按照指定的分数范围删除元素，包含min和max
zrangebyscore key min max withscores //返回指定key中指定分数范围的元素从小大
zrangebyscore key min max withscores limit start end //返回指定key中指定分数范围中指定角标范围的元素,start和end是索引
zincrby key score name //将指定key中指定name的元素的分数在原有基础上加score，并返回增加后的分数
zcount key min max //返回指定key中指定分数范围中元素的个数，包括起始和结束
zrank key name //返回指定key中指定name元素的排名(从小到大)，起始位置从0开始
zrevrank key name //返回指定key中指定name元素的排名(从大到小),起始位置从0开始
```

**适用场景**

1）为所有参与排名的资源建立排序依据。TOP10(歌曲，电影)

2）定时任务执行顺序管理或者过期管理。会员制度（月、季度、年）

(time获取系统当前时间)

**总结**

![1616480744040](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1616480744040.png)

**引用：**

[键值对数据库的学习](https://blog.csdn.net/m0_37602117/article/details/79152220?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-4.control&dist_request_id=1328680.62914.16164661105692345&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-4.control)

[Redis常见问题](https://juejin.cn/post/6844904017387077640)