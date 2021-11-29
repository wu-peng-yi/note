分布式锁是Redis比较常见的使用场景。

**问题场景:**

>  例如一个简单的用户操作,一个线程去修改用户的状态，首先从数据库中读出用户的状态，然后再内存中进行修改，修改完成后，再存回去，在单线程中，没有问题，但在多线程中，由于读取，修改，存 这是三个操作，不是原子操作。所以在多线程中，这样会出问题。

对于这种问题，我们可以使用分布式锁来限制程序的并发执行。



**分布式锁实现的思路很简单，就是进来一个线程先占位，当别的线程进来操作时，发现已经有人占位了，就会放弃或者稍后再试。**



在Redis中，占位一般使用setnx指令，先进来的先占位，线程的操作执行完成后，在调用del指令释放位置。

~~~java
public class LockTest {
    public static void main(String[] args) {
        Redis redis = new Redis();
        redis.execute(jedis -> {
            Long setnx = jedis.setnx("k1", "v1");
            if(setnx == 1){
                //没人占位
                String set = jedis.set("name", "wpy");
                System.out.println(set);
                jedis.del("k1");//释放资源
            }else {
                //有人占位，停止/暂缓 操作
            }
        });
    }
}
~~~

上面代码存在的问题:如果在业务执行过程中抛异常或者挂了，导致del指令没被调用，这样k1无法释放，后来的请求全部阻塞，锁永远无法释放。要解决这个问题，我们可以给锁添加一个过期时间，确保锁在一定的时间之后能够得到释放。改进后的代码如下；

~~~java
public class LockTest {
    public static void main(String[] args) {
        Redis redis = new Redis();
        redis.execute(jedis -> {
            Long setnx = jedis.setnx("k1", "v1");
            if(setnx == 1){
                //没人占位
                //给锁添加过期时间，防止应用过程中抛出异常导致锁无法释放
                jedis.expire("k1",5);
                jedis.set("name", "wpy");
                String name = jedis.get("name");
                System.out.println(name);
                jedis.del("k1");//释放资源
            }else {
                //有人占位，停止/暂缓 操作
            }
        });
    }
}
~~~

这样改造之后，还有一个问题，就是在获取锁和设置过期时间之间如果如果服务器突然挂掉了，这个时候锁被占用，无法及时得到释放，也会造成死锁，因为获取锁和设置过期时间是两个操作，不具备原子性。
为了解决这个问题，从 Redis2.8 开始，setnx 和 expire 可以通过一个命令一起来执行了，我们对上述代码再做改进：

~~~java
public class LockTest {
    public static void main(String[] args) {
        Redis redis = new Redis();
        redis.execute(jedis -> {
            String set = jedis.set("k1", "v1", new SetParams().nx().ex(5));
            if("ok".equals(set)){
                //没人占位
                //给锁添加过期时间，防止应用过程中抛出异常导致锁无法释放
                jedis.expire("k1",5);
                jedis.set("name", "wpy");
                String name = jedis.get("name");
                System.out.println(name);
                jedis.del("k1");//释放资源
            }else {
                //有人占位，停止/暂缓 操作
            }
        });
    }
}

~~~

为了防止业务代码在执行的时候抛出异常，我们给每一个锁添加了一个超时时间，超时之后，锁会被自动释放，但是这也带来了一个新的问题：如果要执行的业务非常耗时，可能会出现紊乱。举个例子：第一个线程首先获取到锁，然后开始执行业务代码，但是业务代码比较耗时，执行了 8 秒，这样，会在第一个线程的任务还未执行成功锁就会被释放了，此时第二个线程会获取到锁开始执行，在第二个线程刚执行了 3 秒，第一个线程也执行完了，此时第一个线程会释放锁，但是注意，它释放的第二个线程的锁，释放之后，第三个线程进来。
对于这个问题，我们可以从两个角度入手：
尽量避免在获取锁之后，执行耗时操作。
可以在锁上面做文章，将锁的 value 设置为一个随机字符串，每次释放锁的时候，都去比较随机字符串是否一致，如果一致，再去释放，否则，不释放。
对于第二种方案，由于释放锁的时候，要去查看锁的 value，第二个比较 value 的值是否正确，第三步释放锁，有三个步骤，很明显三个步骤不具备原子性，为了解决这个问题，我们得引入 Lua 脚本。
Lua 脚本的优势：
使用方便，Redis 中内置了对 Lua 脚本的支持。
Lua 脚本可以在 Redis 服务端原子的执行多个 Redis 命令。
由于网络在很大程度上会影响到 Redis 性能，而使用 Lua 脚本可以让多个命令一次执行，可以有效解决网络给 Redis 带来的性能问题。
在 Redis 中，使用 Lua 脚本，大致上两种思路：
提前在 Redis 服务端写好 Lua 脚本，然后在 Java 客户端去调用脚本（推荐）。
可以直接在 Java 端去写 Lua 脚本，写好之后，需要执行时，每次将脚本发送到 Redis 上去执行。
首先在 Redis 服务端创建 Lua 脚本，内容如下

~~~lua
if redis.call("get",KEYS[1])==ARGV[1] then   
    return redis.call("del",KEYS[1])
else   
    return 0
end

~~~

接下来，可以给 Lua 脚本求一个 SHA1 和，命令如下：

~~~
cat lua/releasewherevalueequal.lua | redis-cli -a javaboy script load --pipe
~~~

script load 这个命令会在 Redis 服务器中缓存 Lua 脚本，并返回脚本内容的 SHA1 校验和，然后在 Java 端调用时，传入 SHA1 校验和作为参数，这样 Redis 服务端就知道执行哪个脚本了。

接下来，在 Java 端调用这个脚本。

~~~java
public class LuaTest {
    public static void main(String[] args) {
        Redis redis = new Redis();
        redis.execute(jedis -> {
            //1. 先获取一个随机字符串
            String value = UUID.randomUUID().toString();
            //2. 获取锁
            String k1 = jedis.set("k1", value, new SetParams().nx().ex(5));
            //3. 判断是否拿到锁
            if("OK".equals(k1)){
                //拿到锁
                String site = jedis.set("site", "www.baidu.com");
                String site1 = jedis.get("site");
                System.out.println(site1);
                //5. 释放锁
                jedis.evalsha("1111", Collections.singletonList("k1"), Collections.singletonList(value));
            }else {
                //没拿到锁
                System.out.println("没拿到锁");
            }
        });
    }
}
~~~

