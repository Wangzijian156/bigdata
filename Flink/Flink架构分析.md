# Flink架构

- [Flink架构](#flink架构)
	- [一、概述](#一概述)
		- [1.1、什么是Flink](#11什么是flink)
		- [1.2、为什么选择Flink](#12为什么选择flink)
		- [1.3、流数据有什么用](#13流数据有什么用)
		- [1.4、数据处理架构](#14数据处理架构)
		- [1.5、流处理框架的演变](#15流处理框架的演变)
		- [1.6、Flink的主要特点](#16flink的主要特点)
		- [1.7、Flink和Spark](#17flink和spark)
	- [二、运行时架构](#二运行时架构)
		- [2.1、Flink运行时的组件](#21flink运行时的组件)
		- [2.2、任务提交流程](#22任务提交流程)
			- [2.2.1、Standalone模式](#221standalone模式)
			- [2.3、Yarn模式](#23yarn模式)
		- [2.3、任务调度流程](#23任务调度流程)
		- [2.4、TaskManger与Slots](#24taskmanger与slots)
		- [2.5、序与数据流（DataFlow）](#25序与数据流dataflow)
		- [2.6、执行图](#26执行图)
		- [2.7、并行度（Parallelism）](#27并行度parallelism)
		- [2.8、任务链（Operator Chains）](#28任务链operator-chains)

## 一、概述

### 1.1、什么是Flink

Apache Flink is a *framework* and *distributed* processing engine for *stateful* computations over *unbounded and bounded data* *streams*.（摘至官网）

```txt
Apache Flink 是一个框架和分布式处理引擎，用于对无界和有界数据流进行状态计算。
```



### 1.2、为什么选择Flink

- 流数据更真实地反映了我们的生活方式（数据不断的产生，有时多有时少）

- 传统的数据结构是i基于有限数据集的
- 特点

```txt
- 低延时
- 高吞吐
- 结果的准确性和良好的容错性
```

（hadoop、spark会先将数据收集在一起再处理）



### 1.3、流数据有什么用

1）电商和市场营销

```txt
数据报表、广告投放、业务流程需要
```

2）物联网（IOT）

```txt
传感器实时数据采集和显示、实时报警、交通运输业
```

3）电信业

```txt
基站流量调配
```

4）银行和金融业

```txt
实时结算和通知推送，实时检测异常行为
```



### 1.4、数据处理架构

1）传统的事务处理架构(传统web应用架构）



![image-20200806121824157](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200806121824157.png)

分为两层：计算层（compute） 和 存储层（Storage）

```txt
- 接受用户Request请求
- 触发事件
- 数据计算（数据是否需要变动）
- 将变动的数据保存到传统关系型数据库
- 最后将计算完的信息包装成Response反馈给用户
```

随着数量的增加，我们需要对数据进行清洗、整理、统计、生成报表，这就衍生出了分析处理架构（传统离线数仓）。



2）分析处理架构（传统离线数仓）

![image-20200806122846648](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200806122846648.png)

```
- 将数据从业务数据库复制到数仓，
- 通过ELT模块进行数据抽取、数据的清洗转换、数据的加载，最后把数据存放在数仓中
- 对数仓中的数据再进行分析和查询（报表、即席查询）
```

分析处理架构的弊端就是时效性太低了，之后衍生出了基于Spark Streaming的实时数仓，但由于Spark Streaming时微批次了，仍然需要攒一批数据，也就限定了实时数仓只能达到秒级，那能不能达到毫秒级的实时数仓，这就得用流式处理架构来解决。



3）有状态的流式处理

![image-20200804102511550](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200804102511550.png)

```txt
- 保证每个数进入流的数据都有一个相应的结果
- 需要查询的数据都存在内存中（之前时存在数据库中）
- 周期性地对数据进行备份（CheckPoint）
```





### 1.5、流处理框架的演变

1）Lambda架构

用两套系统，同时保证低延迟和结果正确

![image-20200804103119177](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200804103119177.png)

Batch Layer：批处理，先把数据收集一起，再处理（确保了数据的正确）

Speed Layer：流处理，速度块，数据不准确，后面的数据可能先发过来的

最终再Application中Batch Layer和Speed Layer结合（维护太麻烦）。



2）各个流式框架对比

- storm（很早以前的框架）

```txt
只是实现了低延时
```

- Spark-Streaming

```txt
- 高吞吐
- 在压力下保持正确
```

- Flink（集大成者）

```txt
- 高吞吐
- 在压力下保持正确
- 操作简单、表现力好
- 低延时
- 时间正确，语义化窗口
```



### 1.6、Flink的主要特点

1）事件驱动

事件驱动型应用是一类具有状态的应用，它从一个或多个事件流提取数据，并根据到来的事件触发计算、状态更新或其他外部动作。

![image-20200806173757042](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200806173757042.png)

2）基于流

```
- 离线数据有界流
- 实时数据无界流
```



3）分层API

![image-20200807092602784](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200807092602784.png)





### 1.7、Flink和Spark

![image-20200807092841531](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200807092841531.png)

![image-20200807092856914](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200807092856914.png)

1）数据模型

```txt
- Spark采用RDD模型，Dstream实际上是一组小批数据RDD的集合
- Flink基本数据模型是数据流，以及事件（Event）序列
```

2）运行时架构

```txt
– spark 是批计算，将 DAG 划分为不同的 stage，一个完成后才可以计算下一个
– flink 是标准的流执行模式，一个事件在一个节点处理完后可以直接发往下一个节点进行处理
```



## 二、运行时架构

### 2.1、Flink运行时的组件

1）作业管理器（JobManager）

```txt
控制一个应用程序执行的主进程，也就是说，每个应用程序都会被一个不同的JobManager所控制执行。
```

2）资源管理器（ResourceManager）

```txt
主要负责管理任务管理器（TaskManager）的插槽（slot），TaskManger插槽是Flink中定义的处理资源单元。
```

3）任务管理器（TaskManager）

```txt
- Flink中的工作进程。
- 通常在Flink中会有多个TaskManager运行，每一个TaskManager都包含了一定数量的插槽（slots）。
- 插槽的数量限制了TaskManager能够执行的任务数量。 
```

4）分发器（Dispatcher）

```txt
- 可以跨作业运行，它为应用提交提供了REST接口。
- 当一个应用被提交执行时，分发器就会启动并将应用移交给一个JobManager。
- 由于是REST接口，所以Dispatcher可以作为集群的一个HTTP接入点，这样就能够不受防火墙阻挡。 
```



### 2.2、任务提交流程

#### 2.2.1、Standalone模式

![image-20200806213627898](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200806213627898.png)

1）提交job

2）分发器Dispatcher将job提交给JM（JobManager）

3）JM向RM请求Slots

4）RM会根据请求中所需Slots的个数，去启动相应个数的TM（TaskManager）

5）TM向RM注册Slots

6）RM发出提供Slots的指令

7）TM给JM提供相应的Slots

8）JM给TM分发任务，任务会在Slot中执行

9）TM之间交换数据（前一个map执行完了，数据传到下一个任务）



#### 2.3、Yarn模式

![image-20200807093049339](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200807093049339.png)



Flink任务提交后

1）Client向HDFS上传Flink的Jar包和配置；

2）向Yarn ResourceManager提交任务；

3）ResourceManager分配Container资源并通知对应的NodeManager启动ApplicationMaster，ApplicationMaster启动后加载Flink的Jar包和配置构建环境，然后启动JobManager；

4）ApplicationMaster向ResourceManager申请资源启动TaskManager，ResourceManager分配Container资源后，由ApplicationMaster通知资源所在节点的NodeManager启动TaskManager

5）NodeManager加载Flink的Jar包和配置构建环境并启动TaskManager；

6）TaskManager启动后向JobManager发送心跳包，并等待JobManager向其分配任务。





### 2.3、任务调度流程

![image-20200807093907250](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200807093907250.png)

1）Flink Program执行Client准备JobGraph（dataflow）并发送给JobManager

```txt
Client
	- Client为提交Job的客户端。
	- 提交Job后，Client可以结束进程（Streaming的任务），也可以不结束并等待结果返回。
```

2）JobManager 再调度任务到各个 TaskManager 去执行

```txt
JobManager 
	- 负责调度 Job 并协调 Task 做 checkpoint。
	- 从Client处接收到Job和JAR包等资源后，会生成优化后的执行计划，并以Task为单元调度到各个TaskManager去执行。
```

3）TaskManager 将心跳和统计信息汇报给 JobManager（TaskManager 之间以流的形式进行数据的传输）

```txt
TaskManager
	- 在启动的时候就设置好了槽位数（Slot），每个slot能启动一个Task。
	- 从JobManager处接收需要部署的Task，部署启动后，与自己的上游建立Netty连接，接收数据并处理。
```

```
Netty: 是由JBOSS提供的一个java开源框架，现为 Github上的独立项目。Netty提供异步的、事件驱动的网络应用程序框架和工具，用以快速开发高性能、高可靠性的网络服务器和客户端程序。
```

```txt
注意： 
	- Client，JobManager，TaskManager都是独立的JVM进程。
	- Task为线程。
```



### 2.4、TaskManger与Slots

1）Flink中每一个Worker(TaskManager)都是一个**JVM进程**，它可能会在**独立的线程**上执行一个或多个SubTask

2）每个Task Slot表示TaskManager拥有资源的**一个固定大小的子集**。

```txt
假如一个TaskManager有三个Slot，那么它会将其管理的内存分成三份给各个Slot。
```

3）通过调整task Slot的数量，允许用户定义SubTask之间如何互相隔离。

```txt
- 如果一个TaskManager一个Slot，那将意味着每个Task Group运行在独立的JVM中（该JVM可能是通过一个特定的容器启动的）。

- 一个TaskManager多个slot意味着更多的subtask可以共享同一个JVM。
	- 在同一个JVM进程中的Task将共享TCP连接（基于多路复用）和心跳消息。
	- Subtask共享Slot，即使它们是不同任务的子任务（前提是它们来自同一个Job）。 这样的结果是，一个Slot可以保存作业的整个管道。
```

4）Task Slot是静态的概念，是指TaskManager具有的并发执行能力，可以通过参数taskmanager.numberOfTaskSlots进行配置

```txt
taskmanager.numberOfTaskSlots(最好设置成与CPU核心数一致)
```

5）并行度Parallelism是动态概念，即TaskManager运行程序时实际使用的并发能力，可以通过参数parallelism.default进行配置



举个例子：假设现在有一个Task流程如下，

```txt
source（加载资源） -> flatmap -> reduce -> sink
```

此Task可以分成三个SubTask

![image-20200807215548404](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200807215548404.png)

那TaskManger中是如何执行的？

- 假设一共有3个TaskManager，设置taskmanager.numberOfTaskSlots=3，每一个TaskManager中的分配3个TaskSlot，也就是每个TaskManager可以接收3个task，一共9个TaskSlot。

![image-20200807220244202](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200807220244202.png)

- 如何设置parallelism.default=1，即运行程序默认的并行度为1，9个TaskSlot只用了1个，有8个空闲，因此，设置合适的并行度才能提高效率。

![image-20200807221231481](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200807221231481.png)



- 设置parallelism=2

```txt
设置并行度
- 代码中
	- 全局设置 env.setParallelism(2)
	- 单个算子设置 flatMap(_.split("\\s")).setParallelism(2)
	
- shell提交job时 ./flink run -c com.atguigu.wc.StreamWordCount –p 2 

- 配置文件flink-conf.yaml,修改 parallelism.default=2
```

![image-20200807221320545](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200807221320545.png)



- 设置parallelism=9，那么9个Solt都有完整的Task在执行

![image-20200807222258329](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200807222258329.png)



- 如果需要将数据写入文件，那么要单独调整sink的并行度（多线程写，会造成写冲突）

![image-20200807222501936](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200807222501936.png)



### 2.5、序与数据流（DataFlow）

所有的Flink程序都是由三部分组成的： **Source** 、**Transformation**和**Sink**。

```txt
- Source负责读取数据源；
- Transformation利用各种算子进行处理加工；
- Sink负责输出。
```

在运行时，Flink上运行的程序会被映射成“逻辑数据流”（DataFlow），它包含了这三部分。每一个DataFlow以一个或多个Sources开始以一个或多个Sinks结束。DataFlow类似于任意的有向无环图（DAG）。



### 2.6、执行图

由Flink程序直接映射成的数据流图是StreamGraph，也被称为逻辑流图，因为它们表示的是计算逻辑的高级视图。

1）StreamGraph：

```txt
是根据用户通过 Stream API 编写的代码生成的最初的图。用来表示程序的拓扑结构。
```

2）JobGraph：

```txt
StreamGraph经过优化后生成了 JobGraph，提交给 JobManager 的数据结构。主要的优化为，将多个符合条件的节点 chain 在一起作为一个节
点，这样可以减少数据在节点之间流动所需要的序列化/反序列化/传输消耗。
```

3）ExecutionGraph：

```txt
JobManager 根据 JobGraph 生成ExecutionGraph。ExecutionGraph是JobGraph的并行化版本，是调度层最核心的数据结构。
```

4）物理执行图：

```txt
JobManager 根据 ExecutionGraph 对 Job 进行调度后，在各个TaskManager 上部署 Task 后形成的“图”，并不是一个具体的数据结构。
```

![img](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/clip_image002.jpg)



### 2.7、并行度（Parallelism）

Flink程序的执行具有**并行、分布式**的特性。

```txt
在执行过程中，一个流（stream）包含一个或多个分区（stream partition），而每一个算子（operator）可以包含一个或多个子任务（operator subtask），这些子任务在不同的线程、不同的物理机或不同的容器中彼此互不依赖地执行。
```

**一个特定算子的子任务（subtask）的个数被称之为其并行度（parallelism）**。



**One-to-one**：那意味着map 算子的子任务看到的元素的个数以及顺序跟source 算子的子任务生产的元素的个数、顺序相同，map、fliter、flatMap等算子都是one-to-one的对应关系。

```txt
类似于spark中的窄依赖
```

**Redistributing**：stream(map()跟keyBy/window之间或者keyBy/window跟sink之间)的分区会发生改变。每一个算子的子任务依据所选择的transformation发送数据到不同的目标任务。

```txt
例如，keyBy() 基于hashCode重分区、broadcast和rebalance会随机重新分区，这些算子都会引起redistribute过程，而redistribute过程就类似于Spark中的shuffle过程。
```

```txt
类似于spark中的宽依赖
```



### 2.8、任务链（Operator Chains）

**相同并行度的one to one操作**，Flink这样相连的算子链接在一起形成一个task，原来的算子成为里面的一部分。

```txt
- 将算子链接成task是非常有效的优化：它能减少线程之间的切换和基于缓存区的数据交换，在减少时延的同时提升吞吐量。
- 任务有先后，但是没有spark中stage阶段的概念，是任务执行图。
```

![image-20200807231135707](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200807231135707.png)