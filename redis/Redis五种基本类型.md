[TOC]



## Redis启动

## 1.String

String是Redis里边最最简单的一种数据结构。在Redis中，所有的key都是字符串，但是不同的key对应的value则具备不同的数据结构，平常说的五种类型是指的Value的数据类型不同。

Redis中的字符串是动态字符串，内部是可以修改的，像JAVA中的Stringbuffer,他采用冗余空间的方式来减少内存的频繁分配。当字符串小于1M时，扩容是在现有的空间基础上加倍，每日扩孔扩1M空间，最大512M。

- append 

  使用append命令时，如果Key存在，则直接在对应的value后追加值，否知新创建 

- decr

  可以实现对value的减一操作，前提value是数字，否则会报错，如果value不存在，则会给一个默认值为0，在默认值基础上减一

- dcerby

  和decr类似，但可以自己设置步长

- get

  获取一个key的value.

- getrange

  用来返回key对应value的子串，类似于java中的subString

- getset

  获取并更新某一个Key，返回旧值，并设置新值给key

- incr

  给key递增

- incrby

  给某一个key的value递增，并能指定步长

- incrbyfloat

  给某一个key的value递增，并能指定步长,步长可以是浮点数

- mget和mset

  批量获取和批量存储

- ttl

  查看Key的有限期，-1表示永久有效，-2表示已经过期

- setex

  再给key设置value的同时，还设置过期时间(秒)

- psetex

  和setex类似，只有时间单位不同,单位为毫秒

- setnx

  默认情况下，set命令会覆盖已经存在key的value，sernx则不会

- msetnx

  批量设置,如果有一个存在，整个操作都会失效

- setrange

  覆盖一个已经存在的key指定index的value

- strlen

  查看字符串长度

  ​

  ​

  ### 1.1 BIT命令

  在Redis中，字符串都是以二进制方式存储，BIT相关命令就是对二进制进行操作。

  - getbit

    返回key对应的value在offset处的value值

  - setbit

    修改key对应的value在offset处的value值

  - bitcount

    统计二进制中1的个数

  ​

  ​


## 2.List

- lpush

  表示将value从左到右依次插入表头

- lrange

  返回指定区间内的元素 

- rpush

  与lpush类似，不同的是从右往左插入到表尾

- rpop

  移除并返回列表的尾元素

- lpop

  移除并返回列表的头元素

- lindex

  返回列表中下标尾Index的元素

- ltrim

  对列表进行修剪

- blpop

  阻塞式的弹出，相当于lpop的阻塞版

## 3.Set

元素不能重复

- sadd

  添加元素到key中

- smembers

  获取一个key下的所有元素

- srem

  移除指定的元素

- sismember

  返回某个元素是否在集合中（1 表示存在，0 不存在）

- scard

  返回集合的数量

- srandmember

  随机返回一个元素

- spop

  随机返回一个元素并且出栈

- smove

  把一个元素从一个集合移到另一个集合中

- sdiff

  返回两个集合的差集

- sinter

  返回两个集合的交集

- sdiffstore

  类似于sdiff，不同的是计算出来的结果会保存在一个新的集合中

- sinterstore

  类似于sinter，不同的是计算出来的结果会保存在一个新的集合中

- sunion

  求并集

- sunionstore

  类似于sunion，不同的是计算出来的结果会保存在一个新的集合中

## 4.Hash

在hash结构中，key是一个字符串，value是key/value键值对

- hset

  添加值

- hget

  获取值

- hmset

  批量设置

- hmget

  批量获取

- hdel

  删除一个指定的field

- hsetnx

  默认情况下,key和field相同，会覆盖已有的value,hsetnx不会

- hvals

  获取所有的value

- hkeys

  获取所有的Key

- hgetall

  获取所有的key和value

- hexists

  返回field是否存在(1 存在，0 不存在)

- hincrby

  给指定value自增 

- hincrbufloat

  能自增浮点数

- hlen

  返回某个key中value的数量

- hstrlen

  返回某个key中某个field的字符串长度

## 5.ZSet

有序集合，能通过socre排序

- zadd

  将指定的元素添加到有序集合中

- zscore

  返回member的score值

- zrange

  返回集合中的子集

- zrevrange

  倒叙返回集合中子集

- zcard

  返回元素个数

- zcount

  返回score在某个区间内的元素个数

- zrangebyscore

  按照score的范围返回元素

- zrank

  返回元素的排名从小到大

- zrevrank

  返回元素排名从大到小

- zincrby

  score自增

- zinterstore

  计算指定个数集合的交集并将value求和后放到指定集合中

- zrem

  弹出一个元素

- zlexcount

  计算有序集合中成员数量

- zrangebylex

  返回指定区间内的成员



## 6.key

- del

  删除一个key/value

- dump

  序列化指定的Key

- exists

  判断一个key是否存在

- ttl

  查看key的有效期

- expire

  设置Key的有效期,如果key在过期之前被重新set了，过期时间会失效

- persist

  移除Key的过期时间

- keys *

  查看所有的key

- pttl

  和ttl类似，不同的是返回的是毫秒值

## 7.补充

1.四种数据类型(String除外)，第一次使用时，如果容器不存在，会自动创建一个

2.四种数据类型(String除外)，如果里面没有元素了，那么立即删除元素释放内存

