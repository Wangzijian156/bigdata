# Hadoop优化

- [Hadoop优化](#hadoop优化)
	- [一、数据压缩](#一数据压缩)
		- [1.1、数据压缩概述](#11数据压缩概述)
		- [1.2、压缩策略和原则](#12压缩策略和原则)
		- [1.3、MR支持的压缩编码](#13mr支持的压缩编码)
		- [1.4、压缩方式选择](#14压缩方式选择)
		- [1.5、压缩位置选择](#15压缩位置选择)
	- [二、Hadoop企业优化](#二hadoop企业优化)
		- [2.1、MapReduce跑得慢](#21mapreduce跑得慢)
		- [2.2、MapReduce数据输入优化](#22mapreduce数据输入优化)
		- [2.3、MapReduce-Map阶段优化](#23mapreduce-map阶段优化)
		- [2.3、MapReduceMap-Reduce阶段优化](#23mapreducemap-reduce阶段优化)
		- [2.4、MapReduceMap-IO传输优化](#24mapreducemap-io传输优化)
		- [2.5、Hadoop整体优化](#25hadoop整体优化)
		- [2.6、数据倾斜问题](#26数据倾斜问题)
		- [2.7、小文件优化](#27小文件优化)





## 一、数据压缩

### 1.1、数据压缩概述

压缩技术能够有效地减少底层存储系统（HDFS）读写字节数。同时压缩也提高网络带框和磁盘空间的效率。在运行MR程序时，I/O操作、网络数据传输、Shuffle和Merge要花大量的时间，尤其是数据规模很大和工作负载密集的情况下，因此使用数据压缩显得非常重要



### 1.2、压缩策略和原则

压缩时提高Hadoop运行效率的一种优化策略。要注意的是，虽然采用压缩技术减少了磁盘IO，但会同时增加了CPU运算负担。所以，压缩特性运用得当能提高性能，但运用不当也可能降低性能。

压缩基本原则：

1）运算密集型的job，少用压缩

2）IO密集型的job，多用压缩



### 1.3、MR支持的压缩编码

MR支持的压缩编码，如下表所示

| 压缩格式 | hadoop自带？ | 算法    | 文件扩展名 | 是否可切分 | 换成压缩格式后，原来的程序是否需要修改 |
| -------- | ------------ | ------- | ---------- | ---------- | -------------------------------------- |
| DEFLATE  | 是，直接使用 | DEFLATE | .deflate   | 否         | 和文本处理一样，不需要修改             |
| Gzip     | 是，直接使用 | DEFLATE | .gz        | 否         | 和文本处理一样，不需要修改             |
| bzip2    | 是，直接使用 | bzip2   | .bz2       | 是         | 和文本处理一样，不需要修改             |
| LZO      | 否，需要安装 | LZO     | .lzo       | 是         | 需要建索引，还需要指定输入格式         |
| Snappy   | 是，直接使用 | Snappy  | .snappy    | 否         | 和文本处理一样，不需要修改             |

为了支持多种压缩/解压缩算法，Hadoop引入了编码/解码器，如下表所示

| 压缩格式 | 对应的编码/解码器                          |
| -------- | ------------------------------------------ |
| DEFLATE  | org.apache.hadoop.io.compress.DefaultCodec |
| gzip     | org.apache.hadoop.io.compress.GzipCodec    |
| bzip2    | org.apache.hadoop.io.compress.BZip2Codec   |
| LZO      | com.hadoop.compression.lzo.LzopCodec       |
| Snappy   | org.apache.hadoop.io.compress.SnappyCodec  |

压缩性能的比较，如下表所示

| 压缩算法 | 原始文件大小 | 压缩文件大小 | 压缩速度 | 解压速度 |
| -------- | ------------ | ------------ | -------- | -------- |
| gzip     | 8.3GB        | 1.8GB        | 17.5MB/s | 58MB/s   |
| bzip2    | 8.3GB        | 1.1GB        | 2.4MB/s  | 9.5MB/s  |
| LZO      | 8.3GB        | 2.9GB        | 49.3MB/s | 74.6MB/s |



### 1.4、压缩方式选择

1）Gzip

```txt
- 优点：
	- 压缩率比较高，而且压缩/解压速度也比较快
	- hadoop本身支持，在应用中处理Gzip格式的文件和直接处理文本一样
	- 大部分Linux系统都自带Gzip命令
	
- 缺点：
	- 不支持Split
	
- 应用场景：
	- 当每个文件压缩之后再130M以内（一个块以内），都可以考虑Gzip压缩格式
```

2）Bzip2

```txt
- 优点：
	- 支持split
	- 具有很高的压缩率，比Gzip压缩率都高
	- Hadoop本身自带、
	
- 缺点：
	- 压缩/加压速度慢
	
- 应用场景：
	- 适合对速度要求不高，但需要较高的压缩率的时候
	- 输出之后的数据比较大，处理之后的数据需要压缩存档减少磁盘空间并且以后数据用得比较少的情况
	- 对单个很大的文本文件想要压缩减少存储空间，同时有需要支持split，而且兼容之前的应用程序的情况
```

3）Lzo

```txt
- 优点：
	- 压缩/解压速度也比较快，合理的压缩率
	- 支持split，是Hadoop中最流行的压缩格式
	- 可以在Linux系统下安装lzop命令，使用方便
	
- 缺点：
	- 压缩率比Gzip低些
	- Hadoop本身不支持，需要安装
	- 应用对lzo格式的文件需要做一些特殊处理（为了支持split需要建索引，还需要指定inputFormat为lzo格式）
	
- 应用场景
	- 一个很大的文本文件，压缩之后还大于200M以上的可以考虑，而且单个文件越大，Lzo优点越明显
```

4）Snappy

```txt
- 优点：
	- 高速压缩速率和合理的压缩率
- 缺点
	- 不支持split
	- 压缩率比Gzip低
	- Hadoop不支持
	
- 应用场景
	- 当MapReduce作业的Map输出的数据比较大的时候，作为Map到Reduce的中间数据的压缩格式
	- 作为一个MapReduce作业输出和另外一个MapReduce作业输入
```



### 1.5、压缩位置选择

![image-20200720111047456](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-optimizing/image-20200720111047456.png)

1）输入端采用压缩：

在有大量数据并计划重复处理的情况下，应该考虑对输入进行压缩。然而，你无须显示指定使用的编解码方式。Hadoop自动检查文件扩展名，如果扩展名能够匹配，就会用恰当的编解码方式对文件进行压缩和解压。否则，Hadoop就不会使用任何编解码器。

2）Mapper输出采用压缩：

当Map任务输出的中间数据量很大时，应考虑在此阶段采用压缩技术。这能显著改善内部数据Shuffle过程，而Shuffle过程在Hadoop处理过程中是资源消耗最多的环节。如果发现数据量大造成网络传输缓慢，应该考虑使用压缩技术。可用于压缩Mapper输出的快速编解码器包括LZO或者Snappy。

3）Reducer输出采用压缩

在此阶段启用压缩技术能够减少要存储的数据量，因此降低所需的磁盘空间。当MapReduce作业形成作业链条时，因为第二个作业的输入也已压缩，所以启用压缩同样有效。



## 二、Hadoop企业优化

### 2.1、MapReduce跑得慢

MapReduce 程序效率的瓶颈在于两点：

1）计算机性能：CPU、内存、磁盘健康、网络

2）I/O 操作优化：

```txt
- 数据倾斜
- Map和Reduce数设置不合理
- Map运行时间太长，导致Reduce等待过久
- 小文件过多
- 大量的不可分块的超大文件
- Spill次数过多
- Merge次数过多等
```



### 2.2、MapReduce数据输入优化

MapReduce优化方法主要从六个方面考虑：数据输入、Map阶段、Reduce阶段、IO传输、数据倾斜问题和常用的调优参数。

```txt
- 合并小文件：在执行MR任务前将小文件进行合并，大量的小文件会产生大量的Map任务，增大Map任务装载次数，
  而任务的装载比较耗时，从而导致MR运行较慢

- 采用CombineTextInputFormat来作为输入，解决输入端大量小文件场景
```



### 2.3、MapReduce-Map阶段优化

```
- 增大环形缓冲区大小。由100m扩大到200m

- 增大环形缓冲区溢写的比例。由80%扩大到90%
  
- 减少合并（Merge）次数：通过调整io.sort.factor参数，增大Merge的文件数目，减少Merge的次数，
  从而缩短MR处理时间。（10个文件，一次20个merge）
  
- 在Map之后，不影响业务逻辑前提下，先进行Combine处理，减少 I/O。
```



### 2.3、MapReduceMap-Reduce阶段优化

- 合理设置Map和Reduce数：两个都不能设置太少，也不能设置太多。太少，会导致Task等待，延长处理时间；太多，会导致Map、Reduce任务间竞争资源，造成处理超时等错误
- 设置Map、Reduce共存：调整slowstart.completedmaps参数，使Map运行到一定程度后，Reduce也开始运行，减少Reduce的等待时间。
- 规避使用Reduce：因为Reduce在用于连接数据集的时候将会产生大量的网络消耗。
- 增加每个Reduce去Map中拿数据的并行数
- 集群性能可以的前提下，增大Reduce端存储数据内存的大小

```txt
默认情况下，数据达到一个阈值的时候，Buffer中的数据就会写入磁盘，然后Reduce会从磁盘中获得所有的数据。
也就是说，Buffer和Reduce是没有直接关联的，中间多次写磁盘->读磁盘的过程，既然有这个弊端，那么就可以通过参数来配置，
使得Buffer中的一部分数据可以直接输送到Reduce，从而减少IO开销：mapreduce.reduce.input.buffer.percent，默认为0.0。
当值大于0的时候，会保留指定比例的内存读Buffer中的数据直接拿给Reduce使用。这样一来，设置Buffer需要内存，读取数据需要内存，
Reduce计算也要内存，所以要根据作业的运行情况进行调整。
```



### 2.4、MapReduceMap-IO传输优化

```txt
- 使用SequenceFile二进制文件
- map输入端主要考虑数据量大小和切片，支持切片的有Bzip2、LZO。注意：LZO要想支持切片必须创建索引；
- map输出端主要考虑速度，速度快的snappy、LZO；
- reduce输出端主要看具体需求，例如作为下一个mr输入需要考虑切片，永久保存考虑压缩率比较大的gzip。
```



### 2.5、Hadoop整体优化

1）NodeManager默认内存8G，需要根据服务器实际配置灵活调整，例如128G内存，配置为100G内存左右，yarn.nodemanager.resource.memory-mb。

2）单任务默认内存8G，需要根据该任务的数据量灵活调整，例如128m数据，配置1G内存，yarn.scheduler.maximum-allocation-mb。

3）mapreduce.map.memory.mb ：控制分配给MapTask内存上限，如果超过会kill掉进程（报：Container is running beyond physical memory limits. Current usage:565MB of512MB physical memory used；Killing Container）。默认内存大小为1G，如果数据量是128m，正常不需要调整内存；如果数据量大于128m，可以增加MapTask内存，最大可以增加到4-5g。

4）mapreduce.reduce.memory.mb：控制分配给ReduceTask内存上限。默认内存大小为1G，如果数据量是128m，正常不需要调整内存；如果数据量大于128m，可以增加ReduceTask内存大小为4-5g。

5）mapreduce.map.java.opts：控制MapTask堆内存大小。（如果内存不够，报：java.lang.OutOfMemoryError）

6）mapreduce.reduce.java.opts：控制ReduceTask堆内存大小。（如果内存不够，报：java.lang.OutOfMemoryError）

7）可以增加MapTask的CPU核数，增加ReduceTask的CPU核数

8）增加每个Container的CPU核数和内存大小

9）在hdfs-site.xml文件中配置多目录（多磁盘）

10）NameNode有一个工作线程池，用来处理不同DataNode的并发心跳以及客户端并发的元数据操作。dfs.namenode.handler.count=![img](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-optimizing/clip_image002.gif)，，比如集群规模为8台时，此参数设置为41。可通过简单的python代码计算该值，代码如下。

```shell
[atguigu@hadoop102 ~]$ python
Python 2.7.5 (default, Apr 11 2018, 07:36:10) 
[GCC 4.8.5 20150623 (Red Hat 4.8.5-28)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import math
>>> print int(20*math.log(8))
41
>>> quit()
```



### 2.6、数据倾斜问题

- 数据倾斜的现象

```txt
- 数据频率倾斜：某个区域的数据要远远大于其他区域
- 数据大小倾斜：部分记录的大小远远大于平均值
```

- 减少数据倾斜的方法

```txt
- 抽样和范围分区：
	- 以通过对原始数据进行抽样得到的结果集来预设分区边界值

- 自定义分区
	- 基于输出键的背景知识进行自定义分区。例如，如果Map输出键的单词来源于一本书。且其中某几个专业词汇较多。
	  那么就可以自定义分区将这这些专业词汇发送给固定的一部分Reduce实例。而将其他的都发送给剩余的
	  Reduce实例。
	  
- Combine
	- 使用Combine可以大量地减小数据倾斜。在可能的情况下，Combine的目的就是聚合并精简数据。
	
- 采用Map Join，尽量避免Reduce Join。
```



### 2.7、小文件优化

1）小文件的弊端

HDFS上每个文件都要在NameNode上建立一个索引，这个索引的大小约为150byte，这样当小文件比较多的时候，就会产   生很多的索引文件一方面会大量占用NameNode的内存空间，另一方面就是索引文件过大使得索引速度变慢。



2）怎么解决

- 采用har归档方式，将小文件归档
- 采用CombineTextInputFormat
- 有小文件场景开启JVM重用；如果没有小文件，不要开启JVM重用，因为会一直占用使用到的task卡槽，直到任务完成才释放。

注意：JVM重用可以使得JVM实例在同一个job中重新使用N次，N的值可以在Hadoop的mapred-site.xml文件中进行配置。通常在10-20之间

```xml
<property>
    <name>mapreduce.job.jvm.numtasks</name>
    <value>10</value>
    <description>How many tasks to run per jvm,if set to -1 ,there is  no limit</description>
</property> 
```

