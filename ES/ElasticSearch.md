# ElasticSearch

- [ElasticSearch](#elasticsearch)
  - [一、概述](#一概述)
    - [1.1、简介](#11简介)
    - [1.2、使用场景](#12使用场景)
    - [1.3、与其他数据存储进行比较](#13与其他数据存储进行比较)
    - [1.4、ElasticSearch的特性](#14elasticsearch的特性)
    - [1.5、倒排索引](#15倒排索引)
    - [1.6、lucene 倒排索引结构](#16lucene-倒排索引结构)
  - [二、ElasticSearch的安装](#二elasticsearch的安装)
    - [2.1、ES安装](#21es安装)
    - [2.2、Linux配置](#22linux配置)
    - [2.3、启动ES](#23启动es)
    - [2.4、Master选举机制](#24master选举机制)
    - [2.5、安装Kibana](#25安装kibana)
    - [2.7、ES、Kibana群起脚本](#27eskibana群起脚本)
    - [](#)
  

## 一、概述

### 1.1、简介

ElasticSearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。

### 1.2、使用场景

1）为用户提供按关键字查询的全文搜索功能；

2）实现企业海量数据的处理分析的解决方案。大数据领域的重要一份子，如著名的ELK框架（ElasticSearch、Logstach（l类似Flume收集日志）、Kibana）。

### 1.3、与其他数据存储进行比较

|               | Redis        | MySql                | ES                                                           | Hive                 | HBase                                                        |
| ------------- | ------------ | -------------------- | ------------------------------------------------------------ | -------------------- | ------------------------------------------------------------ |
| 容量/容量扩展 | 低           | 中                   | 较大                                                         | 海量                 | 海量                                                         |
| 查询时效性    | 极高         | 中等                 | 较高                                                         | 低                   | 中等                                                         |
| 查询灵活性    | 较差 k-v模式 | 非常好，<br/>支持sql | 较好，关联查询较弱，<br/>但是可以全文检索，<br/>DSL语言可以处理过滤、<br/>匹配、排序、聚合等各种操作 | 非常好，<br/>支持sql | 较差，主要靠rowkey, <br/> scan的话性能不行，<br/>或者建立二级索引  <br/>（加上Phoenix <br/>支持sql后中等） |
| 写入速度      | 极快         | 中等                 | 较快（异步写入）                                             | 慢                   | 较快  （异步写入）                                           |
| 一致性、事务  | 弱           | 强                   | 弱                                                           | 弱                   | 弱                                                           |

异步写入：数据先放到缓存中，之后再写入到磁盘（缺点数据一致性的问题，数据容易丢失）



### 1.4、ElasticSearch的特性

1）天然分别、天然集群

es 把数据分成多个shard，下图中的P0-P2，多个shard可以组成一份完整的数据，这些shard可以分布在集群中的各个机器节点中。随着数据的不断增加，集群可以增加多个分片，把多个分片放到多个机子上，已达到负载均衡，横向扩展。

![image-20200721110455624](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es//image-20200721110455624.png)

- 为什么要分片

```txt
数据不能和物理机直接建立关系，否则添加新节点个删除旧节点，就打破之前的存储规律很难保持数据的一致性，容易造成数
据丢失
```

- 分片后数据如何存储

```txt
- 数据存在share中，hash（key） % share数，同时每个share会至少产生一个备份,ElasticSearch会自动把
  share和备份分摊到各个节点上（share不会全都存在一个节点上）；

- 添加节点，ElasticSearch会执行rebalance操作，重新向各个节点分配share和备份文件
  
- 3s1r => 3个share每个share一个备份，最终是6（s + s * r => 3 = 3 * 1 = 6）文件；

- ElasticSearch高可用，3个节点，down一个节点，可以根据剩下的2给节点复原数据。
```

- 分片后读操作如何取数据

```
- ElasticSearch集群的节点中有一个Master，除了管理自己本身的数据外，还负责协调组合其他节点的数据
- 当一个请求过来时，Master会将请求分发到涉及该请求所需数据的share上
```

这种集群分片的机制造就了ElasticSearch强大的数据容量及运算扩展性。



2）天然索引

ES 所有数据都是默认进行索引的，这点和mysql正好相反，mysql是默认不加索引，要加索引必须特别说明，ES只有不加索引才需要说明。ES使用的是倒排索引和Mysql的B+Tree索引不同

这会造成两个问题：

```txt
- 写的压力比较大 （用异步来解决，宕机了就会丢数据）
- 会造成数据膨胀 （数据压缩，解压和解锁又增加了资源的消耗）
```



### 1.5、倒排索引

1）传统关系性数据库

![image-20200721115327815](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es//image-20200721115327815.png)

```txt
- 对于传统的关系性数据库对于关键词的查询，只能逐字逐行的匹配，性能非常差。
- 匹配方式不合理，比如搜索“小密手机” ，如果用like进行匹配， 根本匹配不到。但是考虑使用者的用户体验的话，
  除了完全匹配的记录，还应该显示一部分近似匹配的记录，至少应该匹配到“手机”。
```



2）倒排索引

倒排索引基于分词技术构建的。

举一个例子：红海行动，在数据库中保存的数据如下，现在搜索"红海导演"

![image-20200721121247696](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es//image-20200721121247696.png)

传统关系型数据库是搜不出来结果的，倒排索引是如何处理的？关键在于分词。

- 每个记录保存数据时，都不会直接存入数据库。系统先会对数据进行分词，然后以倒排索引结构保存。如下：

![image-20200721121632658](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es//image-20200721121632658.png)

- 然后等到用户搜索的时候，会把搜索的关键词也进行分词，会把"红海导演"分词分成：红海和导演两个词

- 先用"红海"进行匹配，得到id=1和id=2的记录编号，再用"导演"匹配可以迅速定位id为3的记录，最终123都能查出来。

- 如果搜索的关键词对应的ids有交集，比如"红海"（1，2）、"行动"（1，3）显然1号记录能匹配的次数更多。所以显示的时候以评分进行排序的话，1号记录会排到最前面。而2、3号记录也可以匹配到。





### 1.6、lucene 倒排索引结构

![image-20200721151702313](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es//image-20200721151702313.png)

可以看到 lucene 为倒排索引(Term Dictionary)部分又增加一层Term Index结构，用于快速定位，而这Term Index是缓存在内存中的，但mysql的B+tree不在内存中，所以整体来看ES速度更快，但同时也更消耗资源（内存、磁盘）。



## 二、ElasticSearch的安装

### 2.1、ES安装

1）、解压elasticsearch-6.6.0.tar.gz

```shell
tar -zxvf elasticsearch-6.6.0.tar.gz -C /opt/module/
```

2）修改配置文件/opt/module/elasticsearch-6.6.0/config/elasticsearch.yml

ElasticSearch的配置文件时yml格式的，修改yml配置的注意事项:

```txt
- 每行必须顶格，不能有空格
- “：”后面必须有一个空格
```

3）集群名称，同一集群名称必须相同

```shell
# ---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#
cluster.name: my-es
```

4）单个节点名称

```shell
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
node.name: node-1
```

5）网络部分 改为当前的ip地址 ，端口号保持默认9200就行

```shell
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#
network.host: hadoop102
#
# Set a custom port for HTTP:
#
http.port: 9200
```

6）把bootstrap自检程序关掉

```shell
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#
bootstrap.memory_lock: false
bootstrap.system_call_filter: false
```

7）自发现配置：新节点向集群报到的主机名(高可用)

```shell
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when new node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
discovery.zen.ping.unicast.hosts: ["hadoop102", "hadoop103"]
```

8）ES是用在Java虚拟机中运行的，虚拟机默认启动占用2G内存。但是如果是装在PC机学习用，实际用不了2个G。所以可以改小一点内存。

```shell
vim /opt/module/elasticsearch-6.6.0/config/jvm.options
```

```shell
# Xms represents the initial size of total heap space
# Xmx represents the maximum size of total heap space

-Xms256m
-Xmx256m
```

9）分发到hadoop103、hadoop104节点

```shell
xsync /opt/module/elasticsearch-6.6.0/
```

10）修改hadoop103、hadoop104的节点名称

hadoop103 

```txt
node.name: node-2
network.host: hadoop103
```

hadoop104 

```txt
node.name: node-3
network.host: hadoop104
```



注意：如果先配置号单机的，并且输入了数据，集群启动不了（和redis集群一样），需要把

/opt/module/elasticsearch-6.6.0/data下的nodes全部删除



### 2.2、Linux配置

1）为什么要修改linux配置？

默认elasticsearch是单机访问模式，就是只能自己访问自己。但是我们之后一定会设置成允许应用服务器通过网络方式访问。这时，elasticsearch就会因为嫌弃单机版的低端默认配置而报错，甚至无法启动。所以我们在这里就要把服务器的一些限制打开，能支持更多并发。



2）问题1：max file descriptors [4096] for elasticsearch process likely too low, increase to at least [65536] elasticsearch

原因：系统允许 Elasticsearch 打开的最大文件数需要修改成65536

解决：vi /etc/security/limits.conf

添加内容：

```shell
* soft nofile 65536
* hard nofile 131072
* soft nproc 2048
* hard nproc 65536
```

注意：“*” 不要省略掉

 

3）问题2：max number of threads [1024] for user [judy2] likely too low, increase to at least [4096] （CentOS7.x 不用改）

原因：允许最大进程数修该成4096

解决：vi /etc/security/limits.d/90-nproc.conf  

（CentOS7.x  vi /etc/security/limits.d/20-nproc.conf 已经是4096了）

修改如下内容：

```shell
* soft nproc 1024
```

\#修改为

```shell
* soft nproc 4096
```

 

4）问题3：max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144] （CentOS7.x 不用改）

原因：一个进程可以拥有的虚拟内存区域的数量。

解决： 

在  /etc/sysctl.conf 文件最后添加一行

```shell
vm.max_map_count=262144
```

即可永久修改

重启linux



### 2.3、启动ES

在hadoop102、hadoop103、hadoop104、下同时执行

```shell
/opt/module/elasticsearch-6.6.0/bin/elasticsearch
```

打开浏览器输入：http://hadoop102:9200/_cat/nodes?v

![image-20200721161157808](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es//image-20200721161157808.png)

如果启动未成功，请去查看相关日志

vim  /opt/module/elasticsearch6.6.0/logs/my-es.log



### 2.4、Master选举机制

选举出Master，节点会自动生成一个NodeId，谁的NodeId最小，就选谁时Master。当master宕机后会重新选举。



### 2.5、安装Kibana

1）解压缩

```shell
 tar -zxvf kibana-6.6.0-linux-x86_64.tar.gz  -C /opt/module/
```

![image-20200721162250449](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es//image-20200721162250449.png)

2）修改配置

```shell
vim /opt/module/kibana-6.6.0-linux-x86_64/config/kibana.yml
```

3）修改server.host

```shell
server.host: "0.0.0.0"
```

4）修改elasticsearch.hosts（Kibana是一个工具本身不存数据，必须配置es地址，类似Navicat访问MySql）

```shell
elasticsearch.hosts: ["http://hadoop102:9200"]
```

5）启动

```
/opt/module/kibana-6.6.0-linux-x86_64/bin/kibina
```

6）浏览器输入http://hadoop102:5601/

![image-20200721163259317](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es//image-20200721163259317.png)



### 2.7、ES、Kibana群起脚本

```shell
#!/bin/bash 
es_home=/opt/module/elasticsearch-6.6.0
kibana_home=/opt/module/kibana-6.6.0-linux-x86_64
case $1  in
 "start") {
  for i in hadoop102 hadoop103 hadoop104
  do
    ssh $i  "source /etc/profile;${es_home}/bin/elasticsearch >/dev/null 2>&1 &"
 
   done
   nohup ${kibana_home}/bin/kibana >/home/atguigu/kibana.log 2>&1 & 
};;
"stop") {
  for i in hadoop102 hadoop103 hadoop104
  do
      ssh $i "ps -ef|grep $es_home |grep -v grep|awk '{print \$2}'|xargs kill" >/dev/null 2>&1
  done
  ssh hadoop102 "ps -ef|grep $kibana_home |grep -v grep|awk '{print \$2}'|xargs kill" >/dev/null 2>&1
};;
esac
```





### 

