[TOC]

1.创建多个实例并启动

2.命令行创建集群
~~~
redis-cli --cluster create 
            192.168.204.128:7001 
            192.168.204.128:7002 
            192.168.204.128:7003 
            192.168.204.128:7004 
            192.168.204.128:7005 
            192.168.204.128:7006 
            --cluster-replicas 1
~~~

3.动态添加节点
~~~
 redis-cli --cluster add-node 192.168.204.128:7007 192.168.204.128:7006 -a 123456
~~~

4.添加节点后如何重新分配slot
~~~
redis-cli --cluster reshard 192.168.204.128:7001 
--cluster-from 99819a8677fae9da5f1684125d4004a7f76453c1 
--cluster-to e1d5d39b4a0f4db1eb0862a90b6264f2665e810f 
--cluster-slots 1024 
-a 123456
~~~

## Jedis操作RedisCluster
~~~java
public class RedisCluster {
    public static void main(String[] args) {
        Set<HostAndPort> nodes = new HashSet<>();
        nodes.add(new HostAndPort("192.168.204.128",7001));
        nodes.add(new HostAndPort("192.168.204.128",7002));
        nodes.add(new HostAndPort("192.168.204.128",7003));
        nodes.add(new HostAndPort("192.168.204.128",7004));
        nodes.add(new HostAndPort("192.168.204.128",7005));
        nodes.add(new HostAndPort("192.168.204.128",7006));
        nodes.add(new HostAndPort("192.168.204.128",7007));
        GenericObjectPoolConfig<Object> config = new GenericObjectPoolConfig<>();
        //连接池最大空闲数
        config.setMaxIdle(300);
        //最大连接数
        config.setMaxTotal(1000);
        //连接最大等待时间，单位毫秒,如果是 -1 表示没有限制
        config.setMaxWaitMillis(1000);
        //空闲时监测有效性
        config.setTestOnBorrow(true);
        JedisCluster cluster = new JedisCluster(nodes, 1500, 15000, 5, "123456", config);
        String set = cluster.set("k1", "v1");
        System.out.println(set);
        String k1 = cluster.get("k1");
        System.out.println(k1);
    }
}
~~~