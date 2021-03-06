## HBase-query-use-Fillter-and-Coprosessor

项目说明
---

利用数据生成程序，在本地模拟生成（ip+协议类型+url+时间戳）类型的数据，模拟现实生活中用户对目的链接发起的请求的记录数据，一分钟生成144000条数据，并且模拟流式数据不间断生成20分钟，共计2880000条数据，将本地的数据实时的上传到HDFS，并且监控HDFS文件夹，将数据实时的存进HBase，然后做以下查询：
1. 统计记录全部条数。
2. 查询写入时间开始起的第2:00~3:00这一分钟内协议类型为https的记录条数
3. 查询写入时间开始起的第16:00~18:00这两分钟内，ip范围为[1.1.1.1 ，20.50.253.253]，协议类型为http，url域名为google.com的记录条数
4. 以上查询都需要返回查询消耗时间，并且在保证数据正确的前提下，合理的设计数据表结构和程序参数，优化自己的算法，尽可能的降低写入数据的时间和查询数据的时间。

工程目录
---
~~~

createTable  //创建表程序
dataCreator  //数据生成程序
dataquery    //数据查询程序
DataToHbase  //将数据流式存储到HBase
DataToHdfs   //将数据流式存储到HDFS
TestCoprocessor  //Coprocessor查询方式服务器端jar包（环境为 Hbase0.98）

~~~


HBase表的分区设计
---
由于我们的数据是流式的，有时间序列，因此数据进来的时候相邻时间段的数据会存进同一个Region中，也就是说如果我们的Hadoop环境有多个DataNode的话，此时也只有一个在工作，他们是轮流的，这样我们的写入数据的速度就会慢下来，就不能在规定的时间内完成数据的写入，而且DataNode在某一时间负担过重容易宕机。

我们希望数据能并发的写入，提高数据的写入速度，因此在数据写入时要人为的指定分区，即预分区。原理就是我们提前就划分Region而不是等表内数据到了阀值后自动的划分Region，并且我们指定某一个关键字划分到哪一个Region，在数据写入时就会判断属于哪个Region，我们对关键字可以随机产生，这样就保证了在同一段时间内，各个数据节点是负载均衡的，都能工作，并且数据写入是并发的，大大提高了数据写入时间。

结合我自己的环境情况，我是1个NameNode,2个DataNode，因此我决定采用3个分区。
预分区结构图：

![](http://7xt4et.com2.z0.glb.clouddn.com/HBase-query-use-Fillter-and-Coprosessor1.png)

HBase表的RowKey设计
---

在HBase中数据的排序方式是按照RowKey排序的，因此RowKey的设计直接影响后面查询的效率。我们要尽可能的让我们经常查询的数据排在前面，由问题中的查询条件知道，我们经常查询的是时间戳，其次是ip协议和url。另外RowKey的设计我们要尽可能的保证唯一性和长度一致，因为如果RowKey不唯一会导致不同的数据覆盖的问题，长度不唯一会导致排序混乱，不能按照我们想要的顺序排序。但是url的长度不唯一，因此我们不将它放到RowKey中，每次查询都需要查询时间，因此把时间戳放进RowKey，对于ip和协议来说，如果也放进RowKey会导致后期查询的时候不容易拼接StartKey和StopKey，因而放入列里面。生成的数据时间戳虽然精确到了毫秒，但是还是有很多重复的，因此我在RowKey后面增加了一个累加数来保证RowKey的唯一，然后高位增加了分区号。
因此RowKey的格式如下：

分区号+timestamp+累加数

数据的查询
---

查询使用了Filter和Coprocessor两种方式对每个节点数据查询


