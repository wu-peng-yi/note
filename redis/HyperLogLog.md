[TOC]

一般我们评估一个网站的访问量，有几个主要的参数:

- pv, Page View, 网页的浏览量
- uv, User View, 访问的用户

一般来说，pv或者uv的统计，可以自己来做，也可以借助一些第三方的工具，比如 cnzz,友盟等。

如果自己实现，pv 比较简单，可以直接通过reids计数器就能实现，但是uv 就不一样，uv 涉及到去重。

我们首先需要在前段给每个用户生成一个唯一id ，无论是登录用户还是未登录用户，都要有一个唯一id ，这个id 伴随着请求到达后端，在后端我们通过sadd 集合来存储这个id，最后通过scard 统计集合大小，进而得出uv数据。

如果是千万级别的uv，需要的存储空间非常惊人。而且，像UV统计这种，一般也不需要特别精确，所以我们可以使用今天的主角---HyperLogLog

Redis中提供的HyperLogLog 就是专门用力啊解决这个问题的，HyperLogLog 提供了一套不怎么精确但是够用的去重方案，会有误差，官方给出的误差是 0.81%，这个精确度统计UV够用了。

HyperLogLog  主要提供了两个命令: pfadd 和 pfcount。

pfadd  用来添加记录，类似于sadd，添加过程中，重复的记录会自动去重。

pfcount 则用来统计数据。

~~~java
public class HyperLogLog {
    public static void main(String[] args) {
        Redis redis = new Redis();
        redis.execute(jedis -> {
            for (int i = 0; i < 1000; i++) {
                jedis.pfadd("uv","u"+i,"u"+(i + 1));
            }
            long uv = jedis.pfcount("uv");
            System.out.println(uv);
        });
    }
}
~~~

这段程序打印 994，理论上1001，有误差，但在可以接受的范围内。

除了pfadd 和 pfcount外，还有个命令pfmerge ,用来合并统计结果，合并过程中，会自动去重多个集合中重复的元素。