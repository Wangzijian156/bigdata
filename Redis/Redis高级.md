# Redis高级

- [Redis高级](#redis高级)
  - [一、单线程+IO多路复用](#一单线程io多路复用)
    - [1.1、单线程+IO多路复用](#11单线程io多路复用)
    - [1.2、5种IO模型](#125种io模型)
  - [二、Redis事务](#二redis事务)
    - [2.1、事务概述](#21事务概述)
    - [2.2、主要命令](#22主要命令)
    - [2.3、事务执行](#23事务执行)
    - [2.4、脏读、不可重复读、幻读](#24脏读不可重复读幻读)
    - [2.5、Mysql的四种隔离级别](#25mysql的四种隔离级别)
    - [2.6、锁机制](#26锁机制)
    - [2.7、watch/unwatch](#27watchunwatch)
  - [三、Redis持久化](#三redis持久化)
    - [3.1、持久化](#31持久化)
    - [3.2、RDB(Redis DataBase)](#32rdbredis-database)
    - [3.3、AOF(Append Of File)](#33aofappend-of-file)
  - [四、主从复制](#四主从复制)
    - [4.1、什么是主从复制](#41什么是主从复制)
    - [4.2、有什么用](#42有什么用)
    - [4.3、一主二仆模式](#43一主二仆模式)
    - [4.4、复制原理](#44复制原理)
    - [4.5、薪火相传](#45薪火相传)
    - [4.6、反客为主](#46反客为主)
    - [4.7、哨兵模式](#47哨兵模式)
    - [4.8、Jedis访问哨兵模式](#48jedis访问哨兵模式)
  - [五、集群](#五集群)
    - [5.1、什么是Redis集群](#51什么是redis集群)
    - [5.2、配置redis集群](#52配置redis集群)
    - [5.3、Hash槽（slots）](#53hash槽slots)
    - [5.4、故障恢复](#54故障恢复)
    - [5.5、JedisCluster访问集群](#55jediscluster访问集群)
  - [六、Redis实际应用中的问题](#六redis实际应用中的问题)
    - [6.1、缓存穿透](#61缓存穿透)
    - [6.2、缓存雪崩问题](#62缓存雪崩问题)

## 一、单线程+IO多路复用

### 1.1、单线程+IO多路复用

```txt
- Redis是单线程的，避免了线程切换、加锁等资源消耗
- 虽是单线程，但redis读数据的速度很快,原因是采用了IO多路复用的机制
```

如何理解多路复用，举个实例老师检查学生作业

第一种 **单线程**：按顺序逐个检查，先检查A，然后是B，之后是C、D。。。这中间如果有一个学生卡主，全班都会被耽误。这种模式就好比，老师用循环挨个处理。

第二种 **多线程**：老师创建30个分身，每个分身检查一个学生的答案是否正确。 这种类似于为每一个用户创建一个进程或者线程处理连接。讲台上等。此时E、A又举手，然后去处理E和A。。。 这种就是IO复用模型。

第三种 **IO多路复用**，老师站在讲台上等，谁作业写完谁举手。这时C、D举手，表示他们解答问题完毕，老师下去依次检查C、D的答案，然后继续回到讲台上等。此时E、A又举手，然后去处理E和A。。。

多路复用是指使用一个线程来检查多个文件描述符（Socket）的就绪状态。



### 1.2、5种IO模型

1）什么是IO

IO (Input/Output，输入/输出)即数据的读取（接收）或写入（发送）操作，通常用户进程中的一个完整IO分为两阶段：用户进程空间<-->内核空间、内核空间<-->设备空间（磁盘、网络等）。IO有内存IO、网络IO和磁盘IO三种，通常我们说的IO指的是后两者。

![image-20200715095057380](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200715095057380.png)

对于一个输入操作来说，进程IO系统调用后，内核会先看缓冲区中有没有相应的缓存数据，没有的话再到设备中读取，因为设备IO一般速度较慢，需要等待；内核缓冲区有数据则直接复制到进程空间。

对于一个网络输入操作通常包括两个不同阶段：

- 等待网络数据到达网卡→读取到内核缓冲区，数据准备好；

- 从内核缓冲区复制数据到进程空间。



2）阻塞lO模型

![image-20200715094731624](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200715094731624.png)

进程发起IO系统调用后，进程被阻塞，转到内核空间处理，整个IO处理完毕后返回进程。操作成功则进程获取到数据

- 典型应用：阻塞socket、Java BIO；

- 特点：

```
- 进程阻塞挂起不消耗CPU资源，及时响应每个操作；
- 实现难度低、开发应用较容易；
- 适用并发量小的网络应用开发；
```



3）非阻塞IO模型

![image-20200715101141839](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200715101141839.png)

当用户线程发起一个read操作后，并不需要等待，而是马上就得到了一个结果。如果结果是一个error时，它就知道数据还没有准备好，于是它可以再次发送read操作。一旦内核中的数据准备好了，并且又再次收到了用户线程的请求，那么它马上就将数据拷贝到了用户线程，然后返回。

- 典型应用：socket是非阻塞的方式（设置为NONBLOCK）

- 特点：

```txt
- 进程轮询（重复）调用，消耗CPU的资源；
- 实现难度低、开发应用相对阻塞IO模式较难；
- 适用并发量较小、且不需要及时响应的网络应用开发；
```

```java
 while(true){ 
     data = socket.read(); 
     if(data!= error){ 
         //处理数据 
         break; 
     } 
 }
```



4）IO复用模型

![image-20200715102104487](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200715102104487.png)

IO多路转接是多了一个select函数，select函数有一个参数是文件描述符集合，对这些文件描述符进行循环监听，当某个文件描述符就绪时，就对这个文件描述符进行处理。其中，select只负责等，recvfrom只负责拷贝。IO多路转接是属于阻塞IO，但可以对多个文件描述符进行阻塞监听，所以效率较阻塞IO的高。

- 典型应用：select、poll、epoll三种方案，nginx都可以选择使用这三个方案;Java NIO;

- 特点：

```
- 专一进程解决多个进程IO的阻塞问题，性能好；Reactor模式;
- 实现、开发应用难度较大；
- 适用高并发服务应用开发：一个进程（线程）响应多个请求；
```



5）信号驱动IO模型

![image-20200715102400058](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200715102400058.png)

当进程发起一个IO操作，会向内核注册一个信号处理函数，然后进程返回不阻塞；当内核数据就绪时会发送一个信号给进程，进程便在信号处理函数中调用IO读取数据。

- 特点：回调机制，实现、开发应用难度大；



6）异步IO模型

![image-20200715102514355](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200715102514355.png)

当进程发起一个IO操作，进程返回（不阻塞），但也不能返回果结；内核把整个IO处理完后，会通知进程结果。如果IO操作成功则进程直接获取到数据。

- 典型应用：JAVA7 AIO、高性能服务器应用

- 特点：

```
- 不阻塞，数据一步到位；Proactor模式；
- 需要操作系统的底层支持，LINUX 2.5 版本内核首现，2.6 版本产品的内核标准特性；
- 实现、开发应用难度大；
- 非常适合高性能高并发应用。
```



7）IO模型比较

![image-20200715102749046](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200715102749046.png)

同步IO：用户进程发出IO调用，去获取IO设备数据，双方的数据要经过内核缓冲区同步，完全准备好后，再复制返回到用户进程。而复制返回到用户进程会导致请求进程阻塞，直到I/O操作完成。

异步IO：用户进程发出IO调用，去获取IO设备数据，并不需要同步，内核直接复制到进程，整个过程不导致请求进程阻塞。

所以， 阻塞IO模型、非阻塞IO模型、IO复用模型、信号驱动的IO模型者为同步IO模型，只有异步IO模型是异步IO



## 二、Redis事务

### 2.1、事务概述

```txt
- Redis事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，
  不会被其他客户端发送来的命令请求所打断。
- Redis事务的主要作用就是串联多个命令防止别的命令插队
- Redis的事务没有回滚（和mysql不一样，事务回滚需要消耗大量运行资源，reids追求速度，而放弃了）
```

### 2.2、主要命令	

```txt
- Multi：标记一个事务的开始
- Exec：执行所有事务块内的命令
- discard：取消事务，放弃执行事务块内的所有命令
- watch：监视一个（或多个）key，如果在事务执行之前这些key被其他命令所改动，那么事务会被打断
- unwatch：取消WATCH命令对所有key的监视
```

### 2.3、事务执行

说明：组队阶段出错回滚，执行阶段出错跳过并执行下一行

```txt
- 开启：Multi开启一个事务
- 入队：将多个命令入队到事务中，接到这些命令并不会立即执行，而是放到等待执行的事务队列里
- 执行：由EXEC命令触发执行
```

1）执行一个事务

![image-20200717092539841](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200717092539841.png)

2）放弃事务

![image-20200717092930121](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200717092930121.png)

3）事务异常情况，前面说了Redis中的事务是没有回滚操作的，出现异常分为两种情况

- 添加指令时异常（在multi后，exec前抛出异常），那么事务里面的指令会全部不执行

![image-20200717093035561](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200717093035561.png)

- 执行指令时异常（在exec后抛出异常），此时这种情况会对事务中的可正常执行的指令全部正常执行，异常指令返回异常。

![image-20200717101653435](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200717101653435.png)



### 2.4、脏读、不可重复读、幻读

在并发访问情况下，可能会出现脏读、不可重复读和幻读等读现象。

1）脏读：发生在一个事务A读取了被另一个事务B修改，但是还未提交的数据。假如B回退，则事务A读取的是无效的数据。这跟不可重复读类似，但是第二个事务不需要执行提交。 

![image-20200717152654862](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200717152654862.png)



2）不可重复读：在基于锁的并行控制方法中，如果在执行select时不添加读锁，就会发生不可重复读问题。在多版本并行控制机制中，当一个遇到提交冲突的事务需要回退但却被释放时，会发生不可重复读问题。

![image-20200717153903452](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200717153903452.png)

不可重复读是指在对于数据库中的某个数据，一个事务范围内多次查询却返回了不同的数据值，这是由于在查询间隔，被另一个事务修改并提交了。
不可重复读和脏读的区别是，脏读是某一事务读取了另一个事务未提交的脏数据，而不可重复读则是读取了前一事务提交的数据。



3）幻读发生在当两个完全相同的查询执行时，第二次查询所返回的结果集跟第一个查询不相同。

![image-20200717165830644](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200717165830644.png)



一个事务 1，先发送一条 SQL 语句，里面有一个条件，要查询一批数据出来，如 `SELECT * FROM user WHERE age between 10 and 30`。接着这个时候，别的事务 2往表里插了几条数据，而且事务 B 还提交了，此时多了几行数据。接着事务 1 此时第二次查询，再次按照之前的一模一样的条件执行 `SELECT * FROM user WHERE age between 10 and 30` 这条 SQL 语句，由于其他事务插入了几条数据，导致这次它查询出来了 多了几条数据。



数据库设计了事务隔离机制、MVCC 多版本隔离机制、锁机制，来解决脏读、不可重复读、幻读等问题。



### 2.5、Mysql的四种隔离级别

1）Read Uncommitted（读取未提交内容）

在该隔离级别，所有事务都可以看到其他未提交事务的执行结果。本隔离级别很少用于实际应用，因为它的性能也不比其他级别好多少。读取未提交的数据，也被称之为脏读（Dirty Read）。

 

2）Read Committed（读取提交内容）

这是大多数数据库系统的默认隔离级别（但不是MySQL默认的）。它满足了隔离的简单定义：一个事务只能看见已经提交事务所做的改变。这种隔离级别 也支持所谓的不可重复读（Nonrepeatable Read），因为同一事务的其他实例在该实例处理其间可能会有新的commit，所以同一select可能返回不同结果。

 

3）Repeatable Read（可重读）

这是MySQL的默认事务隔离级别，它确保同一事务的多个实例在并发读取数据时，会看到同样的数据行。不过理论上，这会导致另一个棘手的问题：幻读 （Phantom Read）。简单的说，幻读指当用户读取某一范围的数据行时，另一个事务又在该范围内插入了新行，当用户再读取该范围的数据行时，会发现有新的“幻影” 行。InnoDB和Falcon存储引擎通过多版本并发控制（MVCC，Multiversion Concurrency Control）机制解决了该问题。



4）Serializable（可串行化）

这是最高的隔离级别，它通过强制事务排序，使之不可能相互冲突，从而解决幻读问题。简言之，它是在每个读的数据行上加上共享锁。在这个级别，可能导致大量的超时现象和锁竞争。



| 隔离级别         | 脏读 | 不可重复读 | 幻读 |
| ---------------- | ---- | ---------- | ---- |
| Read Uncommitted | √    | √          | √    |
| Read Committed   | ×    | √          | √    |
| Repeatable Read  | ×    | ×          | √    |
| Serializable     | ×    | ×          | ×    |



### 2.6、锁机制

在并发访问情况下，可能会出现脏读、不可重复读和幻读等读现象，为了应对这些问题，主流数据库都提供了锁机制，并引入了事务隔离级别的概念。

1）悲观锁(Pessimistic Lock), 顾名思义，就是很悲观，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会block直到它拿到锁。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。**使得并发操作串行化**。

2）乐观锁(Optimistic Lock), 顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号等机制。乐观锁适用于多读的应用类型，这样可以提高吞吐量。

```txt
乐观锁策略:提交版本必须大于记录当前版本才能执行更新
```

3）CAS（Check And Set）

多个线程通过CAS尝试修改同一个变量，只有一个线程在同一时刻进行修改，而其他的操作失败，失败的线程不会挂起，告诉失败的线程可以再次尝试。

CAS操作涉及到三个操作数：需要读写的内存位置（V）、进行比较的预期的原值（A）、待写入的新值（B）。

- 第一步：获取位置V的值A  


- 第二步：将A和B同时进行处理,将A和位置V存储值进行比较，如果相等，则将位置V的值由A变更为B,则操作成功;不相等,则不变更为B,继续循环进入第一步。


```txt
乐观锁是一种思想，CAS是这种思想的一种实现方式。
```



### 2.7、watch/unwatch

1）watch指令类似与乐观锁，事务提交时，如果key的值已被别的客户端改变，比如某个list已被别的客户端push/pop，整个事务队列都不会执行。

2）通过watch命令在事务执行之前监控了多个key，倘若在watch之后由任何的值发生了变化，EXEC命令执行的事务都将被放弃，同时返回Nullmulti-back应答已通知调用者事务执行失败。



## 三、Redis持久化

### 3.1、持久化

数据存储在内存中是不安全，一旦发生宕机，数据就会丢失，所以需要定期将数据做持久化处理（也就是保存到磁盘中）。Resdis提供了

2种持久化的方式RDB和AOF。

### 3.2、RDB(Redis DataBase)

1）概念（全量备份）

```txt
在指定的时间间隔内将内存中的数据集快照写入磁盘，也就是行话讲的Snapshot快照，它恢复时是将快照文件直接读到内存里。
```

2）备份如何执行

```txt
Redis会单独创建（fork）一个子进程来进行持久化，会先将数据写入到一个临时文件中，待持久化过程都结束了，
再用这个临时文件替换上次持久化好的文件。
```

3）关于fork

```txt
在Linux程序中，fork()会产生一个和父进程完全相同的子进程，但子进程在此后多会exec系统调用，出于效率考虑，
Linux中引入了“写时复制技术”，一般情况父进程和子进程会共用同一段物理内存，只有进程空间的各段的内容要发生变化时，
才会将父进程的内容复制一份给子进程。
```

5）备份文件

```txt
- 先通过config get dir  查询rdb文件的目录 
- 将*.rdb的文件拷贝到别的地方
```

6）如何恢复

```txt
- 关闭Redis
- 先把备份的文件拷贝到工作目录下
- 启动Redis, 备份数据会直接加载
```

7）rbd的保存策略（什么时候会触发备份）

rdb的自动保存策略在redis.config文件中，可通过更改config配置文件来对rdb的自动保存策略进行修改（周期性备份，也会丢失小部分数据）

```txt
save 900 1 
	- 900s 15min 至少一条数据改变
save 300 10
	- 300s 5min 至少10条数据改变
save 60 10000
	- 60s 只扫10000条数据改变
```

手动保存 ：save会根据上面三个判断条件触发存盘（会阻塞其他业务流程，谨慎使用）

bgsave后台独立备份进程（fork）

8）rdb的优点及缺点

- 优点

```txt
- 节省磁盘空间
- 恢复速度快
```

- 缺点

```txt
- 虽然Redis在fork时使用了写时拷贝技术，但是如果数据庞大时还是比较小号性能
- 周期性备份，如果Redis意外down掉的化，就会丢失最后一次快照的所有修改
```

9）触发存盘的指令

```txt
flushdb  不触发存盘 （以数据库为单位清理)
flushall 触发存盘 （执行后存盘）
	- flashall会触发存盘（先执行flashall，在存盘），存一个空的dump.rbd文件，再次启动数据全为空
shutdown 触发存盘
kill 触发存盘
kill -9 不触发存盘
意外宕机  不触发存盘
```



### 3.3、AOF(Append Of File)

1）概念（AOF增量备份）

```txt
- 以日志的形式来记录每个写操作，将Redis执行过的所有写指令记录下来（读操作不记录）
- 只允许追加文件但不可以改写文件
- Redis重启的话就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作
```

2）AOF的开启

```txt
- 修改配置文件appendonly yes
- 备份文件名字appendfilename "appendonly.aof"
- 通过redis.conf文件配置，保存路径与rdb一致
- AOF和RDB同时开启，系统默认取AOF的数据
```

3）AOF文件故障备份

```txt
AOF的备份机制和性能虽然和RDB不同, 但是备份和恢复的操作同RDB一样，都是拷贝备份文件，需要恢复时再拷贝到Redis工作目录下，
启动系统即加载。
```

注意：**AOF和RDB同时开启，系统默认取AOF的数据**

4）AOF文件故障恢复

```txt
如遇到AOF文件损坏，可通过 redis-check-aof  --fix  appendonly.aof   进行恢复
```

5）AOF的重写

```txt
- 概念：当AOF文件的大小超过所设定的阈值时，Redis就会启动AOF文件的内容压缩
	- bgrewriteaof
	
- 实现方式
	- 将整个内存中的数据库内容用命令的方式重写了一个新的aof文件,和快照优点类似

- 重写条件
	- 系统载入时或者上次重写完毕时，Redis会记录此时AOF大小，设为base_size,如果Redis的AOF
	  当前大小>= base_size +base_size*100% (默认)且当前大小>=64mb(默认)的情况下，
	  Redis会对AOF进行重写
```

6）不停机开启AOF

```txt
- 在已有数据的时候如果直接在配置文件中 设定 appendonly yes  然后再重启，有可能会加载不出数据；
- 已有数据不停机时，在命令行中执行config set appendonly yes；
- 同时记得也要在redis.conf中补充appendonly yes否则重启后appendonly就失效了。
```

7）aof的优点和缺点

- 优点

```txt
- 可读的日志文本，通过操作AOF稳健，可以处理误操作；
- 备份机制更稳健，丢失数据概率更低。
```

- 缺点

```txt
- 比起RDB占用更多的磁盘空间；
- 恢复备份速度要慢；
- 每次读写都同步的话，有一定的性能压力；
- 存在个别Bug，造成恢复不能。
```





## 四、主从复制

### 4.1、什么是主从复制

就是主机数据更新后根据配置和策略，自动同步到备机的master/slaver机制，Master以写为主，Slave以读为主

![image-20200717193502303](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200717193502303.png)

### 4.2、有什么用

1）读写分离，性能扩展

2）容灾快速恢复

### 4.3、一主二仆模式

1）复制三份redis.confg，分别是redis6379.confg(主)、redis6380.confg(仆)、redis6381.confg(仆)

```shell
# redis6379.confg
daemonize yes
save 30 10
port 6379
dir "/home/atguigu/ha_redis"
dbfilename "dump6379.rdb"
logfile "6379.log"
pidfile "/var/run/redis_6379.pid"
protected-mode no
# Generated by CONFIG REWRITE
maxclients 4064


# redis6380.confg
daemonize yes
save 30 10
port 6380
dir "/home/atguigu/ha_redis"
dbfilename "dump6380.rdb"
logfile "6380.log"
pidfile "/var/run/redis_6380.pid"
protected-mode no
# Generated by CONFIG REWRITE
slaveof 192.168.218.102 6379
maxclients 4064


# redis6381.confg
daemonize yes
port 6381
save 30 10
dir "/home/atguigu/ha_redis"
dbfilename "dump6381.rdb"
logfile "6381.log"
pidfile "/var/run/redis_6381.pid"
protected-mode no
slave-priority 10
# Generated by CONFIG REWRITE
maxclients 4064
slaveof 192.168.218.102 6379
```

2）先kill掉redis-server，再重新启动

```txt
redis-server redis6379.conf
redis-server redis6380.conf
redis-server redis6381.conf
```

3）启动三个redis客户端

```txt
redis-cli => 默认是 6379端口
redis-cli -p 6380
redis-cli -p 6381
```

4）在6380（仆）、6381（仆）客户端

```txt
slaveof hadoop102 6379
```

5）在6379（主）

```txt
info replication

# Replication
role:master
connected_slaves:2
slave0:ip=192.168.218.102,port=6380,state=online,offset=239,lag=1
slave1:ip=192.168.218.102,port=6381,state=online,offset=239,lag=0
master_repl_offset:239
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:238
```

6）从机能读取主机上的数据，但是不能写

```shell
127.0.0.1:6381> keys *
1) "k4"
2) "k1"
3) "k2"
4) "k3"

127.0.0.1:6381> set k4 v3
(error) READONLY You can't write against a read only slave.
```

7）一主二仆模式特点

```txt
- slave1、slave2是从头开始复制还是，从切入点开始（所有的key都复制）
- 主机shutdown后，从机原地待命
- 从机down了不影响主机
- 从机回复后不再是slave了（slaveof 只要是down了就失效了）
```

### 4.4、复制原理

```txt
- 每次从机联通后，都会给主机发送sync指令
- 主机立刻进行存盘操作，发送RDB文件，给从机
- 从机收到RDB文件后，进行全盘加载
- 之后每次主机的写操作，都会立刻发送给从机，从机执行相同的命令
```



### 4.5、薪火相传

![image-20200719173925724](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200719173925724.png)

slave同样可以接收其他slaves的连接和同步请求，该slave作为了链条中下一个的master（上一个slave可以是下一个slave的Master）, 

可以有效减轻master的写压力,去中心化降低风险

1）命令

```txt
slaveof <ip> <port>
```

注意：

- 中途变更转向:会清除之前的数据，重新建立拷贝最新的
- 风险是一旦某个slave宕机，后面的slave都没法备份



### 4.6、反客为主

 当一个master宕机后，后面的slave可以立刻升为master，其后面的slave不用做任何修改。

命令：

```txt
slaveof no one  
```



### 4.7、哨兵模式

哨兵（sentinel）模式：反客为主的自动版，能够后台监控主机是否故障，如果故障了根据投票数自动将从库转换为主库（解决了单点问题）。

1）配置哨兵

- 在配置之前一主二仆的/myredis目录下新建sentinel.conf文件

- 在配置文件中填写内容：

```txt
sentinel monitor mymaster 192.168.11.103 6379 1 
protected-mode no	
```

其中mymaster为监控对象起的服务器名称， 1 为 至少有多少个哨兵同意迁移的数量

protected-mode是哨兵有路由的功能，如果不关掉保护模式，方式哨兵需要输入密码.

- 开启一主二仆模式

- 执行哨兵

```txt
redis-sentinel  /myredis/sentinel.conf 
```

- shutdown主机，哨兵会自动将6381切换为主机

注意：在6381的配置问价中，配置了优先级 slave-priority 10（数字越小优先级越高）



2）哨兵模式详解

- 从下线的主服务的所有从服务里面挑选一个从服务，将其转成主服务。选择条件依次为：

```
- 选择优先级靠前的
  	- 如果配置了优先级

- 选择偏移量最大的
	- 偏移量是指从机从主机获得的数据，数据越多偏移量越大

- 选择runid最小的从服务
	- 任何一个redis都一个runid（随机产生）
```

- 挑选出新的主服务之后，sentinel 向原主服务的从服务发送 slaveof 新主服务 的命令，复制新master

- 当已下线的服务重新上线时，sentinel会向其发送slaveof命令，让其成为新主的从机。



### 4.8、Jedis访问哨兵模式

Jedis通过访问哨兵来访问redis集群（如果连主机，主机shutdown又要临时换配置）

1）创建RedisUtil工具类

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPoolConfig;
import redis.clients.jedis.JedisSentinelPool;

import java.util.HashSet;
import java.util.Set;

public class RedisUtil {
    private static JedisSentinelPool jedisSentinelPool = null;
    public static Jedis getJedis() {
        if (jedisSentinelPool == null) {
            JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
            jedisPoolConfig.setMaxTotal(20); // 最大连接数
            jedisPoolConfig.setBlockWhenExhausted(true); // 连接耗尽是否等待
            jedisPoolConfig.setMaxWaitMillis(2000); // 等待时间
            jedisPoolConfig.setMaxIdle(5); // 最大闲置连接数
            jedisPoolConfig.setMinIdle(5); // 最小闲置连接数
            jedisPoolConfig.setTestOnBorrow(true); // 取连接的时候进行一下测试 ping pong
            Set<String> sentinelSet = new HashSet<>();
            sentinelSet.add("hadoop102:26379");
            jedisSentinelPool = new JedisSentinelPool("mymaster", sentinelSet, jedisPoolConfig);
            return jedisSentinelPool.getResource();
        } else {
            return jedisSentinelPool.getResource();
        }
    }

}
```

2）测试代码

```java
public static void main(String[] args) {
    Jedis jedis = RedisUtil.getJedis();
    jedis.set("k1000", "v1000");
    jedis.set("k2000", "v2000");
    System.out.println(jedis.get("k1000"));
    jedis.close();
}
```



## 五、集群

### 5.1、什么是Redis集群

在默认哨兵模式下，任何节点都要承当所有的数据，不做读写分离的化还要承当所有的请求，此时需要建立redis集群。把数据拆成多份分别存储到各个redis节点上。

思考：

- 容量不够，redis如何进行扩容？
- 并发写操作， redis如何分摊？

Redis集群

```txt
- Redis 集群实现了对Redis的水平扩容，即启动N个redis节点，将整个数据库分布存储在这N个节点中，每个节点存储总数据的1/N。

- Redis 同时在集群中也实现了 节点的主从复制。
```

市面上的集群方案

```
- 客户端方案，早期方案通过JedisShardInfo来实现分片
	- 问题：分片规则耦合在客户端
           需要自己实现很多功能
           不提供高可用

- 第三方代理中间件模式：twemproxy、 codis
     - 问题：成为瓶颈和风险点 
     	    版本基本上不再更新了

- redis3.0以后出的官方redis-cluster方案
      问题：有高可用，但没有读写分离
```



### 5.2、配置redis集群

这里采用redis3.0以后出的官方redis-cluster方案，**redis集群是去中心化的集群**

1）安装ruby环境（redis）

```txt
yum -y install ruby-libs ruby ruby-irb ruby-rdoc rubygems 
```

2）拷贝redis-3.2.0.gem到/opt目录下

3）执行在opt目录下执行 gem install --local redis-3.2.0.gem

4）制作6个实例，6379,6380,6381,6389,6390,6391（3主3从）

5）redis6379.conf的配置

```txt
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 15000
port 6379
daemonize yes
dir "/home/atguigu/redis-cluster"
dbfilename dump6379.rdb
pidfile "/var/run/redis_6379.pid"
protected-mode no
save 300 10
```

- cluster-enabled yes  打开集群模式
- cluster-config-file nodes-6379.conf 设定节点配置文件名
- cluster-node-timeout 15000  设定节点失联时间，超过该时间（毫秒），集群自动进行主从切换。

同时按此配置redis6380.conf、 redis6381.conf、redis6389.conf、redis6390.conf、redis6391.conf然后分别启动6个redis-server

6）启动集群（注意集群必须是新的，也就是没有数据，旧数据可以等集群建好后 再导过来）

```shell
redis-server redis-cluster6379.conf
redis-server redis-cluster6380.conf
redis-server redis-cluster6381.conf
redis-server redis-cluster6389.conf
redis-server redis-cluster6390.conf
redis-server redis-cluster6391.conf
```

![image-20200719204441385](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200719204441385.png)

![image-20200719204425184](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200719204425184.png)

7）将6个节点连成集群，找到之前安装redis的解压包，进入src文件夹，找到redis-trib.rb

```txt
/opt/module/redis-3.2.5/src/redis-trib.rb
```

注意不要随意换行

```txt
./redis-trib.rb create --replicas 1 192.168.218.102:6379 192.168.218.102:6380 192.168.218.102:6381 192.168.218.102:6389 192.168.218.102:6390 192.168.218.102:6391
```

![image-20200719211110498](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200719211110498.png)



8）redis cluster 如何分配这六个节点

```
- 一个集群至少要有三个主节点。
- 选项 --replicas 1 表示我们希望为集群中的每个主节点创建一个从节点。
- 分配原则尽量保证每个主数据库运行在不同的IP地址，每个从库和主库不在一个IP地址上
```



### 5.3、Hash槽（slots）

1）当使用redis-cli打开客户端时

```txt
redis-cli
set k1 v1  => 会报错,此key不能由当前redis节点存储
```

使用集群模式客户端启动

```txt
redis-cli -c
set k1 v1 => 会自动转到相应的redis节点存储
```

![image-20200719211258632](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200719211258632.png)

注意：keys *只能查看当前节点的数据



2）为什么会出现这现象？这就得了解数据是如何存储到集群上的

- 启动集群时，会生成16384个Hash槽（slots）

![image-20200719212444219](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200719212444219.png)

- 查看集群节点

```txt
cluster nodes
```

![image-20200719211258632](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200719211450780.png)

- a2bdc86885448efc8c532498d60a00f248e7234a这个是runid（随机生成的），从机后面还跟着主机的runid
- Master：192.168.218.102:6379的Hash槽是0-5460
- Master：192.168.218.102:6380的Hash槽是5461-10922
- Master：192.168.218.102:6381的Hash槽是10923-16383
- 根据key算出一个Hash值（crc64算法），对16384取模，根据余数判断存到哪个Matser上，这样剋以让key比较散列的分布在各个redis节点上



3）slots详解

- 一个 Redis 集群包含 16384 个插槽（hash slot）， 数据库中的每个键都属于这 16384 个插槽的其中一个， 集群使用公式 CRC16(key) % 16384 来计算键 key 属于哪个槽， 其中 CRC16(key) 语句用于计算键 key 的 CRC16 校验和 。
- 集群中的每个节点负责处理一部分插槽。 举个例子， 如果一个集群可以有主节点， 其中：

```txt
- 节点 A 负责处理 0 号至 5500 号插槽。
- 节点 B 负责处理 5501 号至 11000 号插槽。
- 节点 C 负责处理 11001 号至 16383 号插槽
```

- 在redis-cli每次录入、查询键值，redis都会计算出该key应该送往的插槽，如果不是该客户端对应服务器的插槽，redis会报错，并告知应前往的redis实例地址和端口。
- redis-cli客户端提供了 -c 参数实现自动重定向。

```txt
如 redis-cli -c -p 6379 登入后，再录入、查询键值对可以自动重定向。
```

- 不在一个slot下的键值，是不能使用mget，mset等多键操作。

```txt
redis要保证命令的原子性，如果存到不同slot中，可能会出现，一些slot存入成功，一些slot存入失败，这是不允许发生的
```

- 可以通过{}来定义组的概念，从而使key中{}内相同内容的键值对放到一个slot中去。

```txt
mset k1{g1} v1 k2{g1} v2 k3{g1} v3   => 三个key{}内相同，所以会存到一个slot上
```

原因是，如果由{}，那么就用{}内的值进行crc62运算，得到的hash值是一样，当然会分到一个slot上。但是容易造成数据倾斜。



4）slot命令

- 查看key存在哪个slot中

```txt
CLUSTER KEYSLOT <key> 计算键 key 应该被放置在哪个槽上
	- CLUSTER KEYSLOT k4       => 不带{}的
	- CLUSTER KEYSLOT k1{g1}   => 带{}的
```

- CLUSTER COUNTKEYSINSLOT <slot>  返回槽 slot 目前包含的键值对数量。

```txt
CLUSTER COUNTKEYSINSLOT 13591
```

- CLUSTER GETKEYSINSLOT <slot> <count> 返回 count 个 slot 槽中的键。

注意：如果发生数据倾斜，可以用上面三个命令去查询slot的数据，来辨别造成倾斜的原因。



### 5.4、故障恢复

1）如果主节点下线，该主节点的从节点自动升为主节点

2）如果故障的主节点恢复，会自动变成从节点

3）如果所有某一段插槽的主从节点都当掉，这时候2个选择

- 继续提供数据服务
- 不继续提供数据服务

```txt
- redis.conf中的参数  cluster-require-full-coverage 默认是yes ，如果选no那么即使某一部分的slot完全下线(包括从机)，
集群也会继续以现存的数据提供服务。 
```



### 5.5、JedisCluster访问集群

1）在RedisUtil添加集群访问方法

```java
private static JedisCluster jedisCluster = null;

    public static JedisCluster getJedisCluster() {
        if (jedisCluster == null) {
            Set<HostAndPort> hostAndPortSet = new HashSet<>();
            // 其实只需要配置一个节点（从主都行），整个集群就都可访问
            // 配置2、3个节点就行，主要是怕配置的节点突然down掉
            hostAndPortSet.add(new HostAndPort("hadoop102", 6379));
            hostAndPortSet.add(new HostAndPort("hadoop102", 6380));
            hostAndPortSet.add(new HostAndPort("hadoop102", 6381));
            JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
            jedisPoolConfig.setMaxTotal(20);
            jedisPoolConfig.setBlockWhenExhausted(true);
            jedisPoolConfig.setMaxWaitMillis(2000);
            jedisPoolConfig.setMaxIdle(5);
            jedisPoolConfig.setMinIdle(5);
            jedisPoolConfig.setTestOnBorrow(true);
            jedisCluster = new JedisCluster(hostAndPortSet, jedisPoolConfig);
            return jedisCluster;
        } else {
            return jedisCluster;
        }
    }
```

2）测试代码

```java
public static void main(String[] args) {
    JedisCluster jedisCluster = RedisUtil.getJedisCluster();
    jedisCluster.set("k111", "v111");
    jedisCluster.set("k222", "v222");
    jedisCluster.set("k333", "v333");
    System.out.println(jedisCluster.get("k111"));
    System.out.println(jedisCluster.get("k222"));
    System.out.println(jedisCluster.get("k333"));
}
```



## 六、Redis实际应用中的问题

### 6.1、缓存穿透

缓存击穿，是指一个key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库，就像在一个屏障上凿开了一个洞。

解决方案：

1）布隆过滤器

那这个布隆过滤器是如何解决redis中的缓存穿透呢？很简单首先也是对所有可能查询的参数以hash形式存储，当用户想要查询的时候，使用布隆过滤器发现不在集合中，就直接丢弃，不再对持久层查询。

![image-20200720085339632](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200720085339632.png)



2）缓存空对象

当存储层不命中后，即使返回的空对象也将其缓存起来，同时会设置一个过期时间，之后再访问这个数据将会从缓存中获取，保护了后端数据源；

![image-20200720085312064](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200720085312064.png)

但是这种方法会存在两个问题：

```txt
- 如果空值能够被缓存起来，这就意味着缓存需要更多的空间存储更多的键，因为这当中可能会有很多的空值的键；
- 即使对空值设置了过期时间，还是会存在缓存层和存储层的数据会有一段时间窗口的不一致，这对于需要保持一致性的业务会有影响。
```



### 6.2、缓存雪崩问题

缓存雪崩是指，缓存层出现了错误，不能正常工作了。于是所有的请求都会达到存储层，存储层的调用量会暴增，造成存储层也会挂掉的情况。

![image-20200720085232585](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200720085232585.png)

解决方案：

1）redis高可用

这个思想的含义是，既然redis有可能挂掉，那我多增设几台redis，这样一台挂掉之后其他的还可以继续工作，其实就是搭建的集群。

2）限流降级

这个解决方案的思想是，在缓存失效后，通过加锁或者队列来控制读数据库写缓存的线程数量。比如对某个key只允许一个线程查询数据和写缓存，其他线程等待。

3）数据预热

数据加热的含义就是在正式部署之前，我先把可能的数据先预先访问一遍，这样部分可能大量访问的数据就会加载到缓存中。在即将发生大并发访问前手动触发加载缓存不同的key，设置不同的过期时间，让缓存失效的时间点尽量均匀。






参考相关链接：

https://zhuanlan.zhihu.com/p/73361428

https://www.cnblogs.com/xichji/p/11286443.html

https://baijiahao.baidu.com/s?id=1655304940308056733&wfr=spider&for=pc