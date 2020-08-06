# HBase安装

- [HBase安装](#hbase安装)
  - [一、HBase安装](#一hbase安装)
  - [二、HBase服务的启动](#二hbase服务的启动)
  - [三、高可用](#三高可用)

## 一、HBase安装

1）Zookeeper正常部署

首先保证Zookeeper集群的正常部署，并启动之：

```shell
[atguigu@hadoop102 zookeeper-3.5.7]$ bin/zkServer.sh start
[atguigu@hadoop103 zookeeper-3.5.7]$ bin/zkServer.sh start
[atguigu@hadoop104 zookeeper-3.5.7]$ bin/zkServer.sh start
```



2）Hadoop正常部署

Hadoop集群的正常部署并启动：

```shell
[atguigu@hadoop102 hadoop-3.1.3]$ sbin/start-dfs.sh

[atguigu@hadoop103 hadoop-3.1.3]$ sbin/start-yarn.sh
```



3）HBase的解压

解压Hbase到指定目录：

```shell
[atguigu@hadoop102 software]$ tar -zxvf hbase-2.0.5-bin.tar.gz -C /opt/module

[atguigu@hadoop102 software]$ mv /opt/module/hbase-2.0.5 /opt/module/hbase
```

配置环境变量

```shell
[atguigu@hadoop102 ~]$ sudo vim /etc/profile.d/my_env.sh
```

添加

```shell
#HBASE_HOME
export HBASE_HOME=/opt/module/hbase
export PATH=$PATH:$HBASE_HOME/bin
```



4） HBase的配置文件

修改HBase对应的配置文件。

hbase-env.sh修改内容：

```shell
export HBASE_MANAGES_ZK=false
```

hbase-site.xml修改内容：

```
<configuration>
    <property>
        <name>hbase.rootdir</name>
        <value>hdfs://hadoop102:8020/hbase</value>
    </property>

    <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
    </property>

    <property>
        <name>hbase.zookeeper.quorum</name>
        <value>hadoop102,hadoop103,hadoop104</value>
    </property>

    <property>
        <name>hbase.unsafe.stream.capability.enforce</name>
        <value>false</value>
    </property>
    
    <property>
        <name>hbase.wal.provider</name>
        <value>filesystem</value>
    </property>
</configuration>

```

regionservers，vim regionservers

```shell
hadoop102
hadoop103
hadoop104
```

5）HBase远程发送到其他集群

```shell
[atguigu@hadoop102 module]$ xsync hbase/
```



## 二、HBase服务的启动

1）单点启动

```shell
[atguigu@hadoop102 hbase]$ bin/hbase-daemon.sh start master

[atguigu@hadoop102 hbase]$ bin/hbase-daemon.sh start regionserver
```

提示：如果集群之间的节点时间不同步，会导致regionserver无法启动，抛出ClockOutOfSyncException异常。

修复提示：

a、同步时间服务

```shell
vim /home/atguigu/bin/dt.sh
```

```shell
#!/bin/bash

for i in hadoop102 hadoop103 hadoop104
do
    echo "========== $i =========="
    ssh -t $i "sudo date -s $1"
done

```

设置执行权限

```shell
sudo chmod 777 dt.sh
```

```
dt.sh 2020-07-05
```

b、属性：hbase.master.maxclockskew设置更大的值

```xml
 <property>
        <name>hbase.master.maxclockskew</name>
        <value>180000</value>
        <description>Time difference of regionserver from master</description>
</property>

```

2）群启

```shell
[atguigu@hadoop102 hbase]$ bin/start-hbase.sh
```

对应的停止服务：

```shell
[atguigu@hadoop102 hbase]$ bin/stop-hbase.sh
```

HBase客户端

```shell
[atguigu@hadoop102 hbase]$ hbase shell
```

3）查看HBase页面

启动成功后，可以通过“host:port”的方式来访问HBase管理页面，例如：http://hadoop102:16010



## 三、高可用

在HBase中HMaster负责监控HRegionServer的生命周期，均衡RegionServer的负载，如果HMaster挂掉了，那么整个HBase集群将陷入不健康的状态，并且此时的工作状态并不会维持太久。所以HBase支持对HMaster的高可用配置。

1）关闭HBase集群（如果没有开启则跳过此步）

```shell
[atguigu@hadoop102 hbase]$ bin/stop-hbase.sh
```

2）在conf目录下创建backup-masters文件

```shell
[atguigu@hadoop102 hbase]$ touch conf/backup-masters
```

3.在backup-masters文件中配置高可用HMaster节点

```shell
[atguigu@hadoop102 hbase]$ echo hadoop103 > conf/backup-masters
```

4.将整个conf目录scp到其他节点

```shell
[atguigu@hadoop102 hbase]$ xsync /opt/module/hbase/conf/backup-masters
```



