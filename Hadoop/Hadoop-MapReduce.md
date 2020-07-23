# Hadoop-MapReduce

- [Hadoop-MapReduce](#hadoop-mapreduce)
	- [一、基本概念](#一基本概念)
		- [1.1、定义](#11定义)
		- [1.2、特点](#12特点)
		- [1.3、MapReduce核心编程思想](#13mapreduce核心编程思想)
		- [1.4、MapReduce进程](#14mapreduce进程)
		- [1.5、WordCount三阶段](#15wordcount三阶段)
	- [二、Hadoop序列化](#二hadoop序列化)
	- [三、FileInputFormat数据输入](#三fileinputformat数据输入)
		- [3.1、MapReduce数据流](#31mapreduce数据流)
		- [3.2、切片与MapTask并行度决定机制](#32切片与maptask并行度决定机制)
		- [3.3、FileInputFormat切片机制](#33fileinputformat切片机制)
		- [3.4、CombineTextInputFormat切片机制](#34combinetextinputformat切片机制)
		- [3.5、数据读取](#35数据读取)
	- [四、MapTask](#四maptask)
	- [五、Shuffle](#五shuffle)
		- [5.1、shuffle总流程图](#51shuffle总流程图)
		- [5.2、Partition分区](#52partition分区)
		- [5.3、WritableComparable排序](#53writablecomparable排序)
		- [5.4、Combiner](#54combiner)
	- [六、ReduceTask](#六reducetask)
		- [6.1、ReduceTask工作机制](#61reducetask工作机制)
		- [6.2、OutputFormat](#62outputformat)
	- [七、Join机制](#七join机制)
	- [八、数据清理](#八数据清理)







## 一、基本概念

### 1.1、定义

1）MapReduce时分布式运算程序的编程框架

2）核心功能时将  **用户编写的业务逻辑代码**  和  **自带默认组件** 整合成一个完整的分布式运算程序

### 1.2、特点

1）优点

```txt
- 易于编程、利用接口
- 基于HDFS良好的扩展性
- 高容错性
- 适合大数据的离线处理
```

2）缺点

```txt
- 不擅长实时计算
- 不擅长流式计算（spark擅长流式计算）
- 不擅长DAG（有向图）计算
```

### 1.3、MapReduce核心编程思想

整个过程切分为Map（切片，（k, v）对应工作），和Reduce（计算统计阶段）。MapReduce编程模型只能包含一个Map阶段和一个Reduce阶段，如果用户的业务逻辑非常复杂，那就只能多个MapReduce程序，串行运行。

### 1.4、MapReduce进程

MRAppMaster：负责程序过程调度 和 状态协调

MapTask：负责map阶段的整个数据处理流程

ReduceTask：负责Reduce阶段的整个数据处理流程

### 1.5、WordCount三阶段

1）Map阶段

```txt
- 继承父类Mapper规定泛型<k, v>
- 重写map方法，规定入参格式
- 利用write方法写出排序归类后的<k, v>
```

2）Reduce阶段

```txt
- 继承父类Reduce规定泛型<k, v>
- 重写reduce方法，规定入参格式（其中有迭代器作计数）
- 利用write方法写出统计后的<k, v>
```

3）Driver阶段

```txt
- 统合架构，启动MR的工具
- 获取实例
- 设置驱动路径
- 设置Mapper和Reducer的路径
- 设置输出类型
- 提交任务
```



## 二、Hadoop序列化

为什么不适用Java的序列化：Java序列化附带其他信息，内容较多，不便于传输

自定义对象实现序列化接口流程：

1）统计的Bean对象必须实现Writable接口

2）编写Mapper类

3）编写Reducer类

4）编写Driver驱动类





## 三、FileInputFormat数据输入

### 3.1、MapReduce数据流

![image-20200713211359146](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-mr/image-20200713211359146.png)

### 3.2、切片与MapTask并行度决定机制

1）数据块：是HDFS物理上把数据分成一块一块；

2）数据切片：数据切片只是在逻辑上对输入进行分片，并不会在磁盘上将其切分成片进行存储。

3）切片与MapTask并行度决定机制：

```txt
- 一个job的Map阶段并行度由客户端在提交Job的切片数决定
- 每一个Split切片分配一个MapTask并行实例处理
- 默认情况下，切片大小=BlockSize
	（当切片大小小于BlockSize，就会出现一个文件存在两个datanode上，这样会影响读取效率）
- 切片时不考虑数据集整体，而是逐个针对每一个文件单独切片
```



### 3.3、FileInputFormat切片机制

针对每一个文件单独切片，若剩下的部分是否大于Blocksize的1.1倍（设定这个数字防止切片过小）



### 3.4、CombineTextInputFormat切片机制

目的：防止大量小切片产生大量MapTask降低效率

应用：Driver中进行设定 

```java
CombineTextInputFormat.setMaxInputSplitSize(job, 2000000);
```



### 3.5、数据读取

1）FileInputFormat实现类

```txt
- TextInputFormat（逐行读取的String类型读取格式）
- KeyValueTextInputFormat（键值对读取）
- NLineInputFormat（多行读取，指定行数为N）
- 自定义InputFormat（根据需求，应用举例时我们用小文件举例）
```

2）应用：在Driver中进行设定

```txt
- 通过Configuration属性的set方法设定切割符
- 通过job实例的方法setInputFormat设置输入格式
```



## 四、MapTask

![image-20200713204612035](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-mr/image-20200713204612035.png)



1）Read阶段

```txt
MapTask通过用户编写的RecordReader，从输入InputSplit中解析出一个个key/value
```

2）Map阶段

```txt
该阶段主要是将解析出的key/value交给用户编写map()函数处理，并产生一系列新的key/value
```

3）Collect收集阶段

```txt
在用户编写map()函数中，当数据处理完成后，一般会调用OutputCollector.collect()输出结果。
在该函数内部，它会将生成的key/value分区（调用Partitioner），并写入一个环形内存缓冲区中
```

4）Spill阶段

```txt
即“溢写”，当环形缓冲区满后，MapReduce会将数据写到本地磁盘上，生成一个临时文件。需要注意的是，
将数据写入本地磁盘之前，先要对数据进行一次本地排序，并在必要时对数据进行合并、压缩等操作。
```

步骤：

- 利用快速排序算法对缓存区内的数据进行排序，排序方式是，先按照分区编号Partition进行排序，然后按照key进行排序。这样，经过排序后，数据以分区为单位聚集在一起，且同一分区内所有数据按照key有序。
- 按照分区编号由小到大依次将每个分区中的数据写入任务工作目录下的临时文件output/spillN.out（N表示当前溢写次数）中。如果用户设置了Combiner，则写入文件之前，对每个分区中的数据进行一次聚集操作。
- 将分区数据的元信息写到内存索引数据结构SpillRecord中，其中每个分区的元信息包括在临时文件中的偏移量、压缩前数据大小和压缩后数据大小。如果当前内存索引大小超过1MB，则将内存索引写到文件output/spillN.out.index中。	

5）Combine

```txt
当所有数据处理完成后，MapTask对所有临时文件进行一次合并，以确保最终只会生成一个数据文件
```



## 五、Shuffle

MapReduce过程：

maptask - sort - copy - sort -reducetask

maptask -           shuffle        -reducetask

### 5.1、shuffle总流程图

![image-20200713214725558](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-mr/image-20200713214725558.png)



 

### 5.2、Partition分区

1）默认Partitioner分区

```txt
- 默认Partitioner分区为key的hashCode和Integer.max_value进行与运算，并以numReduceTasks取余
```

2）自定义Partitioner

```txt
- 自定义类继承Patitioner, 重写getPartition方法
- Driver
	- setPartitionerClass()-设置自定义的Partitioner类
	- setNumReduceTasks()-设定相应数量的ReduceTasks
```

3）总结

```txt
- 分区号从0开始
- 如果分区数大于ReduceTasks数，会产生空文件
- 如果ReduceTasks数小于分区数，则溢出
```

### 5.3、WritableComparable排序

1）默认依靠字典顺序快排

2）排序类型

```txt
- 部分排序
- 全排序，只设置一个ReduceTask，处理大型文件效率很低，输出结果只有一个文件，文件内部有序
- 辅助排序（GroupingComparator），在Reduce端对key分组
- 二次排序，自定义排序中compareTo判断条件为两个时为二次排序
```

3）自定义WritableComparable

- 全排序

```txt
如果是bean对象，则在该类中实现（implements）WritableComparable接口，并在此接口中重写compareTo方法
```

- 区内排序（Partition内）

```txt
- 实操需求：不同省份输出不同的手机号，在不同的省份内又用流量内部排序
- 实操步骤
	- 自定义Partition，重写getPartition方法
	- Driver类中setPartitionerClass，setNumReduceTasks
```

4）GroupingComparator



### 5.4、Combiner

1）相当于Map阶段的Reduce

2）自定义Combiner实现的步骤

```
- 继承Reducer，重写reduce方法
	- 使用一个count计数，遍历相同key的值进行累加
	- 写出
- Driver类中setCombinerClass
```



## 六、ReduceTask

### 6.1、ReduceTask工作机制

![image-20200713214110573](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-mr/image-20200713214110573.png)

1）Copy阶段：ReduceTask从各个MapTask上远程拷贝一片数据，并针对某一片数据，如果其大小超过一定阈值，则写到磁盘上，否则直接放到内存中。

2）Merge阶段：在远程拷贝数据的同时，ReduceTask启动了两个后台线程对内存和磁盘上的文件进行合并，以防止内存使用过多或磁盘上文件过多。

3）Sort阶段：按照MapReduce语义，用户编写reduce()函数输入数据是按key进行聚集的一组数据。为了将key相同的数据聚在一起，Hadoop采用了基于排序的

策略。由于各个MapTask已经实现对自己的处理结果进行了局部排序，因此，ReduceTask只需对所有数据进行一次归并排序即可。

4）Reduce阶段：reduce()函数将计算结果写到HDFS上。



### 6.2、OutputFormat

目的：控制最终文件的输出路径和输出格式

自定义OutputFormat



## 七、Join机制

1）Reduce Join

实现思路（主要依靠reduce方法）

- Map端负责以连接字段做key，将需要的内容作为value，进行输出
- Reduce端负责计算合并

缺点

- Map资源利用率不高，只进行分类工作
- Reduce阶段压力过大，且容易产生数据倾斜

解决方式：Map Join


2）Map Join

使用场景

- Map Join适用于一张表十分小、一张表很大的场景。

思路：采用DistributedCache

- 在Mapper的setup阶段，将文件读取到缓存集合中
- 在驱动函数中加载缓存
  

## 八、数据清理

 计数器应用与数据清洗（ETL，数据清洗过程中，利用计数器记录各种数据