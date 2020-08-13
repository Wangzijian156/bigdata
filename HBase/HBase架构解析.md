# HBase结构解析

- [HBase结构解析](#hbase结构解析)
	- [一、概述](#一概述)
		- [1.1、什么是HBase](#11什么是hbase)
		- [1.2、HBase逻辑结构](#12hbase逻辑结构)
		- [1.3、HBase物理结构](#13hbase物理结构)
		- [1.4、数据模型](#14数据模型)
	- [二、HBase架构](#二hbase架构)
		- [2.1、HBase基本架构](#21hbase基本架构)
		- [2.2、Region Server架构](#22region-server架构)
		- [2.3、写操作流程](#23写操作流程)
		- [2.4、MemStore Flush](#24memstore-flush)
		- [2.5、读操作流程](#25读操作流程)
			- [2.5.1、整体流程](#251整体流程)
			- [2.5.2、Merge细节](#252merge细节)
			- [2.5.3、布隆过滤器](#253布隆过滤器)
		- [2.6、StoreFile Compaction](#26storefile-compaction)
		- [2.7、Region Split](#27region-split)
	- [三、HBase优化](#三hbase优化)
		- [3.1、预分区](#31预分区)
			- [3.1.1、shell手动设置预分区](#311shell手动设置预分区)
			- [3.1.2、生成16进制序列预分区](#312生成16进制序列预分区)
			- [3.1.3、按照文件中设置的规则预分区](#313按照文件中设置的规则预分区)
			- [3.1.4、使用JavaAPI创建预分区](#314使用javaapi创建预分区)
		- [3.2、 RowKey设计](#32-rowkey设计)
			- [3.2.1、基础操作](#321基础操作)
			- [3.2.2、统计网站每分钟的访问次数](#322统计网站每分钟的访问次数)
		- [3.3、内存优化](#33内存优化)
		- [3.4、基础优化](#34基础优化)

## 一、概述

### 1.1、什么是HBase

[Apache](https://www.apache.org/) HBase™ is the [Hadoop](https://hadoop.apache.org/) database, a distributed, scalable, big data store.Use Apache HBase™ when you need random, realtime read/write access to your Big Data. （摘至HBase官网）

```txt
- HBase是基于Hadoop的，是一个分布式、可扩展、支持海量数据存储的数据库（NoSql数据库）。
- 当需要 随机、实时的 读写时可以使用HBase
```



### 1.2、HBase逻辑结构

HBase的逻辑结果如下图所示：

![image-20200805103937362](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805103937362.png)

分析：

1）列族(Column Family)：一个列族可以包含多个列(column)，在底层物理存储中，一个列族的数据会存在一起，不同列族的数据分开存放。

2）列(Column Qualifier)：也叫做列限定符，HBase中建表的时候，不需要声明列，只需要声明列族，可以在加入数据时再去声明列。

3）行键(Row Key)：类似于mysql中的主键，HBase的row_key是有序的（字典序），从图中可以发现 "row_key11" 要排在 "row_key2" 的前面，是因为字典序是 **一个字符一个字符** 进行对比 。

```txt
注意：
- 在Hbase中查询数据时，一般都使用row_key去查（select * from where row_key='roe_key2'),尽量不要使用列去查（select * 
from where name='王五'），这是因为不使用row_key会导致扫描全表，而HBase一般存放了海量的数据，全表扫描性能很差。

- 所以在设计row_key，可以把列中的内容联合到row_key中（类似联合主键）
```

4）分区(Region)：HBase基于HDFS支持分区，也就是当表中的数据不断增大时，会分割成多个文件来存储。

```txt
注意：
- Region会按照row_key范围来进行分割，也就是Region内的数据也是有序的。

- 相同的RowKey不可以跨Region，言外之意就是一行的数据只能存储在一个分区里面。
```

5）存储的单元(store)：HBase的最小存储单元，在Region的基础上根据列族再进行一个分割，因为row_key所以数据再store中也是有序的。

![image-20200805145612163](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805145612163.png)



### 1.3、HBase物理结构

逻辑上，HBase的数据模型同关系型数据库很类似，数据存储在一张表中，有行有列。但从HBase的底层物理存储结构（K-V）来看，HBase更像是一个multi-dimensional （多维）map。物理结构如下图：

![image-20200805145955487](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805145955487.png)

分析：

1）HBase的一行数据，可以映射成一张表，这就是HBase的多维Map。

2）HBase多维Map如何删除和修改数据：

因为HBase中存放了海量的数据，如果像Mysql那样的删除和修改，十分损耗性能。HBase是以追加新数据（磁盘顺序写的速度很快）的方式来实现删除和修改，也就是牺牲读性能来换取写性能的极大提升。

```txt
- HBase中维护了TimeStamp和Type两个字段
	- TimeStamp：用于记录操作时间
	- Type：用于记录操作类型（add、update -> Put, delete -> Delete）
	
- 读取数据时如果Type=Put,则根据TimeStamp取最新的数据。如果Type=Delete,则返回空表示已删除

- Delete类型数据不会永久保留，HBase引入了版本的概念，就是时间戳，可以设定版本的个数，超出版本限定的数据会自动删除
```



### 1.4、数据模型

| 数据模型       | 含义                                                         |
| -------------- | ------------------------------------------------------------ |
| **Name Space** | 类似于关系型数据库的database概念，每个命名空间下有多个表     |
| **Table**      | 类似于关系型数据库的表概念。不同的是，HBase定义表时只需要声明列族即可，不需要声明具体的列。 |
| **Row**        | HBase表中的每行数据都由一个RowKey和多个Column（列）组成，<br/>数据是按照RowKey的字典顺序存储的，并且查询数据时只能根据RowKey进行检索 |
| **Column**     | HBase中的每个列都由Column Family(列族)和Column Qualifier（列限定符）进行限定 |
| **Time Stamp** | 用于标识数据的不同版本（version），每条数据写入时，系统会自动为其加上该字段，其值为写入HBase的时间。 |
| **Cell**       | 由{rowkey, column Family：column Qualifier, time Stamp} 唯一确定的单元。<br/>cell中的数据全部是字节码形式存贮。 |



## 二、HBase架构

### 2.1、HBase基本架构

![image-20200805153431645](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805153431645.png)

1）Region Server

```txt
- Region Server为 Region的管理者，其实现类为HRegionServer，主要作用如下:
	- 对于数据的操作：get, put, delete；
	- 对于Region的操作：splitRegion(分区)、compactRegion（合并）。
```

2）Master

```txt
- Master是所有Region Server的管理者，其实现类为HMaster，主要作用如下：
	- 对于表的操作：create, delete, alter

- 对于RegionServer的操作：分配regions到每个RegionServer，监控每个RegionServer的状态，负载均衡和故障转移。
```

3）Zookeeper

```txt
HBase通过Zookeeper来做master的高可用、RegionServer的监控、元数据的入口以及集群配置的维护等工作。
```

4）HDFS

```txt
HDFS为Hbase提供最终的底层数据存储服务，同时为HBase提供高可用的支持。
```



### 2.2、Region Server架构

![image-20200805155351099](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805155351099.png)

1）Region Server中的4个主要模块

- StoreFile

```txt
保存实际数据的物理文件，StoreFile以Hfile的形式存储在HDFS上。每个Store会有一个或多个StoreFile（HFile），数据在每个StoreFile中都是有序的。
```

- MemStore

```txt
写缓存，由于HFile中的数据要求是有序的，所以数据是先存储在MemStore中，排好序后，等到达刷写时机才会刷写到HFile，每次刷写
都会形成一个新的HFile。
```

- WAL（Write-Ahead logfile）

```txt
由于数据要经MemStore排序后才能刷写到HFile，但把数据保存在内存中会有很高的概率导致数据丢失，为了解决这个问题，数据会先写在一个叫做
Write-Ahead logfile的文件中，然后再写入MemStore中。所以在系统出现故障的时候，数据可以通过这个日志文件重建。
```

- BlockCache

```txt
读缓存，每次查询出的数据会缓存在BlockCache中，方便下次查询。
```

2）每个RegionSerer只有1个BlockCache和1个WAL，可以服务于多个Region。

3）每个Store对应一个列族，包含MemStore和StoreFile，StoreFile以Hfile的形式存储在HDFS上。



### 2.3、写操作流程

![image-20200805193041525](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805193041525.png)

写流程：

1）Client先访问zookeeper，获取hbase:meta表位于哪个Region Server。

2）访问对应的Region Server，获取hbase:meta表，根据读请求的namespace:table/rowkey，查询出目标数据位于哪个Region Server中的哪个Region中。并将该table的region信息以及meta表的位置信息缓存在客户端的meta cache，方便下次访问。

3）与目标Region Server进行通讯；

4）将数据顺序写入（追加）到WAL；

5）将数据写入对应的MemStore，数据会在MemStore进行排序；

6）向客户端发送ack；

7）等达到MemStore的刷写时机后，将数据刷写到HFile。



### 2.4、MemStore Flush

![image-20200805162927903](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805162927903.png)

Region中有多个Store，每个Store中都有一个MemStore，写数据时，数据会先缓存到MemStore中，那什么时候MemStore会将数据写入StoreFile？

HBase提供了4种Flush（刷写）机制：

1）某个MemStore的大小达到阙值

```properties
hbase.hregion.memstore.flush.size=128（默认值）
hbase.hregion.memstore.block.multiplier=4（默认值）
```

当Region中的某个MemStore的大小达到了hbase.hregion.memstore.flush.size，会触发Flush（整个Region中的所有MemStore都会开始刷写）。此时MemStore在往HDFS中写数据的同时，写操作仍在往MemSore中写数据。当MemStore的大小达到

```txt
hbase.hregion.memstore.flush.size * hbase.hregion.memstore.block.multiplier
```

会停止继续往该memstore写数据。

```txt
注意：
当Resgion中一个MemStore触发Flush，其余的MemStore可能远未达到阙值，同时写入，会造成小文件过多的情况，所以Region中的MemStore应该尽可能少，MemStore => Store => 列族，最终认定在设计表时列族应尽可能少（最多2个）
```



2）MemStore的总大小阙值

```properties
java_heapsize=java堆栈的大小
hbase.regionserver.global.memstore.size=0.4（默认值）
hbase.regionserver.global.memstore.size.lower.limit=0.95（默认值）
```

当Region Server中Memstore的总大小达到

```txt
java_heapsize *
hbase.regionserver.global.memstore.size *
hbase.regionserver.global.memstore.size.lower.limit
```

Region会按照其所有Memstore的大小顺序（由大到小）依次进行刷写。直到Region Server中所有Memstore的总大小减小到上述值以下。当Region Server中memstore的总大小达到

```txt
java_heapsize*
hbase.regionserver.global.memstore.size
```

时，会阻止继续往所有的memstore写数据。



3）自动刷写的机制

到达自动刷写的时间，也会触发memstore flush。自动刷新的时间间隔由该属性进行配置

```txt
hbase.regionserver.optionalcacheflushinterval（默认1小时）
```



4）WAL文件的数量达到阙值

当WAL文件的数量超过

```txt
hbase.regionserver.max.logs（该属性名已经废弃，现无需手动设置，最大值为32）
```

region会按照时间顺序依次进行刷写，直到WAL文件数量减小到hbase.regionserver.max.log以下。



### 2.5、读操作流程

#### 2.5.1、整体流程

![image-20200805193211086](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805193211086.png)

读流程：

- Client先访问zookeeper，获取hbase:meta表位于哪个Region Server；

- 访问对应的Region Server，获取hbase:meta表，根据读请求的namespace:table/rowkey，查询出目标数据位于哪个Region Server中的哪个Region中。并将该table的region信息以及meta表的位置信息缓存在客户端的meta cache，方便下次访问；

- 与目标Region Server进行通讯；

- 分别在MemStore和Store File（HFile）中查询目标数据，并将查到的所有数据进行合并。此处所有数据是指同一条数据的不同版本（time stamp）或者不同的类型（Put/Delete；

- 将查询到的新的数据块（Block，HFile数据存储单元，默认大小为64KB）缓存到Block Cache；

- 将合并后的最终结果返回给客户端。



#### 2.5.2、Merge细节

![image-20200805193428016](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805193428016.png)

读取数据顺序

- 从Block Cache中读取
- 如果Block Cache，再去MemStore中读取
- 如果MemStore还没有，最终去HDFS上读取HFile

```txt
HBase中存放着海量的数据，如果从HDFS一个一个读取HFile，太慢了。HBase增加三个条件来减少查询范围，从而提升读取的效率。
- 时间范围
- RowKey范围
- 布隆过滤器
```



#### 2.5.3、布隆过滤器

布隆过滤器是一个 bit 向量或者说 bit 数组，长这样：

![image-20200805195443586](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805195443586.png)

如果我们要映射一个值到布隆过滤器中，我们需要使用多个不同的哈希函数生成多个哈希值，并对每个生成的哈希值指向的 bit 位置 1，例如针对值 “Hello World” 和三个不同的哈希函数分别生成了哈希值 1、5、7，则上图转变为：

![image-20200805195242751](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805195242751.png)

Ok，我们现在再存一个值 “Java”，如果哈希函数返回 3、5、8 的话，图继续变为：

![image-20200805195629875](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805195629875.png)

值得注意的是，4 这个 bit 位由于两个值的哈希函数都返回了这个 bit 位，因此它被覆盖了。所以这就造成了布隆过滤器的特殊性：

只能

```txt
0 0 0 => 不存在
```

而不能

```txt
1 1 1 => 存在 （因为会被覆盖）
```



### 2.6、StoreFile Compaction

由于memstore每次刷写都会生成一个新的HFile，且同一个字段的不同版本（timestamp）和不同类型（Put/Delete）有可能会分布在不同的HFile中，因此查询时需要遍历所有的HFile。为了减少HFile的个数，以及清理掉过期和删除的数据，会进行StoreFile Compaction。

Compaction分为两种，分别是Minor Compaction和Major Compaction，如下图所示：

![image-20200805200210027](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805200210027.png)



1）Minor Compaction

```
将临近的若干个较小的HFile合并成一个较大的HFile，并清理掉部分过期和删除的数据。
```

2）Major Compaction（默认7天合并一次）

```txt
Major Compaction会将一个Store下的所有的HFile合并成一个大HFile，并且会清理掉所有过期和删除的数据。
```



### 2.7、Region Split

默认情况下，每个Table起初只有一个Region，随着数据的不断写入，Region会自动进行拆分。刚拆分时，两个子Region都位于当前的Region Server，但处于负载均衡的考虑，HMaster有可能会将某个Region转移给其他的Region Server。

![image-20200805200408414](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805200408414.png)

1）当1个region中的某个Store下所有StoreFile的总大小超过

```txt
hbase.hregion.max.filesize=128（默认值）
```

该Region就会进行拆分（0.94版本之前）。

2）当1个region中的某个Store下所有StoreFile的总大小超过

```txt
Min(initialSize*R^3 ,hbase.hregion.max.filesize)
```

该Region就会进行拆分。

其中：

```
- initialSize = 2*hbase.hregion.memstore.flush.size（默认）

- R为当前Region Server中属于该Table的Region个数（0.94版本之后）
```

具体的切分策略为：

第一次split：1^3 * 256 = 256MB 

第二次split：2^3 * 256 = 2048MB 

第三次split：3^3 * 256 = 6912MB 

第四次split：4^3 * 256 = 16384MB > 10GB，因此取较小的值10GB 

后面每次split的size都是10GB了。

3）Hbase 2.0引入了新的split策略，如果当前RegionServer上该表只有一个Region，按照

```txt
2 * hbase.hregion.memstore.flush.size = 256 （默认）
```

分裂，否则按照hbase.hregion.max.filesize分裂。



## 三、HBase优化

### 3.1、预分区

思考一：

```txt
HBase新创建一个表时只有一个Region，按照Region Split的机制，只有达到阙值了才会逐个分裂，也就说在开始的一段时间内只有少量的Region，整个HBase的读写并发度很低（Region却多并发度越高）。
```

思考二：

```txt
每一个Region都有一个startRowKey和endRowKey，当一个Region分裂成Region1, region2时，会从[startRowKey,endRowKey]中间生
成一个中间的middleRowKey。Region1[startRowKey,middleRowKey],Region2[middleRowKey,endRowKey]但是这个过程自动的不可
控。如果用时间戳作为Rowkey会造成新来的数据都存到region2，这就是region热点（数据倾斜）问题。
```

所谓预分区，就是在们建立表的时候指明，我们这张表要有几个分区。

#### 3.1.1、shell手动设置预分区

```shell
hbase> create 'tableName','columnFamily1','columnFamily2',SPLITS => ['1000','2000','3000','4000']
```

```
create 'staff1','info1','info2',SPLITS => ['1000','2000','3000','4000']
```

4个分区键['1000','2000','3000','4000']，会生成5个分区，登录http://hadoop102:16010/查看

![image-20200805223446821](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805223446821.png)

```
[-∞， 1000]
[1000, 2000]
[2000, 3000]
[3000, 4000]
[4000, ∞]
```

![image-20200805223446821](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805223446821.png)
![image-20200805223610211](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805223610211.png)



#### 3.1.2、生成16进制序列预分区

```shell
create 'staff2','info','partition2',{NUMREGIONS => 15, SPLITALGO => 'HexStringSplit'}
```

NUMREGIONS => 15创建15个分区，SPLITALGO => 'HexStringSplit'按照16进制字符串分割。

```txt
八位16进制数最大就是八个f，现在是15个分区，15个分区再16进制里除以f，每个区的大小时11111111(如果分成十个分区除以10就好了，八个f除以a)
```

![image-20200805224104761](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805224104761.png)

假设有一个"abc"这样的RowKey，那么它会进入到哪个Region？答案是[aaaaaaaa, bbbbbbbb]。要注意如果rowkey是字符串，会出现region热点（数据倾斜）问题

```txt
[99999999, aaaaaaaa] 以a开头的字符串（部分）
[aaaaaaaa, bbbbbbbb] 以a开头的字符串（部分）、以b开头的字符串（部分）
[bbbbbbbb, cccccccc] 以b开头的字符串（部分）、以c开头的字符串（部分）
[cccccccc, dddddddd] 以c开头的字符串（部分）、以d开头的字符串（部分）
[dddddddd, eeeeeeee] 以d开头的字符串（部分）、以e开头的字符串（部分）
[eeeeeeee, ] 出上面以外，字母开头的的字符串，都会到这个区
```

那么假设RowKey就想用"abc"，怎么处理呢？将abc转成16进制。

```java
Bytes.toHex(Bytes.toBytes("abc"))  
// 输出结果：616263 
```



#### 3.1.3、按照文件中设置的规则预分区

分区键写到文件中，创建splits.txt

```shell
vim /opt/module/hbase/bin/splits.txt
```

文件内容如下（不能有空行）：

```
bbbb 
aaaa  
cccc  
dddd  
```

会自动按照字典序排序。

启动Hbase Shell的时候也要在splits.txt所在路径

然后执行：

```shell
create 'staff3','info',SPLITS_FILE => 'splits.txt'
```

![image-20200805231455830](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hbase/image-20200805231455830.png)

#### 3.1.4、使用JavaAPI创建预分区

```java
//自定义算法，产生一系列Hash散列值存储在二维数组中
byte[][] splitKeys = 某个散列值函数
//创建HbaseAdmin实例
HBaseAdmin hAdmin = new HBaseAdmin(HbaseConfiguration.create());
//创建HTableDescriptor实例
HTableDescriptor tableDesc = new HTableDescriptor(tableName);
//通过HTableDescriptor实例和散列值二维数组创建带有预分区的Hbase表
hAdmin.createTable(tableDesc, splitKeys);
```



### 3.2、 RowKey设计

#### 3.2.1、基础操作

1）生成随机数、hash、散列值

```txt
原本rowKey为1001的，SHA1后变成：dd01903921ea24941c26a48f2cec24e0bb0e8cc7
原本rowKey为3001的，SHA1后变成：49042c54de64a1e9bf0b33e00245660ef92dc7bd
原本rowKey为5001的，SHA1后变成：7b61dec07e02c188790670af43e717f0f46e8913
```

2）字符串反转

```txt
20170524000001转成10000042507102
20170524000002转成20000042507102
```

这样也可以在一定程度上散列逐步put进来的数据。

3）字符串拼接

```txt
20170524000001_a12e
20170524000001_93i7
```



#### 3.2.2、统计网站每分钟的访问次数

如何统计？

```
去表里找到每分钟的访问记录，然后进行一个统计，比如14:42有多少访问量，14:43又有多少个访问量，统计一个值，然后形成折线图或者其他的图都可以。
```

要求：不能有Region热点问题，一起查询的数据要放在一起（使用scan查询）

RowKey如何设计？

```txt
同一分钟的数据放在一起 => 只要RowKey前缀相同就会挨着存放
```

1、以时间戳为RowKey

```
yyyyMMddHHmmssSSS
```

问题：

```txt
- 会造成Region热点问题，随着时间的流逝，分区后新的数据只会添加到最新的分区中（递增的RowKey要避免）。
- 高并发会出现同时加入多条数据的现象(唯一性问题)
```

2、时间戳 (倒叙) + UserID 作为RowKey

```txt
mmHHddMMyyyy_userId
```

```txt
- 加入userId解决了唯一性
- 时间戳 (倒叙)解决了递增问题
```

3、公司中常用的方式：加盐、Hash

1）yyyyMMddHHmmssSSS_userId 有递增问题，加个前缀。

```txt
RowKey进入哪个分区是由前面的字符决定的。
```

2）rand()_yyyyMMddHHmmssSSS_userId，前缀加随机数。

```txt
虽然解决了Region热点问题，但是同一分钟内的数据无法存在一起。
```

3）前缀需要加一个有规律的随机数

```txt
"yyyyMMddHHmm”.hasCode()_yyyyMMddHHmmssSSS_userId
```

同一分钟的hashCode()的值是一样的，这就保证了同一分钟内的数据存在一起。那如何均匀分区？可以对hashcode取余

```
"yyyyMMddHHmm”.hasCode()%num_yyyyMMddHHmmssSSS_userId
```

num为分区数，得到的结果并不多(余数要么0，要么1，要么2 ...)，就是一个数字而已。对RowKey的长度也没有很大的影响。

4）假设num=5，那分区键如何设计 ：

```
[-∞， 1]
[1, 2]
[2, 3]
[3, 4]
[4, ∞]
```

5）如何查数据

```txt
scan("202006261555".hashCode()%5_202006261555,
"202006261555".hashCode()%5_202006261555|)
```

注意：

startRowKey="202006261555".hashCode()%5_202006261555，endRowKey如何设计？只需要在startRowKey后面加一个较大的字符就行了，一般'|'就够了。



### 3.3、内存优化

HBase操作过程中需要大量的内存开销，毕竟Table是可以缓存在内存中的，但是不建议分配非常大的堆内存，因为GC过程持续太久会导致RegionServer处于长期不可用状态，一般16~36G内存就可以了，如果因为框架占用内存过高导致系统内存不足，框架一样会被系统服务拖死。



### 3.4、基础优化

1）Zookeeper会话超时时间

```txt
hbase-site.xml
属性：zookeeper.session.timeout
解释：默认值为90000毫秒（90s）。当某个RegionServer挂掉，90s之后Master才能察觉到。可适当减小此值，以加快Master响应，可调整至600000毫秒。
```

2）设置RPC监听数量

```txt
hbase-site.xml
属性：hbase.regionserver.handler.count  解释：默认值为30，用于指定RPC监听的数量，可以根据客户端的请求数进行调整，读写请求较多时，增加此值。
```

3）手动控制Major Compaction

```txt
hbase-site.xml
属性：hbase.hregion.majorcompaction
解释：默认值：604800000秒（7天）， Major Compaction的周期，若关闭自动Major Compaction，可将其设为0
```

4）优化HStore文件大小

```txt
hbase-site.xml
属性：hbase.hregion.max.filesize  解释：默认值10737418240（10GB），如果需要运行HBase的MR任务，可以减小此值，因为一个region对应一个map任务，如果单个region过大，会导致map任务执行时间过长。该值的意思就是，如果HFile的大小达到这个数值，则这个region会被切分为两个Hfile。  
```

5）优化HBase客户端缓存

```txt
hbase-site.xml
属性：hbase.client.write.buffer  解释：默认值2097152bytes（2M）用于指定HBase客户端缓存，增大该值可以减少RPC调用次数，但是会消耗更多内存，反之则反之。一般我们需要设定一定的缓存大小，以达到减少RPC次数的目的。  
```

6）指定scan.next扫描HBase所获取的行数

```txt
hbase-site.xml
属性：hbase.client.scanner.caching  解释：用于指定scan.next方法获取的默认行数，值越大，消耗内存越大。
```

7.BlockCache占用RegionServer堆内存的比例

```txt
hbase-site.xml
属性：hfile.block.cache.size
解释：默认0.4，读请求比较多的情况下，可适当调大
```

8.MemStore占用RegionServer堆内存的比例

```txt
hbase-site.xml
属性：hbase.regionserver.global.memstore.size
解释：默认0.4，写请求较多的情况下，可适当调大
```

