# Table API 与SQL
- [Table API 与SQL](#table-api-与sql)
  - [一、概述](#一概述)
  - [二、API的调用](#二api的调用)
    - [2.1、基本程序结构](#21基本程序结构)
    - [2.2、创建表环境](#22创建表环境)
    - [2.3、在Catalog中注册表](#23在catalog中注册表)
      - [2.3.1、Flink表的概述](#231flink表的概述)
      - [2.3.2、将 DataStream 转换成表](#232将-datastream-转换成表)
      - [2.3.3、创建临时视图](#233创建临时视图)
      - [2.3.4、连接到文件系统](#234连接到文件系统)
      - [2.3.5、从kafka中读取数据](#235从kafka中读取数据)
    - [2.4、表的查询](#24表的查询)
      - [2.4.1、表的查询](#241表的查询)
      - [2.4.2、更新模式](#242更新模式)
    - [2.5、输出表](#25输出表)
      - [2.5.1、输出到文件](#251输出到文件)
      - [2.5.2、输出到Kafka](#252输出到kafka)
      - [2.5.3、输出到ES](#253输出到es)
      - [2.5.4、输出到Mysql](#254输出到mysql)
    - [2.6、将表转换成DataStream](#26将表转换成datastream)
    - [2.7、查看执行计划](#27查看执行计划)
  - [三、流处理中的特殊概念](#三流处理中的特殊概念)
    - [3.1、流处理和关系代数的区别](#31流处理和关系代数的区别)
    - [3.2、动态表](#32动态表)
    - [3.3、流式持续查询的过程](#33流式持续查询的过程)
      - [3.3.1、流式持续查询总流程](#331流式持续查询总流程)
      - [3.3.2、将流转换成动态表](#332将流转换成动态表)
      - [3.3.3、持续查询](#333持续查询)
      - [3.3.4、将动态表转换成 DataStream](#334将动态表转换成-datastream)
    - [3.4、时间特性](#34时间特性)
      - [3.4.1、处理时间](#341处理时间)
      - [3.4.2、事件时间](#342事件时间)

## 一、概述

1）Flink 对批处理和流处理，提供了统一的上层 API。

2）Table API 是一套内嵌在 Java 和 Scala 语言中的查询API，它允许以非常直观的方式组合来自一些关系运算符的查询。

3）Flink 的 SQL 支持基于实现了 SQL 标准的 Apache Calcite。

![image-20200811105551761](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200811105551761.png)



## 二、API的调用

引包

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-planner_2.12</artifactId>
    <version>1.10.1</version>
</dependency>
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-api-scala-bridge_2.12</artifactId>
    <version>1.10.1</version>
</dependency>
```

### 2.1、基本程序结构

首先创建执行环境，然后定义source、transform和sink。

```scala
// 1、创建表的执行环境
val tableEnv = ...     

// 2、根据source创建表
// 创建一张表，用于读取数据
tableEnv.connect(source).createTemporaryTable("inputTable")
// 注册一张表，用于把计算结果输出
tableEnv.connect(source).createTemporaryTable("outputTable")

// 通过 Table API 查询算子，得到一张结果表
val result = tableEnv.from("inputTable").select(...)
// 通过 SQL查询语句，得到一张结果表
val sqlResult  = tableEnv.sqlQuery("SELECT ... FROM inputTable ...")

// 将结果表写入输出表中
result.insertInto("outputTable")
```



### 2.2、创建表环境

表环境（TableEnvironment）是flink中集成Table API & SQL的核心概念。它负责:

```txt
- 注册catalog
- 在内部 catalog 中注册表
- 执行 SQL 查询
- 注册用户自定义函数
- 将 DataStream 或 DataSet 转换为表
- 保存对 ExecutionEnvironment 或 StreamExecutionEnvironment 的引用
```

1）基于老版本的流处理

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
// 基于老版本的流处理
val settings = EnvironmentSettings.newInstance()
    .useOldPlanner()
    .inStreamingMode()
    .build()
// 默认情况下调用oldPlanner 
val oldStreamTableEnv = StreamTableEnvironment.create(env, settings)
```

2）基于老版本的批处理

```scala
val batchEnv = ExecutionEnvironment.getExecutionEnvironment
val oldBatchTableEnv = BatchTableEnvironment.create(batchEnv)
```

3）基于blink planner的流处理

```scala
val blinkStreamSettings = EnvironmentSettings.newInstance()
    .useBlinkPlanner()
    .inStreamingMode()
    .build()
val blinkStreamTableEnv = StreamTableEnvironment.create(env, blinkStreamSettings)
```

4）基于blink planner的批处理

```scala
val blinkBatchSettings = EnvironmentSettings.newInstance()
    .useBlinkPlanner()
    .inBatchMode()
    .build()
val blinkBatchTableEnv = TableEnvironment.create(blinkBatchSettings)
```



### 2.3、在Catalog中注册表

#### 2.3.1、Flink表的概述

TableEnvironment 可以注册目录 Catalog，并可以基于 Catalog 注册表。表（Table）是由一个“标识符”（identifier）来指定的，由3部分组成：Catalog名、数据库（database）名和对象名。

```txt
- 表可以是常规的，也可以是虚拟的（视图，View）；
- 常规表（Table）一般可以用来描述外部数据，比如文件、数据库表或消息队列的数据，也可以直接从 DataStream转换而来；
- 视图（View）可以从现有的表中创建，通常是 table API 或者 SQL 查询的一个结果集。
```



#### 2.3.2、将 DataStream 转换成表

1）对于一个 DataStream，可以直接转换成 Table，进而方便地调用 Table API 做转换操作

```scala
val dataStream: DataStream[SensorReading] = ...
val sensorTable: Table = tableEnv.fromDataStream(dataStream)
```

默认转换后的 Table schema 和 DataStream 中的字段定义一一对应，也可以单独指定出来

```scala
val dataStream: DataStream[SensorReading] = ...
val sensorTable = tableEnv.fromDataStream(dataStream, 'id, 'timestamp, 'temperature)
```

2）数据类型与 Schema 的对应

- 基于名称（name-based）

```scala
val sensorTable = tableEnv.fromDataStream(dataStream, 
                  'timestamp as 'ts, 'id as 'myId, 'temperature)
```

- 基于位置（position-based）

```scala
val sensorTable = tableEnv.fromDataStream(dataStream, 'myId, 'ts)
```

Flink也支持将表转换成DataStream，详情见   [2.6、将表转换成DataStream](#将表转换成DataStream)



#### 2.3.3、创建临时视图

1）基于 DataStream 创建临时视图

```scala
tableEnv.createTemporaryView("sensorView", dataStream)
tableEnv.createTemporaryView("sensorView", dataStream, 'id, 'temperature, 'timestamp as 'ts)
```

2）基于 Table 创建临时视图

```scala
tableEnv.createTemporaryView("sensorView", sensorTable)
```





#### 2.3.4、连接到文件系统

连接外部系统在Catalog中注册表，直接调用tableEnv.connect()就可以，里面参数要传入一个ConnectorDescriptor，也就是connector描述器。对于文件系统的connector而言，flink内部已经提供了，就叫做FileSystem()。

1）引入Csv的包

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-csv</artifactId>
    <version>1.10.1</version>
</dependency>
```

2）sensor.txt的内容如下

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

3）代码

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)

// 读取数据
val inputPath = "input/sensor.txt"
val tableEnv = StreamTableEnvironment.create(env)
tableEnv.connect(new FileSystem().path(inputPath)) // 定义到文件系统的连接
.withFormat(new Csv()) // 定义以csv格式进行数据格式化
.withSchema(new Schema()
            .field("id", DataTypes.STRING())
            .field("timestamp", DataTypes.BIGINT())
            .field("temperature", DataTypes.DOUBLE())
           ) // 定义表结构
.createTemporaryTable("inputTable") // 创建临时表

// 打印数据
val inputTable = tableEnv.from("inputTable")
// 将表转换成DataStream
inputTable.toAppendStream[(String, Long, Double)].print()
env.execute("file test")
```

inputTable.toAppendStream就是将表转换成DataStream，详情见   [2.6、将表转换成DataStream](#将表转换成DataStream)



#### 2.3.5、从kafka中读取数据

1）代码

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
// 读取数据
val inputPath = "input/sensor.txt"
val tableEnv = StreamTableEnvironment.create(env)
tableEnv.connect(new Kafka()
                 .version("0.11")
                 .topic("sensor")
                 .property("zookeeper.connect", "hadoop102:2181")
                 .property("bootstrap.servers", "hadoop102:9092")
                )
.withFormat(new Csv())
.withSchema(new Schema()
            .field("id", DataTypes.STRING())
            .field("timestamp", DataTypes.BIGINT())
            .field("temperature", DataTypes.DOUBLE())
           )
.createTemporaryTable("kafkaInputTable")

// 打印数据
val inputTable = tableEnv.from("kafkaInputTable")
// 将表转换成DataStream
inputTable.toAppendStream[(String, Long, Double)].print()
env.execute("kafka test")
```

inputTable.toAppendStream就是将表转换成DataStream，详情见   [2.6、将表转换成DataStream](#将表转换成DataStream)

2）启动kafka，往sensor中生成数据



### 2.4、表的查询

#### 2.4.1、表的查询

查询2.3中通过文件系统创建的表"inputTable"

1）Table API

```txt
Table API 基于代表“表”的 Table 类，并提供一整套操作处理的方法 API；这些方法会返回一个新的 Table 对象，表示对输入表应用转换操作的结果。有些关系型转换操作，可以由多个方法调用组成，构成链式调用结构。
```

```scala
val sensorTable = tableEnv.from("inputTable")
// 使用table api
val resultTable = sensorTable
    .select('id, 'temperature)
    .filter('id === "sensor_1")
resultTable.toAppendStream[(String, Double)].print("result")
```

2）SQL

```
- Flink 的 SQL 集成，基于实现 了SQL 标准的 Apache Calcite；
- 在 Flink 中，用常规字符串来定义 SQL 查询语句；
- SQL 查询的结果，也是一个新的 Table。
```

```scala
val resultSQLTable = tableEnv.sqlQuery(
    """
        |select id, temperature
        |from inputTable
        |where id = 'sensor_1'
        |""".stripMargin
)
resultSQLTable.toAppendStream[(String, Double)].print("sql")
```

3）聚合查询

```scala
// 3.2 聚合转换
val aggTable = sensorTable
    .groupBy('id)    // 基于id分组
    .select('id, 'id.count as 'count)
aggTable.toRetractStream[Row].print("agg")
```

```txt
备注：
	- toAppendStream：追加流，一个一个往后加
	- toRetractStream：撤回流，有Insert和Delete两类操作，得到的数据会增加一个Boolean类型的标识位（返回的第一个字段），来表示
	  到底是新增的数据（Insert），还是被删除的数据（老数据， Delete）
```

![image-20200811202228820](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200811202228820.png)

#### 2.4.2、更新模式

对于流式查询，需要声明如何在表和外部连接器之间执行转换。与外部系统交换的消息类型，由更新模式（Update Mode）指定

1）追加（Append）模式

```txt
表只做插入操作，和外部连接器只交换插入（Insert）消息
```

2）撤回（Retract）模式

```txt
- 表和外部连接器交换添加（Add）和撤回（Retract）消息
- 插入操作（Insert）编码为Add消息；删除（Delete）编码为Retract消息；更新（Update）编码为上一条的Retract和下一条的Add消息
```

3）更新插入（Upsert）模式

```txt
更新和插入都被编码为 Upsert 消息；删除编码为 Delete 消息
```



### 2.5、输出表

表的输出，是通过将数据写入 TableSink 来实现的。TableSink 是一个通用接口，可以支持不同的文件格式、存储数据库和消息队列。

具体实现，输出表最直接的方法，就是通过 Table.insertInto() 方法将一个 Table 写入注册过的 TableSink 中。

#### 2.5.1、输出到文件

```scala
// 1. 创建环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
val tableEnv = StreamTableEnvironment.create(env)

// 2. 连接外部系统，读取数据，注册表
val filePath = "input/sensor.txt"
tableEnv.connect(new FileSystem().path(filePath))
    .withFormat(new Csv())
    .withSchema(new Schema()
                .field("id", DataTypes.STRING())
                .field("timestamp", DataTypes.BIGINT())
                .field("temp", DataTypes.DOUBLE())
               )
    .createTemporaryTable("inputTable")

// 3. 转换操作
val sensorTable = tableEnv.from("inputTable")
val resultTable = sensorTable
    .select('id, 'temp)
    .filter('id === "sensor_1")

// 4. 输出到文件
// 注册输出表
val outputPath = "output/output.txt"
tableEnv.connect(new FileSystem().path(outputPath))
    .withFormat(new Csv())
    .withSchema(new Schema()
                .field("id", DataTypes.STRING())
                .field("temperature", DataTypes.DOUBLE())
                //        .field("cnt", DataTypes.BIGINT()) // count是关键字
               )
    .createTemporaryTable("outputTable")

resultTable.insertInto("outputTable")
env.execute()
```



#### 2.5.2、输出到Kafka

不支持Retract模式、Upsert模式

```scala
 // 1. 创建环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)

val tableEnv = StreamTableEnvironment.create(env)

// 2. 从kafka读取数据
tableEnv.connect(new Kafka()
                 .version("0.11")
                 .topic("sensor")
                 .property("zookeeper.connect", "hadoop102:2181")
                 .property("bootstrap.servers", "hadoop102:9092")
                )
    .withFormat(new Csv())
    .withSchema(new Schema()
                .field("id", DataTypes.STRING())
                .field("timestamp", DataTypes.BIGINT())
                .field("temperature", DataTypes.DOUBLE())
               )
    .createTemporaryTable("kafkaInputTable")

// 3. 查询转换
// 3.1 简单转换
val sensorTable = tableEnv.from("kafkaInputTable")
val resultTable = sensorTable
    .select('id, 'temperature)
    .filter('id === "sensor_1")

// 3.2 聚合转换
val aggTable = sensorTable
    .groupBy('id)    // 基于id分组
    .select('id, 'id.count as 'count)

// 4. 输出到kafka
tableEnv.connect(new Kafka()
                 .version("0.11")
                 .topic("sinktest")
                 .property("zookeeper.connect", "hadoop102:2181")
                 .property("bootstrap.servers", "hadoop102:9092")
                )
    .withFormat(new Csv())
    .withSchema(new Schema()
                .field("id", DataTypes.STRING())
                .field("temp", DataTypes.DOUBLE())
               )
    .createTemporaryTable("kafkaOutputTable")

resultTable.insertInto("kafkaOutputTable")

env.execute("kafka pipeline test")
```



#### 2.5.3、输出到ES

支持Retract模式、Upsert模式

```scala
// 1. 创建环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)

val tableEnv = StreamTableEnvironment.create(env)

// 2. 连接外部系统，读取数据，注册表
val filePath = "input/sensor.txt"

tableEnv.connect(new FileSystem().path(filePath))
    .withFormat(new Csv())
    .withSchema(new Schema()
                .field("id", DataTypes.STRING())
                .field("timestamp", DataTypes.BIGINT())
                .field("temp", DataTypes.DOUBLE())
               )
    .createTemporaryTable("inputTable")

// 3. 转换操作
val sensorTable = tableEnv.from("inputTable")
// 3.1 简单转换
val resultTable = sensorTable
    .select('id, 'temp)
    .filter('id === "sensor_1")

// 3.2 聚合转换
val aggTable = sensorTable
    .groupBy('id) // 基于id分组
    .select('id, 'id.count as 'count)

// 4. 输出到es
tableEnv.connect(new Elasticsearch()
                 .version("6")
                 .host("localhost", 9200, "http")
                 .index("sensor")
                 .documentType("temperature")
                )
    .inUpsertMode() // 配置更新模式
    .withFormat(new Json())
    .withSchema(new Schema()
                .field("id", DataTypes.STRING())
                .field("count", DataTypes.BIGINT())
               )
    .createTemporaryTable("esOutputTable")

aggTable.insertInto("esOutputTable")

env.execute("es output test")
```



#### 2.5.4、输出到Mysql

需要引入flink-jdbc

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-jdbc_2.12</artifactId>
    <version>1.10.1</version>
</dependency>
```

```scala
// 1. 创建环境
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)

val tableEnv = StreamTableEnvironment.create(env)

// 2. 连接外部系统，读取数据，注册表
val filePath = "input/sensor.txt"

tableEnv.connect(new FileSystem().path(filePath))
    .withFormat(new Csv())
    .withSchema(new Schema()
                .field("id", DataTypes.STRING())
                .field("timestamp", DataTypes.BIGINT())
                .field("temp", DataTypes.DOUBLE())
               )
    .createTemporaryTable("inputTable")

// 3. 转换操作
val sensorTable = tableEnv.from("inputTable")

// 聚合转换
val aggTable = sensorTable
    .groupBy('id) // 基于id分组
    .select('id, 'id.count as 'cnt)

// 4. 输出到 Mysql，先在Mysql建立相应的表结构再插入数据
// jdbcOutputTable实在flink执行环境中创建的表，是一个tablesink
// sensor_count才是mysql中的表
val sinkDDL: String =
	"""
        |create table jdbcOutputTable (
        |  id varchar(20) not null,
        |  cnt bigint not null
        |) with (
        |  'connector.type' = 'jdbc',
        |  'connector.url' = 'jdbc:mysql://hadoop102:3306/test',
        |  'connector.table' = 'sensor_count',
        |  'connector.driver' = 'com.mysql.jdbc.Driver',
        |  'connector.username' = 'root',
        |  'connector.password' = '123456'
        |)
      """.stripMargin

tableEnv.sqlUpdate(sinkDDL)
aggTable.insertInto("jdbcOutputTable")

env.execute("mysql output test")
```



### 2.6、将表转换成DataStream

Flink中表可以转换为DataStream或DataSet。这样，自定义流处理或批处理程序就可以继续在 Table API或SQL查询的结果上运行了。

```txt
- 将表转换为 DataStream 或 DataSet 时，需要指定生成的数据类型，即要将表的每一行转换成的数据类型；
- 表作为流式查询的结果，是动态更新的；
- 转换有两种转换模式：追加（Appende）模式和撤回（Retract）模式。
```

1）追加模式（Append Mode）

```txt
– 用于表只会被插入（Insert）操作更改的场景
```

```scala
val resultStream: DataStream[Row] = tableEnv.toAppendStream[Row](resultTable)
```

2）撤回模式（Retract Mode）

```txt
– 用于任何场景。有些类似于更新模式中 Retract 模式，它只有 Insert 和 Delete 两类操作。
– 得到的数据会增加一个Boolean类型的标识位（返回的第一个字段），用它来表示到底是新增的数据（Insert），还是被删除的数据（Delete）
```

```scala
val aggResultStream: DataStream[(Boolean, (String, Long))] = tableEnv
       .toRetractStream[(String, Long)](aggResultTable)
```

```txt
思考：为什么没有toUpsetStream方法,更新插入（Upsert）模式？
	- Upsert模式的更新和插入都是一条信息，在web系统中会根据key进行对比数据不一致这是更新操作，但流中无法判断更新还是插入比较麻烦，
	  flink自己没有实现相应的API
```



### 2.7、查看执行计划

Table API 提供了一种机制来解释计算表的逻辑和优化查询计划。查看执行计划，可以通过 TableEnvironment.explain(table) 方法或 TableEnvironment.explain() 方法完成，返回一个字符串，描述三个计划（类比SQL中explain）。

```txt
- 优化的逻辑查询计划；
- 优化后的逻辑查询计划；
- 实际执行计划。
```

```scala
val explaination: String = tableEnv.explain(resultTable)
println(explaination)
```



## 三、流处理中的特殊概念

### 3.1、流处理和关系代数的区别

![image-20200812102853339](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200812102853339.png)

```txt
注意：关系型数据库中的sql一般都是用来批处理的，表是静态表。而Flink中的表是动态表，查询是持续查询。
```



### 3.2、动态表

1）概念：

​		因为流处理面对的数据，是连续不断的，这和我们熟悉的关系型数据库中保存的 "表" 完全不同。把流数据转换成Table，然后执行类似于table的select操作，随着新数据的到来会不停更新。这样得到的表，在Flink Table API概念里，就叫做“**动态表**”（Dynamic Tables）。

```
- 动态表是 Flink 对流数据的 Table API 和 SQL 支持的核心概念
- 与表示批处理数据的静态表不同，动态表是随时间变化的
```

2）持续查询（Continuous Query）

```txt
- 动态表可以像静态的批处理表一样进行查询，查询一个动态表会产生持续查询（Continuous Query）
- 连续查询永远不会终止，并会生成另一个动态表
- 查询会不断更新其动态结果表，以反映其动态输入表上的更改
```



### 3.3、流式持续查询的过程

#### 3.3.1、流式持续查询总流程

![image-20200812104123525](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200812104123525.png)

流式持续查询的过程为：

- 流被转换为动态表。

- 对动态表计算持续查询，生成新的动态表。

```txt
注意：持续查询的过程中会不断地保存状态，这样新的动态表不用再去统计一遍所有数据，而是直接在状态地基础上进行计算
```

- 生成的动态表被转换回流。



#### 3.3.2、将流转换成动态表

```txt
- 为了处理带有关系查询的流，必须先将其转换为表
- 从概念上讲，流的每个数据记录，都被解释为对结果表的插入（Insert）修改操作
```

![image-20200812105257237](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200812105257237.png)

#### 3.3.3、持续查询

持续查询会在动态表上做计算处理，并作为结果生成新的动态表

![image-20200812105336293](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200812105336293.png)

#### 3.3.4、将动态表转换成 DataStream

与常规的数据库表一样，动态表可以通过插入（Insert）、更新（Update）和删除（Delete）更改，进行持续的修改。

```txt
- 仅追加（Append-only）流
- 撤回（Retract）流
- Upsert（更新插入）流
```

```txt
注意：”Upsert（更新插入）流" 和2.6中 ”表转换成DataStream中没有toUpsetStream“ 不一样。flink中没有实现相应的UpsetStream的API，所以toUpsetStream不行（转换成UpsetStream后接下来要调用UpsetStream的API）。但是Flink为了支持第三方的扩展，支持”Upsert（更新插入）流"，第三方Sink支持UpsetStream就可以使用”Upsert（更新插入）流"。
```

![image-20200812110546748](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200812110546748.png)

![image-20200812110558841](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200812110558841.png)



### 3.4、时间特性

基于时间的操作（比如Table API和SQL中窗口操作），需要定义相关的时间语义和时间数据来源的信息。所以，Table可以提供一个逻辑上的时间字段，用于在表处理程序中，指示时间和访问相应的时间戳。

```txt
- 时间属性，可以是每个表schema的一部分。一旦定义了时间属性，它就可以作为一个字段引用，并且可以在基于时间的操作中使用。
- 时间属性的行为类似于常规时间戳，可以访问，并且进行计算。
```

#### 3.4.1、处理时间

处理时间语义（Processing Time）下，允许表处理程序根据机器的本地时间生成结果。它是时间的最简单概念。它既不需要提取时间戳，也不需要生成watermark。

1）DataStream转化成Table时指定

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
//env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime)

val settings = EnvironmentSettings.newInstance()
    .useBlinkPlanner()
    .inStreamingMode()
    .build()
val tableEnv = StreamTableEnvironment.create(env, settings)

// 读取数据
val inputPath = "input/sensor.txt"
val inputStream = env.readTextFile(inputPath)
val dataStream: DataStream[SensorReading] = inputStream
    .map(data => {
        val dataArray = data.split(",")
        SensorReading(dataArray(0), dataArray(1).toLong, dataArray(2).toDouble)
    })

// 将 DataStream转换为 Table，并指定时间字段
val sensorTable = tableEnv.fromDataStream(dataStream, 'id, 'temperature, 'timestamp, 'pt.proctime)
sensorTable.printSchema()  // 打印表结构
sensorTable.toAppendStream[Row].print() // 按行打印

env.execute("time test")
```

输出结果：

```txt
root
 |-- id: STRING
 |-- temperature: DOUBLE
 |-- timestamp: BIGINT
 |-- pt: TIMESTAMP(3) *PROCTIME*
 
sensor_1,35.8,1547718199,2020-08-12T03:47:13.365
sensor_6,15.4,1547718201,2020-08-12T03:47:13.365
sensor_7,6.7,1547718202,2020-08-12T03:47:13.365
```

2）定义Table Schema时指定（会报错）

3）创建表的DDL中指定

```scala
// 读取数据
val sinkDDL: String =
	"""
        |create table inputTable (
        |  id varchar(20) not null,
        |  ts bigint,
        |  temperature double,
        |  pt AS PROCTIME()
        |) with (
        |  'connector.type' = 'filesystem',
        |  'connector.path' = 'file:///E:\\projoect\\flink-project\\input\\sensor.txt',
        |  'format.type' = 'csv'
        |)
  	""".stripMargin

tableEnv.sqlUpdate(sinkDDL) // 执行 DDL

val sensorTable = tableEnv.from("inputTable")
sensorTable.toAppendStream[Row].print()

```

#### 3.4.2、事件时间

事件时间语义（Event Time），允许表处理程序根据每个记录中包含的时间生成结果。这样即使在有乱序事件或者延迟事件时，也可以获得正确的结果。

```txt
- 处理无序事件，并区分流中的准时和迟到事件；
- 从事件数据中，提取时间戳，并用来推进事件时间的进展（watermark）。
```

1）DataStream转化成Table时指定

在 DataStream 转换成 Table，使用 .rowtime 可以定义事件时间属性

```scala
// 将 DataStream转换为 Table，并指定时间字段
val sensorTable = tableEnv.fromDataStream(dataStream, 'id, 'timestamp.rowtime, 'temperature)
// 或者，直接追加时间字段
val sensorTable = tableEnv.fromDataStream(dataStream, 'id, 'temperature, 'timestamp, 'rt.rowtime)
```

代码：

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
// 指定使用事件时间
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

val settings = EnvironmentSettings.newInstance()
    .useBlinkPlanner()
    .inStreamingMode()
    .build()

val tableEnv = StreamTableEnvironment.create(env, settings)

// 读取数据
val inputPath = "input/sensor.txt"
val inputStream = env.readTextFile(inputPath)
// 先转换成样例类类型（简单转换操作）
val dataStream = inputStream
    .map(data => {
        val arr = data.split(",")
        SensorReading(arr(0), arr(1).toLong, arr(2).toDouble)
    })
    .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[SensorReading](Time.seconds(1)) {
        // 要精确到毫秒
        override def extractTimestamp(element: SensorReading): Long = element.timestamp * 1000L
    })

val sensorTable = tableEnv.fromDataStream(dataStream, 'id, 'temperature, 'timestamp.rowtime as 'ts)
sensorTable.printSchema()
sensorTable.toAppendStream[Row].print()

env.execute("event time test")
```



2）定义 Table Schema 时指定

```scala
val inputPath = "input/sensor.txt"
tableEnv.connect(new FileSystem().path(inputPath)) // 定义到文件系统的连接
    .withFormat(new Csv()) // 定义以csv格式进行数据格式化
    .withSchema(new Schema()
                .field("id", DataTypes.STRING())
                .field("timestamp", DataTypes.BIGINT())
                .rowtime(new Rowtime()
                         .timestampsFromField("timestamp") // 从字段中提取时间戳
                         .watermarksPeriodicBounded(1000)    // watermark延迟1秒
                        )
                .field("temperature", DataTypes.DOUBLE())
               ) // 定义表结构
    .createTemporaryTable("inputTable") // 创建临时表


val sensorTable = tableEnv.from("inputTable")
sensorTable.printSchema()
sensorTable.toAppendStream[Row].print()
```



3）在创建表的 DDL 中定义

```scala
val sinkDDL: String =
	"""
        |
        |create table inputTable (
        |  id varchar(20) not null,
        |  ts bigint,
        |  temperature double,
        |  rt AS TO_TIMESTAMP( FROM_UNIXTIME(ts) ),
        |  watermark for rt as rt - interval '1' second
        |) with (
        |  'connector.type' = 'filesystem',
        |  'connector.path' = 'input/sensor.txt',
        |  'format.type' = 'csv'
        |)  
    """.stripMargin
tableEnv.sqlUpdate(sinkDDL)
val sensorTable = tableEnv.from("inputTable")
sensorTable.printSchema()
sensorTable.toAppendStream[Row].print()
```

