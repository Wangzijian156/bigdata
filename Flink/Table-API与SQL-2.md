# Table API 与SQL（二）

- [Table API 与SQL（二）](#table-api-与sql二)
  - [四、分组窗口](#四分组窗口)
    - [4.1、Group Windows](#41group-windows)
      - [4.1.1、滚动窗口](#411滚动窗口)
      - [4.1.2、滑动窗口](#412滑动窗口)
      - [4.1.3、会话窗口](#413会话窗口)
    - [4.2、Over Windows](#42over-windows)
      - [4.2.1、无界 Over Windows](#421无界-over-windows)
      - [4.2.2、有界 Over Windows](#422有界-over-windows)
    - [4.3、SQL中窗口的定义](#43sql中窗口的定义)
      - [4.3.1、SQL 中的 Group Windows](#431sql-中的-group-windows)
      - [4.3.2、SQL 中的 Over Windows](#432sql-中的-over-windows)
    - [4.4、代码实现](#44代码实现)
  - [五、函数](#五函数)
    - [5.1、内置函数](#51内置函数)
      - [5.1.1、比较函数](#511比较函数)
      - [5.1.2、逻辑函数](#512逻辑函数)
      - [5.1.3、算数函数](#513算数函数)
      - [5.1.4、字符串函数](#514字符串函数)
      - [5.1.5、时间函数](#515时间函数)
      - [5.1.6、聚合函数](#516聚合函数)
    - [5.2、用户自定义函数（UDF）](#52用户自定义函数udf)
      - [5.2.1、标量函数](#521标量函数)
      - [5.2.2、表函数](#522表函数)
      - [5.2.3、聚合函数](#523聚合函数)
      - [5.2.4、表聚合函数](#524表聚合函数)

## 四、分组窗口

分组窗口（Group Windows）会根据时间或行计数间隔，将行聚合到有限的组（Group）中，并对每个组的数据执行一次聚合函数。

```txt
注意：时间语义，要配合窗口操作才能发挥作用
```

在 Table API 和 SQL 中，主要有两种窗口

```txt
- Group Windows（分组窗口）
	- 根据时间或行计数间隔，将行聚合到有限的组（Group）中，并对每个组的数据执行一次聚合函数
- Over Windows
	- 针对每个输入行，计算相邻行范围内的聚合
```



### 4.1、Group Windows

1）Group Windows 是使用 window（w:GroupWindow）子句定义的，并且必须由as子句指定一个别名。

2）为了按窗口对表进行分组，窗口的别名必须在 group by 子句中，像常规的分组字段一样引用

例如：

```scala
val table = inputStream
    .window([w: GroupWindow] as 'w) // 定义窗口，别名为 w
    .groupBy('w, 'a)      // 按照字段 a和窗口 w分组
    .select('a, 'b.sum)  // 聚合
```

#### 4.1.1、滚动窗口

滚动窗口要用 Tumble 类来定义

```scala
// Tumbling Event-time Window
.window(Tumble over 10.minutes on 'rowtime as 'w)
// Tumbling Processing-time Window
.window(Tumble over 10.minutes on 'proctime as 'w)
// Tumbling Row-count Window
.window(Tumble over 10.rows on 'proctime as 'w)
```



#### 4.1.2、滑动窗口

滑动窗口要用 Slide 类来定义

```scala
// Sliding Event-time Window
.window(Slide over 10.minutes every 5.minutes on 'rowtime as 'w)
// Sliding Processing-time window 
.window(Slide over 10.minutes every 5.minutes on 'proctime as 'w)
// Sliding Row-count window
.window(Slide over 10.rows every 5.rows on 'proctime as 'w)
```



#### 4.1.3、会话窗口

会话窗口要用 Session 类来定义

```scala
// Session Event-time Window
.window(Session withGap 10.minutes on 'rowtime as 'w)
// Session Processing-time Window
.window(Session withGap 10.minutes on 'proctime as 'w)
```





### 4.2、Over Windows

Over window 聚合是标准 SQL 中已有的（over 子句），可以在查询的 SELECT 子句中定义。

```txt
- Over window 聚合，会针对每个输入行，计算相邻行范围内的聚合
- Over windows 使用 window（w:overwindows*）子句定义，并在 select（）方法中通过别名来引用
```

例如：

```scala
val table = input
    .window([w: OverWindow] as 'w)
    .select('a, 'b.sum over 'w, 'c.min over 'w)
```

#### 4.2.1、无界 Over Windows

可以在事件时间或处理时间，以及指定为时间间隔、或行计数的范围内，定义 Over windows（全量数据，到目前为止的所有数据）。

无界的 over window 是使用常量指定的。

```scala
// 无界的事件时间 over window
.window(Over partitionBy 'a orderBy 'rowtime preceding UNBOUNDED_RANGE as 'w)
//无界的处理时间 over window
.window(Over partitionBy 'a orderBy 'proctime preceding UNBOUNDED_RANGE as 'w)
// 无界的事件时间 Row-count over window
.window(Over partitionBy 'a orderBy 'rowtime preceding UNBOUNDED_ROW as 'w)
// 无界的处理时间 Row-count over window
.window(Over partitionBy 'a orderBy 'proctime preceding UNBOUNDED_ROW as 'w)
```

```txt
备注：UNBOUNDED_ROW与UNBOUNDED_RANGE差不多，无界都是全量数据
```



#### 4.2.2、有界 Over Windows

有界的 over window 是用间隔的大小指定的

```scala
// 有界的事件时间 over window，一分钟之内的所有数据聚合起来
.window(Over partitionBy 'a orderBy 'rowtime preceding 1.minutes as 'w)
// 有界的处理时间 over window
.window(Over partitionBy 'a orderBy 'proctime preceding 1.minutes as 'w)
// 有界的事件时间 Row-count over window
.window(Over partitionBy 'a orderBy 'rowtime preceding 10.rows as 'w)
// 有界的处理时间 Row-count over window
.window(Over partitionBy 'a orderBy 'proctime preceding 10.rows as 'w)
```



### 4.3、SQL中窗口的定义

#### 4.3.1、SQL 中的 Group Windows

Group Windows 定义在 SQL 查询的 Group By 子句中。

1）TUMBLE(time_attr, interval)

```scala
定义一个滚动窗口，第一个参数是时间字段，第二个参数是窗口长度
```

2）HOP(time_attr, interval, interval)

```scala
定义一个滑动窗口，第一个参数是时间字段，第二个参数是窗口滑动步长，第三个是窗口长度
```

3）SESSION(time_attr, interval)

```scala
定义一个会话窗口，第一个参数是时间字段，第二个参数是窗口间隔
```





#### 4.3.2、SQL 中的 Over Windows

用 Over 做窗口聚合时，所有聚合必须在同一窗口上定义，也就是说必须是相同的分区、排序和范围。

```txt
- 目前仅支持在当前行范围之前的窗口
- ORDER BY 必须在单一的时间属性上指定
```

```sql
SELECT COUNT(amount) OVER (
    PARTITION BY user
    ORDER BY proctime
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
FROM Orders
```



### 4.4、代码实现

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(1)
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
// 定义 Event Time
.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[SensorReading](Time.seconds(1)) {
    override def extractTimestamp(element: SensorReading): Long = element.timestamp * 1000L
})
// timestamp是关键字，防止冲突
val sensorTable = tableEnv.fromDataStream(dataStream, 'id, 'temperature, 'timestamp.rowtime as 'ts)

// Group Windows
// 1.1 table api
val resultTable = sensorTable
    .window(Tumble over 10.seconds on 'ts as 'tw)     // 每10秒统计一次，滚动时间窗口
    .groupBy('id, 'tw)
	// select 中可以使用聚合函数
    // 也可以访问窗口信息 'tw.end
    .select('id, 'id.count, 'temperature.avg, 'tw.end)

// 1.2 sql
tableEnv.createTemporaryView("sensor", sensorTable)
// tumble(ts, interval '10' second)因为是函数不能使用别名
val resultSqlTable = tableEnv.sqlQuery(
    """
        |select
        |  id,
        |  count(id),
        |  avg(temperature),
        |  tumble_end(ts, interval '10' second)
        |from sensor
        |group by
        |  id,
        |  tumble(ts, interval '10' second)
      """.stripMargin)

resultTable.toAppendStream[Row].print("result")
resultSqlTable.toRetractStream[Row].print("sql")

// 2. Over window：统计每个sensor每条数据，与之前两行数据的平均温度
// 2.1 table api
val overResultTable = sensorTable
	.window( Over partitionBy 'id orderBy 'ts preceding 2.rows as 'ow )
	.select('id, 'ts, 'id.count over 'ow, 'temperature.avg over 'ow)

// 2.2 sql
val overResultSqlTable = tableEnv.sqlQuery(
    """
        |select
        |  id,
        |  ts,
        |  count(id) over ow,
        |  avg(temperature) over ow
        |from sensor
        |window ow as (
        |  partition by id
        |  order by ts
        |  rows between 2 preceding and current row
        |)
      """.stripMargin)

// 转换成流打印输出
overResultTable.toAppendStream[Row].print("result")
overResultSqlTable.toRetractStream[Row].print("sql")
```



## 五、函数

### 5.1、内置函数

Flink Table API 和 SQL 为用户提供了一组用于数据转换的内置函数。SQL 中支持的很多函数，Table API 和 SQL 都已经做了实现。

#### 5.1.1、比较函数

SQL

```txt
-  value1 = value2
-  value1 > value2
```

对应的Table API

```txt
-  ANY1 === ANY2
-  ANY1 > ANY2
```

#### 5.1.2、逻辑函数

SQL：

```txt
- boolean1 OR boolean2  
- boolean IS FALSE
- NOT boolean
```

Table API：

```txt
- BOOLEAN1 || BOOLEAN2
- BOOLEAN.isFalse
- !BOOLEAN
```

```
注意：none.isFalse => false , !none => none
```

#### 5.1.3、算数函数

SQL：

```txt
- numeric1 + numeric2
- POWER(numeric1, numeric2)
```

•Table API：

```txt
- NUMERIC1 + NUMERIC2
- NUMERIC1.power(NUMERIC2)
```

#### 5.1.4、字符串函数

SQL：

```txt
- string1 || string2
- UPPER(string)
- CHAR_LENGTH(string)
```

Table API：

```txt
- STRING1 + STRING2
- STRING.upperCase()
- STRING.charLength()
```

#### 5.1.5、时间函数

SQL：

```txt
- DATE string
- TIMESTAMP string
- CURRENT_TIME
- INTERVAL string range
```

Table API：

```txt
- STRING.toDate
- STRING.toTimestamp
- currentTime()
- NUMERIC.days
- NUMERIC.minutes
```

#### 5.1.6、聚合函数

SQL：

```txt
- COUNT(*)
- SUM(expression)
- RANK()
- ROW_NUMBER()
```

Table API：

```txt
- FIELD.count
- FIELD.sum
- FIELD.sum0
```

```txt
备注：sum0是如果是null返回一个0，Table API目前还没有实现RANK()，ROW_NUMBER()
```



### 5.2、用户自定义函数（UDF）

用户定义函数（User-defined Functions，UDF）是一个重要的特性，它们显著地扩展了查询的表达能力。在大多数情况下，用户定义的函数必须先注册，然后才能在查询中使用。

```txt
备注：函数通过调用 registerFunction（）方法在 TableEnvironment 中注册。当用户定义的函数被注册时，它被插入到 
TableEnvironment的函数目录中，这样Table API 或 SQL 解析器就可以识别并正确地解释它。
```

#### 5.2.1、标量函数

标量函数（Scalar Functions）

- 用户定义的标量函数，可以将0、1或多个标量值，映射到新的标量值（类似map）。
- 为了定义标量函数，必须在 org.apache.flink.table.functions 中扩展基类Scalar Function，并实现（一个或多个）求值（eval）方
  法。
- 标量函数的行为由求值方法决定，求值方法必须公开声明并命名为 eval。

```scala
class HashCode( factor: Int ) extends ScalarFunction {
    def eval( s: String ): Int = {
    	s.hashCode * factor
    }
}
```

代码实例：

```scala
object ScalarFunctionTest {
  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

    val settings = EnvironmentSettings.newInstance()
      .useBlinkPlanner()
      .inStreamingMode()
      .build()
    val tableEnv = StreamTableEnvironment.create(env, settings)

    // 读取数据
    val inputPath = "input/sensor.txt"
    val inputStream = env.readTextFile(inputPath)
    //    val inputStream = env.socketTextStream("localhost", 7777)

    // 先转换成样例类类型（简单转换操作）
    val dataStream = inputStream
      .map(data => {
        val arr = data.split(",")
        SensorReading(arr(0), arr(1).toLong, arr(2).toDouble)
      })
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[SensorReading](Time.seconds(1)) {
        override def extractTimestamp(element: SensorReading): Long = element.timestamp * 1000L
      })

    val sensorTable = tableEnv.fromDataStream(dataStream, 'id, 'temperature, 'timestamp.rowtime as 'ts)

    // 调用自定义hash函数，对id进行hash运算
    // 1. table api
    // 首先new一个UDF的实例
    val hashCode = new HashCode(23)
    val resultTable = sensorTable
      .select('id, 'ts, hashCode('id))

    // 2. sql
    // 需要在环境中注册UDF
    tableEnv.createTemporaryView("sensor", sensorTable)
    tableEnv.registerFunction("hashCode", hashCode)
    val resultSqlTable = tableEnv.sqlQuery("select id, ts, hashCode(id) from sensor")

    resultTable.toAppendStream[Row].print("result")
    resultSqlTable.toAppendStream[Row].print("sql")

    env.execute("scalar function test")
  }
}

// 自定义标量函数
class HashCode( factor: Int ) extends ScalarFunction{
  def eval( s: String ): Int = {
    s.hashCode * factor - 10000
  }
}
```

#### 5.2.2、表函数

表函数（Table Functions）

- 用户定义的表函数，也可以将0、1或多个标量值作为输入参数；与标量函数不同的是，它可以返回任意数量的行作为输出，而不是单个值。

```txt
备注：一行数据转成多行（类似爆炸函数）
```

- 为了定义一个表函数，必须扩展 org.apache.flink.table.functions 中的基类 TableFunction 并实现（一个或多个）求值方法。
- 表函数的行为由其求值方法决定，求值方法必须是 public 的，并命名为 eval。

代码实例：

```scala
object TableFunctionTest {
  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)
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
        override def extractTimestamp(element: SensorReading): Long = element.timestamp * 1000L
      })

    val sensorTable = tableEnv.fromDataStream(dataStream, 'id, 'temperature, 'timestamp.rowtime as 'ts)

    // 1. table api
    val split = new Split("_")     // new一个UDF实例
    val resutTable = sensorTable
      // 类似sql中的join用法
      .joinLateral( split('id) as ('word, 'length))
      .select('id, 'ts, 'word, 'length)


    // 2. sql
    tableEnv.createTemporaryView("sensor", sensorTable)
    tableEnv.registerFunction("split", split)
    val resultSqlTable = tableEnv.sqlQuery(
      """
        |select
        |  id, ts, word, length
        |from
        |  sensor, lateral table( split(id) ) as splitid(word, length)
      """.stripMargin)

    resutTable.toAppendStream[Row].print("result")
    resultSqlTable.toAppendStream[Row].print("sql")

    env.execute("table function test")

  }
}

// 自定义TableFunction
class Split(separator: String) extends TableFunction[(String, Int)]{
  def eval(str: String): Unit ={
    str.split(separator).foreach(
      word => collect((word, word.length))
    )
  }
}
```



#### 5.2.3、聚合函数

用户自定义聚合函数（User-Defined Aggregate Functions，UDAGGs）可以把一个表中的数据，聚合成一个标量值。用户定义的聚合函数，是通过继承 AggregateFunction 抽象类实现的。（多对一）

![image-20200813093637838](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200813093637838.png)



1）AggregationFunction要求必须实现的方法：

```txt
– createAccumulator()
– accumulate()
– getValue()
```

2）AggregateFunction 的工作原理如下：

- 首先，它需要一个累加器（Accumulator），用来保存聚合中间结果的数据结构；可以通过调用 createAccumulator() 方法创建空累加器；

- 随后，对每个输入行调用函数的 accumulate() 方法来更新累加器；

- 处理完所有行后，将调用函数的 getValue() 方法来计算并返回最终结果。

3）代码

```scala
object AggregateFunctionTest {
  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

    val settings = EnvironmentSettings.newInstance()
      .useBlinkPlanner()
      .inStreamingMode()
      .build()
    val tableEnv = StreamTableEnvironment.create(env, settings)

    // 读取数据
    val inputPath = "input/sensor.txt"
    val inputStream = env.readTextFile(inputPath)
    //    val inputStream = env.socketTextStream("localhost", 7777)

    // 先转换成样例类类型（简单转换操作）
    val dataStream = inputStream
      .map(data => {
        val arr = data.split(",")
        SensorReading(arr(0), arr(1).toLong, arr(2).toDouble)
      })
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[SensorReading](Time.seconds(1)) {
        override def extractTimestamp(element: SensorReading): Long = element.timestamp * 1000L
      })

    val sensorTable = tableEnv.fromDataStream(dataStream, 'id, 'temperature, 'timestamp.rowtime as 'ts)

    // 创建一个聚合函数实例
    val avgTemp = new AvgTemp()

    // Table API的调用
    val resultTable = sensorTable.groupBy('id)
      .aggregate(avgTemp('temperature) as 'avgTemp)
      .select('id, 'avgTemp)

    // SQL的实现
    tableEnv.createTemporaryView("sensor", sensorTable)
    tableEnv.registerFunction("avgTemp", avgTemp)
    val resultSqlTable = tableEnv.sqlQuery(
      """
        |SELECT
        |id, avgTemp(temperature)
        |FROM
        |sensor
        |GROUP BY id
  """.stripMargin)

    // 转换成流打印输出
    // 做了聚合操作，就要使用撤回流toRetractStream
    resultTable.toRetractStream[(String, Double)].print("agg temp")
    resultSqlTable.toRetractStream[Row].print("agg temp sql")


    env.execute("table function test")
  }
}

// 定义一个类，专门用于表示聚合的状态
case class AvgRemAcc(var sum: Double, var count: Int) {
}


// 自定义TableFunction
class AvgTemp extends AggregateFunction[Double, AvgRemAcc] {
  override def getValue(accumulator: AvgRemAcc): Double = accumulator.sum / accumulator.sum

  override def createAccumulator(): AvgRemAcc = AvgRemAcc(0.0, 0)

  // 还要实现一个具体的处理计算函数，accumulate
  def accumulate(accumulator: AvgRemAcc, temperature: Double): Unit = {
    accumulator.sum += temperature
    accumulator.count += 1

  }
}
```



#### 5.2.4、表聚合函数

用户定义的表聚合函数（User-Defined Table Aggregate Functions，UDTAGGs），可以把一个表中数据，聚合为具有多行和多列的结果表。用户定义表聚合函数，是通过继承 TableAggregateFunction 抽象类来实现的。（做Top N）

![image-20200813102358349](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200813102358349.png)

代码：

```txt
表聚合函数的sql实现太麻烦了，暂时使用Table Api就行
```



```scala
object TableAggregateFunctionTest {
  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

    val settings = EnvironmentSettings.newInstance()
      .useBlinkPlanner()
      .inStreamingMode()
      .build()
    val tableEnv = StreamTableEnvironment.create(env, settings)

    // 读取数据
    val inputPath = "input/sensor.txt"
    val inputStream = env.readTextFile(inputPath)
    //    val inputStream = env.socketTextStream("localhost", 7777)

    // 先转换成样例类类型（简单转换操作）
    val dataStream = inputStream
      .map(data => {
        val arr = data.split(",")
        SensorReading(arr(0), arr(1).toLong, arr(2).toDouble)
      })
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[SensorReading](Time.seconds(1)) {
        override def extractTimestamp(element: SensorReading): Long = element.timestamp * 1000L
      })

    val sensorTable = tableEnv.fromDataStream(dataStream, 'id, 'temperature, 'timestamp.rowtime as 'ts)

    // 创建一个表聚合函数实例
    val top2Temp = new Top2Temp()

    // Table API的调用
    val resultTable = sensorTable.groupBy('id)
      .flatAggregate(top2Temp('temperature) as('temp, 'rank))
      .select('id, 'temp, 'rank)

    // 转换成流打印输出
    resultTable.toRetractStream[(String, Double, Int)].print("agg temp")


    env.execute("table function test")
  }
}

// 定义一个类，用来表示表聚合函数的状态
case class Top2TempAcc(var highestTemp: Double = Double.MinValue,
                       var secondHighestTemp: Double = Double.MinValue) {

}

// 自定义TableFunction。提取所有温度中最高的两个温度,输出二元组（temp, rank）
class Top2Temp extends TableAggregateFunction[(Double, Int), Top2TempAcc] {
  override def createAccumulator(): Top2TempAcc = Top2TempAcc()

  def accumulate(acc: Top2TempAcc, temp: Double) = {
    // 判断当前温度值，是否比状态中的值大
    if (temp > acc.highestTemp) {
      // 如果比最高温度还高，排在第一，原来的第一顺到第二
      acc.secondHighestTemp = acc.highestTemp
      acc.highestTemp = temp
    } else if (temp > acc.secondHighestTemp) {
      // 如果比最高温度和第二高温度之间，直接替换第二高的温度
      acc.secondHighestTemp = temp

    }
  }

  def emitValue(acc: Top2TempAcc, out: Collector[(Double, Int)]): Unit = {
    out.collect(acc.highestTemp, 1)
    out.collect(acc.secondHighestTemp, 2)
  }
 
}
```

