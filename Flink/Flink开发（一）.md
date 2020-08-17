# Flink开发（上）

- [Flink开发（上）](#flink开发上)
  - [一、基础操作](#一基础操作)
    - [1.1、创建项目](#11创建项目)
    - [1.2、Wordcount的实现](#12wordcount的实现)
      - [1.2.1、批处理](#121批处理)
      - [1.2.2、流处理](#122流处理)
      - [1.2.3、设置并行度](#123设置并行度)
    - [1.3、提交job](#13提交job)
  - [二、DataSream API](#二datasream-api)
    - [2.1、Environment](#21environment)
    - [2.2、Source](#22source)
    - [2.3、Transform](#23transform)
      - [2.3.1、map、flatMap、filter](#231mapflatmapfilter)
      - [2.3.2、KeyBy](#232keyby)
      - [2.3.3、滚动聚合算子](#233滚动聚合算子)
      - [2.3.4、Reduce](#234reduce)
      - [2.3.5、Split 和 Select](#235split-和-select)
      - [2.3.6、Connect和 CoMap](#236connect和-comap)
      - [2.3.7、Union](#237union)
    - [2.4、基础数据类型](#24基础数据类型)
    - [2.5、实现UDF函数](#25实现udf函数)
      - [2.5.1、函数类](#251函数类)
      - [2.5.2、匿名函数](#252匿名函数)
      - [2.5.3、富函数](#253富函数)
    - [2.6、Sink](#26sink)
      - [2.6.1、File Sink](#261file-sink)
      - [2.6.2、Kafka Sink](#262kafka-sink)
      - [2.6.3、Redis Sink](#263redis-sink)
      - [2.6.4、Elasticsearch Sink](#264elasticsearch-sink)
      - [2.6.5、JDBC Sink](#265jdbc-sink)
  - [三、Window](#三window)
    - [3.1、Window概述](#31window概述)
    - [3.2、Window类型](#32window类型)
    - [3.3、Window API](#33window-api)
      - [3.3.1、概述](#331概述)
      - [3.3.2、创建窗口](#332创建窗口)
      - [3.3.3、窗口函数](#333窗口函数)
      - [3.3.4、其它可选 API](#334其它可选-api)
  - [四、时间语义与Wartermark](#四时间语义与wartermark)
    - [4.1、时间语义概述](#41时间语义概述)
    - [4.2、在代码中设置 Event Time](#42在代码中设置-event-time)
    - [4.3、Watermark](#43watermark)
    - [4.4、Watermark传递机制](#44watermark传递机制)
    - [4.5、代码引入Watermark](#45代码引入watermark)
    - [4.6、窗口起始点如何计算](#46窗口起始点如何计算)

## 一、基础操作

### 1.1、创建项目

1）创建maven工程，pom文件如下

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.atguigu.flink</groupId>
    <artifactId>FlinkTutorial</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-scala_2.12</artifactId>
            <version>1.10.1</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.flink/flink-streaming-scala -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-scala_2.12</artifactId>
            <version>1.10.1</version>
        </dependency>
    </dependencies>

<build>
    <plugins>
    <!-- 该插件用于将Scala代码编译成class文件 -->
    <plugin>
        <groupId>net.alchim31.maven</groupId>
        <artifactId>scala-maven-plugin</artifactId>
        <version>3.4.6</version>
        <executions>
            <execution>
                <!-- 声明绑定到maven的compile阶段 -->
                <goals>
                    <goal>compile</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-assembly-plugin</artifactId>
            <version>3.0.0</version>
            <configuration>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
            </configuration>
            <executions>
                <execution>
                    <id>make-assembly</id>
                    <phase>package</phase>
                    <goals>
                        <goal>single</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
</project>
```

2）添加scala框架 和 scala文件夹



### 1.2、Wordcount的实现

#### 1.2.1、批处理

```scala
import org.apache.flink.api.scala._

object WordCount {
  def main(args: Array[String]): Unit = {
    // 创建一个批处理执行环境
    val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment

    // 从文件中读取数据
    val inputPath: String = "input/hello.txt"
    val inputDataSet = env.readTextFile(inputPath)

    val resultDataSet = inputDataSet
      .flatMap(_.split(" "))
      .map((_, 1))
      .groupBy(0) // 以第一个元素作为key进行分组
      .sum(1) // 对所有数据的第二个元素求和

    resultDataSet.print()

  }
}
```

#### 1.2.2、流处理

```scala
import org.apache.flink.api.java.utils.ParameterTool
import org.apache.flink.api.scala._
import org.apache.flink.streaming.api.scala.{DataStream, StreamExecutionEnvironment}

object WordCountStream {
  def main(args: Array[String]): Unit = {
    // 创建一个批处理执行环境
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    // 从外部命令中获取参数
    val params: ParameterTool =  ParameterTool.fromArgs(args)
    val host: String = params.get("host")
    val port: Int = params.getInt("port")

    // 接收一个socket文本流
    // nc -lk 7777
    val inputStream: DataStream[String] = env.socketTextStream(host, port)

    val resultStream: DataStream[(String, Int)] = inputStream
      .flatMap(_.split("\\s"))
      .filter(_.nonEmpty)
      .map((_, 1)) 
      .keyBy(0)
      .sum(1)
    resultStream.print()

    // 启动任务执行
    env.execute()
  }
}
```

注意：这里使用ParameterTool来获取main方法的参数，查看fromArgs源码

```scala
else if (args[i].startsWith("--") || args[i].startsWith("-")) {
    // the argument cannot be a negative number because we checked earlier
    // -> the next argument is a parameter name
    map.put(key, NO_VALUE_KEY);
} 
```

默认获取"--" 或"-"开头的参数，设置main方法的启动配置Program arguments

```txt
--host hadoop102 --port 7777
```

最后在hadoop102上启动netcat

```shell
nc -lk 7777
```

传一下数据观察下规律的，如下图：

![image-20200806175137328](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200806175137328.png)

"hello"一直都是3号线程处理，"hbase"一直都是4号线程处理，因为keyBy，数据进行了重分区，优点类似shuffle

![image-20200806180357751](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200806180357751.png)

如何确定是由哪个sum来统计，其实是对key进行hash再对并行度取模

```txt
"hello".hashCode() % 8
```



#### 1.2.3、设置并行度

并行度就是接受数据时用几个线程来处理

1）并行度设置

- 全局设置


```scala
 // 创建一个批处理执行环境
 val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
 // 设置并行度（不设置并行度，默认使用cpu的核数）
 env.setParallelism(2)
```

- 单个算子指定由某一个并行度执行（默认为1）

```scala
map((_, 1)).setParallelism(2) // 指定某个并行度执行
```

- 给print指定由某一个并行度执行


```scala
resultStream.print().setParallelism(1)
```

假如最后的结果要写入文件中，如果不指定并行度，多线程写文件会出错

- 提交job时，在网站上指定


2）并行度优先级：算子指定 > 全局设置 > 网站上指定 > 配置文件默认值



### 1.3、提交job



![image-20200806203125210](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200806203125210.png)





## 二、DataSream API

### 2.1、Environment

![image-20200808092653140](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200808092653140.png)

 1）批处理执行环境

```scala
val env = ExecutionEnvironment.getExecutionEnvironment
```

2）流处理执行环境

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
```

3）返回本地执行环境，需要在调用时指定默认的并行度

```scala
val env = StreamExecutionEnvironment.createLocalEnvironment(1)
```

4）返回集群执行环境，将Jar提交到远程服务器。需要在调用时指定JobManager的IP和端口号，并指定要在集群中运行的Jar包

```scala
val env = ExecutionEnvironment.createRemoteEnvironment("jobmanage-hostname", 6123,"YOURPATH//wordcount.jar")
```



### 2.2、Source

定义样例类

```scala
// 定义样例类，温度传感器
case class SensorReading( id: String, timestamp: Long, temperature: Double )
```

1）从集合中读取数据

```scala
 // 创建执行环境
 val env = StreamExecutionEnvironment.getExecutionEnvironment
 // 1、从集合中读取数据
 val dataList = List(
     SensorReading("sensor_1", 1547718199, 35.8),
     SensorReading("sensor_6", 1547718201, 15.4),
     SensorReading("sensor_7", 1547718202, 6.7),
     SensorReading("sensor_10", 1547718205, 38.1)
 )
 val stream = env.fromCollection(dataList)
 stream.print()
```

2）从文件读取数据

```scala
// 2、从文件中读取数据
val inputPath = "input/hello.txt"
val stream2 = env.readTextFile(inputPath)
stream2.print()
```

3）从kafka中读取数据

- 需要引入kafka连接器的依赖，pom.xml

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka-0.11_2.12</artifactId>
    <version>1.10.1</version>
</dependency>
```

- 具体代码如下

```scala
// 3、从kafka中读取数据
val properties = new Properties()
properties.setProperty("bootstrap.servers", "hadoop102:9092")
properties.setProperty("group.id", "consumer-group")
properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")
properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")
properties.setProperty("auto.offset.reset", "latest")

// bin/kafka-console-producer.sh --broker-list hadoop102:9092 --topic sensor
val stream3 = env.addSource(new FlinkKafkaConsumer011[String]("sensor", new SimpleStringSchema(), properties))
stream3.print()
```



4）自定义Source

除了以上的source数据来源，我们还可以自定义source。需要做的，只是传入一个SourceFunction就可以。具体调用如下

- 自定义SourceFunction

```scala
// 自定义SourceFunction
class MySensorSource() extends SourceFunction[SensorReading]{
  // 定义一个标识位flag，用来表示数据源是否正常运行发出数据
  var running: Boolean = true

  override def cancel(): Unit = running = false

  override def run(ctx: SourceFunction.SourceContext[SensorReading]): Unit = {
    // 定义一个随机数发生器
    val rand = new Random()
    // 随机生成一组（10个）传感器的初始温度: （id，temp）
    var curTemp = 1.to(10).map( i => ("sensor_" + i, rand.nextDouble() * 100) )
    // 定义无限循环，不停地产生数据，除非被cancel
    while(running){
      // 在上次数据基础上微调，更新温度值
      curTemp = curTemp.map(
        data => (data._1, data._2 + rand.nextGaussian())
      )
      // 获取当前时间戳，加入到数据中，调用ctx.collect发出数据
      val curTime = System.currentTimeMillis()
      curTemp.foreach(
        data => ctx.collect(SensorReading(data._1, curTime, data._2))
      )
      // 间隔500ms
      Thread.sleep(500)
    }
  }
}
```

- 调用SourceFunction

```scala
 // 4. 自定义Source
val stream4 = env.addSource( new MySensorSource() )
stream4.print()
env.execute()
```



### 2.3、Transform

#### 2.3.1、map、flatMap、filter

用法与spark中一致

#### 2.3.2、KeyBy

```txt
DataStream => KeyedStream
```

![image-20200808094719534](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200808094719534.png)

对key取hash再对分区取模 => 相同的key在同一个分区上，一个分区可以包含多个key



#### 2.3.3、滚动聚合算子

滚动聚合算子（Rolling Aggregation），这些算子可以针对KeyedStream的每一个支流做聚合。

```txt
sum()
min()
max()
minBy()
maxBy()
```

实例：创建sensor.txt，找出最低温度

```txt
sensor_1,1547718199,35.8
sensor_6,1547718201,15.4
sensor_7,1547718202,6.7
sensor_10,1547718205,38.1
sensor_1,1547718206,32
sensor_1,1547718208,36.2
sensor_1,1547718210,29.7
sensor_1,1547718213,30.9
```

代码

```scala
 val inputStream = env.readTextFile("input/sensor.txt")
// 1、先转换成样例类类型（简单转换操作）
val dataStream = inputStream.map(data => {
    val strs = data.split(",")
    SensorReading(strs(0), strs(1).toLong, strs(2).toDouble)
})

// dataStream -> keyStream
// 2、聚合
val aggStream = dataStream.keyBy(_.id).minBy("temperature")
// min：只会找出temperature最小值（id、timestamp首次加载后不在变动）
// minBy：会找出temperature最小值，及其对应的id、timestamp整条数据
aggStream.print()
```



#### 2.3.4、Reduce

**KeyedStream** **→** **DataStream**： 分组之后的聚合。

实例：复杂需求，查询到现在为止，出现过的最低温度（tempstamp要不断刷新，temperature取最小值）

1）lamda表达式

2）自定义ReduceFunction

```scala
class MyReduceFunction extends ReduceFunction[SensorReading] {
  override def reduce(value1: SensorReading, value2: SensorReading): SensorReading =
    SensorReading(value1.id, value2.timestamp, value1.temperature.min(value2.temperature))
}
```

```scala
val resultStream = dataStream
      .keyBy("id")
	  // 方法一：lamda表达式
      //      .reduce( (curState, newData) =>
      //        SensorReading( curState.id, newData.timestamp, curState.temperature.min(newData.temperature) )
      //      )
      .reduce(new MyReduceFunction)
// min max reduce这些聚合函数都在keyStream中
resultStream.print()
```



#### 2.3.5、Split 和 Select

1）Split（分流）：

```txt
DataStream => SplitStream：根据某些特征把一个DataStream拆分成两个或者多个DataStream。
```

![image-20200808095918347](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200808095918347.png)

2）select

```txt
SplitStream→DataStream：从一个SplitStream中获取一个或者多个DataStream。
```

![image-20200808095939867](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200808095939867.png)

实例：（需求）传感器数据按照温度高低（以30度为界），拆分成两个流。

```scala
val splitStream = stream2
  .split( sensorData => {
    if (sensorData.temperature > 30) Seq("high") else Seq("low")
  } )

val high = splitStream.select("high")
val low = splitStream.select("low")
val all = splitStream.select("high", "low")
```



#### 2.3.6、Connect和 CoMap

1）connect（合流）

```scala
DataStream,DataStream => ConnectedStreams：连接两个保持他们类型的数据流，两个数据流被Connect之后，只是被放在了一个同一个流中，内部依然保持各自的数据和形式不发生任何变化，两个流相互独立。
```

![image-20200808100552991](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200808100552991.png)

2）CoMap,CoFlatMap

```scala
ConnectedStreams → DataStream：作用于ConnectedStreams上，功能与map和flatMap一样，对ConnectedStreams中的每一个Stream分别进行map和flatMap处理。
```

![image-20200808100636443](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200808100636443.png)

```scala
// 4.2 合流，connect
val warningStream = highTempStream.map(data => (data.id, data.temperature))
val connectedStreams = warningStream.connect(lowTempStream)

// 用coMap对数据进行分别处理
val coMapResultStream: DataStream[Any] = connectedStreams
.map(
    waringData => (waringData._1, waringData._2, "warning"),
    lowTempData => (lowTempData.id, "healthy")
)
coMapResultStream.print("coMap")
```

输出结果：

```txt
coMap:7> (sensor_1,30.9,warning)
coMap:11> (sensor_6,healthy)
coMap:13> (sensor_7,healthy)
coMap:1> (sensor_1,32.0,warning)
coMap:15> (sensor_10,38.1,warning)
coMap:10> (sensor_1,35.8,warning)
coMap:5> (sensor_1,healthy)
coMap:3> (sensor_1,36.2,warning)
```

3）合流在实际开发中有哪些运用场景

```txt
火情的检测，只检测温度不合理，高温不一定着火，冒烟了才算着火
高温 + 烟雾指数 => 合流
```



#### 2.3.7、Union

```txt
DataStream => DataStream：对两个或者两个以上的DataStream进行union操作，产生一个包含所有DataStream元素的新DataStream。
 	- 注意： union合流,数据类型必须一样
```

![image-20200808101317580](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200808101317580.png)

```scala
val unionStream = highTempStream.union(lowTempStream, allTempStream)
```



### 2.4、基础数据类型

1）Flink支持所有的Java和Scala基础数据类型，Int, Double, Long, String, …

2）Java和Scala元组（Tuples）

```txt
flink用java语言实现了元组功能
```

3）Scala样例类（case classes）

```scala
case class Person(name: String, age: Int) 
val persons: DataStream[Person] = env.fromElements(
Person("Adam", 17), 
Person("Sarah", 23) )
persons.filter(p => p.age > 18)
```

4）Java简单对象（POJOs）

```java
public class Person {
public String name;
public int age;
  public Person() {}
  public Person(String name, int age) { 
    this.name = name;      
    this.age = age;  
  }
}
DataStream<Person> persons = env.fromElements(   
new Person("Alex", 42),   
new Person("Wendy", 23));
```

5）其它（Arrays, Lists, Maps, Enums, 等等）

```txt
Flink对Java和Scala中的一些特殊目的的类型也都是支持的，比如Java的ArrayList，HashMap，Enum等等。
```



### 2.5、实现UDF函数

实现UDF函数（更细粒度的控制流）

#### 2.5.1、函数类

函数类（Function Classes）Flink暴露了所有udf函数的接口(实现方式为接口或者抽象类)。例如MapFunction, FilterFunction, ProcessFunction等等

1）FilterFunction接口

```scala
class FilterFilter extends FilterFunction[String] {
      override def filter(value: String): Boolean = {
        value.contains("flink")
      }
}
val flinkTweets = tweets.filter(new FlinkFilter)
```

2）我们filter的字符串"flink"还可以当作参数传进去

```scala
val tweets: DataStream[String] = ...
val flinkTweets = tweets.filter(new KeywordFilter("flink"))

class KeywordFilter(keyWord: String) extends FilterFunction[String] {
    override def filter(value: String): Boolean = {
    	value.contains(keyWord)
    }
}
```

3）匿名类

```scala
val flinkTweets = tweets.filter(
    new RichFilterFunction[String] {
        override def filter(value: String): Boolean = {
        	value.contains("flink")
        }
    }
)
```

#### 2.5.2、匿名函数

匿名函数（Lambda Functions）

```scala
val tweets: DataStream[String] = ...
val flinkTweets = tweets.filter(_.contains("flink"))
```

#### 2.5.3、富函数

富函数（Rich Function）是DataStream API提供的一个函数类的接口，所有Flink函数类都有其Rich版本。它与常规函数的不同在于，可以获取运行环境的上下文，并拥有一些生命周期方法，所以可以实现更复杂的功能。

Rich Function有一个生命周期的概念。典型的生命周期方法有：

```txt
- open()方法是rich function的初始化方法，当一个算子例如map或者filter被调用之前open()会被调用。
- close()方法是生命周期中的最后一个调用的方法，做一些清理工作。
- getRuntimeContext()方法提供了函数的RuntimeContext的一些信息，例如函数执行的并行度，任务的名字，以及state状态
```

实例：

```scala
// 富函数，可以获取到运行时上下文，还有一些生命周期
class MyRichMapper extends RichMapFunction[SensorReading, String]{

  // 在构造函数执行完成后，数据加载之前执行
  override def open(parameters: Configuration): Unit = {
    // 做一些初始化操作，比如数据库的连接
    //    getRuntimeContext
  }

  override def map(value: SensorReading): String = value.id + " temperature"

  override def close(): Unit = {
    //  一般做收尾工作，比如关闭连接，或者清空状态
  }
}
```



### 2.6、Sink

Flink没有类似于spark中foreach方法，让用户进行迭代的操作。所有对外的输出操作都要利用Sink完成。

```scala
 stream.addSink(new MySink(xxxx)) 
```

#### 2.6.1、File Sink

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
// 读取数据
val inputPath = "D:\\Projects\\BigData\\FlinkTutorial\\src\\main\\resources\\sensor.txt"
val inputStream = env.readTextFile(inputPath)

// 先转换成样例类类型（简单转换操作）
val dataStream = inputStream
.map( data => {
    val arr = data.split(",")
    SensorReading(arr(0), arr(1).toLong, arr(2).toDouble)
} )

dataStream.print()
//    dataStream.writeAsCsv("D:\\Projects\\BigData\\FlinkTutorial\\src\\main\\resources\\out.txt")
dataStream.addSink(
    StreamingFileSink.forRowFormat(
        new Path("D:\\Projects\\BigData\\\\FlinkTutorial\\src\\main\\resources\\out1.txt"),
        new SimpleStringEncoder[SensorReading]()
    ).build()
)

env.execute("file sink test")
```



#### 2.6.2、Kafka Sink

1）pom.xml添加flink-connector-kafka依赖

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka-0.11_2.12</artifactId>
    <version>1.10.1</version>
</dependency>
```

2）代码：从sensor中读取数据，再将数据写入到sinktest中

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
// 从kafka读取数据
val properties = new Properties()
properties.setProperty("bootstrap.servers", "hadoop102:9092")
properties.setProperty("group.id", "consumer-group")
val stream = env.addSource( new FlinkKafkaConsumer011[String]("sensor", new SimpleStringSchema(), properties) )


// 先转换成样例类类型（简单转换操作）
val dataStream = stream
.map( data => {
    val arr = data.split(",")
    SensorReading(arr(0), arr(1).toLong, arr(2).toDouble).toString
} )

dataStream.addSink( new FlinkKafkaProducer011[String]("hadoop102:9092", "sinktest", new SimpleStringSchema()) )
env.execute("kafka sink test")
```

- sensor生成数据

```shell
bin/kafka-console-producer.sh --broker-list hadoop102:9092 --topic sensor
```

- 消费sensortest

```shell
/opt/module/kafka/bin/kafka-console-consumer.sh --bootstrap-server hadoop102:9092 --from-beginning --topic sensortest
```



#### 2.6.3、Redis Sink

1）pom.xml添加flink-connector-redis依赖

```xml
<dependency>
    <groupId>org.apache.bahir</groupId>
    <artifactId>flink-connector-redis_2.11</artifactId>
    <version>1.0</version>
</dependency>
```

2）实例：从文件中读取数据，存到redis中

- 定义一个RedisMapper

```scala
class MyRedisMapper extends RedisMapper[SensorReading]{
  // 定义保存数据写入redis的命令，HSET 表名 key value
  override def getCommandDescription: RedisCommandDescription = {
    new RedisCommandDescription(RedisCommand.HSET, "sensor_temp")
  }

  // 将温度值指定为value
  override def getValueFromData(data: SensorReading): String = data.temperature.toString

  // 将id指定为key
  override def getKeyFromData(data: SensorReading): String = data.id
}
```

- 从文件中读取数据，存到redis中

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
// 读取数据
val inputPath = "input/sensor.txt"
val inputStream = env.readTextFile(inputPath)

// 先转换成样例类类型（简单转换操作）
val dataStream = inputStream
  .map( data => {
    val arr = data.split(",")
    SensorReading(arr(0), arr(1).toLong, arr(2).toDouble)
  } )

// 定义一个FlinkJedisConfigBase
val conf = new FlinkJedisPoolConfig.Builder()
  .setHost("hadoop102")
  .setPort(6379)
  .build()

dataStream.addSink( new RedisSink[SensorReading]( conf, new MyRedisMapper ) )

env.execute("redis sink test")
```



#### 2.6.4、Elasticsearch Sink

1）pom.xml添加flink-connector-elasticsearch依赖

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-elasticsearch6_2.12</artifactId>
    <version>1.10.1</version>
</dependency>
```

2）

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
// 读取数据
val inputPath = "D:\\Projects\\BigData\\FlinkTutorial\\src\\main\\resources\\sensor.txt"
val inputStream = env.readTextFile(inputPath)

// 先转换成样例类类型（简单转换操作）
val dataStream = inputStream
.map(data => {
    val arr = data.split(",")
    SensorReading(arr(0), arr(1).toLong, arr(2).toDouble)
})

// 定义HttpHosts
val httpHosts = new util.ArrayList[HttpHost]()
httpHosts.add(new HttpHost("hadoop102", 9200))

// 自定义写入es的EsSinkFunction
val myEsSinkFunc = new ElasticsearchSinkFunction[SensorReading] {
    override def process(t: SensorReading, runtimeContext: RuntimeContext, requestIndexer: RequestIndexer): Unit = {
        // 包装一个Map作为data source
        val dataSource = new util.HashMap[String, String]()
        dataSource.put("id", t.id)
        dataSource.put("temperature", t.temperature.toString)
        dataSource.put("ts", t.timestamp.toString)

        // 创建index request，用于发送http请求
        val indexRequest = Requests.indexRequest()
        .index("sensor")
        .`type`("readingdata")
        .source(dataSource)

        // 用indexer发送请求
        requestIndexer.add(indexRequest)

    }
}

dataStream.addSink(new ElasticsearchSink
                   .Builder[SensorReading](httpHosts, myEsSinkFunc)
                   .build()
                  )

env.execute("es sink test")
```



#### 2.6.5、JDBC Sink

1）pom.xml添加mysql-connector-java依赖

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.44</version>
</dependency>
```

2） 定义一个RichSinkFunction

```scala
class MyJdbcSinkFunc() extends RichSinkFunction[SensorReading]{
  // 定义连接、预编译语句
    
  var conn: Connection = _
  var insertStmt: PreparedStatement = _
  var updateStmt: PreparedStatement = _

  override def open(parameters: Configuration): Unit = {
    conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/test", "root", "123456")
    insertStmt = conn.prepareStatement("insert into sensor_temp (id, temp) values (?, ?)")
    updateStmt = conn.prepareStatement("update sensor_temp set temp = ? where id = ?")
  }

  override def invoke(value: SensorReading): Unit = {
    // 先执行更新操作，查到就更新
    updateStmt.setDouble(1, value.temperature)
    updateStmt.setString(2, value.id)
    updateStmt.execute()
    // 如果更新没有查到数据，那么就插入
    if( updateStmt.getUpdateCount == 0 ){
      insertStmt.setString(1, value.id)
      insertStmt.setDouble(2, value.temperature)
      insertStmt.execute()
    }
  }

  override def close(): Unit = {
    insertStmt.close()
    updateStmt.close()
    conn.close()
  }
}
```

3）MyJdbcSinkFunc的使用

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
// 读取数据
val inputPath = "D:\\Projects\\BigData\\FlinkTutorial\\src\\main\\resources\\sensor.txt"
val inputStream = env.readTextFile(inputPath)

val stream = env.addSource( new MySensorSource() )

// 先转换成样例类类型（简单转换操作）
val dataStream = inputStream
.map(data => {
    val arr = data.split(",")
    SensorReading(arr(0), arr(1).toLong, arr(2).toDouble)
})

stream.addSink( new MyJdbcSinkFunc() )

env.execute("jdbc sink test")
}
```



## 三、Window

### 3.1、Window概述

window是一种切割无限数据为有限块进行处理的手段。

```txt
- streaming流式计算用于处理无限数据集的数据处理引擎，无限数据集是指一种不断增长的本质上无限的数据集;
- Window将一个无限的stream拆分成有限大小的”buckets”桶，我们可以在这些桶上做计算操作。
```



### 3.2、Window类型

1）CountWindow：按照指定的数据条数生成一个Window，与时间无关.

2）TimeWindow：按照时间生成Window。

- 滚动窗口（Tumbling Windows）：时间对齐，窗口长度固定，没有重叠

![image-20200809102637565](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200809102637565.png)

- 滑动窗口（Sliding Windows）：时间对齐，窗口长度固定，有重叠

![image-20200809102657902](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200809102657902.png)

- 会话窗口（Session Windows）：时间无对齐

![image-20200809102736758](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200809102736758.png)

### 3.3、Window API

#### 3.3.1、概述

如何使用window：可以用 .window() 来定义一个窗口，然后基于这个 window 去做一些聚合或者其它处理操作。

```txt
注意:
	- window()方法必须在 keyBy 之后才能用（每个key会对应一个窗口）。
	- window后面一般会接上reduce方法
```



#### 3.3.2、创建窗口

1）窗口分配器（window assigner）

```txt
- window() 方法接收的输入参数是一个 WindowAssigner
- WindowAssigner 负责将每条输入的数据分发到正确的 window 中
```

2）创建不同类型的窗口

- 滚动时间窗口（tumbling time window）

```scala
.timeWindow(Time.seconds(15))
```

- 滑动时间窗口（sliding time window）

```scala
.timeWindow(Time.seconds(15), Time.seconds(3))
```

- 会话窗口（session window）

```scala
.window(EventTimeSessionWindows.withGap(Time.seconds(10))) 
```

- 滚动计数窗口（tumbling count window）

```scala
.countWindow(1)
```

- 滑动计数窗口（sliding count window）

```scala
.countWindow(10, 2)
```



#### 3.3.3、窗口函数

窗口函数（window function）

1）window function 定义了要对窗口中收集的数据做的计算操作

2）可以分为两类

```txt
- 增量聚合函数（incremental aggregation functions）（建议多使用）
	- 每条数据到来就进行计算，保持一个简单的状态
 	- ReduceFunction, AggregateFunction

- 全窗口函数（full window functions）
	- 先把窗口所有数据收集起来，等到计算的时候会遍历所有数据
 	- ProcessWindowFunction
```



#### 3.3.4、其它可选 API

```txt
- trigger() —— 触发器，定义 window 什么时候关闭，触发计算并输出结果
- evictor() —— 移除器，定义移除某些数据的逻辑
- allowedLateness() —— 允许处理迟到的数据
- sideOutputLateData() —— 将迟到的数据放入侧输出流
- getSideOutput() —— 获取侧输出流
```

 

## 四、时间语义与Wartermark

### 4.1、时间语义概述

在Flink的流式处理中，会涉及到时间的不同概念，

**Event Time**

```txt
是事件创建的时间。它通常由事件中的时间戳描述，例如采集的日志数据中，每一条日志都会记录自己的生成时间，Flink通过时间戳分配器访问事件时间戳。
```

**Ingestion Time**：

```txt
是数据进入Flink的时间。
```

**Processing Time**：

```txt
是每一个执行基于时间操作的算子的本地系统时间，与机器(服务器)相关。
```

```txt
注意：
	- 默认的时间属性就是Processing Time
	- 但是我们往往更关心事件时间（Event Time）	
```

实例：假如地铁上玩游戏，到8:22:45时已同通关到第三关，8:22:45~8:23:20又通关了5关，但是由于地铁进隧道没有信号，数据没有及时传送到服务器上，该游每通关三层会有奖励，最终服务器应该如何计算奖励。

![image-20200809210819955](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200809210819955.png)

1）如果按照Processing Time来统计数据那么玩家只通关了三层，奖励一次，但实际上顽疾已经通关到第8层应该奖励2次，这显然不符合逻辑

2）所以数据统计应该按照Event Time也就是玩家的时间来统计



### 4.2、在代码中设置 Event Time

1）我们可以直接在代码中，对执行环境调用 setStreamTimeCharacteristic 方法，设置流的时间特性

2）具体的时间，还需要从数据中提取时间戳（timestamp）

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
```



### 4.3、Watermark

什么是Watermark？就是到达窗口的时间。

```txt
备注：在大部分情况下，流处理的数据都是按照事件产生的时间顺序来的，但由于网络、分布式等原因，会导致乱序的产生。Watermark专门来解决乱序的问题。
```

```
- Watermark是一种衡量Event Time进展的机制；
- Watermark是用于处理乱序事件的，而正确的处理乱序事件，通常用Watermark机制结合Window来实现；
- 数据流中的Watermark用于表示Timestamp小于Watermark的数据，都已经到达了，因此，window的执行也是由Watermark触发的；
- Watermark 用来让程序自己平衡延迟和结果正确性。
```

Watermark特点：

![image-20200809213711825](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200809213711825.png)

```txt
- Watermark 是一条特殊的数据记录
- Watermark 必须单调递增，以确保任务的事件时间时钟在向前推进，而不是在后退
- Watermark 与数据的时间戳相关
```



### 4.4、Watermark传递机制

实例：上游有4个并行任务，中间是Task（4个分区），下游有三个并行任务，如下图所示

![image-20200809213553109](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200809213553109.png)

1）Task中的每个分区都会存放一个Watermark

2）当上游有的Watermark传递过来时，替换Task所有分区中的最小Watermark（minWatermark）

3）将取出的当前最小minWatermark与Event-time Clock进行对比，如果minWatermark大，则更新Event-time Clock

```
Event-time Clock=minWatermark
```

4）将更后的Event-time Clock以轮询的方式发送给下游

```txt
表示当前Event-time Clock时间段之前的数据已全部到达
```



### 4.5、代码引入Watermark

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
// 从调用时刻开始给env创建的每一个stream追加时间特征：EventTime
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
// 每隔5毫秒产生一个watermark
env.getConfig.setAutoWatermarkInterval(50)

// 读取数据
val inputStream = env.socketTextStream("hadoop102", 7778)

// 先转换成样例类类型（简单转换操作）
val dataStream = inputStream
.map( data => {
    val arr = data.split(",")
    SensorReading(arr(0), arr(1).toLong, arr(2).toDouble)
} )
//      .assignAscendingTimestamps(_.timestamp * 1000L)    // 升序数据提取时间戳
// 要把数据中哪个字段提取成时间戳
.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[SensorReading](Time.seconds(3)) {
    override def extractTimestamp(element: SensorReading): Long = element.timestamp * 1000L
})
// 定义测输出流
val latetag = new OutputTag[(String, Double, Long)]("late")
// 每15秒统计一次，窗口内各传感器所有温度的最小值，以及最新的时间戳
val resultStream = dataStream
.map( data => (data.id, data.temperature, data.timestamp) )
.keyBy(_._1)    // 按照二元组的第一个元素（id）分组
.timeWindow(Time.seconds(15))  // 滚动窗口 15秒 滚动一次
.allowedLateness(Time.minutes(1)) // 允许窗口处理迟到数据，窗口先不关
.sideOutputLateData(latetag) // 窗口已经关闭才来，放到侧输出流里（单独收集起来，再去找之前窗口的统计结果，手动导入）
.reduce( (curRes, newData) => (curRes._1, curRes._2.min(newData._2), newData._3) )

// 从侧输出流中取数据
resultStream.getSideOutput(latetag).print("late")
resultStream.print("result")

env.execute()
```



### 4.6、窗口起始点如何计算

源码

```txt
timeWindow -> avaStream.timeWindow -> TumblingProcessingTimeWindows -> assignWindows
-> TimeWindow.getWindowStartWithOffset
```

```scala
public static long getWindowStartWithOffset(long timestamp, long offset, long windowSize) {
   return timestamp - (timestamp - offset + windowSize) % windowSize;
}
```

假设：第一条数据sensor_1,1547718199,35.8

```txt
timestamp：1547718199000
offset：0（默认值）
windowSize：15000（15秒）
timestamp - (timestamp - offset + windowSize) % windowSize = 1,547,718,195,000
所以第一窗口是：[195, 210],而不是[199, 214]
```



