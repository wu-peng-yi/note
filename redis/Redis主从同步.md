[TOC]

## 1.CAP
在分布式环境下，CAP原理是一个非常基础的东西，所有的分布式存储系统，都只能在CAP中选择两项实现。

c：consitent 一致性

a: avaliability 可用性

p: partition tolerance 分布式容忍性

在一个分布式系统中，这三个只能满足两个:在一个分布式系统中, P 肯定是要实现的，c 和 a 只能选择其中一个。大部分情况下，大多数网站架构选择了 ap。

在Redis中，实际上就是保证最终一致性。

Redis中，当搭建了主从服务之后，如果主从之间的连接断开了，Redis依然是可以操作的，相当于它满足了可用性，但是此时主从两个节点中的数据会有差异，相当于牺牲了一致性，但是Redis保持最终一致，当网络恢复时，从机会追赶主机，尽量保持数据一致。

### 2.1配置方式


## 3.Jedis操作哨兵模式
准备工作:

1. 所有的实例均配置masterauth (在redis.conf中)
2. 所有实例均需要配置绑定地址: bind 192.168.204.128

做好准备工作，然后启动三个Redis实例，同时启动哨兵。哨兵配置的时候，监控的Mater 不要直接写127.0.0.1，按如下方式写:
~~~java
192.168.204.128
~~~

~~~java
public class Sentinel {
    public static void main(String[] args) {
        JedisPoolConfig config = new JedisPoolConfig();
        config.setMaxTotal(10);
        config.setMaxWaitMillis(1000);
        String master = "mymaster";
        Set<String> sentinels = new HashSet<>();
        sentinels.add("192.168.204.128:26379");
        JedisSentinelPool sentinelPool = new JedisSentinelPool(master, sentinels, config, "123456");
        Jedis jedis = null;
        while (true) {
            try {
                jedis = sentinelPool.getResource();
                String k1 = jedis.get("k1");
                System.out.println(k1);
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                if (jedis != null) {
                    jedis.close();
                }
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {

                }
            }

        }
    }
}
~~~

## 4.SpringBoot
SpringBoot 操作哨兵模式