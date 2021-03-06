[TOC]
Reids3.2 开始提供了 GEO模块。该模块也使用了 GeoHash 算法。
## 1.GeoHash
核心思想:GeoHash 是一种地址编码方法，使用这种方式，能够将二维的空间经纬度数据编码成一个一维字符串。

地球上经纬度的划分:

以经过伦敦格林尼治天文台旧址的经线为 0 度经线，向东就是东经，向西就是西经。如果我们将西经定义为负，经度的范围就是[-180，180]。

维度北纬90度到南纬90度，如果我们将南纬定义为负，则纬度的范围就是[-90，90]。

接下来，以本初子午线和赤道为界，我们可以将地球上的点分配到一个二维坐标中。

GeoHash就是基于这样的思想，划分的次数越多，区域越多，每个区域的面积就更小，精确度就会提高。

GeoHash 具体算法:

以天安门广场为例(39.90960456049752,116.3972282409668):

1.纬度的范围在(-90,90)之间，中间值为0，对于39.90960456049752值落在(0,90)，因此得到的值为1

2.(0,90)的中间值为 45，39.90960456049752落在(0,45)之间，因此得到一个 0 

3.(0,45)的中间值为 22.5，39.90960456049752 落在(22.5,45)之间，因此得到一个 1 

... 以此类推，

这样的话，我们得到的纬度二进制是 101

按照同样的步骤，我们可以算出来经度的二进制是 110

接下来将经纬度合并(经度占偶数位，纬度占奇数位):

111001

按照Base32(0-9,b-z(去掉a i l o))对合并后的二进制数据进行编码,编码的时候，先将二进制转换为十进制，然后进行编码。

将编码得到的字符串，可以拿去geohash.org上解析。

GeoHash有哪些特点:

1.用一个字符串表示经纬度

2.GeoHash 表示的是一个区域而不是一个点。

3.编码格式有规律，例如:一个地址编码之后的格式是 123,另一个地址编码之后的格式是 123456,从字符串上就可以看出来，123456 处于 123 之中。

## 2.Redis中使用
添加地址:
~~~
127.0.0.1:6379> GEOADD city 116.3972282409668 39.90960456049752 beijing
127.0.0.1:6379> GEOADD city 114.085947 22.547 shenzhen
~~~
查看两个地址之间的距离
~~~
127.0.0.1:6379> geodist city beijing shenzhen km
"1943.4481"
~~~
获取元素的位置:
~~~
127.0.0.1:6379> geopos city beijing
1) 1) "116.39723092317581177"
   2) "39.90960415014365736"
~~~

获取hash值
~~~
127.0.0.1:6379> geohash city beijing
1) "wx4g09mfk10"
~~~

查看附近的人:
~~~
127.0.0.1:6379> georadiusbymember city beijing 200 km count 3 asc
1) "beijing"
~~~

以北京为中心，方圆两百千米以内的城市找出3个按照远近顺序排列，这个命令不会排除北京

当然，也可以根据经纬度来查询(将member 换成对应的经纬度):
~~~
127.0.0.1:6379> georadius city 116.3972282409668 39.90960456049752 2000 km withdist withhash withcoord count 4 desc
1) 1) "shenzhen"
   2) "1943.4481"
   3) (integer) 4046433733682118
   4) 1) "114.08594459295272827"
      2) "22.54699993773966327"
2) 1) "jinan"
   2) "352.2092"
   3) (integer) 4065932780085610
   4) 1) "117.05215662717819214"
      2) "36.78490889792045238"
3) 1) "shijiazhuang"
   2) "268.4335"
   3) (integer) 4068411363356987
   4) 1) "114.45938318967819214"
      2) "38.02411632967314148"
4) 1) "beijing"
   2) "0.0002"
   3) (integer) 4069885544163409
   4) 1) "116.39723092317581177"
      2) "39.90960415014365736"
~~~
