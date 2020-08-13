# Flink安装

- [Flink安装](#flink安装)
  - [一、Standalone模式](#一standalone模式)
  - [二、Yarn模式](#二yarn模式)
  - [三、配置文件解析](#三配置文件解析)

## 一、Standalone模式

1）解压缩  flink-1.10.1-bin-scala_2.12.tgz

```shell
tar -zxvf flink-1.10.1-bin-scala_2.12.tgz -C /opt/module/
```

2）修改flink-conf.yaml

```shell
vim /opt/module/flink-1.10.1/conf/conf/flink-conf.yaml
```

修改内容如下

```
jobmanager.rpc.address: hadoop102
```

3）修改 /conf/slaves

```txt
hadoop103
hadoop104
```

4）分发给另外两台机子

```shell
xsync /opt/module/flink-1.10.1/
```

5）启动（ /opt/module/flink-1.10.1/）

```txt
bin/start-cluster.sh
```

6）访问http://hadoop102:8081可以对flink集群和任务进行监控管理

7）将打包好的jar包，上传到flink

![image-20200806185019185](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200806185019185.png)



![image-20200806185731734](https://gitee.com/wangzj6666666/bigdata-img/raw/master/flink/image-20200806185731734.png)



## 二、Yarn模式

1）启动hadoop集群

2)  启动yarn-session

```shell
./yarn-session.sh  -n 2 -s 2 -jm 1024 -tm 1024 -nm test -d  
```

其中：

```txt
-n(--container)：TaskManager的数量。
-s(--slots)：	每个TaskManager的slot数量，默认一个slot一个core，默认每个taskmanager的slot的个数为1，有时可以多一些taskmanager，做冗余。
-jm：JobManager的内存（单位MB)。
-tm：每个taskmanager的内存（单位MB)。
-nm：yarn 的appName(现在yarn的ui上的名字)。 
-d：后台执行。
```

3）执行任务

```
./flink run -c com.atguigu.wc.StreamWordCount  FlinkTutorial-1.0-SNAPSHOT-jar-with-dependencies.jar --host hadoop102 –port 7777
```

4）取消yarn-session

```shell
yarn application --kill application_1577588252906_0001
```



## 三、配置文件解析

flink-conf.yaml

1）基础配置

```yaml
# jobManager 的IP地址
jobmanager.rpc.address: localhost

# JobManager 的端口号
jobmanager.rpc.port: 6123

# JobManager JVM heap 内存大小
jobmanager.heap.size: 1024m

# TaskManager JVM heap 内存大小
taskmanager.heap.size: 1024m

# 每个 TaskManager 提供的任务 slots 数量大小
taskmanager.numberOfTaskSlots: 1

# 程序默认并行计算的个数
parallelism.default: 1

# 文件系统来源
# fs.default-scheme
```

2）高可用性配置

```yaml
# 可以选择 'NONE' 或者 'zookeeper'.
# high-availability: zookeeper

# 文件系统路径，让 Flink 在高可用性设置中持久保存元数据
# high-availability.storageDir: hdfs:///flink/ha/

# zookeeper 集群中仲裁者的机器 ip 和 port 端口号
# high-availability.zookeeper.quorum: localhost:2181

# 默认是 open，如果 zookeeper security 启用了该值会更改成 creator
# high-availability.zookeeper.client.acl: open
```

3）容错和检查点 配置

```yaml
# 用于存储和检查点状态
# state.backend: filesystem

# 存储检查点的数据文件和元数据的默认目录
# state.checkpoints.dir: hdfs://namenode-host:port/flink-checkpoints

# savepoints 的默认目标目录(可选)
# state.savepoints.dir: hdfs://namenode-host:port/flink-checkpoints

# 用于启用/禁用增量 checkpoints 的标志
# state.backend.incremental: false
```

4）web 前端配置

```yaml
# 基于 Web 的运行时监视器侦听的地址.
#jobmanager.web.address: 0.0.0.0

#  Web 的运行时监视器端口
rest.port: 8081

# 是否从基于 Web 的 jobmanager 启用作业提交
# jobmanager.web.submit.enable: false
```

5）高级配置

```yaml
# io.tmp.dirs: /tmp

# 是否应在 TaskManager 启动时预先分配 TaskManager 管理的内存
# taskmanager.memory.preallocate: false

# 类加载解析顺序，是先检查用户代码 jar（“child-first”）还是应用程序类路径（“parent-first”）。 默认设置指示首先从用户代码 jar 加载类
# classloader.resolve-order: child-first


# 用于网络缓冲区的 JVM 内存的分数。 这决定了 TaskManager 可以同时拥有多少流数据交换通道以及通道缓冲的程度。 如果作业被拒绝或者您收到系统没有足够缓冲区的警告，请增加此值或下面的最小/最大值。 另请注意，“taskmanager.network.memory.min”和“taskmanager.network.memory.max”可能会覆盖此分数

# taskmanager.network.memory.fraction: 0.1
# taskmanager.network.memory.min: 67108864
# taskmanager.network.memory.max: 1073741824
```

6）Flink 集群安全配置

```yaml
# 指示是否从 Kerberos ticket 缓存中读取
# security.kerberos.login.use-ticket-cache: true

# 包含用户凭据的 Kerberos 密钥表文件的绝对路径
# security.kerberos.login.keytab: /path/to/kerberos/keytab

# 与 keytab 关联的 Kerberos 主体名称
# security.kerberos.login.principal: flink-user

# 以逗号分隔的登录上下文列表，用于提供 Kerberos 凭据（例如，`Client，KafkaClient`使用凭证进行 ZooKeeper 身份验证和 Kafka 身份验证）
# security.kerberos.login.contexts: Client,KafkaClient
```

