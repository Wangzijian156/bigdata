# Kafka总结

- [Kafka总结](#kafka总结)
  - [一、Kafka概述](#一kafka概述)
    - [1.1、Kafka是什么](#11kafka是什么)
    - [1.2、消息队列](#12消息队列)
    - [1.3、Kafka基础架构](#13kafka基础架构)
    - [1.4、Kafka的用途](#14kafka的用途)
  - [二、Kafka架构深入](#二kafka架构深入)
    - [2.1、Kakfa工作流程](#21kakfa工作流程)
    - [2.2、文件存储机制](#22文件存储机制)
    - [2.3、Kafka生产者](#23kafka生产者)
      - [2.3.1、分区策略](#231分区策略)
      - [2.3.2、数据可靠性保证](#232数据可靠性保证)
        - [1）ack机制](#1ack机制)
        - [2）ack应答机制](#2ack应答机制)
        - [3）ISR](#3isr)
        - [4）故障处理细节](#4故障处理细节)
      - [2.3.3、Exactly Once语义](#233exactly-once语义)
    - [2.4、Kafka消费者](#24kafka消费者)
      - [2.4.1、消费方式](#241消费方式)
      - [2.4.2、分区分配策略](#242分区分配策略)
      - [2.4.3、offset的维护](#243offset的维护)
  - [三、Kafka高效读写机制](#三kafka高效读写机制)
  - [四、Zookeeper在Kafka中的作用](#四zookeeper在kafka中的作用)
  - [五、Kafka事务](#五kafka事务)
  - [六、Kafka API](#六kafka-api)
    - [6.1、Producer API](#61producer-api)
      - [6.1.1、异步发送API](#611异步发送api)
      - [6.1.2、同步发送API](#612同步发送api)
    - [6.2、Consumer API](#62consumer-api)
      - [6.2.1、动提交offset](#621动提交offset)
      - [6.2.2、手动提交offset](#622手动提交offset)
        - [1）同步提交offset](#1同步提交offset)
      - [2）异步提交offset](#2异步提交offset)
  - [七、Kafka监控（Kafka Eagle）](#七kafka监控kafka-eagle)

## 一、Kafka概述

### 1.1、Kafka是什么

Kafka是一个分布式的基于发布/订阅模式的消息队列，主要应用于大数据实时处理领域。

### 1.2、消息队列

1）消息队列的应用

```txt
- 异步
- 消峰
- 解耦
```

2）消息队列的模式

- 点对点模式

```txt
- 一对一，消费者主动拉取数据，消息收到后消息清除;
- Queue支持存在多个消费者，但是对一个消息而言，只会有一个消费者可以消费;
- 消息被消费以后，queue中不再有存储，所以消息消费者不可能消费到已经被消费的消息。
```

![image-20200813165320606](https://gitee.com/wangzj6666666/bigdata-img/raw/master/kafka/image-20200813165320606.png)

- 发布 / 订阅模式

```txt
- 一对多，消费者消费数据之后不会清除消息;
- 消息生产者发布）将消息发布到topic中，同时有多个消息消费者订阅）消费该消息。和点对点方式不同，发布到topic的消息会被所有订阅者
  消费。
```

![image-20200813165512945](https://gitee.com/wangzj6666666/bigdata-img/raw/master/kafka/image-20200813165512945.png)

### 1.3、Kafka基础架构

![image-20200813165633157](https://gitee.com/wangzj6666666/bigdata-img/raw/master/kafka/image-20200813165633157.png)

1）Producer 

```txt
消息生产者，就是向kafka broker发消息的客户端；
```

2）Consumer 

```txt
消息消费者，向kafka broker取消息的客户端；
```

3）Consumer Group（CG）

```txt
消费者组，由多个consumer组成。消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个消费者消费；消费者组之间互不影响。
所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。
```

4）Broker (节点)

```txt
一台kafka服务器就是一个broker。一个集群由多个broker组成。一个broker可以容纳多个topic。
```

5）Topic 

```txt
可以理解为一个队列，生产者和消费者面向的都是一个topic。
```

6）Partition

```txt
为了实现扩展性，一个非常大的topic可以分布到多个broker即服务器）上，一个topic可以分为多个partition，
每个partition是一个有序的队列。
```

7）Replica

```txt
副本，为保证集群中的某个节点发生故障时，该节点上的partition数据不丢失，且kafka仍然能够继续工作，kafka提供了副本机制，
一个topic的每个分区都有若干个副本，一个leader和若干个follower。
```

8）leader

```txt
每个分区多个副本的“主”，生产者发送数据的对象，以及消费者消费数据的对象都是leader。
```

9）follower

```txt
每个分区多个副本中的“从”，实时从leader中同步数据，保持和leader数据的同步。leader发生故障时，某个follower会成为新的leader。
```



### 1.4、Kafka的用途

1）日志收集

```txt
一个公司可以用Kafka可以收集各种服务的log，通过kafka以统一接口服务的方式开放给各种consumer，例如hadoop、Hbase、Solr等
```

2）消息系统

```txt
解耦和生产者和消费者、缓存消息等。
```

3）用户活动跟踪

```txt
Kafka经常被用来记录web用户或者app用户的各种活动，如浏览网页、搜索、点击等活动，这些活动信息被各个服务器发布到kafka的topic中，
然后订阅者通过订阅这些topic来做实时的监控分析，或者装载到hadoop、数据仓库中做离线分析和挖掘。
```

4）运营指标

```
Kafka也经常用来记录运营监控数据。包括收集各种分布式应用的数据，生产各种操作的集中反馈，比如报警和报告。
```

5）流式处理

```txt
比如spark streaming和storm。
```

6）事件源



## 二、Kafka架构深入

### 2.1、Kakfa工作流程

![image-20200816094022080](https://gitee.com/wangzj6666666/bigdata-img/raw/master/kafka/image-20200816094022080.png)



1）Producer负责生成数据并指明存储到哪个Topic中，与此同时也会指明存储在哪个Partiton（Leader）上（hash值取模），Producer按批发送数据。

```txt
备注：
	- Producer发送数据是按批发送的（连接Kafka也会消耗性能，如果数据以一条一条的形式传输，那么连接Kafka所要的性能会大幅度提升）。
	- 每批数据发送完毕会记录一个Offset，下批数据来时直接在Offset的位置往下写（大幅度提高写操作的效率）。
	- 因为有Offset，所以数据在Partion上是有序的，但在Partition之间是无序的。
```

2）Kafka集群上（假设集群节点数为3）会将一个Topic分成3个Partition（Leader），分别存储在各个节点上，于此同时每个Partition也会有 一个备份（Follower）存储在除本身节点外的任一节点。

```txt
备注：
	- 不手动创建Topic会自动生成Topic（默认创建的Topic是一个分区，一个副本）。
	- Partiton发送数据到Leader中，Follower会主动去Leader上同步数据。
```

3）消费者组负责从Topic中消费数据

```txt
备注：
	- 消费者中Partion（Leader）中读取数据，而不是Follower。
```



### 2.2、文件存储机制

![image-20200816120954084](https://gitee.com/wangzj6666666/bigdata-img/raw/master/kafka/image-20200816120954084.png)

1）topic是逻辑上的概念，而partition是物理上的概念，每个partition对应于一个log文件，该log文件中存储的就是producer生产的数据。

2）Producer生产的数据会被不断追加到该log文件末端，且每条数据都有自己的offset。

```txt
备注：
	- 消费者组中的每个消费者，都会实时记录自己消费到了哪个offset，以便出错恢复时，从上次的位置继续消费。
	- 磁盘的顺序写入速度很快，按顺序写入可以提高写操作的速度。
```

3）由于消息会不断追加到log文件末尾，为防止log文件过大导致数据定位效率低下，Kafka采取了分片和索引机制，将每个partition分为多个segment。每个segment对应两个文件——“.index”文件和“.log”文件。

```txt
备注：
	- Kafka的保存数据时间默认为7天
	- log默认超过1G就会切分。
```

![image-20200816122700150](https://gitee.com/wangzj6666666/bigdata-img/raw/master/kafka/image-20200816122700150.png)



4）Segment的命名是以0开始，后续每个segment文件名为上一个全局partion的最大offset(偏移message数)。index文件和log文件的结构如下图所示。

![image-20200816123255565](https://gitee.com/wangzj6666666/bigdata-img/raw/master/kafka/image-20200816123255565.png)

```txt
备3意味着在该Index中它处于第三个的位置，756意味着（position）这个消息的物理偏移地址
```



### 2.3、Kafka生产者

#### 2.3.1、分区策略

1）为什么要分区

```txt
- 方便在集群中扩展，Partition可以调整适应机器，从而适应任意大小的数据；
- 可以提高并发，以Partition为单位读写。
```

2）如何分区

- 指明Partition情况下，直接使用指明的值；
- 未指明但有key的情况下，keyhash和topic的partition数取余；
- round-robin算法：既没有Partition有没有key时，随机生成整数自增与与topic的partition取余。

#### 2.3.2、数据可靠性保证

##### 1）ack机制

topic的每个partition收到producer发送的数据后，都向producer发送ack，收到则继续发送，未收到则重新发送。

```txt
思考：
	1）什么时候发送ack
		- 确保Follower和Leader同步完成再发送，这样再Leader挂掉后可以根据Follwoer选出新的Leader。 
	2）多少个Follower同步完成后发送Aack
```

副本数据同步策略

| 方案                        | 优点                                               | 缺点                                                |
| --------------------------- | -------------------------------------------------- | --------------------------------------------------- |
| 半数以上完成同步，就发送ack | 延迟低                                             | 选举新的leader时，容忍n台节点的故障，需要2n+1个副本 |
| 全部完成同步，才发送ack     | 选举新的leader时，容忍n台节点的故障，需要n+1个副本 | 延迟高                                              |

Kafka选择了第二种方案，原因如下：

- 同样为了容忍n台节点的故障，第一种方案需要2n+1个副本，而第二种方案只需要n+1个副本，而Kafka的每个分区都有大量的数据，第一种方案会造成大量数据的冗余。
- 虽然第二种方案的网络延迟会比较高，但网络延迟对Kafka的影响较小。



##### 2）ack应答机制

参数配置

- 0

```txt
Producer不等待Broker的ack，这一操作提供了一个最低的延迟，Broker一接收到还没有写入磁盘就已经返回，当Broker故障时有可能丢失数据。
```

- 1

```txt
Producer等待Broker的ack，Partition的leader落盘成功后返回ack，如果在Follower同步成功之前Leader故障，那么将会丢失数据
```

- -1

```txt
Producer等待Broker的ack，Partition的Leader和Follower全部落盘成功后才返回ack。但是如果在Follower同步完成后，Broker发送ack之前，Leader发生故障，那么会造成数据重复。
```

```txt
注意：Brokr就是Kafka集群中的每个节点。
```



##### 3）ISR

leader收到数据，所有follower都开始同步数据，但有一个follower，因为某种故障，迟迟不能与leader进行同步，那leader就要一直等下去，直到它完成同步，才能发送ack。这个问题怎么解决呢？

- Leader维护了一个动态的in-sync replica set (ISR)，意为和leader保持同步的follower集合。
- 当ISR中的follower完成数据的同步之后，leader就会给producer发送ack。
- 如果follower长时间未向leader同步数据，则该follower将被踢出ISR，该时间阈值由replica.lag.time.max.ms参数设定。
- Leader发生故障之后，就会从ISR中选举新的leader。



##### 4）故障处理细节

log文件中有 HW（High Watermark） 和 LEO（Log End Offset），用于处理当故障挂掉的Follower、Leader重新启动后的数据恢复。

1）follower故障：先踢出，恢复后读取leader的LEO，重新加入；

2）leader故障：ISR重新选举，各follower截掉高于HW的部分，从新leader同步数据。

```txt
注意：这只能保证副本之间的数据一致性，并不能保证数据不丢失或者不重复。
```



#### 2.3.3、Exactly Once语义

1）幂等性机制（idempotent）：无论发送几次，只落盘一次

2）幂等性机制 + at least once = exactly once

3）如何使用Exactly Once

```txt
- enable.idompotence设置为true
- acks设置为-1
```



### 2.4、Kafka消费者

#### 2.4.1、消费方式

消费者(Consumer)采用pull模式从broker中读取数据

```txt
备注:
	- 因为push很难适应消费速率不同的消费者，但pull不足在于如果kafka没有数据，consumer会返回空数据
	- 针对于上一点，kafka传入timeout，若无数据，consumer等待后再返回
```

​		

#### 2.4.2、分区分配策略

1）roundrobin：轮循消费者

2）range

```txt
- 总数做除法，前几分配给第一个consumer，以此类推。
- 按Topic分配，容易造成数据倾斜。
```

​		 

#### 2.4.3、offset的维护

```txt
- Kafka 0.9以前，consumer将offset保存在zookeeper中。
- Kafka0.9以后，保存在内置的topic中。
```


## 三、Kafka高效读写机制

1）顺序写磁盘，一直追加在log文件的末尾。

2）零复制技术

- Linux的分层

```txt
application
kernel
hardware
```

![image-20200816144152982](https://gitee.com/wangzj6666666/bigdata-img/raw/master/kafka/image-20200816144152982.png)

再Linux操作系统中，用户无法直接操作硬件层。比如要读取一个文件，必须要用一个App，将硬盘中的数据加载到PageCache中，再读取到App，最后通过App进行网络传输。

```txt
PageCache的好处
	- I/O Scheduler 会将连续的小块写组装成大块的物理写从而提高性能。
	- I/O Scheduler 会尝试将一些写操作重新按顺序排好，从而减少磁盘头的移动时间。
```

```txt
NIC：网络接口控制器
```

- 零拷贝

不用再将数据读取到App中了，PageCache可以直接将数据进行网路传输。

![image-20200816144252615](https://gitee.com/wangzj6666666/bigdata-img/raw/master/kafka/image-20200816144252615.png)





## 四、Zookeeper在Kafka中的作用

1）Kafka Controller

```txt
- 管理broker的上下线
- Topic的分区副本分配
- leader选举
```

2）Kafka Controller依赖于Zookeeper

3）Leader选举机制举例

```txt
- KafkaController监听Zookeeper
- broker故障
- Kafka Controller从Zookeeper获取ISR
- Kafka Controller更新leader及ISR在zookeeper上
```

4）Kafka Controller的形成机制-先到先得



## 五、Kafka事务

Kafka从0.11版本开始引入了事务支持。事务可以保证Kafka在Exactly Once语义的基础上，生产和消费可以跨分区和会话，要么全部成功，要么全部失败。



## 六、Kafka API

### 6.1、Producer API

Kafka的Producer发送消息采用的是异步发送的方式。在消息发送的过程中，涉及到了两个线程。main线程和Sender线程，以及一个线程共享变量 RecordAccumulator。

```txt
main线程将消息发送给RecordAccumulator，Sender线程不断从RecordAccumulator中拉取消息发送到Kafka broker。
```

![image-20200816192910761](https://gitee.com/wangzj6666666/bigdata-img/raw/master/kafka/image-20200816192910761.png)

batch.size：只有数据积累到batch.size之后，sender才会发送数据。

linger.ms：如果数据迟迟未达到batch.size，sender等待linger.time之后就会发送数据。

#### 6.1.1、异步发送API

1）导入依赖

```xml
<dependency>
	<groupId>org.apache.kafka</groupId>
	<artifactId>kafka-clients</artifactId>
	<version>2.4.1</version>
</dependency>
```

2）编写代码

需要用到的类：

```txt
KafkaConsumer：需要创建一个消费者对象，用来消费数据

ConsumerConfig：获取所需的一系列配置参数

ConsuemrRecord：每条数据都要封装成一个ConsumerRecord对象
```

为了使我们能够专注于自己的业务逻辑，Kafka提供了自动提交offset的功能。 

自动提交offset的相关参数：

```txt
enable.auto.commit：是否开启自动提交offset功能
auto.commit.interval.ms：自动提交offset的时间间隔
```

以下为自动提交offset的代码：

```java
package com.atguigu.kafka;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.Arrays;
import java.util.Properties;

public class CustomConsumer {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "hadoop102:9092");
        props.put("group.id", "test");
        props.put("enable.auto.commit", "true");
        props.put("auto.commit.interval.ms", "1000");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Arrays.asList("first"));
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(100);
            for (ConsumerRecord<String, String> record : records)
                System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
        }
    }
}

```

#### 6.1.2、同步发送API

同步发送的意思就是，一条消息发送之后，会阻塞当前线程，直至返回ack。

由于send方法返回的是一个Future对象，根据Futrue对象的特点，我们也可以实现同步发送的效果，只需在调用Future对象的get方发即可。

```java
package com.atguigu.kafka;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.util.Properties;
import java.util.concurrent.ExecutionException;

public class CustomProducer {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Properties props = new Properties();
        props.put("bootstrap.servers", "hadoop102:9092");//kafka集群，broker-list
        props.put("acks", "all");
        props.put("retries", 1);//重试次数
        props.put("batch.size", 16384);//批次大小
        props.put("linger.ms", 1);//等待时间
        props.put("buffer.memory", 33554432);//RecordAccumulator缓冲区大小
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        Producer<String, String> producer = new KafkaProducer<>(props);
        for (int i = 0; i < 100; i++) {
            producer.send(new ProducerRecord<String, String>("first", Integer.toString(i), Integer.toString(i))).get();
        }
        producer.close();
    }
}

```



### 6.2、Consumer API

Consumer消费数据时的可靠性是很容易保证的，因为数据在Kafka中是持久化的，故不用担心数据丢失问题。

由于consumer在消费过程中可能会出现断电宕机等故障，consumer恢复后，需要从故障前的位置的继续消费，所以consumer需要实时记录自己消费到了哪个offset，以便故障恢复后继续消费。

所以offset的维护是Consumer消费数据是必须考虑的问题。

#### 6.2.1、动提交offset

1）导入依赖

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>2.4.1</version>
</dependency>
```

2）编写代码

需要用到的类：

```txt
KafkaConsumer：需要创建一个消费者对象，用来消费数据
ConsumerConfig：获取所需的一系列配置参数
ConsuemrRecord：每条数据都要封装成一个ConsumerRecord对象
```

为了使我们能够专注于自己的业务逻辑，Kafka提供了自动提交offset的功能。 

自动提交offset的相关参数：

```txt
enable.auto.commit：是否开启自动提交offset功能
auto.commit.interval.ms：自动提交offset的时间间隔
```

以下为自动提交offset的代码：

```java
package com.atguigu.kafka;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.Arrays;
import java.util.Properties;

public class CustomConsumer {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "hadoop102:9092");
        props.put("group.id", "test");
        props.put("enable.auto.commit", "true");
        props.put("auto.commit.interval.ms", "1000");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Arrays.asList("first"));
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(100);
            for (ConsumerRecord<String, String> record : records)
                System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
        }
    }
}

```

#### 6.2.2、手动提交offset

手动提交offset的方法有两种：分别是commitSync（同步提交）和commitAsync（异步提交）。

两者的相同点是，都会将本次poll的一批数据最高的偏移量提交；不同点是，commitSync阻塞当前线程，一直到提交成功，并且会自动失败重试（由不可控因素导致，也会出现提交失败）；而commitAsync则没有失败重试机制，故有可能提交失败。

##### 1）同步提交offset

由于同步提交offset有失败重试机制，故更加可靠，

```java
package com.atguigu.kafka.consumer;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.Arrays;
import java.util.Properties;

/**
 * @author liubo
 */
public class CustomComsumer {

    public static void main(String[] args) {

        Properties props = new Properties();
        props.put("bootstrap.servers", "hadoop102:9092");//Kafka集群
        props.put("group.id", "test");//消费者组，只要group.id相同，就属于同一个消费者组
        props.put("enable.auto.commit", "false");//关闭自动提交offset
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Arrays.asList("first"));//消费者订阅主题

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(100);//消费者拉取数据
            for (ConsumerRecord<String, String> record : records) {
                System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
            }
            consumer.commitSync();//同步提交，当前线程会阻塞知道offset提交成功
        }
    }
}

```

#### 2）异步提交offset

虽然同步提交offset更可靠一些，但是由于其会阻塞当前线程，直到提交成功。因此吞吐量会收到很大的影响。因此更多的情况下，会选用异步提交offset的方式。

```java
package com.atguigu.kafka.consumer;

import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;

import java.util.Arrays;
import java.util.Map;
import java.util.Properties;

/**
 * @author liubo
 */
public class CustomConsumer {

    public static void main(String[] args) {

        Properties props = new Properties();
        props.put("bootstrap.servers", "hadoop102:9092");//Kafka集群
        props.put("group.id", "test");//消费者组，只要group.id相同，就属于同一个消费者组
        props.put("enable.auto.commit", "false");//关闭自动提交offset
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Arrays.asList("first"));//消费者订阅主题

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(100);//消费者拉取数据
            for (ConsumerRecord<String, String> record : records) {
                System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
            }
            consumer.commitAsync(new OffsetCommitCallback() {
                @Override
                public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, Exception exception) {
                    if (exception != null) {
                        System.err.println("Commit failed for" + offsets);
                    }
                }
            });//异步提交
        }
    }
}

```



## 七、Kafka监控（Kafka Eagle）

1）修改kafka启动命令

修改kafka-server-start.sh命令中

```shell
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
  export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
fi
```

为

```shell
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
  export KAFKA_HEAP_OPTS="-server -Xms2G -Xmx2G -XX:PermSize=128m -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:ParallelGCThreads=8 -XX:ConcGCThreads=5 -XX:InitiatingHeapOccupancyPercent=70"
  export JMX_PORT="9999"
  \#export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
fi
```

注意：修改之后在启动Kafka之前要分发之其他节点

2）上传压缩包kafka-eagle-bin-1.3.7.tar.gz到集群/opt/software目录

3）解压到本地

```shell
[atguigu@hadoop102 software]$ tar -zxvf kafka-eagle-bin-1.3.7.tar.gz
```

4）将kafka-eagle-web-1.3.7-bin.tar.gz解压至/opt/module

```shell
[atguigu@hadoop102 kafka-eagle-bin-1.3.7]$ tar -zxvf kafka-eagle-web-1.3.7-bin.tar.gz -C /opt/module/
```

5）修改名称

```shell
[atguigu@hadoop102 module]$ mv kafka-eagle-web-1.3.7/ eagle
```

7）给启动文件执行权限

```shell
[atguigu@hadoop102 eagle]$ cd bin/
[atguigu@hadoop102 bin]$ ll
总用量 12
-rw-r--r--. 1 atguigu atguigu 1848 8月 22 2017 ke.bat
-rw-r--r--. 1 atguigu atguigu 7190 7月 30 20:12 ke.sh
[atguigu@hadoop102 bin]$ chmod 777 ke.sh
```

8）修改配置文件

```properties
######################################
# multi zookeeper&kafka cluster list
######################################
kafka.eagle.zk.cluster.alias=cluster1
cluster1.zk.list=hadoop102:2181,hadoop103:2181,hadoop104:2181

######################################
# kafka offset storage
######################################
cluster1.kafka.eagle.offset.storage=kafka

######################################
# enable kafka metrics
######################################
kafka.eagle.metrics.charts=true
kafka.eagle.sql.fix.error=false

######################################
# kafka jdbc driver address
######################################
kafka.eagle.driver=com.mysql.jdbc.Driver
kafka.eagle.url=jdbc:mysql://hadoop102:3306/ke?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
kafka.eagle.username=root
kafka.eagle.password=000000
```

9）添加环境变量

```shell
export KE_HOME=/opt/module/eagle
export PATH=$PATH:$KE_HOME/bin
```

10）启动

```shell
[atguigu@hadoop102 eagle]$ bin/ke.sh start
... ...
... ...
*******************************************************************
* Kafka Eagle Service has started success.
* Welcome, Now you can visit 'http://192.168.9.102:8048/ke'
* Account:admin ,Password:123456
*******************************************************************
* <Usage> ke.sh [start|status|stop|restart|stats] </Usage>
* <Usage> https://www.kafka-eagle.org/ </Usage>
*******************************************************************
[atguigu@hadoop102 eagle]$
```

11）登录页面查看监控数据

http://192.168.9.102:8048/ke

![image-20200816193714633](https://gitee.com/wangzj6666666/bigdata-img/raw/master/kafka/image-20200816193714633.png)