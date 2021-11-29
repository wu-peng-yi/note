[TOC]

正常来说，一个可以商用的数据库往往都有比较完善的事务支持，Redis当然也不例外。相对于关系型数据库中的事务模型，Redis中的事务要简单许多。因为简单，所以Redis中的事务模型不太严格，所以我们不能像使用关系型数据库中的事务那样来使用Redis。

在关系型数据库中，和事务相关的三个指令分别是:
- begin
- commit
- rollback

在Redis中，当然也有对应的指令:
- multi
- exec
- discard


## 1.原子性
注意，**Redis中的事务并不能算作原子性。它仅仅具备隔离性，也就是说当前的输入可以不被其他事务打断。**

由于每一次事务操作涉及到的指令比较多，为了提高执行效率，我们在使用客户端时，可以通过pipeline来优化指令的执行。

Redis中还有一个watch 指令，watch可以用来监控一个key,通过这种监控，我们可以确保在exec 之前，watch 监控的键没有被修改过。

## 2.Java代码实现
~~~java
public class TransactionTest {
    public static void main(String[] args) {
        Redis redis = new Redis();
        redis.execute(jedis ->{
            new TransactionTest().saveMoney(jedis,"wpy",1000);
        });
    }

    public Integer saveMoney(Jedis jedis, String userId, Integer money) {
        while (true) {
            jedis.watch(userId);
            int v = Integer.parseInt(jedis.get(userId)) + money;
            Transaction tx = jedis.multi();
            tx.set(userId, String.valueOf(v));
            List<Object> exec = tx.exec();
            if (exec != null) {
                break;
            }
        }
        System.out.println("dsadas");
        return Integer.parseInt(jedis.get(userId));
    }
}
~~~