[TOC]

Redis 是一个缓存工具，也叫做 NoSQL 数据库。既然是数据库，必然支持数据的持久化操作。在Redis中，数据库持久化一共有两种方案:

1.快照方式

2.AOF 日志

## 1.快照

### 1.1 原理
Redis 使用操作系统的多进程机制来实现快照持久化:Redis 在持久化时，会调用 glibc 函数 fork 一个子进程，然后将快照持久化操作完全交给子进程去处理，而父进程则继续处理客户端请求。在这个过程中，子进程能够看到的内存中的数据在子进程产生的一瞬间就固定下来了，再也不会改变。这就是为什么 Redis 持久化叫做快照。

### 1.2 具体配置

在Redis中，默认情况下，快照持久化的方式就是开启的。

默认情况下会产生一个 dump.rdb 文件，这个文件就是备份下来的文件。当Redis启动时，会自动加载这个文件，从该文件恢复数据。

具体配置在redis.conf中。
~~~
save 900 1
save 300 10
save 60 10000

#执行快照出错后，是否继续处理客户端的写命令
stop-writes-on-bgsave-error yes
#是否对快照文件进行压缩
rdbcompression yes
#快照文件名
dbfilename dump.rdb
#表示生成的快照文件存储位置
dir ./
~~~

### 1.3.备份流程
1.在Redis运行过程中，我们可以向Redis 发送一条save命令来创建一个快照。但是需要注意，save 是一个阻塞命令，Redis在收到save命令，开始处理备份操作之后，在处理完成之前，将不再处理其他的请求。其他命令被挂起，所以save 使用的并不多。

2.我们一般可以使用 bgsave, bgsave 会fork 一个子进程去处理备份请求，不影响父进程处理客户端请求。

3.如果我们定义的备份规则，有一个满足，也会触发bgsave。

4.当我们执行shutdown 时，也会触发 save 命令，备份完成后，将Redis关闭。

5.用Redis 搭建主从复制时，在从机连上主机之后，会自动发送一条 sync同步命令，主机收到命令之后，首先执行bgsave 对数据进行快照，然后才会给丛机发送快照数据进行同步。

## 2.AOF
与快照持久化不同，AOF 持久化是将被执行的命令追加到 aof 文件末尾，在恢复时，只需要把记录下来的命令从头到尾执行一遍即可。

默认情况下，AOF 是没有开启的，我们需要手动开启:
~~~
# 开启 aof配置
appendonly yes

# AOF 文件名
appendfilename "appendonly.aof"

#备份的时机，总是/每秒/从不
# appendfsync always
appendfsync everysec
# appendfsync no

#表示aof文件压缩时是否还继续进行同步操作
no-appendfsync-on-rewrite no

#表示当目前aof文件大小超过上一次重写时的 aof 文件大小的百分之多少的时候，再次进行重写
auto-aof-rewrite-percentage 100

# 如果之前没有重写过，则以启动时的aof大小为依据，同时要求aof 文件至少大于64M
auto-aof-rewrite-min-size 64mb
~~~
同时为了避免快照备份的影响，可以先将rdb备份关闭