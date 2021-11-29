[TOC]

## 11.1 预备知识
Pipeline(管道) 本质上是由客户端提供的一种操作。Pipeline通过调整指令列表的读写顺序，可以大幅度的节省IO 时间，提高效率。

## 11.2 简单限流
~~~java
public class RateLimiter {
    private Jedis jedis;

    public RateLimiter(Jedis jedis) {
        this.jedis = jedis;
    }

    /**
     * 限流方法
     * @param user 操作的用户，相当于限流的对象
     * @param action 具体的操作
     * @param period 时间窗，限流的周期
     * @param maxCount 限流的次数
     * @return
     */
    public boolean isAllowed(String user,String action,int period,int maxCount){
        //1.数据用zset保存，首先生成一个key
        String key = user + "-" + action;
        //2.获取当前时间戳
        long nowTime = System.currentTimeMillis();
        //3. 建立管道
        Pipeline pipelined = jedis.pipelined();
        pipelined.multi();
        //4.将当前的操作存下来
        pipelined.zadd(key,nowTime,String.valueOf(nowTime));
        //5. 移除时间窗之外的数据
        pipelined.zremrangeByScore(key,0,nowTime - period * 1000L);
        //6. 统计剩下的key
        Response<Long> response = pipelined.zcard(key);
        //7. 将当前key 设置一个过期时间，过期时间就是时间窗
        pipelined.expire(key,period + 1);
        //关闭管道
        pipelined.exec();
        pipelined.close();

        //8. 比较时间窗内的操作数
        return response.get() <= maxCount;
    }

    public static void main(String[] args) {
        Redis redis = new Redis();
        redis.execute(jedis -> {
            RateLimiter rateLimiter = new RateLimiter(jedis);
            for (int i = 0; i < 20; i++) {
                System.out.println(rateLimiter.isAllowed("wpy", "publish", 3, 3));
            }
        });
    }
}
~~~

## 11.3 深入限流操作
Redis4.0开始提供了一个Redis-Cell模块，这个模块使用**漏斗算法**，提供了一个非常好用的限流指令。

漏斗算法就像名字一样，是一个漏斗，请求从大口进，小口出进入到系统中，这样，无论是多大的访问量，最终进入到系统中的请求都是固定的。
使用漏斗算法，需要我们首先安装Redis-Cell模块：

- [https://github.com/brandur/redis-cell](https://github.com/brandur/redis-cell)

docker安装步骤:
~~~
//拉取镜像
docker pull hsz1273327/redis-cell
//运行
docker run --name wpy-redis-cell -d -p 6379:6379 hsz1273327/redis-cell --requirepass 123456
//进入容器
docker exec -it wpy-redis-cell redis-cli -a 123456
~~~

redis启动成功后，如果存在CL.THROTTLE命令，说明redis-cell安装成功了

CL.THROTTLE命令一共有五个参数:

1.第一个参数是key

2.第二个参数是漏斗的容量

3.第三个参数是时间窗内可以操作的次数

4.第四个参数是时间窗

5.每次漏出数量(可选)

执行完成后，返回值也有五个
- 第一个参数 0 表示允许，1 表示拒绝
- 第二个参数是漏斗的容量
- 第三个参数是漏斗的剩余空间
- 第四个参数是如果拒绝了，多长时间后可以再试
- 第五个参数是多长时间后，漏斗会完全空出来

## 11.4 Lettuce扩展
首先定义一个命令接口
~~~java
public interface RedisCommandInterface extends Commands {
    @Command("CL.THROTTLE ?0 ?1 ?2 ?3 ?4")
    List<Object> throttle(String key,Long init,Long count,Long period,Long quota);
}
~~~
定义完成后直接调用即可:
~~~java
public class ThrottleTest {
    public static void main(String[] args) {
        RedisClient redisClient = RedisClient.create("redis://123456@192.168.204.128");
        StatefulRedisConnection<String, String> connect = redisClient.connect();
        RedisCommandFactory factory = new RedisCommandFactory(connect);
        RedisCommandInterface commands = factory.getCommands(RedisCommandInterface.class);
        List<Object> throttle = commands.throttle("wpy-publish", 10L, 10L, 60L, 1L);
        System.out.println(throttle);
    }
}
~~~