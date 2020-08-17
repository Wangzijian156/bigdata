# Redis基础

- [Redis基础](#redis基础)
	- [一、Redis简介](#一redis简介)
		- [1.1、什么是Redis](#11什么是redis)
		- [1.2、Redis与Memcache的区别](#12redis与memcache的区别)
		- [1.3、Redis与HBase的区别](#13redis与hbase的区别)
		- [1.4、Redis能干嘛](#14redis能干嘛)
	- [二、Redis安装与启动](#二redis安装与启动)
	- [三、Redis基本操作](#三redis基本操作)
		- [3.1、Redis的数据库与选择](#31redis的数据库与选择)
		- [3.2、Redis五大数据类型](#32redis五大数据类型)
		- [3.3、位运算](#33位运算)
		- [3.4、常见配置](#34常见配置)
		- [3.5、Jedis的使用](#35jedis的使用)

## 一、Redis简介

### 1.1、什么是Redis

Redis是用C语言开发的，遵循BSD协议，是一个高性能的（Key/Value）分布式内存数据库，基于内存运行。

### 1.2、Redis与Memcache的区别

- Redis几乎覆盖了Memcached的绝大部分功能

- Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载
- Redis不仅仅支持简单的KV类型数据，同时还支持list、set、zset、hash等数据结构的存储
- Redis支持数据的备份，即master-slave模式的数据备份

### 1.3、Redis与HBase的区别

- 共同点

```txt
基于key-value方式的存储，不适合进行分析查询
```

- 不同点

```txt
- 数据量：HBase数据量远远大于redis
- 性能：Redis存取效率更高，使用经常变化的数据
- 数据存储时效性：Redis更适合用于高频访问的临时数据，HBase更适合长期存储
```

### 1.4、Redis能干嘛

- 配合关系型数据库做高速缓存

```txt
- 高频次，热门访问的数据，降低数据库IO
- 分布式结构，做session共享
```

- 实时计算中常常用于存储临时数据

```txt
- 高频次
- 读写时效性高
- 总数居量不大
- 临时性
```

- 由于其拥有持久化能力，利用其多样的数据结构存储特定的数据

```txt
- 最新N个数据：通过List实现，按自然时间排序的数据
- 排行榜，TopN：利用zset（有序集合）
- 时效性的数据，比如手机验证码：Expire过期
- 计数器，秒杀：原子性，自增方法INCR，DECR
- 去除大量数据中的重复数据：利用set
- 构建队列：利用list集合
- 发布订阅消息系统：pub/sub模式
```



## 二、Redis安装与启动

1）下载获取redis-3.2.5.tar.gz后将它放入我们的Linux目录/opt

2）解压命令：tar -zxfv redis-3.2.5.tar.gz -C /opt/module

3）解压完成进入目录cd /opt/module/redis-3.2.5

4）如果出现 "gcc：命令未找到"

```txt
sudo yum install gcc-c++
```

5）在redis-3.2.5目录下执行make（需安装gcc），继续执行make install

```txt
Jemalloc/jemalloc.h：没有那个文件
- 解决方案：运行make distclean之后再 make
```

6）查看安装目录/usr/local/bin

```txt
- Redis-benchmark: 性能测试工具
- Redis-check-aof: AOF文件
- Redis-check-dump：修复有问题的dump.rdb文件
- redis-server:Redis服务器启动命令
- Redis-cli：客户端，操作入口
```

7）备份redis.conf：拷贝一份redis.conf到其他目录

8）修改redis.conf文件将里面daemonize no 改成 yes，让服务在后台启动

9）启动命令：执行redis-server /myredis/redis.conf

10）用客户端访问：redis-cli

![image-20200714222523582](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis//image-20200714222523582.png)

11）ping

![image-20200714222545754](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis//image-20200714222545754.png)





## 三、Redis基本操作

### 3.1、Redis的数据库与选择

```
- 默认16个数据库
- 统一密码管理，所有库都是同样密码，要么都OK要么一个也连接不上。
- 使用命令 select   <dbid>  来切换数据库。如: select 8 
```

### 3.2、Redis五大数据类型

1）key概述

```txt
- keys * 查询库里所有键
- type <key>查看类型
- del <key>删除某个键
- expire <key> <seconds> 设置过期时间
- ttl <key> 查看还有多少秒过期，-1表示永不过期，-2表示已过期
- dbsize 查看当前数据库的key的数量
- flushdb 清空当前库
- flushall 通杀全部库 
```

2）String

- String类型是二进制安全的。意味着Redis的String可以包含任何数据。比如jpg图片或者序列化对象。
- String类型是Redis最基本的数据类型，一个Redis中字符串value最多可以是512M
- 命令

```txt
- get <key>  
	- 查询对应键值

- set <key> <value> 
	- 添加键值对
	
- setnx <key> <value>
	- (key不存在时设置key)

- append <key> <value> 
	- 指定的 key 追加值
	- 如果key 已经存在并且是一个字符串， APPEND 命令将 value 追加到 key 原来的值的末尾。
	- 如果 key 不存在， APPEND 就简单地将给定 key 设为 value ，就像执行 SET key value 一样

- strlen <key> 
	- 获取指定 key 所储存的字符串值的长度

- incr <key>
	- (自增)

- decr <key>
	- (自减)

- incrby/decrby <key> <步长>
	- (规定自减自增多少)

- mset <key1> <value1> <key2> <value2> ...  
	- 同时设置一个或多个 key-value对  

- mget <key1> <key2> <key3> ... 
	- 同时获取一个或多个 value  

- msetnx <key1> <value1> <key2> <value2> ...
	- 所有给定 key 都不存在时，同时设置一个或多个 key-value 对。
	- 如果key存在执行失败

- getrange <key> <start> <end>
	- 获取 key 中字符串的子字符串

- setrange <key> <offset> <str>
	- 覆盖 key 所储存的字符串值，覆盖的位置从偏移量 offset 开始

- setex <key> 
	- <过期时间> <value> 设置key的同时，设置国企时间，单位秒

- getset <key> <value> 
	- 以新换旧，设置了新值同时活得旧值
```

- 原子性：所谓原子操作是指不会被线程调度机制打断的操作；这种操作一旦开始，就一直运行到结束，中间不会有任何 context switch （切换到另一个线程）

```txt
- 在单线程中， 能够在单条指令中完成的操作都可以认为是" 原子操作"，因为中断只能发生于指令之间
- 在多线程中，不能被其它进程（线程）打断的操作就叫原子操作
```

3）list：双向链表，对两端的操作性能很高，通过索引下标的操作中间的节点性能会较差

```txt
- lpush/rpush <key> <value1> <value2>...
	- 从左边/右边插入一个或多个值
	
- lpop/rpop <key> 
	- 从左边/右边吐出一个值
	- 值在键在，值亡键亡
	
- rpoplpush <key1> <key2>  
	-key1右边吐一个到key2左边
	
- lrange <key> <start> <stop> 
	- 遍历start到stop
	
- lindex <key> <index> 
	- 按照索引下标获得元素(从左到右)

- llen <key>
	- 获得列表长度 

- linsert <key> before <value> <newvalue>  
	- 在<value>的前面插入<newvalue>
	
- lrem <key> <n> <value> 
	- 从左边删除n个value（从左到右）
```

4）set：自动去重，并且set提供了判断某个成员是否在一个set集合内的重要接口，这个也是list所不能提供的。

Redis的Set是string类型的无序集合。它底层其实是一个value为null的hash表,所以添加，删除，查找的复杂度都

是O(1)。

```txt
- sadd <key> <value1> <value2> .....   
	- 将一个或多个 member 元素加入到集合 key 当中，已经存在于集合的 member 元素将被忽略
	
- smembers <key>
	- 取出该集合的所有值

- sismember <key>  <value>
 	- 判断集合<key>是否为含有该<value>值，有返回1，没有返回0
 	
- srem <key> <value1> <value2> ....
	- 删除集合中的某个元素

- sinter <key1> <key2>  
	- 返回两个集合的交集元素

- sunion <key1> <key2>  
	- 返回两个集合的并集元素

- sdiff <key1> <key2>  
 	- 返回两个集合的差集元素
```

5）hash

- Redis hash是一个string类型的field和value的映射表，hash特别适合用于存储对象。
- 类似Java里面的Map<String,Object>
- 命令

```txt
- hset <key>  <field>  <value>
	- 给<key>集合中的  <field>键赋值<value>
	
- hget <key1>  <field>   
	- 从<key1>集合<field> 取出 value 
	
- hmset <key1>  <field1> <value1> <field2> <value2>...   
	- 批量设置hash的值
	
- hexists key  <field>
	- 查看哈希表 key 中，给定域 field 是否存在
	
- hgetall <key>   
	- 列出该hash集合的所有field和values

- hincrby <key> <field>  <increment> 
	- 为哈希表 key 中的域 field 的值加上增量 increment 
```

注意：尽量一次性把数据批量录入，不要单个多次录入（效率低）

hash实例：用户ID为查找的key，存储的value用户对象包含姓名，年龄，生日等信息，如果用普通的key/value结构来存储，主要有以下2种存储方式：

![image-20200716123304310](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis//image-20200716123304310.png)

方法一：每次修改用户的某个属性需要，先反序列化改好后再序列化回去。开销较大

方法二：用户ID数据冗余

![image-20200716123426306](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis//image-20200716123426306.png)

方法三：通过 key(用户ID) + field(属性标签) 就可以操作对应属性数据了，既不需要重复存储数据，也不会带来序列化和并发修改控制的问题



6）zset：和set相似没有重复元素，不同的是zset的所有成员都关联了一个评分（score），这个评分（score）被用来按照从最低分到最高分的方式排序集合中的成员。集合的成员是唯一的，但是评分可以是重复的

```
- zadd  <key> <score1> <value1>  <score2> <value2>...
	- 将一个或多个 member 元素及其 score 值加入到有序集 key 当中
	
- zrange <key>  <start> <stop>  [WITHSCORES]   
	- 返回有序集 key 中，下标在<start> <stop>之间的元素
	- 带WITHSCORES，可以让分数一起和值返回到结果集。
	- 有序输出从小到大

- zrevrange <key>  <start> <stop>  [WITHSCORES]   
	- 同上，逆序按评分从大到小

- zincrby <key> <increment> <value>
	- 为元素的score加上增量

-  zrem  <key>  <value>  
	- 删除该集合下，指定值的元素 

- zcount <key>  <min>  <max> 
	- 统计该集合，分数区间内的元素个数 

- zrank <key>  <value> 
	- 返回该值在集合中的排名，从0开始
```

实例：如何利用zset实现一个文章访问量的排行榜？

``` txt
key		=> article_top:day
value	=> id:topic
scor	=> 访问量

用户点击(访问量自增1)
zincrby article_top:daye  acti_0101:xxxx 1

展示排行榜Top10
zrevrange article_top:daye 0 9
```



### 3.3、位运算

```txt
getbit key offset
setbit key offset
```

键值对如下：

```
("mykey", "hello")
```

```txt
getbit mykey 0  => 0
getbit mykey 1  => 1
getbit mykey 2  => 1
getbit mykey 3  => 0
getbit mykey 4	=> 1     => 0X68 (十六进制) => 'h' (ASCII) => 一个字符一个字节（8bits）
getbit mykey 5	=> 0
getbit mykey 6	=> 0
getbit mykey 7	=> 0
```

修改位上的值 =>最终value也会发生改变

```txt
setbit mykey 2 1
setbit mykey 4 0
setbit mykey 5 1
```



![image-20200817100518238](https://gitee.com/wangzj6666666/bigdata-img/raw/master/redis/image-20200817100518238.png)



### 3.4、常见配置

redis.conf 配置项说明如下：

1）Redis默认不是以守护进程的方式运行，可以通过该配置项修改，使用yes启用守护进程

```txt
daemonize no
```

2）当Redis以守护进程方式运行时，Redis默认会把pid写入/var/run/redis.pid文件，可以通过pidfile指定

```txt
pidfile /var/run/redis.pid
```

3）指定Redis监听端口，默认端口为6379，作者在自己的一篇博文中解释了为什么选用6379作为默认端口，因为6379在手机按键上MERZ对应的号码，而MERZ取自意大利歌女Alessia Merz的名字

```txt
port 6379
```

4）绑定的主机地址

```
bind 127.0.0.1
- 默认情况bind=127.0.0.1只能接受本机的访问请求
- 不写的情况下，无限制接受任何ip地址的访问
- 生产环境肯定要写应用服务器的地址
```

注意：如果开启了protected-mode，那么在没有设定bind ip且没有设密码的情况下，Redis只允许接受本机的相应

5）当 客户端闲置多长时间后关闭连接，如果指定为0，表示关闭该功能

```txt
timeout 300
```

6）指定日志记录级别，Redis总共支持四个级别：debug、verbose、notice、warning，默认为verbose

```txt
loglevel verbose
```

7）日志记录方式，默认为标准输出，如果配置Redis为守护进程方式运行，而这里又配置为日志记录方式为标准输出，则日志将会发送给/dev/null

```txt
logfile stdout
```

8）设置数据库的数量，默认数据库为0，可以使用SELECT <dbid>命令在连接上指定数据库id

```txt
databases 16
```

9）指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合

```txt
save <seconds> <changes>
Redis默认配置文件中提供了三个条件：
save 900 1
save 300 10
save 60 10000
分别表示900秒（15分钟）内有1个更改，300秒（5分钟）内有10个更改以及60秒内有10000个更改。
```

10）指定存储至本地数据库时是否压缩数据，默认为yes，Redis采用LZF压缩，如果为了节省CPU时间，可以关闭该选项，但会导致数据库文件变的巨大

```txt
rdbcompression yes
```

11）指定本地数据库文件名，默认值为dump.rdb

```txt
dbfilename dump.rdb
```

12）指定本地数据库存放目录

```txt
dir ./
```

13）设置当本机为slav服务时，设置master服务的IP地址及端口，在Redis启动时，它会自动从master进行数据同步

```
slaveof <masterip> <masterport>
```

14）当master服务设置了密码保护时，slav服务连接master的密码

```txt
masterauth <master-password>
```

13）设置Redis连接密码，如果配置了连接密码，客户端在连接Redis时需要通过AUTH <password>命令提供密码，默认关闭

```txt
requirepass foobared
- 修改myredis.config配置文件，添加 requirepass 密码
- 重新启动redis-server
- 重新开启redis-cli，执行set k1 v1报错 (error) NOAUTH Authentication required
- 输入密码Auth 123456（本次连接不用再输）
```

14）设置同一时间最大客户端连接数，默认无限制，Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件描述符数，如果设置 maxclients 0，表示不作限制。当客户端连接数到达限制时，Redis会关闭新的连接并向客户端返回max number of clients reached错误信息

```txt
maxclients 128
```

15）指定Redis最大内存限制，Redis在启动时会把数据加载到内存中，达到最大内存后，Redis会先尝试清除已到期或即将到期的Key，当此方法处理 后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。Redis新的vm机制，会把Key存放内存，Value会存放在swap区

```txt
maxmemory <bytes>
```

16）指定是否在每次更新操作后进行日志记录，Redis在默认情况下是异步的把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失。因为 redis本身同步数据文件是按上面save条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认为no

```txt
appendonly no
```

17）指定更新日志文件名，默认为appendonly.aof

```txt
appendfilename appendonly.aof
```

指定更新日志条件，共有3个可选值： 

```txt
no：表示等操作系统进行数据缓存同步到磁盘（快） 
always：表示每次更新操作后手动调用fsync()将数据写到磁盘（慢，安全） 
everysec：表示每秒同步一次（折衷，默认值）
appendfsync everysec
```

18）指定是否启用虚拟内存机制，默认值为no，简单的介绍一下，VM机制将数据分页存放，由Redis将访问量较少的页即冷数据swap到磁盘上，访问多的页面由磁盘自动换出到内存中（在后面的文章我会仔细分析Redis的VM机制）

```txt
vm-enabled no
```

19）虚拟内存文件路径，默认值为/tmp/redis.swap，不可多个Redis实例共享

```txt
vm-swap-file /tmp/redis.swap
```

20）将所有大于vm-max-memory的数据存入虚拟内存,无论vm-max-memory设置多小,所有索引数据都是内存存储的(Redis的索引数据 就是keys),也就是说,当vm-max-memory设置为0的时候,其实是所有value都存在于磁盘。默认值为0

```txt
vm-max-memory 0
```

21）Redis swap文件分成了很多的page，一个对象可以保存在多个page上面，但一个page上不能被多个对象共享，vm-page-size是要根据存储的 数据大小来设定的，作者建议如果存储很多小对象，page大小最好设置为32或者64bytes；如果存储很大大对象，则可以使用更大的page，如果不 确定，就使用默认值

```txt
vm-page-size 32
```

22）设置swap文件中的page数量，由于页表（一种表示页面空闲或使用的bitmap）是在放在内存中的，，在磁盘上每8个pages将消耗1byte的内存。

```txt
vm-pages 134217728
```

23）设置访问swap文件的线程数,最好不要超过机器的核数,如果设置为0,那么所有对swap文件的操作都是串行的，可能会造成比较长时间的延迟。默认值为4

```txt
vm-max-threads 4
```

24）设置在向客户端应答时，是否把较小的包合并为一个包发送，默认为开启

```txt
glueoutputbuf yes
```

25）指定在超过一定的数量或者最大的元素超过某一临界值时，采用一种特殊的哈希算法

```txt
hash-max-zipmap-entries 64
hash-max-zipmap-value 512
```

26）指定是否激活重置哈希，默认为开启（后面在介绍Redis的哈希算法时具体介绍）

```txt
activerehashing yes
```

27）指定包含其它的配置文件，可以在同一主机上多个Redis实例之间使用同一份配置文件，而同时各个实例又拥有自己的特定配置文件

```t'x't
include /path/to/local.conf
```



### 3.5、Jedis的使用

1）idea创建空的maven项目

2）导入jar包

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.2.0</version>
</dependency>
```

3）禁用Linux的防火墙：Linux里执行命令 **service** **iptables** **stop**

4）myredis.conf中注释掉bind 127.0.0.1 ,然后 protect-mode no

5）Jedis的使用

```java
 public static void main(String[] args) {
     //连接本地的 Redis 服务
     Jedis jedis = new Jedis("hadoop102", 6379);
     //查看服务是否运行，打出pong表示OK
     System.out.println("connection is OK==========>: " + jedis.ping());

     // 查看说有key
     Set<String> keys = jedis.keys("*");
     for (Iterator iterator = keys.iterator(); iterator.hasNext(); ) {
         String key = (String) iterator.next();
         System.out.println(key);
     }
     System.out.println("jedis.exists====>" + jedis.exists("k2"));
     System.out.println(jedis.ttl("k1"));

     // String
     System.out.println(jedis.get("k1"));
     jedis.set("k4", "k4_Redis");
     System.out.println("----------------------------------------");
     jedis.mset("str1", "v1", "str2", "v2", "str3", "v3");
     System.out.println(jedis.mget("str1", "str2", "str3"));

     // List
     List<String> list = jedis.lrange("mylist", 0, -1);
     for (String element : list) {
         System.out.println(element);
     }

     // zset
     jedis.zadd("zset01",60d,"v1");
     jedis.zadd("zset01",70d,"v2");
     jedis.zadd("zset01",80d,"v3");
     jedis.zadd("zset01",90d,"v4");
     Set<String> s1 = jedis.zrange("zset01",0,-1);
     for (Iterator iterator = s1.iterator(); iterator.hasNext();) {
         String string = (String) iterator.next();
         System.out.println(string);
     }

     jedis.close();
 }
```



















































redis大部分服务于短期内读操作（短期内写操作很少）



退出redis-cli，不关闭redis-server

quit 

ctrl + c

关闭redis-server

- redis-cli中输入shutdown，在退出

- redis-cli shutdown