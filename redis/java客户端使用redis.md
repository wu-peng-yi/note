[TOC]

## 1.开启远程连接

Redis默认不支持远程连接，需要手动修改配置文件开启。

## 2.Jedis

jedis github地址[https://github.com/redis/jedis](https://github.com/redis/jedis)

### 2.1 Jedis连接

- 首先创建一个普通maven项目

- 添加jedis依赖

  ~~~xml
  <dependency>
      <groupId>redis.clients</groupId>
      <artifactId>jedis</artifactId>
      <version>3.5.1</version>
      <type>jar</type>
      <scope>compile</scope>
  </dependency>
  ~~~

  测试方法

~~~java
		//构造一个jedis对象，使用默认端口6379可以不用写
        Jedis jedis = new Jedis("192.168.204.128");
        //密码验证
        jedis.auth("123456");
        //测试连接
        String ping = jedis.ping();
        //返回Pong表示连接成功
        System.out.println(ping);
~~~



一旦连接上redis，剩下操作就很简单了,因为jedis的方法API和Redis的命令高度一致，直接使用即可

### 2.2 连接池

在实际应用中，Jedis实例一般都通过连接池来获取，由于Jedis不是线程安全的，所有当我们使用Jedis对象时，从连接池获取，使用完成后，交还给连接池。

~~~java
public class JedisPoolTest {
    public static void main(String[] args) {
        //1.构造一个Jedis连接池
        JedisPool jedisPool = new JedisPool("192.168.204.128",6379);
        //2.从连接池获取一个连接
        Jedis jedis = jedisPool.getResource();
        //3.jedis操作
        jedis.auth("123456");
        String ping = jedis.ping();
        System.out.println(ping);
        //4.归还连接
        jedis.close();
    }
}
~~~

如果第三步抛出异常的话，会导致第四步无法执行，所以我们要对代码进行改进，确保第四步执行。

~~~java
public class JedisPoolTest {
    public static void main(String[] args) {
        Jedis jedis = null;
        try {
            //1.构造一个Jedis连接池
            JedisPool jedisPool = new JedisPool("192.168.204.128",6379);
            //2.从连接池获取一个连接
            jedis = jedisPool.getResource();
            //3.jedis操作
            jedis.auth("123456");
            String ping = jedis.ping();
            System.out.println(ping);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            //4.归还连接
            if (jedis != null) {
                jedis.close();
            }
        }

    }
}
~~~

通过finally确保jedis一定被关闭。

利用JDK1.7中的try-with-resource进行改造

~~~java
public class JedisPoolTest {
    public static void main(String[] args) {
        //1.构造一个Jedis连接池
        JedisPool jedisPool = new JedisPool("192.168.204.128", 6379);
        //2.从连接池获取一个连接
        try (Jedis jedis = jedisPool.getResource();) {
            //3.jedis操作
            jedis.auth("123456");
            String ping = jedis.ping();
            System.out.println(ping);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
~~~

但是这段代码无法实现强约束，可以做进一步改进

~~~java
public interface CallWithJedis {
    void call(Jedis jedis);
}


public class Redis {
    private JedisPool pool;

    public Redis() {
        GenericObjectPoolConfig<Object> config = new GenericObjectPoolConfig<>();
        //连接池最大空闲数
        config.setMaxIdle(300);
        //最大连接数
        config.setMaxTotal(1000);
        //连接最大等待时间，单位毫秒,如果是 -1 表示没有限制
        config.setMaxWaitMillis(1000);
        //空闲时监测有效性
        config.setTestOnBorrow(true);
        /**
         * 1. redis地址
         * 2. redis端口
         * 3. 超时时间
         * 4. 密码
         */
        pool = new JedisPool(config,"192.168.204.128",6379,30,"123456");
    }

    public void execute(CallWithJedis callWithJedis){
        try(Jedis jedis = pool.getResource();){
            callWithJedis.call(jedis);
        }catch (Exception e){

        }
    }
}

public class JedisPoolTest {
    public static void main(String[] args) {
       

        Redis redis = new Redis();
        redis.execute(jedis -> {
            System.out.println(jedis.ping());
        });

    }
}
~~~



## 3.Lettuce

github地址[https://github.com/lettuce-io/lettuce-core](https://github.com/lettuce-io/lettuce-core)

Lettuce与Jedis的比较:

1.Jedis在实现过程中是直接连接Redis的，在多个线程之间共享一个Jedis实例，这是线程不安全的，如果想在多线程场景下使用Jedis，就得使用连接池，这样，每个线程都有自己的Jedis实例。

2.Lettuce 基于目前很火的Netty NIO 框架来构建，所以克服了Jedis线程不安全的问题，支持同步，异步，以及响应式调用，多个线程可以共享一个连接实例。



使用Lettuce，首先创建一个Maven项目，添加Lettuce依赖

~~~XML
<dependency>
            <groupId>io.lettuce</groupId>
            <artifactId>lettuce-core</artifactId>
            <version>5.2.2.RELEASE</version>
        </dependency>
~~~

简单测试案例

~~~java
public class LettuceTest {
    public static void main(String[] args) {
        RedisClient redisClient = RedisClient.create("redis://123456@192.168.204.128");
        StatefulRedisConnection<String, String> connect = redisClient.connect();
        RedisCommands<String, String> sync = connect.sync();
        String set = sync.set("name", "wpy");
        String name = sync.get("name");
        System.out.println(name);
    }
}
~~~

注意:密码传递方式，直接卸载连接地址里面。