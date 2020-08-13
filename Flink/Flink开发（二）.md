# Flink开发（中）

- [Flink开发（中）](#flink开发中)
  - [五、状态编程和容错机制](#五状态编程和容错机制)
    - [5.1、什么是状态](#51什么是状态)
    - [5.2、Flink中的状态](#52flink中的状态)
      - [5.2.1、算子状态](#521算子状态)
      - [5.2.2、键控状态](#522键控状态)
    - [5.3、代码中定义状态](#53代码中定义状态)
    - [5.4、状态后端](#54状态后端)
    - [5.5、状态一致性](#55状态一致性)
      - [5.5.1、一致性检查点](#551一致性检查点)
      - [5.5.2、检查点算法](#552检查点算法)
      - [5.5.3、容错机制代码设置](#553容错机制代码设置)
      - [5.5.4、保存点](#554保存点)
    - [5.6、端到端 exactly-once](#56端到端-exactly-once)
      - [5.6.1、幂等写入](#561幂等写入)
      - [5.6.2、事务写入](#562事务写入)
      - [5.6.3、预写日志（WAL）](#563预写日志wal)
      - [5.6.4、两阶段提交（2PC）](#564两阶段提交2pc)
      - [5.6.5、一致性对比](#565一致性对比)
      - [5.6.7、Kafka 状态一致性的保证](#567kafka-状态一致性的保证)
  - [六、ProcessFunction API](#六processfunction-api)
    - [6.1、概述](#61概述)
    - [6.2、KeyedProcessFunction](#62keyedprocessfunction)
    - [6.3、侧输出流](#63侧输出流)


## 五、状态编程和容错机制

### 5.1、什么是状态

流式计算分为无状态和有状态两种情况。

1）无状态流处理每次只转换一条输入记录，并且仅根据最新的输入记录输出结果

```txt
例如，流处理应用程序从传感器接收温度读数，并在温度超过90度时发出警告。
```

2）有状态流处理维护所有已处理记录的状态值，并根据每条新输入的记录更新状态，因此输出记录(灰条)反映的是综合考虑多个事件之后的结果

```
- 计算过去一小时的平均温度。
- 若在一分钟内收到两个相差20度以上的温度读数，则发出警告。
```

流与流之间的所有关联操作，以及流与静态表或动态表之间的关联操作，都是有状态的计算。



### 5.2、Flink中的状态

![image-20200810103109113](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200810103109113.png)

Flink中由一个任务维护，并且用来计算某个结果的所有数据，都属于这个任务的状态。

```txt
备注：可以认为状态就是一个本地变量，可以被任务的业务逻辑访问，一般会和特定的算子绑定。
```

Flink有两种类型的状态：算子状态、键控状态。



#### 5.2.1、算子状态

算子状态（Operator State，例如：map、flatMap）结构图如下：



![image-20200810103924205](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200810103924205.png)

- 特点

```
- 算子状态的作用范围限定为算子任务，由同一并行任务所处理的所有数据都可以访问到相同的状态
- 状态对于同一子任务而言是共享的
- 算子状态不能由相同或不同算子的另一个子任务访问
```

- 算子状态数据结构

```txt
- 列表状态（List state）
	- 将状态表示为一组数据的列表
	
- 联合列表状态（Union list state）
	- 也将状态表示为数据的列表。它与常规列表状态的区别在于，在发生故障时，或者从保存点（savepoint）启动应用
	  程序时如何恢复
	  
- 广播状态（Broadcast state）
	- 如果一个算子有多项任务，而它的每项任务状态又都相同，那么这种特殊情况最适合应用广播状态
```



#### 5.2.2、键控状态

键控状态（Keyed State，例如keyBy），结构图如下：

![image-20200810104341946](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200810104341946.png)

- 特点

```txt
- 根据输入数据流中定义的键（key）来维护和访问的
- Flink 为每个 key 维护一个状态实例，并将具有相同键的所有数据，都分区到同一个算子任务中，这个任务会维护和处
  理这个 key 对应的状态
- 当任务处理一条数据时，它会自动将状态的访问范围限定为当前数据的 key
```

- 键控状态数据结构

```txt
- 值状态（Value state）
	- 将状态表示为单个的值
	
- 列表状态（List state）
	- 将状态表示为一组数据的列表
	
- 映射状态（Map state）
	- 将状态表示为一组 Key-Value 对
	
- 聚合状态（Reducing state & Aggregating State）
	- 将状态表示为一个用于聚合操作的列表
```

```txt
注意：大多数情况下使用键控状态（Keyed State）
```



### 5.3、代码中定义状态

1）声明一个状态有两种方式：必须定义在RichFunction中，因为需要运行时上下文

- 第一种：全局变量

```scala
var valueState: ValueState[Double] = _
//在这是获取不到上下文的，富函数只有在调用open方法时才会生成下上文
//val valueState: ValueState[Double] = getRuntimeContext.getState( new ValueStateDescriptor[Double]("valuestate", classOf[Double]))
 override def open(parameters: Configuration): Unit = {
    valueState = getRuntimeContext.getState(new ValueStateDescriptor[Double]("valuestate", classOf[Double]))
  }
```

- 第二种：lazy延迟加载

```scala
lazy val listState: ListState[Int] = getRuntimeContext
    .getListState( new ListStateDescriptor[Int]("liststate", classOf[Int]) )
```

2）各种数据类型的使用（list、map、reduce）

```scala
class MyRichMapper1 extends RichMapFunction[SensorReading, String] {

  var valueState: ValueState[Double] = _
  lazy val listState: ListState[Int] = getRuntimeContext
    .getListState( new ListStateDescriptor[Int]("liststate", classOf[Int]) )
  lazy val mapState: MapState[String, Double] = getRuntimeContext
    .getMapState(new MapStateDescriptor[String, Double]("mapstate", classOf[String], classOf[Double]))
  lazy val reduceState: ReducingState[SensorReading] = getRuntimeContext
    .getReducingState(new ReducingStateDescriptor[SensorReading]("reducestate", new MyReducer, classOf[SensorReading]))

  override def open(parameters: Configuration): Unit = {
    valueState = getRuntimeContext.getState(new ValueStateDescriptor[Double]("valuestate", classOf[Double]))
  }

  override def map(value: SensorReading): String = {
    val myV = valueState.value()
    valueState.update(value.temperature)
    listState.add(1)
    val list = new util.ArrayList[Int]()
    list.add(2)
    list.add(3)
    listState.addAll(list)
    listState.update(list)
    listState.get()

    mapState.contains("sensor_1")
    mapState.get("sensor_1")
    mapState.put("sensor_1", 1.3)

    reduceState.get()
    reduceState.add(value)

    value.id
  }
}
```

2）需求：对于温度传感器温度值跳变，超过10度，报警

- 第一种实现方式：自定义富函数

```scala
object StateTest {
  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)
    // 读取数据
    val inputStream = env.socketTextStream("hadoop102", 7778)

    // 先转换成样例类类型（简单转换操作）
    val dataStream = inputStream
      .map( data => {
        val arr = data.split(",")
        SensorReading(arr(0), arr(1).toLong, arr(2).toDouble)
      } )

    // 需求：对于温度传感器温度值跳变，超过10度，报警
    val alertStream = dataStream
      .keyBy(_.id)
      .flatMap( new TempChangeAlert(10.0) )
    
    alertStream.print()

    env.execute("state test")
  }
}

// 实现自定义RichFlatmapFunction
class TempChangeAlert(threshold: Double) extends RichFlatMapFunction[SensorReading, (String, Double, Double)] {
  // 定义状态保存上一次的温度值
  lazy val lastTempState: ValueState[Double] = getRuntimeContext.getState(new ValueStateDescriptor[Double]("last-temp", classOf[Double]))
  // 第一个数据,lastTempState.value()为0，如果第一次的温度超过10度， (value.temperature - lastTemp).abs肯定大于10，会报警
  // 定义一个标志位，忽略第一次的温度
  lazy val flagState: ValueState[Boolean] = getRuntimeContext.getState(new ValueStateDescriptor[Boolean]("flag", classOf[Boolean]))

  override def flatMap(value: SensorReading, out: Collector[(String, Double, Double)]): Unit = {
    // 获取上次的温度值
    val lastTemp = lastTempState.value()
    // 跟最新的温度值求差值作比较
    val diff = (value.temperature - lastTemp).abs
    if (!flagState.value() && diff > threshold)
      out.collect((value.id, lastTemp, value.temperature))

    // 更新状态
    lastTempState.update(value.temperature)
    flagState.update(true)
  }
}
```

- 第一种实现方式：flatMapWithState

```scala
def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)
    // 读取数据
    val inputStream = env.socketTextStream("hadoop102", 7778)

    // 先转换成样例类类型（简单转换操作）
    val dataStream = inputStream
    .map( data => {
        val arr = data.split(",")
        SensorReading(arr(0), arr(1).toLong, arr(2).toDouble)
    } )

    // 需求：对于温度传感器温度值跳变，超过10度，报警
    val alertStream = dataStream
    .keyBy(_.id)
    //      .flatMap( new TempChangeAlert(10.0) )
    .flatMapWithState[(String, Double, Double), Double] {
        case (data: SensorReading, None) => ( List.empty, Some(data.temperature) )
        case (data: SensorReading, lastTemp: Some[Double]) => {
            // 跟最新的温度值求差值作比较
            val diff = (data.temperature - lastTemp.get).abs
            if( diff > 10.0 )
            ( List((data.id, lastTemp.get, data.temperature)), Some(data.temperature) )
            else
            ( List.empty, Some(data.temperature) )
        }
    }

    alertStream.print()

    env.execute("state test")
}
```





### 5.4、状态后端

每传入一条数据，有状态的算子任务都会读取和更新状态。由于有效的状态访问对于处理数据的低延迟至关重要，因此每个并行任务都会在本地维护其状态，以确保快速的状态访问。

1）定义：状态的存储、访问以及维护，由一个可插入的组件决定，这个组件就叫做**状态后端**（state backend）。

状态后端主要负责两件事：

```txt
- 本地的状态管理；
- 以及将检查点（checkpoint）状态写入远程存储
```

2）状态后端类型

- MemoryStateBackend（开发环境）内存级的状态后端

```txt
本地状态存储在TaskManager的JVM堆上；而将checkpoint存储在JobManager的内存中。
	- 特点：快速、低延迟，但不稳定
```

- FsStateBackend（稳定）

```txt
将checkpoint存到远程的持久化文件系统（FileSystem）上，本地状态存储在TaskManager的JVM堆上。
	- 同时拥有内存级的本地访问速度，和更好的容错保证
```

- RocksDBStateBackend

```txt
将所有状态序列化后，存入本地的RocksDB中存储。
```

3）代码

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStateBackend(new FsStateBackend(""))
// setStateBackend已弃用，以后用StreamExecutionEnvironment.setStateBackend(StateBackend)
```





### 5.5、状态一致性

当在分布式系统中引入状态时，自然也引入了一致性问题。

```txt
- 有状态的流处理，内部每个算子任务都可以有自己的状态
- 对于流处理器内部，状态一致性即计算结果要准确
	- exactly once(精确一次消费，数仓实战中接触过)
	- 容灾恢复
```

在流处理中，一致性可以分为3个级别：

```txt
- at-most-once: 最多计算一次，如果发生故障不会去处理，不会去恢复丢失。
- at-least-once: 这表示计数结果可能大于正确值，但绝不会小于正确值。也就是说，计数程序在发生故障后可能多算，但是绝不会少算。
- exactly-once: 这指的是系统保证在发生故障后得到的计数结果与正确值一致。
```

Flink的一个重大价值在于，它既保证了exactly-once，也具有低延迟和高吞吐的处理能力。



#### 5.5.1、一致性检查点

Flink 故障恢复机制的核心，就是应用状态的一致性检查点。有状态流应用的一致检查点，其实就是所有任务的状态，在某个时间点的一份拷贝（一份快照）；

```txt
备注：这个时间点，应该是所有任务都恰好处理完一个相同的输入数据的时候
```

![image-20200810171831745](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200810171831745.png)

如图中所示：InputStream 传入数字 ，Source收集后，按照奇数偶数分别求和，如何存储状态？

1）此时sum_even=2+4，sum_odd=1+3+5

2）直接存储5、6、9，表示5之前的数据已经统计完，偶数是6，基数是9

```txt
不存储sum_even=2+4，sum_odd=1+3+5详细信息的原因是
	- 实现逻辑太复杂、难度太大
```



#### 5.5.2、检查点算法

一种简单的想法

```txt
暂停应用，保存状态到检查点，再重新恢复应用
```

Flink 的改进实现

```txt
- 基于 Chandy-Lamport 算法的分布式快照
- 将检查点的保存和数据处理分离开，不暂停整个应用
```

Flink检查点算法流程：

1）JobManager 会向每个 source 任务发送一条带有新检查点 ID 的消息，通过这种方式来启动检查点

![image-20200810214511297](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200810214511297.png)

2）数据源（Source）将它们的状态写入检查点，并发出一个检查点 barrier

3）状态后端在状态存入检查点之后，会返回通知给 Source任务，Source 任务就会向 JobManager 确认检查点完成

![image-20200810214549310](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200810214549310.png)

4）分界线对齐：barrier 向下游传递，sum 任务会等待所有输入分区的 barrier 到达

5）对于barrier已经到达的分区，继续到达的数据会被缓存

6）而barrier尚未到达的分区，数据会被正常处理

![image-20200810214624749](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200810214624749.png)

7）当收到所有输入分区的 barrier 时，任务就将其状态保存到状态后端的检查点中，然后将 barrier 继续向下游转发

![image-20200810214747203](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200810214747203.png)

8）向下游转发检查点 barrier 后，任务继续正常的数据处理

![image-20200810214801517](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200810214801517.png)

9）Sink 任务向 JobManager 确认状态保存到 checkpoint 完毕

10）当所有任务都确认已成功将状态保存到检查点时，检查点就真正完成了

![image-20200810214818933](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200810214818933.png)

#### 5.5.3、容错机制代码设置

默认情况下checkpoint不开启

```scala
// 开启checkpoint，已经弃用
// 使用enableCheckpointing(interval : Long, mode: CheckpointingMode)
// CheckpointingMode： "exactly once" and "at least once"
env.enableCheckpointing()

// checkpoint的配置
val chkpConfig = env.getCheckpointConfig
// 设置连续两个checkpoint之间的时间间隔
chkpConfig.setCheckpointInterval(10000L) 
// 设置CheckpointingMode，CheckpointingMode.EXACTLY_ONCE精确一次消费
chkpConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE) 
// 超时时间，超过时间checkpoint就不继续做了，等下次存盘在做
chkpConfig.setCheckpointTimeout(60000)
// 最大同时并行的Checkpoint
chkpConfig.setMaxConcurrentCheckpoints(2)
// 上一个Checkpoint执行完毕 到 下一个checkponit触发执行 之间的时间间隔
chkpConfig.setMinPauseBetweenCheckpoints(500L)
// 状态恢复方式 true: checkpoint（自动存盘机制）、false: savepoint（手动存盘机制）
chkpConfig.setPreferCheckpointForRecovery(true)
// 允许多少次checkpoint失败，达到次数上限报错，默认-1
chkpConfig.setTolerableCheckpointFailureNumber(0)
```



```scala
// 重启策略配置
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(3, 10000L))//重启三次每隔10秒重启一次
env.setRestartStrategy(RestartStrategies.failureRateRestart(5, Time.minutes(5), Time.seconds(10)))//5分钟内重启5次，没两次之间间隔10秒
```



#### 5.5.4、保存点

保存点（SavePoint）

Flink 还提供了可以自定义的镜像保存功能，就是保存点（savepoints）。原则上，创建保存点使用的算法与检查点完全相同，因此保存点可以认为就是具有一些额外元数据的检查点。

- Flink不会自动创建保存点，因此用户（或者外部调度程序）必须明确地触发创建操作 。
- 保存点是一个强大的功能。除了故障恢复外，保存点可以用于：有计划的手动备份，更新应用程序，版本迁移，暂停和重启应用，等等。



### 5.6、端到端 exactly-once

目前我们看到的一致性保证都是由流处理器实现的，也就是说都是在 Flink 流处理器内部保证的；而在真实应用中，流处理应用除了流处理器以外还包含了数据源（例如 Kafka）和输出到持久化系统

```txt
- 端到端的一致性保证，意味着结果的正确性贯穿了整个流处理应用的始终；每一个组件都保证了它自己的一致性；
- 整个端到端的一致性级别取决于所有组件中一致性最弱的组件。
```

1）内部保证 —— checkpoint

2）source 端 —— 可重设数据的读取位置

3）sink 端 —— 从故障恢复时，数据不会重复写入外部系统

```txt
- 幂等写入
- 事务写入
```



#### 5.6.1、幂等写入

所谓幂等操作，是说一个操作，可以重复执行很多次，但只导致一次结果更改，也就是说，后面再重复执行就不起作用了。

```txt
例如：key-value型数据，对key求hash值，在根据hash值确定存放位置，那么key相同的数据，无论怎么存都会存放到同一个位置
```



#### 5.6.2、事务写入

事务（Transaction）

```
- 应用程序中一系列严密的操作，所有操作必须成功完成，否则在每个操作中所作的所有更改都会被撤消；
- 具有原子性：一个事务中的一系列的操作要么全部成功，要么一个都不做。
```

Flink中的事务写入实现思想

```txt
构建的事务对应着 checkpoint，等到 checkpoint 真正完成的时候，才把所有对应的结果写入 sink 系统中
```

实现方式

```txt
- 预写日志
- 两阶段提交
```



#### 5.6.3、预写日志（WAL）

预写日志（Write-Ahead-Log，WAL，Hbase中也有）

```txt
- 把结果数据先当成状态保存，然后在收到 checkpoint 完成的通知时，一次性写入 sink 系统；
- DataStream API 提供了一个模板类：GenericWriteAheadSink，来实现这种事务性 sink。
```

```txt
注意：
	- 一批写入sink，时效性降低了
	- 一批写入sink，写到一半挂掉，数据需要重新写入，如果sink没有使用exactly-once，会造成数据重复
```



#### 5.6.4、两阶段提交（2PC）

两阶段提交（Two-Phase-Commit，2PC）

```txt
- 对于每个 checkpoint，sink 任务会启动一个事务，并将接下来所有接收的数据添加到事务里
- 然后将这些数据写入外部 sink 系统，但不提交它们 —— 这时只是“预提交”
- 当它收到 checkpoint 完成的通知时，它才正式提交事务，实现结果的真正写入
```

```txt
这种方式真正实现了 exactly-once，它需要一个提供事务支持的外部 sink 系统。Flink 提供了 TwoPhaseCommitSinkFunction 接口。
```

2PC 对外部 sink 系统的要求

```txt
- sink必须提供事务支持
- 在收到 checkpoint 完成的通知之前，事务必须是“等待提交”的状态。在故障恢复的情况下，这可能需要一些时间。如果这个时候sink系统关闭
  事务（例如超时了），那么未提交的数据就会丢失
- sink 任务必须能够在进程失败后恢复事务，提交事务必须是幂等操作
```

#### 5.6.5、一致性对比

不同 Source 和 Sink 的一致性对比

![image-20200811103607912](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200811103607912.png)



#### 5.6.7、Kafka 状态一致性的保证

Flink+Kafka 端到端状态一致性的保证

```txt
内部：checkpoint
source：kafka consumer（保存偏移量）
sink：kafka producer
```

 Kafka Exactly-once 两阶段提交步骤

- 第一条数据来了之后，开启一个 kafka 的事务（transaction），正常写入 kafka 分区日志但标记为未提交，这就是“预提交”
- jobmanager 触发 checkpoint 操作，barrier 从 source 开始向下传递，遇到 barrier 的算子将状态存入状态后端，并通知 jobmanager
- sink 连接器收到 barrier，保存当前状态，存入 checkpoint，通知 jobmanager，并开启下一阶段的事务，用于提交下个检查点的数据
- jobmanager 收到所有任务的通知，发出确认信息，表示 checkpoint 完成
- sink 任务收到 jobmanager 的确认信息，正式提交这段时间的数据
- 外部kafka关闭事务，提交的数据可以正常消费了。




## 六、ProcessFunction API

### 6.1、概述

之前学习的**转换算子**是无法访问事件的时间戳信息和水位线信息的。基于此，DataStream API提供了一系列的Low-Level转换算子。可以**访问时间戳、watermark以及注册定时事件**。还可以输出**特定的一些事件**，例如超时事件等。

Flink提供了8个Process Function：

```txt
- ProcessFunction （DataStream）
- KeyedProcessFunction （KeySteam）
- CoProcessFunction 
- ProcessJoinFunction
- BroadcastProcessFunction
- KeyedBroadcastProcessFunction
- ProcessWindowFunction
- ProcessAllWindowFunction
```



### 6.2、KeyedProcessFunction

用来操作KeyedStream，会处理流的每一个元素。

1）所有的Process Function都继承自RichFunction接口，所以都有open()、close()和getRuntimeContext()等
方法。

2）KeyedProcessFunction[KEY, IN, OUT]还额外提供了两个方法:processElement、onTimer

```txt
- processElement流中的每一个元素都会调用这个方法，可以访问元素的时间戳，元素的key，以及TimerService时间
  服务
  
- 定时器触发时调用
```

代码：

```scala
// KeyedProcessFunction功能测试
class MyKeyedProcessFunction extends KeyedProcessFunction[String, SensorReading, String]{
  var myState: ValueState[Int] = _

  override def open(parameters: Configuration): Unit = {
    myState = getRuntimeContext.getState(new ValueStateDescriptor[Int]("mystate", classOf[Int]))
  }

  override def processElement(value: SensorReading, ctx: KeyedProcessFunction[String, SensorReading, String]#Context, out: Collector[String]): Unit = {
    ctx.getCurrentKey
    ctx.timestamp()
    ctx.timerService().currentWatermark()
    ctx.timerService().registerEventTimeTimer(ctx.timestamp() + 60000L)
  }

  override def onTimer(timestamp: Long, ctx: KeyedProcessFunction[String, SensorReading, String]#OnTimerContext, out: Collector[String]): Unit = {

  }
}
```





3）需求：10秒之内温度时连续上升的化报警

- 如果用滚动窗口，会出现下面的情况，应该报警却未报警的现象

![image-20200810154855580](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200810154855580.png)

- 如果用滚动窗口，同样会出现应该报警却未报警的现象

![image-20200810155335550](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200810155335550.png)



- 使用KeyedProcessFunction，当确定第一个上升温度时，定义一个定时器（判断未来10秒内是否连续上升），如果有下降温度删除定时器

```scala
object ProcessFunctionTest {
  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)

    // 读取数据
    val inputStream = env.socketTextStream("localhost", 7777)

    // 先转换成样例类类型（简单转换操作）
    val dataStream = inputStream
      .map( data => {
        val arr = data.split(",")
        SensorReading(arr(0), arr(1).toLong, arr(2).toDouble)
      } )
    //      .keyBy(_.id)
    //      .process( new MyKeyedProcessFunction )

    val warningStream = dataStream
      .keyBy(_.id)
      .process( new TempIncreWarning(10000L) )

    warningStream.print()

    env.execute("process function test")
  }
}

// 实现自定义的KeyedProcessFunction
class TempIncreWarning(interval: Long) extends KeyedProcessFunction[String, SensorReading, String]{
  // 定义状态：保存上一个温度值进行比较，保存注册定时器的时间戳用于删除
  lazy val lastTempState: ValueState[Double] = getRuntimeContext.getState(new ValueStateDescriptor[Double]("last-temp", classOf[Double]))
  lazy val timerTsState: ValueState[Long] = getRuntimeContext.getState(new ValueStateDescriptor[Long]("timer-ts", classOf[Long]))

  override def processElement(value: SensorReading, ctx: KeyedProcessFunction[String, SensorReading, String]#Context, out: Collector[String]): Unit = {
    // 先取出状态
    val lastTemp = lastTempState.value()
    val timerTs = timerTsState.value()

    // 更新温度值
    lastTempState.update(value.temperature)

    // 当前温度值和上次温度进行比较
    if( value.temperature > lastTemp && timerTs == 0 ){
      // 如果温度上升，且没有定时器，那么注册当前时间10s之后的定时器
      val ts = ctx.timerService().currentProcessingTime() + interval
      ctx.timerService().registerProcessingTimeTimer(ts)
      timerTsState.update(ts)
    } else if( value.temperature < lastTemp ){
      // 如果温度下降，那么删除定时器
      ctx.timerService().deleteProcessingTimeTimer(timerTs)
      timerTsState.clear()
    }
  }

  override def onTimer(timestamp: Long, ctx: KeyedProcessFunction[String, SensorReading, String]#OnTimerContext, out: Collector[String]): Unit = {
    out.collect("传感器" + ctx.getCurrentKey + "的温度连续" + interval/1000 + "秒连续上升")
    timerTsState.clear()
  }
}
```



### 6.3、侧输出流

```scala
object SideOutputTest {
  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)
    env.setStateBackend(new FsStateBackend(""))
    //    env.setStateBackend(new RocksDBStateBackend(""))

    // 读取数据
    val inputStream = env.socketTextStream("localhost", 7777)

    // 先转换成样例类类型（简单转换操作）
    val dataStream = inputStream
      .map( data => {
        val arr = data.split(",")
        SensorReading(arr(0), arr(1).toLong, arr(2).toDouble)
      } )

    val highTempStream = dataStream
      .process( new SplitTempProcessor(30.0) )

    // 主输出流
    highTempStream.print("high")
    // 侧输出流
    highTempStream.getSideOutput(new OutputTag[(String, Long, Double)]("low")).print("low")

    env.execute("side output test")
  }
}

// 实现自定义ProcessFunction，进行分流
class SplitTempProcessor(threshold: Double) extends ProcessFunction[SensorReading, SensorReading]{
  override def processElement(value: SensorReading, ctx: ProcessFunction[SensorReading, SensorReading]#Context, out: Collector[SensorReading]): Unit = {
    if( value.temperature > threshold ){
      // 如果当前温度值大于30，那么输出到主流
      out.collect(value)
    } else {
      // 如果不超过30度，那么输出到侧输出流
      ctx.output(new OutputTag[(String, Long, Double)]("low"), (value.id, value.timestamp, value.temperature))
    }
  }
}

```

