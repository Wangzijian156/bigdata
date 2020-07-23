# Hadoop-Yarn

- [Hadoop-Yarn](#hadoop-yarn)
  - [一、概述](#一概述)
  - [二、Yarn基本框架](#二yarn基本框架)
  - [三、Yarn工作机制](#三yarn工作机制)
  - [四、资源调度器](#四资源调度器)
  - [五、Yarn任务推测执行](#五yarn任务推测执行)




## 一、概述

Yarn是一个资源调度平台，负责为运算程序提供服务器运算资源，相当于一个分布式的操作系统平台

## 二、Yarn基本框架 

![image-20200713232247038](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-yarn/image-20200713232247038.png)

1）Resource Manager(RM)

```txt
- 处理客户端请求
- 监控Nodemanager
- 启动或监控ApplicationMaster
- 资源的分配与调度
```

2）NodeManager（NM）

```txt
- 管理单个节点上的资源
- 处理来自Resouremanager的命令
- 处理来自ApplicationMaster的命令
```

3）ApplicationMaster(AM)

```txt
- 负责数据的切分
- 为应用程序申请资源分配任务
- 任务的监控与容器
```

4）Container

```txt
- 资源抽象-封装某个节点上的多维资源、如内存、CPU、磁盘、网络
```



## 三、Yarn工作机制

![image-20200713232319577](https://gitee.com/wangzj6666666/bigdata-img/raw/master/hadoop-yarn/image-20200713232319577.png)

1）MR程序提交到客户端所在的节点。

2）YarnRunner向ResourceManager申请一个Application。

3）RM将该应用程序的资源路径返回给YarnRunner。

4）该程序将运行所需资源提交到HDFS上。

5）程序资源提交完毕后，申请运行mrAppMaster。

6）RM将用户的请求初始化成一个Task。

7）其中一个NodeManager领取到Task任务。

8）该NodeManager创建容器Container，并产生MRAppmaster。

9）Container从HDFS上拷贝资源到本地。

10）MRAppmaster向RM 申请运行MapTask资源。

11）RM将运行MapTask任务分配给另外两个NodeManager，另两个NodeManager分别领取任务并创建容器。

12）MR向两个接收到任务的NodeManager发送程序启动脚本，这两个NodeManager分别启动MapTask，MapTask对数据分区排序。

13）MrAppMaster等待所有MapTask运行完毕后，向RM申请容器，运行ReduceTask。

14）ReduceTask向MapTask获取相应分区的数据。

15）程序运行完毕后，MR会向RM申请注销自己。



## 四、资源调度器

1）FIFO：单队列，First In First Out

2）Capacity Scheduler

- 多队列，FIFO
- 如何选择进入哪一个队列
  - 正在运行的任务数/应分得的计算资源
  - 得出比值最小即最闲的队列

3）Fair Scheduler

- 多队列，可同时运行多个任务
- 优先级概念：理想获得资源-实际获得资源



## 五、Yarn任务推测执行

1）基本假设，作业完成时间取决于最慢的任务完成时间

2）推测执行机制

- 前提条件
  - 每个Task只能有一个备份任务
  - 当前Job已完成Task必须不少于5%
  - 开启推测执行参数设置：mapred-site.xml

- 不能使用推测执行机制的情况
  - 任务间存在严重的负载倾斜
  - 特殊任务，比如任务向数据库中写数据
- 算法原理