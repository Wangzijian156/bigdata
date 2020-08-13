# ClickHouse

- [ClickHouse](#clickhouse)
  - [一、概述](#一概述)
    - [1.1、简介](#11简介)
    - [1.2、列式存储](#12列式存储)
    - [1.3、DBMS的功能](#13dbms的功能)
    - [1.4、写操作](#14写操作)
    - [1.5、读操作](#15读操作)
      - [1.5.1、语句级多线程](#151语句级多线程)
      - [1.5.2、稀疏索引](#152稀疏索引)
    - [1.6、 生命周期管理](#16-生命周期管理)
  - [二、ClickHouse安装](#二clickhouse安装)
  - [三、数据类型](#三数据类型)
    - [3.1、整型](#31整型)
    - [3.2、浮点型](#32浮点型)
    - [3.3、布尔型](#33布尔型)
    - [3.4、Decimal型](#34decimal型)
    - [3.5、字符串](#35字符串)
    - [3.6、枚举型](#36枚举型)
    - [3.7、时间类型](#37时间类型)
    - [3.8、数组](#38数组)
  - [四、表引擎](#四表引擎)
    - [4.1、表引擎的使用](#41表引擎的使用)
    - [4.2 TinyLog](#42-tinylog)
    - [4.3  Memory](#43-memory)
    - [4.4 MergeTree](#44-mergetree)
      - [4.4.1 partition by 分区 （可选项）](#441-partition-by-分区-可选项)
      - [4.4.2 primary key主键(可选)](#442-primary-key主键可选)
      - [4.4.3 order by （必选）](#443-order-by-必选)
      - [4.4.4 二级索引](#444-二级索引)
      - [4.4.5 数据TTL](#445-数据ttl)
    - [4.5 ReplacingMergeTree](#45-replacingmergetree)
    - [4.6 SummingMergeTree](#46-summingmergetree)
  - [五、SQL操作](#五sql操作)
    - [5.1、Insert](#51insert)
    - [5.2、Update 和 Delete](#52update-和-delete)
    - [5.3、查询操作](#53查询操作)
    - [5.4、alter操作](#54alter操作)
    - [5.5 导出数据](#55-导出数据)
  - [六、副本](#六副本)
  - [七、集群](#七集群)
    - [7.1、分片](#71分片)
    - [7.2、读写原理](#72读写原理)
    - [7.3、如何配置](#73如何配置)

## 一、概述

### 1.1、简介

ClickHouse 是俄罗斯的Yandex于2016年开源的列式存储数据库（DBMS），使用C++语言编写，主要用于在线分析处理查询（OLAP），能够使用SQL查询实时生成分析数据报告。 



### 1.2、列式存储

以下面的表为例：
 
| Id   | Name | Age  |
| ---- | ---- | ---- |
| 1    | 张三 | 18   |
| 2    | 李四 | 22   |
| 3    | 王五 | 34   |

采用行式存储时，数据在磁盘上的组织结构为：

![image-20200728212427086](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200728212427086.png)

好处是想查某个人所有的属性时，可以通过一次磁盘查找加顺序读取就可以。但是当想查所有人的年龄时，需要不停的查找，或者全表扫描才行，遍历的很多数据都是不需要



### 1.3、DBMS的功能

几乎覆盖了标准SQL的大部分语法，包括 DDL和 DML ,以及配套的各种函数。

用户管理及权限管理

数据的备份与恢复



### 1.4、写操作

ClickHouse采用类LSM Tree的结构，数据写入后定期在后台合并。通过类LSM tree的结构，但是没有内存表，没有预写日志，ClickHouse在数据导入时全部是顺序append写入磁盘，在后台周期性合并数据到主数据段。

![image-20200728213509234](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200728213509234.png)

特点：

1）临时分区中也会进行排序（值排序临时分区中的数据）；

2）每次写入都会产生一个临时分区，写入频次不能特别高（临时分区过多也会增加服务器的负担），尽量按批次写入；

 3）不支持常规意义的修改行和删除行数据（olap数据库布怎么修改数据，更多的是查询分析）；

 3）不支持事务。



### 1.5、读操作

#### 1.5.1、语句级多线程

```txt
redis 		单线程 
mysql 		多线程	  语句单线程		 高dps，少量数据，复杂度低
clickhouse 	多线程	  单语句多线程	低dps，大量数据，复杂度高
```

ClickHouse将数据划分为多个partition，每个partition再进一步划分为多个index granularity，然后通过多个CPU核心分别处理其中的一部分来实现并行数据处理。在这种设计下，单条Query就能利用整机所有CPU。极致的并行处理能力，极大的降低了查询延时。

**弊端：**

clickhouse即使对于大量数据的查询也能够化整为零平行处理。但是有一个弊端就是对于单条查询使用多cpu，就不利于同时并发多条查询。所以对于高qps（每秒查询）的查询业务，clickhouse并不是强项。



#### 1.5.2、稀疏索引

1）每个稀疏索引条目都包含一个搜索键值和一个指向具有该搜索键值的第一个数据记录的指针。

2）要定位一个记录，需要找到包含小于等于待查找的搜索键值的最大的搜索键值的索引条目。从该索引条目指向的记录开始，沿着文件中的指针开始遍历，直到找到所需记录。

![image-20200728215209539](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200728215209539.png)

例如：查找`ID=22222`的记录
首先，找到小于等于`ID=22222`的最大索引条目：`ID=10101`的索引条目；
然后，从`ID=10101`的索引条目中记录的指针指向的记录`ID=10101`开始沿着文件里的指针遍历，直到找到`ID=22222`的记录；



clickhouse使用稀疏索引，索引之间的颗粒度（连续两个稀疏索引之间相隔，默认8192行）。使用稀疏索引范围查询块，点对点查询一般。



### 1.6、 生命周期管理

支持对数据的生存周期进行管理，可以像Redis那样失效掉过期的数据，维持数据的新陈代谢。



## 二、ClickHouse安装

1）CentOS取消打开文件数限制

在/etc/security/limits.conf、/etc/security/limits.d/20-nproc.conf这2个文件的末尾加入一下内容：

```shell
[root@hadoop102 ~]$  vim /etc/security/limits.conf
# 在文件末尾添加：
* soft nofile 65536 
* hard nofile 65536 
* soft nproc 131072 
* hard nproc 131072
```

 2）CentOS取消SELINUX

修改/etc/selinux/config中的SELINUX=disabled后重启

```shell
[root@hadoop102 ~]# vim /etc/selinux/config
SELINUX=disabled
```

3）关闭防火墙

```shell
systemctl stop firewalld.service
systemctl status firewalld.service # 在下方出现disavtive（dead），表示关闭成功
```

4）安装依赖

```shell
yum install -y libtool
yum install -y *unixODBC*
```

5）rpm安装clickhouse（三台服务器都安装上）

```txt
[root@hadoop102 software]# ls
clickhouse/clickhouse-common-static-20.4.4.18-2.x86_64.rpm
clickhouse/clickhouse-common-static-dbg-20.4.4.18-2.x86_64.rpm
clickhouse/clickhouse-client-20.4.4.18-2.noarch.rpm
clickhouse/clickhouse-server-20.4.4.18-2.noarch.rpm
```

6）修改配置文件

```shell
sudo vim /etc/clickhouse-server/config.xml
```

把 **<listen_host>::</listen_host>** 的注解打开，这样的话才能让clickhouse被除本机以外的服务器访问

7）启动ClickServer

```txt
sudo systemctl start clickhouse-server
```

8）使用client连接server

```txt
clickhouse-client -m
```

9）关闭开机自启

```txt
sudo systemctl disable clickhouse-server
```





## 三、数据类型

### 3.1、整型

固定长度，包括有符号整型或无符号整型。

```txt
- 有符号整型：范围 -2^n-1 ~ 2^(n-1)-1
- 无符号整型：范围 0 ~ 2^(n-1)
```



### 3.2、浮点型

建议尽可能以整数形式存储数据。浮点型进行计算时可能引起四舍五入的误差

```shell
:) select 1-0.9
┌───────minus(1, 0.9)─┐
│ 0.09999999999999998 │
└─────────────────────┘
```

**使用场景：一般数据值比较小，不涉及大量的统计计算，精度要求不高的时候。比如保存商品的重量。**



### 3.3、布尔型

没有单独的类型来存储布尔值。可以使用 UInt8 类型，取值限制为 0 或 1



### 3.4、Decimal型

可在加、减和乘法运算过程中保持精度，对于除法，最低有效数字会被丢弃（不舍入）。

**使用场景：一般金额字段、汇率、利率等字段为了保证小数点精度，都使用Decimal进行存储。**



### 3.5、字符串

```txt
- String：字符串可以任意长度的。它可以包含任意的字节集，包含空字节。
- FixedString(N)：固定长度 N 的字符串，N 必须是严格的正自然数。
```

（注意：当服务端读取长度小于 N 的字符串时候，通过在字符串末尾添加空字节来达到 N 字节长度。 当服务端读取长度大于 N 的字符串时候，将返回错误消息。）



### 3.6、枚举型

Enum 保存 'string'= integer 的对应关系。包括 Enum8 和 Enum16 类型。

```txt
Enum8 用 'String'= Int8 对描述。
Enum16 用 'String'= Int16 对描述。
```

创建一个带有一个枚举 Enum8('hello' = 1, 'world' = 2) 类型的列：

```txt
CREATE TABLE t_enum
(
    x Enum8('hello' = 1, 'world' = 2)
)
ENGINE = TinyLog
```

这个 `x` 列只能存储类型定义中列出的值：`'hello'`或`'world'`。如果尝试保存任何其他值，ClickHouse 抛出异常。

```shell
:) INSERT INTO t_enum VALUES ('hello'), ('world'), ('hello')

INSERT INTO t_enum VALUES
Ok.
3 rows in set. Elapsed: 0.002 sec.

:) insert into t_enum values('a')

INSERT INTO t_enum VALUES

Exception on client:
Code: 49. DB::Exception: Unknown element 'a' for type Enum8('hello' = 1, 'world' = 2)
```

从表中查询数据时，ClickHouse 从 `Enum` 中输出字符串值。

```sql
SELECT * FROM t_enum

┌─x─────┐
│ hello │
│ world │
│ hello │
└───────┘
```

如果需要看到对应行的数值，则必须将 `Enum` 值转换为整数类型。

```sql
SELECT CAST(x, 'Int8') FROM t_enum

┌─CAST(x, 'Int8')─┐
│               1 │
│               2 │
│               1 │
└─────────────────┘
```

**使用场景：对一些状态、类型的字段算是一种空间优化，也算是一种数据约束。但是实际使用中往往因为一些数据内容的变化增加一定的维护成本，甚至是数据丢失问题。所以谨慎使用。**



### 3.7、时间类型

目前clickhouse 有三种时间类型

Date 接受 **年-月-日** 的字符串比如 ‘2019-12-16’

Datetime 接受 **年-月-日 时:分:秒** 的字符串比如 ‘2019-12-16 20:50:10’

Datetime64 接受 **年-月-日 时:分:秒.亚秒** 的字符串比如 ‘2019-12-16 20:50:10.66’



### 3.8、数组

**Array(T)**：由 T 类型元素组成的数组。

T 可以是任意类型，包含数组类型。 但不推荐使用多维数组，ClickHouse 对多维数组的支持有限。例如，不能在 MergeTree 表中存储多维数组。

可以使用array函数来创建数组：

```txt
array(T)
```

也可以使用方括号：

```txt
[]
```

创建数组案例：

```sql
:) SELECT array(1, 2) AS x, toTypeName(x)

SELECT
  [1, 2] AS x,
  toTypeName(x)

┌─x─────┬─toTypeName(array(1, 2))─┐

│ [1,2] │ Array(UInt8)      │

└───────┴─────────────────────────┘

1 rows in set. Elapsed: 0.002 sec.

 
:) SELECT [1, 2] AS x, toTypeName(x)

SELECT
  [1, 2] AS x,
  toTypeName(x)

┌─x─────┬─toTypeName([1, 2])─┐

│ [1,2] │ Array(UInt8)    │

└───────┴────────────────────┘
 
1 rows in set. Elapsed: 0.002 sec.
```



## 四、表引擎

### 4.1、表引擎的使用

表引擎是clickhouse的一大特色。可以说， 表引擎决定了如何存储标的数据。包括：

1）数据的存储方式和位置 

2）并发数据访问。

3）索引的使用。

4）是否可以执行多线程请求。

5）数据如何拷贝副本。

 

表引擎的使用方式就是必须显形在创建表时定义该表使用的引擎，以及引擎使用的相关参数。如：

```txt
create table t_tinylog ( id String, name String) engine=TinyLog;
```

**特别注意：引擎的名称大小写敏感**



### 4.2 TinyLog

 以列文件的形式保存在磁盘上，不支持索引，没有并发控制。一般保存少量数据的小表，生产环境上作用有限。可以用于平时练习测试用。



### 4.3  Memory

内存引擎，数据以未压缩的原始形式直接保存在内存当中，服务器重启数据就会消失。读写操作不会相互阻塞，不支持索引。简单查询下有非常非常高的性能表现（超过10G/s）。

一般用到它的地方不多，除了用来测试，就是在需要非常高的性能，同时数据量又不太大（上限大概 1 亿行）的场景。

 

### 4.4 MergeTree

Clickhouse 中最强大的表引擎当属 MergeTree （合并树）引擎及该系列（MergeTree）中的其他引擎。地位可以相当于innodb之于Mysql。 而且基于MergeTree，还衍生除了很多小弟，也是非常有特色的引擎。

建表语句

```txt
create table t_order_mt(
  id UInt32,
  sku_id String,
  total_amount Decimal(16,2),
  create_time Datetime
 ) engine =MergeTree
partition by toYYYYMMDD(create_time)
primary key (id)
order by (id,sku_id)


insert into t_order_mt
values(101,'sku_001',1000.00,'2020-06-01 12:00:00') ,
(102,'sku_002',2000.00,'2020-06-01 11:00:00'),
(102,'sku_004',2500.00,'2020-06-01 12:00:00'),
(102,'sku_002',2000.00,'2020-06-01 13:00:00')
(102,'sku_002',12000.00,'2020-06-01 13:00:00')
(102,'sku_002',600.00,'2020-06-02 12:00:00')
```

 MergeTree其实还有很多参数(绝大多数用默认值即可)，但是三个参数是更加重要的，也涉及了关于MergeTree的很多概念。



#### 4.4.1 partition by 分区 （可选项）

作用： 学过hive的应该都不陌生，分区的目的主要是降低扫描的范围，优化查询速度。

如果不填：只会使用一个分区。

分区目录：MergeTree 是以列文件+索引文件+表定义文件组成的，但是如果设定了分区那么这些文件就会保存到不同的分区目录中。

并行：分区后，面对涉及跨分区的查询统计，clickhouse会以分区为单位并行处理。



**数据写入与分区合并：**

任何一个批次的数据写入都会产生一个临时分区，不会纳入任何一个已有的分区。写入后的某个时刻（大概10-15分钟后），clickhouse会自动执行合并操作（等不及也可以手动通过optimize执行），把临时分区的数据，合并到已有分区中。

```txt
optimize table xxxx [final]
```

手动触发合并，除了合并分区还有很多别的事件会触发。

加入final选项，保证即使数据已经合并完成，也会强行合并（主要是可以保证触发其他事件）。否则的话，如果数据已经合并完成，则不会合并，也不会触发其他事件。

 

#### 4.4.2 primary key主键(可选)

clickhouse中的主键，和其他数据库不太一样，**它只提供了数据的一级索引，但是却不是唯一约束**。这就意味着是可以存在相同primary key的数据的。

主键的设定主要依据是查询语句中的where 条件。

根据条件通过对主键进行某种形式的二分查找，能够定位到对应的index granularity,避免了全表扫描。

index granularity： 直接翻译的话就是索引粒度，指在稀疏索引中两个相邻索引对应数据的间隔。clickhouse中的MergeTree默认是8192。官方不建议修改这个值，除非该列存在大量重复值，比如在一个分区中几万行才有一个不同数据。

稀疏索引：稀疏索引的好处就是可以用很少的索引数据，定位更多的数据，代价就是只能定位到索引粒度的第一行，然后再进行进行一点扫描。

 

 

#### 4.4.3 order by （必选）

order by 设定了**分区内**的数据按照哪些字段顺序进行有序保存。

order by是MergeTree中唯一一个必填项，甚至比primary key 还重要，因为当用户不设置主键的情况，很多处理会依照order by的字段进行处理（比如后面会讲的去重和汇总）。

**要求：主键必须是order by字段的前缀字段。**

比如order by 字段是 (id,sku_id) 那么主键必须是id 或者(id,sku_id)

 

#### 4.4.4 二级索引

 目前在clickhouse的官网上二级索引的功能是被标注为**实验性**的。

所以使用二级索引前需要增加设置·

```sql
set allow_experimental_data_skipping_indices=1;
```

```sql
create table t_order_mt2(
id UInt32,
sku_id String,
total_amount Decimal(16,2),
create_time Datetime,
INDEX a total_amount TYPE minmax GRANULARITY 5
) engine =MergeTree
partition by toYYYYMMDD(create_time)
primary key (id)
order by (id, sku_id)
```

其中**GRANULARITY N** 是设定二级索引对于一级索引粒度的粒度。

那么在使用下面语句进行测试，可以看出二级索引能够为非主键字段的查询发挥作用。

```shell
[bigdata@hdp1 t_order_mt]$ clickhouse-client --send_logs_level=trace <<< 'select * from test1.t_order_mt where total_amount > toDecimal32(900., 2)'
```

 

#### 4.4.5 数据TTL

 

TTL即Time To Live，MergeTree提供了可以管理数据或者列的生命周期的功能。

**必须靠触发合并操作才能实现数据的时效。**

1）列级别TTL

```sql
create table t_order_mt3(
id UInt32,
sku_id String,
total_amount Decimal(16,2) TTL create_time+interval 10 SECOND,
create_time Datetime 
) engine =MergeTree
partition by toYYYYMMDD(create_time)
primary key (id)
order by (id, sku_id)
```

插入数据

```sql
insert into t_order_mt3
values(106,'sku_001',1000.00,'2020-06-12 22:52:30') ,
(107,'sku_002',2000.00,'2020-06-12 22:52:30'),
(110,'sku_003',600.00,'2020-06-13 12:00:00')
```

2）表级TTL：针对整张表

下面的这条语句是数据会在create_time 之后10秒丢失

```sq
alter table t_order_mt3 MODIFY TTL create_time + INTERVAL 10 SECOND;
```

涉及判断的字段必须是Date或者Datetime类型，推荐使用分区的日期字段。

能够使用的时间周期：

```txt
- SECOND
- MINUTE
- HOUR
- DAY
- WEEK
- MONTH
- QUARTER
- YEAR 
```



### 4.5 ReplacingMergeTree

ReplacingMergeTree是MergeTree的一个变种，它存储特性完全继承MergeTree，只是多了一个去重的功能。

尽管MergeTree可以设置主键，但是primary key其实没有唯一约束的功能。如果你想处理掉重复的数据，可以借助这个ReplacingMergeTree。

**去重时机**：数据的去重只会在合并的过程中出现。合并会在未知的时间在后台进行，所以你无法预先作出计划。有一些数据可能仍未被处理。

**去重范围**：如果表经过了分区，去重只会在分区内部进行去重，不能执行跨分区的去重。

 

所以ReplacingMergeTree能力有限， ReplacingMergeTree 适用于在后台清除重复的数据以节省空间，但是它不保证没有重复的数据出现。

```sql
create table t_order_rmt(
id UInt32,
sku_id String,
total_amount Decimal(16,2) ,
create_time Datetime 
) engine =ReplacingMergeTree(create_time)
partition by toYYYYMMDD(create_time)
primary key (id)
order by (id, sku_id)
```

 **ReplacingMergeTree()** **填入的参数为版本字段，重复数据保留版本字段值最大的。**

**如果不填版本字段，默认保留最后一条。**  

```sql
insert into t_order_rmt
values(101,'sku_001',1000.00,'2020-06-01 12:00:00') ,
(102,'sku_002',2000.00,'2020-06-01 11:00:00'),
(102,'sku_004',2500.00,'2020-06-01 12:00:00'),
(102,'sku_002',2000.00,'2020-06-01 13:00:00')
(102,'sku_002',12000.00,'2020-06-01 13:00:00')
(102,'sku_002',600.00,'2020-06-02 12:00:00')
```

```sql
SELECT *
FROM t_order_rmt
```

![image-20200729222951210](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200729222951210.png)

```sql
OPTIMIZE TABLE t_order_rmt FINAL
```

```sql
SELECT *
FROM t_order_rmt
```

![image-20200729223005537](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200729223005537.png)

通过测试得到结论：

```txt
- 实际上是使用order by 字段作为唯一键。
- 去重不能跨分区。
- 只有合并分区才会进行去重。
- 认定重复的数据保留，版本字段值最大的。
- 如果版本字段相同则保留最后一笔。
```

 

### 4.6 SummingMergeTree

 

对于不查询明细，只关心以维度进行汇总聚合结果的场景。如果只使用普通的MergeTree的话，无论是存储空间的开销，还是查询时临时聚合的开销都比较大。

Clickhouse 为了这种场景，提供了一种能够“预聚合”的引擎，SummingMergeTree.

表定义

```sql
create table t_order_smt(
id UInt32,
sku_id String,
total_amount Decimal(16,2) ,
create_time Datetime 
) engine =SummingMergeTree(total_amount)
partition by toYYYYMMDD(create_time)
primary key (id)
order by (id,sku_id )
```

 插入数据

```sql
insert into t_order_smt
values(101,'sku_001',1000.00,'2020-06-01 12:00:00') ,
(102,'sku_002',2000.00,'2020-06-01 11:00:00'),
(102,'sku_004',2500.00,'2020-06-01 12:00:00'),
(102,'sku_002',2000.00,'2020-06-01 13:00:00')
(102,'sku_002',12000.00,'2020-06-01 13:00:00')
(102,'sku_002',600.00,'2020-06-02 12:00:00')
```

![image-20200729223026346](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200729223026346.png)

```sql
optimize table t_order_smt final;
```

![image-20200729223045778](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200729223045778.png)

通过结果可以得到以下结论：

```txt
- 以SummingMergeTree（）中指定的列作为汇总数据列。可以填写多列必须数字列，如果不填，以所有非维度列且为数字	列的字段为汇总数据列。
- 以order by 的列为准，作为维度列。
- 其他的列保留第一行。
- 不在一个分区的数据不会被聚合。
```

 

设计聚合表的话，唯一键值、流水号可以去掉，所有字段全部是维度、度量或者时间戳。

能不能直接 select total_amount from province_name=’’ and create_date=’xxx’ 来得到汇总值？

不行，可能会包含一些还没来得及聚合的临时明细

select sum(total_amount) from province_name=’’ and create_date=’xxx’

即使使用SummingMergeTree 引擎也要手工进行sum， 聚合的效率肯定远远高于没有预聚合数据库或者其他引擎。

SummingMergeTree 是非幂等的。



##  五、SQL操作

### 5.1、Insert

基本与标准SQL（MySQL）基本一致

包括标准 insert into [table_name] values(…),(….) 



### 5.2、Update 和 Delete

ClickHouse提供了Delete 和Update的能力，这类操作被称为Mutation查询，它可以看做Alter 的一种。

虽然可以实现修改和删除，但是和一般的OLTP数据库不一样，**Mutation**语句是一种很“重”的操作，而且不支持事务。

“重”的原因主要是每次修改或者删除都会导致放弃目标数据的原有分区，重建新分区。所以尽量做批量的变更，不要进行频繁小数据的操作。

删除操作

```sql
alter table t_order_smt delete where sku_id ='sku_001';
```

修改操作

```sql
alter table t_order_smt 
update total_amount=toDecimal32(2000.00,2) 
where id =102;
```

由于操作比较“重”，所以 Mutation语句分两步执行，同步执行的部分其实只是进行新增数据新增分区和并把旧分区打上逻辑上的失效标记。知道触发分区合并的时候，才会删除旧数据释放磁盘空间。



### 5.3、查询操作

clickhouse基本上与标准SQL 差别不大。

```
- 支持子查询
- 支持CTE(with 子句) 
- 支持各种JOIN， 但是JOIN操作无法使用缓存，所以即使是两次相同的JOIN语句，Clickhouse也会视为两条新SQL。
- 支持窗口函数。

- 不支持自定义函数。
-  GROUP BY 操作增加了 with rollup、with cube、with total 用来计算小计和总计。
```

模拟数据

```sql
insert into t_order_mt
values(101,'sku_001',1000.00,'2020-06-01 12:00:00') ,
(102,'sku_002',2000.00,'2020-06-01 12:00:00'),
(103,'sku_004',2500.00,'2020-06-01 12:00:00'),
(104,'sku_002',2000.00,'2020-06-01 12:00:00')
(105,'sku_003',600.00,'2020-06-02 12:00:00'),
(106,'sku_001',1000.00,'2020-06-04 12:00:00'),
(107,'sku_002',2000.00,'2020-06-04 12:00:00'),
(108,'sku_004',2500.00,'2020-06-04 12:00:00'),
(109,'sku_002',2000.00,'2020-06-04 12:00:00'),
(110,'sku_003',600.00,'2020-06-01 12:00:00')
```

1）with rollup : 从右至左去掉维度进行小计。

（group 1,2,3 上卷  [1,2]  [1]  [0]，0代表总计）

```sql
select id , sku_id,sum(total_amount) from t_order_mt group by id,sku_id with rollup;
```

![image-20200729224720714](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200729224720714.png)



2） with cube : 从右至左去掉维度进行小计，再从左至右去掉维度进行小计。

（group 1,2,3 上卷  [1,2]   [2,3]  [1,3]  [1]  [2]  [3]  [0]）

```sql
select id , sku_id,sum(total_amount) from  t_order_mt group by id,sku_id with cube;
```

![image-20200729224757817](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200729224757817.png)

3） with totals: 只计算合计

```sql
select id , sku_id,sum(total_amount) from  t_order_mt group by id,sku_id with totals;
```

![image-20200729224832171](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200729224832171.png)



### 5.4、alter操作

同mysql的修改字段基本一致，

新增字段

```sql
alter table tableName add column newcolname String after col1
```

修改字段类型

```sql
alter table tableName modify column newcolname String;
```

删除字段

```
alter table tableName drop column newcolname;
```

 

### 5.5 导出数据

```shell
clickhouse-client --query   "select toHour(create_time) hr ,count(*) from test1.order_wide where dt='2020-06-23' group by hr" --format CSVWithNames> ~/rs1.csv
```

 

## 六、副本

1、副本写入流程

![image-20200730155748352](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200730155748352.png)



2、配置如下

1）需要启动zookeeper集群 和另外一台clickhouse 服务器，另外一台clickhouse服务器的安装完全和第一台一直即可。

2）在两台服务器的/etc/clickhouse-server/config.d目录下创建一个名为metrika.xml的配置文件

```xml
<?xml version="1.0"?>
<yandex>
  <zookeeper-servers>
     <node index="1">
        <host>hadoop102</host>
        <port>2181</port>
     </node>
     <node index="2">
        <host>hadoop103</host>
        <port>2181</port>
     </node>
     <node index="3">
        <host>hadoop104</host>
        <port>2181</port>
     </node>

  </zookeeper-servers>
</yandex>
```

3）在 /etc/clickhouse-server/config.xml 中增加

```xml
<include_from>/etc/clickhouse-server/config.d/metrika.xml</include_from>
```

![image-20200730155953046](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200730155953046.png)

重启服务

```txt
sudo systemctl restart clickhouse-server
```

4）再两个服务器上分别启动ClickHouse客户端

A机器

```sql
create table rep_t_order_mt_0105 (
  id UInt32,
  sku_id String,
  total_amount Decimal(16,2),
  create_time Datetime
 ) engine =ReplicatedMergeTree('/clickhouse/tables/01/rep_t_order_mt_0105','rep_hdp1')
 partition by toYYYYMMDD(create_time)
  primary key (id)
  order by (id,sku_id);
```

B机器

```sql
 create table rep_t_order_mt_0105 (
    id UInt32,
    sku_id String,
    total_amount Decimal(16,2),
    create_time  Datetime
 ) engine =ReplicatedMergeTree('/clickhouse/tables/01/rep_t_order_mt_0105','rep_hdp2')
 partition by toYYYYMMDD(create_time)
   primary key (id)
   order by (id,sku_id);
```

参数解释

ReplicatedMergeTree 中，第一参数是分片的zk_path，一般按照： /clickhouse/table/{shard}/{table_name} 的格式写，如果只有一个分片就写01即可。第二个参数是副本名称，相同的分片副本名称不能相同。



## 七、集群

### 7.1、分片

副本虽然能够提高数据的可用性，降低丢失风险，但是对数据的横向扩容没有解决。每台机子实际上必须容纳全量数据。

要解决数据水平切分的问题，需要引入分片的概念。通过分片把一份完整的数据进行切分，不同的分片分布到不同的节点上。在通过Distributed表引擎把数据拼接起来一同使用。

Distributed表引擎本身不存储数据，有点类似于MyCat之于MySql，成为一种中间件，通过分布式逻辑表来写入、分发、路由来操作多台节点不同分片的分布式数据。



### 7.2、读写原理

![image-20200730184755966](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200730184755966.png)

- distribute（假设安装再hdp1节点上）：

```
- 分布式表（逻辑表不存数据），会安装再某个节点上；
- 相当于一个代理人，客户端的请求会直接发送到distribute上；
```

- hdp1 ~ hdp6：本地表

模式一（没有副本）：

1）客户端发送请求

2）distribute接受到请求后，分析数据是存放在哪个节点上，如果是存放再自身节点（hdp1），直接将数据存入就行，如果是其余节点，则将数据存入缓存remote-shard-data-temp-partition中

3）distribute异步地将remote-shard-data-temp-partition中的数据发送给相应的节点，数据发送完后，删除缓存remote-shard-data-temp-partition



模式二（三主三从）：

1）模式一的流程

2）internal_replication=true，数据发送到主从中任意节点就行，之后主从之间会自动进行数据备份（发一次）。 internal_replication=false，数据会先发送给主，再发送给从（发两次，无法保证数据一致性，万一第二次发送失败）。

注意：官方推荐使用internal_replication=true



思考：数据发送到主从节点对，到底由谁去接受数据呢？

![image-20200730191840368](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200730191840368.png)



集群中每个节点都维护了一个errors_count，代表了发送失败的次数，errors_count越小表示该节点月稳定，当数据发送到主从节点对时，errors_count小的节点负责接收数据。如果errors_count相同的话由随机、顺序（配置问价中的顺序）、随机（优先第一顺位）、host名称近似等四种方式。



### 7.3、如何配置

集群架构如下（由于ClickHouse比较耗资源，无法向redis那样一个节点配置两个服务），hdp1和hdp2（为主从节点），hdp3负责分片。

![image-20200730192517672](https://gitee.com/wangzj6666666/bigdata-img/raw/master/clickhouse/image-20200730192517672.png)

1）修改配置文件

```txt
sudo vim  /etc/clickhouse-server/config.d/metrika.xml
```

```xml
<yandex>
    <clickhouse_remote_servers>
        <gmall_cluster> <!-- 集群名称-->
            <shard>         <!--集群的第一个分片-->
                <internal_replication>true</internal_replication>
                <replica>    <!--片的第一个副本-->
                    <host>hadoop102</host>
                    <port>9000</port>
                </replica>
                <replica>    <!--片的第二个副本-->
                    <host>hadop103</host>
                    <port>9000</port>
                </replica>
            </shard>

            <shard>  <!--集群的第二个分片-->
                <internal_replication>true</internal_replication>
                <replica>    <!--片的第一个副本-->
                    <host>hadoop104</host>
                    <port>9000</port>
                </replica>
            </shard>

        </gmall_cluster>
    </clickhouse_remote_servers>

    <zookeeper-servers>
        <node index="1">
            <host>hadoop102</host>
            <port>2181</port>
        </node>

        <node index="2">
            <host>hadoop103</host>
            <port>2181</port>
        </node>
        <node index="3">
            <host>hadoop104</host>
            <port>2181</port>
        </node>
    </zookeeper-servers>

    <macros>
        <shard>01</shard>   <!--机器放的分片数不一样-->
        <replica>rep_1_1</replica>  <!--机器放的副本数不一样-->
    </macros>

</yandex>
```

macros是宏，三个节点配置不一样，如下

| hdp1                                                         | hdp2                                                         | hdp3                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| <macros>  <shard>01</shard>   <replica>rep_1_1</replica>  </macros> | <macros>  <shard>01</shard>   <replica>rep_1_2</replica>  </macros> | <macros>  <shard>02</shard>   <replica>rep_2_1</replica>  </macros> |

 2）添加本地表（在任意节点上添加三个节点都会创建）

```sql
    create table st_order_mt_0213 on cluster gmall_cluster (
    id UInt32,
    sku_id String,
    total_amount Decimal(16,2),
    create_time  Datetime
 ) engine =ReplicatedMergeTree('/clickhouse/tables/{shard}/st_order_mt_0105','{replica}')
 partition by toYYYYMMDD(create_time)
   primary key (id)
   order by (id,sku_id);
```

3）添加分布式表（在任意节点上添加三个节点都会创建）

```sql
create table st_order_mt_0213_all on cluster gmall_cluster
(
    id UInt32,
    sku_id String,
    total_amount Decimal(16,2),
    create_time  Datetime
)engine = Distributed(gmall_cluster,test0213, st_order_mt_0213,hiveHash(sku_id));
```

4）测试数据

```sql
insert into  st_order_mt_0213_all 
values(201,'sku_001',1000.00,'2020-06-01 12:00:00') ,
(202,'sku_002',2000.00,'2020-06-01 12:00:00'),
(203,'sku_004',2500.00,'2020-06-01 12:00:00'),
(204,'sku_002',2000.00,'2020-06-01 12:00:00')
(205,'sku_003',600.00,'2020-06-02 12:00:00');
```



相关资料：https://www.jianshu.com/p/017fb664fa63