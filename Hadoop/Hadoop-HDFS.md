# Hadoop-HDFS

- [Hadoop-HDFS](#hadoop-hdfs)
	- [一、基本概念](#一基本概念)
		- [1.1、产生背景](#11产生背景)
		- [1.2、定义](#12定义)
		- [1.3、特点](#13特点)
		- [1.4、组成架构](#14组成架构)
	- [二、HDFS的读写流程](#二hdfs的读写流程)
		- [2.1、写操作流程](#21写操作流程)
		- [2.2、网络拓扑](#22网络拓扑)
		- [2.3、机架感知](#23机架感知)
		- [2.4、HDFS读数据流程](#24hdfs读数据流程)
	- [三、NN和2NN](#三nn和2nn)
		- [3.1、NN与2NN工作机制](#31nn与2nn工作机制)
		- [3.2、Fsimage和Edits解析](#32fsimage和edits解析)
		- [3.3、NameNode多目录配置](#33namenode多目录配置)
	- [四、DataNode](#四datanode)
		- [4.1、DataNode工作机制](#41datanode工作机制)
		- [4.2、数据完整性](#42数据完整性)
		- [4.3、掉线时限参数设置](#43掉线时限参数设置)
		- [4.4、服役新数据节点](#44服役新数据节点)
		- [4.5、退役旧数据节点](#45退役旧数据节点)
				- [4.5.1、白名单与黑名单](#451白名单与黑名单)
				- [4.5.2、配置白、黑名单](#452配置白黑名单)
				- [4.5.3、退役节点](#453退役节点)
		- [4.6、DataNode多目录配置](#46datanode多目录配置)
	- [五、小文件存档](#五小文件存档)
		- [5.1、小文件的弊端](#51小文件的弊端)
		- [5.2、解决小文件的方案](#52解决小文件的方案)
	- [六、纠删码](#六纠删码)
		- [6.1、为什么要引入纠删码](#61为什么要引入纠删码)
		- [6.2、纠删码](#62纠删码)





## 一、基本概念

### 1.1、产生背景

随着数据量越来越大，在一个OS存不下所有的数据，那么就分配到更多的OS管理的磁盘中，需要一种系统来管理多台机器上的文件，

这就是分布式文件管理系统。HDFS是分布式管理系统中的一种。



### 1.2、定义

​	分布式文件系统，适合一次写入多次读出的场景，不支持文件的修改。

### 1.3、特点

1）优点

```txt
- 高容错性（自动保存多个副本）
- 适合处理大数据
- 可构建在廉价的机器上（万元级别）
```

2）缺点

```txt
- 不适合低延时数据访问
- 无法高效的对大量小文件进行存储
- 不支持并发写入、文件随机修改
```

### 1.4、组成架构

 ![image-20200712221213208](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-hdfs/image-20200712221213208.png)

1）NameNode（leader，缩写NN）

```txt
- 管理HDFS的名称空间
- 配置副本策略
- 管理数据块（Block）的映射信息
- 处理客户端Client读写请求
```

2）DataNode（slave，缩写DN，NameNode下达命令，DataNode执行实际操作）

```txt
- 存储实际的数据块
- 执行数据块的读/写操作
```

3）Client（客户端）

```txt
- 文件切分。文件上传到HDFS时，Client将文件切分成一个一个的Block，然后进行上传
- 与NameNode交互，获取文件的位置信息
- 与DataNode交互，读取或者写入数据
- Client提供一些命令来管理HDFS，比如NameNode格式化
- Client可以通过一些命令来访问HDFS，比如对HDFS增删改查操作
```

4）Secondary NameNode（不是NN的热备，缩写2NN）

```txt
- 辅助NN分担工作量，定期合并Fsimage和Edits，推送给NameNode
```

5）Block文件块

```
- 块的大小通过配置参数dfs.blockSize来规定，Block大小的设置主要取决于磁盘的传输速度
	=> 太小，会增加寻址时间
	=> 太大，会导致处理数据时太慢
- 默认128M，老版本64M
```

注意：Block大小计算公式

- 如果寻址时间约为10ms，即查找到目标block的时间为10ms；
- 寻址时间为传输时间的1%时，则为最佳状态，因此，传输时间 10ms / 0.01 = 1s
- block的大小1s * 100MB/s （磁盘传输速度）= 100MB => 128M (取整，计算机中128才是整数)



## 二、HDFS的读写流程

### 2.1、写操作流程

![image-20200712212718140](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-hdfs/image-20200712212718140.png)

1）客户端通过Distributed FileSystem模块向NameNode请求上传文件，NameNode检查目标文件是否已存在，父目录是否存在。

2）NameNode返回是否可以上传。

3）客户端请求第一个 Block上传到哪几个DataNode服务器上。

4）NameNode返回3个DataNode节点，分别为dn1、dn2、dn3。

5）客户端通过FSDataOutputStream模块请求dn1上传数据，dn1收到请求会继续调用dn2，然后dn2调用dn3，将这个通信管道建立完成。

6）dn1、dn2、dn3逐级应答客户端。

7）客户端开始往dn1上传第一个Block（先从磁盘读取数据放到一个本地内存缓存），以Packet为单位，dn1收到一个Packet就会传给dn2，dn2传给dn3；dn1每传一个packet会放入一个应答队列等待应答。

8）当一个Block传输完成之后，客户端再次请求NameNode上传第二个Block的服务器。（重复执行3-7步）。



### 2.2、网络拓扑

在HDFS写数据的过程中，NameNode会选择距离待上传数据最近距离的DataNode接收数据。那么这个最近距离怎么计算呢？

![image-20200712213336527](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-hdfs/image-20200712213336527.png)

**节点距离：两个节点到达最近的共同祖先的距离总和**

- Distance（/d1/r1/n0, /d1/r1/n0）= 0，同一节点的进程
- Distance（/d1/r1/n1, /d1/r1/n2）= 2，同一机架上的不同节点
- Distance（/d1/r2/n0, /d1/r3/n2）= 4，同一数据中心不同机架上的节点
- Distance（/d1/r2/n1, /d2/r4/n1）= 6，不同数据中心的节点



### 2.3、机架感知

机架感知（副本存储节点选择）

![image-20200712214150027](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-hdfs/image-20200712214150027.png)

1）第一个副本在client所处的节点上，如果client在集群外，随机选一个

2）第二个副本在另一个机架的随机一个节点

3）第三个副本在第二个副本所在机架的随机节点



### 2.4、HDFS读数据流程

![image-20200712213101011](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-hdfs/image-20200712213101011.png)

1）客户端通过Distributed FileSystem向NameNode请求下载文件，NameNode通过查询元数据，找到文件块所在的DataNode地址。

2）挑选一台DataNode（就近原则，然后随机）服务器，请求读取数据。

3）DataNode开始传输数据给客户端（从磁盘里面读取数据输入流，以Packet为单位来做校验）。

4）客户端以Packet为单位接收，先在本地缓存，然后写入目标文件



## 三、NN和2NN

### 3.1、NN与2NN工作机制

NameNode中的元数据是如何存储的？

1）元数据需要存放在内存中来提高读取速度；

2）为了防止元数据丢失，在磁盘中对元数据进行备份（备份到FsImage中）；

3）引入Edits文件(只进行追加操作，效率很高)，每当元数据有更新或者添加时，同时将数据追加到Edits中；

4）SecondaryNamenode定期合并FsImage和Edits。

```txt
思考：为什么要这样设计：
- NN经常需要进行随机访问，还有响应客户请求，数据存在磁盘中，必然是效率过低。因此，元数据需要存放在内存中；
- 但如果只存在内存中，一旦断电，元数据丢失，整个集群就无法工作了。因此产生在磁盘中备份元数据的FsImage；
- 如果更新数据同时更新FsImage，就会导致效率过低，但如果不更新，就会发生一致性问题，一旦NameNode节点断电，就会产生数据丢失。
  因此，引入Edits文件；
- 如果长时间添加数据到Edits中，会导致该文件数据过大，效率降低，而且一旦断电，恢复元数据需要的时间过长。
  因此，需要定期进行FsImage和Edits的合并，如果这个操作由NN节点完成，又会效率过低。因此，引入2NN。
```

![image-20200712223114416](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-hdfs/image-20200712223114416.png)

第一阶段：NameNode启动

```txt
1）第一次启动NameNode格式化后，创建Fsimage和Edits文件。如果不是第一次启动，直接加载编辑日志和镜像文件到内存。
2）客户端对元数据进行增删改的请求。
3）NameNode记录操作日志，更新滚动日志。
4）NameNode在内存中对元数据进行增删改。
```

第二阶段：Secondary NameNode工作

```txt
1）Secondary NameNode询问NameNode是否需要CheckPoint。直接带回NameNode是否检查结果。
2）Secondary NameNode请求执行CheckPoint。
3）NameNode滚动正在写的Edits日志。
4）将滚动前的编辑日志和镜像文件拷贝到Secondary NameNode。
5）Secondary NameNode加载编辑日志和镜像文件到内存，并合并。
6）生成新的镜像文件fsimage.chkpoint。
7）拷贝fsimage.chkpoint到NameNode。
8）NameNode将fsimage.chkpoint重新命名成fsimage。
```

### 3.2、Fsimage和Edits解析

1）Fsimage：NN内存中元数据序列化后形成的文件。里面包含文件系统所有目录和inode的序列化信息

2）Edits：记录客户端更新元数据信息的每一步操作（可通过Edits运算出元数据）。

3） CheckPoint（在hdfs-default.xml中配置）

- 通常情况下，SecondaryNameNode每隔一小时执行一次
- 一分钟检查一次操作次数，当操作次数达到1百万时，2NN执行一次

4）NameNode故障处理

- 将2NN中数据拷贝到NN存储数据的目录
- 使用-importCheckpoint选项启动NN守护进程，从而将2NN中数据拷贝到NN目录中。

### 3.3、NameNode多目录配置

NameNode的本地目录可以配置成多个，且每个目录存放内容相同，增加了可靠性

1）在hdfs-site.xml文件中修改如下内容

```xml
<property>
  <name>dfs.namenode.name.dir</name>
  <value>file:///${hadoop.tmp.dir}/name1,file:///${hadoop.tmp.dir}/name2</value>
</property>
```

2）停止集群，删除data和logs中所有数据。

```shell
[atguigu@hadoop102 hadoop-3.1.3]$ rm -rf data/ logs/
[atguigu@hadoop103 hadoop-3.1.3]$ rm -rf data/ logs/
[atguigu@hadoop104 hadoop-3.1.3]$ rm -rf data/ logs/
```

3）格式化集群并启动。

```shell
[atguigu@hadoop102 hadoop-3.1.3]$ bin/hdfs namenode –format
[atguigu@hadoop102 hadoop-3.1.3]$ sbin/start-dfs.sh
```

4）查看结果

```shell
[atguigu@hadoop102 dfs]$ ll
总用量 12
drwx------. 3 atguigu atguigu 4096 12月 11 08:03 data
drwxrwxr-x. 3 atguigu atguigu 4096 12月 11 08:03 name1
drwxrwxr-x. 3 atguigu atguigu 4096 12月 11 08:03 name2
```

 

## 四、DataNode

### 4.1、DataNode工作机制

![image-20200712224322961](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-hdfs/image-20200712224322961.png)

1）一个数据块在DataNode上以文件形式存储在磁盘上，包括两个文件，一个是数据本身，一个是元数据包括数据块的长度，块数据的校验和，以及时间戳。

2）DataNode启动后向NameNode注册，通过后，周期性（1小时）的向NameNode上报所有的块信息。

3）心跳是每3秒一次，心跳返回结果带有NameNode给该DataNode的命令如复制块数据到另一台机器，或删除某个数据块。如果超过10分钟没有收到某个DataNode的心跳，则认为该节点不可用。

4）集群运行中可以安全加入和退出一些机器。



### 4.2、数据完整性

计算数据完整性的流程：

1）DataNode读取BLock的时候，计算CheckSum

- 一致则读取
- 不一致则计算其他DataNode的CheckSum

2）DataNode周期性验证CheckSum

3）使用crc校验（常见的校验算法 crc（32） 、md5（128）、  sha1（160） ）



### 4.3、掉线时限参数设置

![image-20200712225303668](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-hdfs/image-20200712225303668.png)

1）DataNode进程死亡或者网络故障造成DataNode无法与NameNode通信；

2）NameNode不会立即把该节点判定为死亡，要经过一段时间，这段时间暂称作超时时长；

3）HDFS默认的超时时长为10分钟 + 30秒

4）超时时长计算公式

```txt
TimuOut = 2 * dfs.namenode.heartbeat.rechech - interval + 10 * dfs.heartbeat.interval
```

 默认的dfs.namenode.heartbeat.rechech - interval大小为5分钟，

默认的dfs.heartbeat.interval大小为3秒。



### 4.4、服役新数据节点

​	随着公司业务的增长，数据量越来越大，原有的数据节点的容量已经不能满足存储数据的需求，需要在原有

集群基础上动态添加新的数据节点。

1）环境准备

- 在hadoop104主机上再克隆一台hadoop105主机
- 修改IP地址和主机名称
- 删除原来HDFS文件系统留存的文件（/opt/module/hadoop-3.1.3/data和log）
- source一下配置文件

```shell
[atguigu@hadoop105 hadoop-3.1.3]$ source /etc/profile
```

2）服役新节点具体步骤

直接启动DataNode，即可关联到集群

```shell
[atguigu@hadoop105 hadoop-3.1.3]$ hdfs --daemon start datanode
[atguigu@hadoop105 hadoop-3.1.3]$yarn -–daemon start nodemanager
```



### 4.5、退役旧数据节点

##### 4.5.1、白名单与黑名单

- 添加到白名单的主机节点，都允许访问NameNode，不在白名单的主机节点，都会被直接退出。
- 添加到黑名单的主机节点，不允许访问NameNode，会在数据迁移后退出。.

注意：实际情况下，白名单用于确定允许访问NameNode的DataNode节点，内容配置一般与workers文件内容一致。 黑名单用于在集群运行过程中退役

DataNode节点

##### 4.5.2、配置白、黑名单

1）在NameNode的/opt/module/hadoop-3.1.3/etc/hadoop目录下分别创建whitelist 和blacklist文件

```txt
[atguigu@hadoop102 hadoop]$ pwd
/opt/module/hadoop-3.1.3/etc/hadoop
[atguigu@hadoop102 hadoop]$ touch whitelist
[atguigu@hadoop102 hadoop]$ touch blacklist
```

在whitelist中添加如下主机名称,假如集群正常工作的节点为102 103 104 105 

```txt
hadoop102
hadoop103
hadoop104
hadoop105
```

黑名单暂时为空。

2）在NameNode的hdfs-site.xml配置文件中增加dfs.hosts 和 dfs.hosts.exclude配置

```xml
<property>
	<name>dfs.hosts</name>
	<value>/opt/module/hadoop-3.1.3/etc/hadoop/whitelist</value>
</property>

<property>
	<name>dfs.hosts.exclude</name>
	<value>/opt/module/hadoop-3.1.3/etc/hadoop/blacklist</value>
</property>
```

3）配置文件分发

```shell
[atguigu@hadoop102 hadoop]$ xsync hdfs-site.xml
```

4）重新启动集群

```shell
[atguigu@hadoop102 hadoop-3.1.3]$ stop-dfs.sh
[atguigu@hadoop102 hadoop-3.1.3]$ start-dfs.sh
```

注意: 因为workers中没有配置105,需要单独在105启动DN

5）在web端查看目前正常工作的DN节点



##### 4.5.3、退役节点

退役通常使用黑名单，白名单退役会直接将节点抛弃，没有迁移数据的过程，会造成数据丢失。

1）准备使用黑名单退役105，编辑blacklist文件，添加105  

```shell
[atguigu@hadoop102 hadoop] vim blacklist
hadoop105
```

2）刷新NameNode

```shell
[atguigu@hadoop102 hadoop] hdfs dfsadmin -refreshNodes
```

3）在web端查看DN状态，105 正在退役中…进行数据的迁移

![image-20200712231014228](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-hdfs/image-20200712231014228.png)



### 4.6、DataNode多目录配置

DataNode也可以配置成多个目录，每个目录存储的数据不一样。

1）在hdfs-site.xml中修改如下内容:

```xml
<property>
	<name>dfs.datanode.data.dir</name>
	<value>file:///${hadoop.tmp.dir}/data1,file:///${hadoop.tmp.dir}/data2</value>
</property>
```

2）停止集群，删除data和logs中所有数据。

```shell
[atguigu@hadoop102 hadoop-3.1.3]$ rm -rf data/ logs/
[atguigu@hadoop103 hadoop-3.1.3]$ rm -rf data/ logs/
[atguigu@hadoop104 hadoop-3.1.3]$ rm -rf data/ logs/
```

3）格式化集群并启动。

```shell
[atguigu@hadoop102 hadoop-3.1.3]$ bin/hdfs namenode –format
[atguigu@hadoop102 hadoop-3.1.3]$ sbin/start-dfs.sh
```

4）查看结果

```shell
[atguigu@hadoop102 dfs]$ ll
总用量 12
drwx------. 3 atguigu atguigu 4096 4月  4 14:22 data1
drwx------. 3 atguigu atguigu 4096 4月  4 14:22 data2
drwxrwxr-x. 3 atguigu atguigu 4096 12月 11 08:03 name1
drwxrwxr-x. 3 atguigu atguigu 4096 12月 11 08:03 name2
```



## 五、小文件存档

### 5.1、小文件的弊端

1）大量的小文件会耗尽NameNode的大部分内存（每个文件均会按块存储，每个块的元数据存储在NameNode的内存中）；

2）索引文件过大使得索引速度变慢。

注意：存储小文件所需要的磁盘容量和数据块的大小无关。一个1MB的文件设置为128M的块，实际使用的时

1MB的磁盘空间。



### 5.2、解决小文件的方案

1）采用har归档方式，将小文件归档

2）采用CombineTextInputFormat

3）有小文件场景开启JVM重用；如果没有小文件，不要开启JVM重用，因为会一直占用使用到的task卡槽，直到任务完成才释放。

JVM重用可以使得JVM实例在同一个job中重新使用N次，N的值可以在Hadoop的mapred-site.xml文件中进行配置。通常在10-20之间

 



## 六、纠删码

### 6.1、为什么要引入纠删码

纠删码是hadoop3.x新加入的功能，之前的hdfs都是采用副本方式容错，默认情况下，一个文件有3个副

本，可以容忍任意2个副本（datanode）不可用，这样提高了数据的可用性，但也带来了2倍的冗余开销。例

如3TB的空间，只能存储1TB的有效数据。



### 6.2、纠删码

1）RS-10-4-1024k：使用RS编码，每10个数据单元（cell），生成4个校验单元，共14   个单元，也就是说：这14个单元中，只要有任意的10个单元存在（不管是

数据单元还 是校验单元，只要总数=10），就可以得到原始数据。每个单元的大小是    1024k=1024*1024=1048576。

2）RS-3-2-1024k：使用RS编码，每3个数据单元，生成2个校验单元，共5个单元，也  就是说：这5个单元中，只要有任意的3个单元存在（不管是数据单元还是校

验单元，  只要总数=3），就可以得到原始数据。每个单元的大小是1024k=1024*1024=1048576。

3）RS-6-3-1024k：使用RS编码，每6个数据单元，生成3个校验单元，共9个单元，也  就是说：这9个单元中，只要有任意的6个单元存在（不管是数据单元还是校

验单元，只要总数=6），就可以得到原始数据。每个单元的大小是1024k=1024*1024=1048576。

4）RS-LEGACY-6-3-1024k：策略和上面的RS-6-3-1024k一样，只是编码的算法用的是rs- legacy。

5）XOR-2-1-1024k：使用XOR编码（速度比RS编码快），每2个数据单元，生成1个校  验单元，共3个单元，也就是说：这3个单元中，只要有任意的2个单元存在（不管是  数据单元还是校验单元，只要总数=2），就可以得到原始数据。每个单元的大小是    1024k=1024*1024=1048576。