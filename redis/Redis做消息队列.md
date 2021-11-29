[TOC]

我们平时说到消息队列，一般都是指RabbitMQ,RocketMQ,ActiveMQ以及大数据里的kafka,这些事常见的消息中间件，作为专业消息中间件，它里面提供了许多功能。

但是，当我们需要使用消息中间件的时候，并非每次都需要专业的消息中间件，假如我们只有一个消息队列，只有一个消费者，那就没有必要使用上面的专业的消息中间件，这种情况可以直接使用Redis来做消息队列。

Redis的消息队列不是特别专业，没有很多高级特性，适用于简单场景，如果对于消息可靠性有着极高的追求，不适合使用Redis做消息队列。

## 1.消息队列

Redis做消息队列，使用它的List数据结构就可以实现，我们可以使用lpush/rpush操作来实现入列，使用lpop/rpop来实现出列。

在客户端(例如java端)，我们会维护一个死循环来不停地从队列中读取消息并处理，如果队列中有消息，则直接获取到，如果没有消息，就会陷入死循环，直到下一次有消息进入，这种死循环会造成大量资源浪费，这个时候，我们可以使用之前讲的blpop/brpop。

## 2.延迟消息队列

延迟队列可以通过zset实现，因为zset中有score，我们可以把时间作为score,将value存到redis中，然后通过轮询方式去不断地读取消息出来。

首先，如果消息是一个字符串，直接发送即可，如果是一个对象，则需要对对象进行序列化，这里使用JSON实现序列化和反序列化。

所以，首先在项目中添加json依赖:

~~~xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId
	<artifactId>jackson-databind</artifactId>
    <version>2.11.3</version>
</dependency>
~~~

接下来，构造一个消息对象

~~~java
public class WpyMessage {
    private String id;
    private Object data;

    public WpyMessage() {
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public Object getData() {
        return data;
    }

    public void setData(Object data) {
        this.data = data;
    }

    @Override
    public String toString() {
        return "WpyMessage{" +
                "id='" + id + '\'' +
                ", data=" + data +
                '}';
    }
}
~~~

接下来封装一个消息队列

~~~java
public class delayMessageQueue {
    private Jedis jedis;
    private String queue;

    public delayMessageQueue(Jedis jedis, String queue) {
        this.jedis = jedis;
        this.queue = queue;
    }

    /**
     * 消息入队
     * @param data 要发送的消息
     */
    public void queue(Object data){
        //构造一个消息对象
        WpyMessage msg = new WpyMessage();
        msg.setId(UUID.randomUUID().toString());
        msg.setData(data);
        //序列化
        try {
            String s = new ObjectMapper().writeValueAsString(msg);
            System.out.println("msg publish" + new Date());
            //消息发送，score 延迟5秒
            jedis.zadd(queue,System.currentTimeMillis()+5000,s);
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
    }

    /**
     * 消息消费
     */
    public void loop(){
        while (!Thread.interrupted()){
            //读取score在0 到当前时间戳之前的消息
            Set<String> zrange = jedis.zrangeByScore(queue, 0, System.currentTimeMillis(), 0, 1);
            if(zrange.isEmpty()){
                //消息是空的，休息500毫秒，然后继续
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    break;
                }
                continue;
            }
            //如果读取到了消息则直接读取消息出来
            String next = zrange.iterator().next();
            if(jedis.zrem(queue,next) > 0){
                //抢到了这条消息，接下来处理业务
                try {
                    WpyMessage wpyMessage = new ObjectMapper().readValue(next, WpyMessage.class);
                    System.out.println("receive msg"+wpyMessage);
                } catch (JsonProcessingException e) {
                    
                }
            }
        }
    }
}
~~~

~~~测试

~~~

