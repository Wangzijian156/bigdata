# HBase 基本操作

- [HBase 基本操作](#hbase-基本操作)
  - [一、HBase基本Shell操作](#一hbase基本shell操作)
    - [1.1、客户端命令](#11客户端命令)
    - [1.2、表的操作](#12表的操作)
  - [二、HBase API](#二hbase-api)
    - [2.1、环境准备](#21环境准备)
    - [2.2、DDL](#22ddl)
      - [2.2.1、判断表是否存在](#221判断表是否存在)
      - [2.2.2、创建表](#222创建表)
      - [2.2.3、删除表](#223删除表)
      - [2.2.4、创建命名空间](#224创建命名空间)
    - [2.3 DML](#23-dml)
      - [2.3.1 插入数据](#231-插入数据)
      - [2.3.2 单条数据查询](#232-单条数据查询)
      - [2.3.3 扫描数据](#233-扫描数据)
      - [2.3.4 删除数据](#234-删除数据)

## 一、HBase基本Shell操作

### 1.1、客户端命令

1）进入HBase客户端命令行

```shell
[atguigu@hadoop102 hbase]$ bin/hbase shell
```

2）查看帮助命令

```shell
hbase(main):001:0> help
```

3）查看当前数据库中有哪些表

```shell
hbase(main):002:0> list
```

### 1.2、表的操作

1）创建表

```shell
hbase(main):002:0> create 'student','info'
```

2）插入数据到表

```shell
hbase(main):003:0> put 'student','1001','info:sex','male'
hbase(main):004:0> put 'student','1001','info:age','18'
hbase(main):005:0> put 'student','1002','info:name','Janna'
hbase(main):006:0> put 'student','1002','info:sex','female'
hbase(main):007:0> put 'student','1002','info:age','20'
```

3）扫描查看表数据

```shell
hbase(main):008:0> scan 'student'
hbase(main):009:0> scan 'student',{STARTROW => '1001', STOPROW => '1001'}
hbase(main):010:0> scan 'student',{STARTROW => '1001'}
```

4）查看表结构

```shell
hbase(main):011:0> describe 'student'
```

5）更新指定字段的数据

```shell
hbase(main):012:0> put 'student','1001','info:name','Nick'
hbase(main):013:0> put 'student','1001','info:age','100'
```

6）查看“指定行”或“指定列族:列”的数据

```shell
hbase(main):014:0> get 'student','1001'
hbase(main):015:0> get 'student','1001','info:name'
```

7）统计表数据行数

hbase(main):021:0> count 'student'

8）删除数据

删除某rowkey的全部数据：

```shell
hbase(main):016:0> deleteall 'student','1001'
```

删除某rowkey的某一列数据：

```shell
hbase(main):017:0> delete 'student','1002','info:sex'
```

清空表数据

```shell
hbase(main):018:0> truncate 'student'
```

提示：清空表的操作顺序为先disable，然后再truncate。

9）删除表

首先需要先让该表为disable状态：

```shell
hbase(main):019:0> disable 'student'
```

然后才能drop这个表：

```shell
hbase(main):020:0> drop 'student'
```

提示：如果直接drop表，会报错：ERROR: Table student is enabled. Disable it first.

10）变更表信息

将info列族中的数据存放3个版本：

```shell
hbase(main):022:0> alter 'student',{NAME=>'info',VERSIONS=>3}
hbase(main):022:0> get 'student','1001',{COLUMN=>'info:name',VERSIONS=>3}
```



## 二、HBase API

### 2.1、环境准备

新建项目后在pom.xml中添加依赖

```xml
<dependency>
    <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-server</artifactId>
    <version>2.0.5</version>
</dependency>

<dependency>
    <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-client</artifactId>
    <version>2.0.5</version>
</dependency>

```



### 2.2、DDL

#### 2.2.1、判断表是否存在

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.NamespaceDescriptor;
import org.apache.hadoop.hbase.NamespaceExistException;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;

import java.io.IOException;

public class HBase_DDL {

    //TODO 判断表是否存在
    public static boolean isTableExist(String tableName) throws IOException {

        //1.创建配置信息并配置
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.quorum", "hadoop102,hadoop103,hadoop104");

        //2.获取与HBase的连接
        Connection connection = ConnectionFactory.createConnection(configuration);

        //3.获取DDL操作对象
        Admin admin = connection.getAdmin();

        //4.判断表是否存在操作
        boolean exists = admin.tableExists(TableName.valueOf(tableName));

        //5.关闭连接
        admin.close();
        connection.close();

        //6.返回结果
        return exists;
    }

}
```

#### 2.2.2、创建表

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.NamespaceDescriptor;
import org.apache.hadoop.hbase.NamespaceExistException;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;

import java.io.IOException;

public class HBase_DDL {

    //TODO 创建表
    public static void createTable(String tableName, String... cfs) throws IOException {

        //1.判断是否存在列族信息
        if (cfs.length <= 0) {
            System.out.println("请设置列族信息！");
            return;
        }

        //2.判断表是否存在
        if (isTableExist(tableName)) {
            System.out.println("需要创建的表已存在！");
            return;
        }

        //3.创建配置信息并配置
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.quorum", "hadoop102,hadoop103,hadoop104");

        //4.获取与HBase的连接
        Connection connection = ConnectionFactory.createConnection(configuration);

        //5.获取DDL操作对象
        Admin admin = connection.getAdmin();

        //6.创建表描述器构造器
        TableDescriptorBuilder tableDescriptorBuilder = TableDescriptorBuilder.newBuilder(TableName.valueOf(tableName));

        //7.循环添加列族信息
        for (String cf : cfs) {
            ColumnFamilyDescriptorBuilder columnFamilyDescriptorBuilder = ColumnFamilyDescriptorBuilder.newBuilder(Bytes.toBytes(cf));
            tableDescriptorBuilder.setColumnFamily(columnFamilyDescriptorBuilder.build());
        }

        //8.执行创建表的操作
        admin.createTable(tableDescriptorBuilder.build());

        //9.关闭资源
        admin.close();
        connection.close();
    }

}

```



#### 2.2.3、删除表

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.NamespaceDescriptor;
import org.apache.hadoop.hbase.NamespaceExistException;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;

import java.io.IOException;

public class HBase_DDL {

    //TODO 删除表
    public static void dropTable(String tableName) throws IOException {

        //1.判断表是否存在
        if (!isTableExist(tableName)) {
            System.out.println("需要删除的表不存在！");
            return;
        }

        //2.创建配置信息并配置
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.quorum", "hadoop102,hadoop103,hadoop104");

        //3.获取与HBase的连接
        Connection connection = ConnectionFactory.createConnection(configuration);

        //4.获取DDL操作对象
        Admin admin = connection.getAdmin();

        //5.使表下线
        TableName name = TableName.valueOf(tableName);
        admin.disableTable(name);

        //6.执行删除表操作
        admin.deleteTable(name);

        //7.关闭资源
        admin.close();
        connection.close();
    }

}

```



#### 2.2.4、创建命名空间

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.NamespaceDescriptor;
import org.apache.hadoop.hbase.NamespaceExistException;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;

import java.io.IOException;

public class HBase_DDL {

    //TODO 创建命名空间
    public static void createNameSpace(String ns) throws IOException {

        //1.创建配置信息并配置
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.quorum", "hadoop102,hadoop103,hadoop104");

        //2.获取与HBase的连接
        Connection connection = ConnectionFactory.createConnection(configuration);

        //3.获取DDL操作对象
        Admin admin = connection.getAdmin();

        //4.创建命名空间描述器
        NamespaceDescriptor namespaceDescriptor = NamespaceDescriptor.create(ns).build();

        //5.执行创建命名空间操作
        try {
            admin.createNamespace(namespaceDescriptor);
        } catch (NamespaceExistException e) {
            System.out.println("命名空间已存在！");
        } catch (Exception e) {
            e.printStackTrace();
        }

        //6.关闭连接
        admin.close();
        connection.close();

    }
}
```



### 2.3 DML

#### 2.3.1 插入数据

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.Cell;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;

import java.io.IOException;

public class HBase_DML {

    //TODO 插入数据
    public static void putData(String tableName, String rowKey, String cf, String cn, String value) throws IOException {

        //1.获取配置信息并设置连接参数
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.quorum", "hadoop102,hadoop103,hadoop104");

        //2.获取连接
        Connection connection = ConnectionFactory.createConnection(configuration);

        //3.获取表的连接
        Table table = connection.getTable(TableName.valueOf(tableName));

        //4.创建Put对象
        Put put = new Put(Bytes.toBytes(rowKey));

        //5.放入数据
        put.addColumn(Bytes.toBytes(cf), Bytes.toBytes(cn), Bytes.toBytes(value));

        //6.执行插入数据操作
        table.put(put);

        //7.关闭连接
        table.close();
        connection.close();
    }

}

```

 

#### 2.3.2 单条数据查询

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.Cell;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;

import java.io.IOException;

public class HBase_DML {

    //TODO 单条数据查询(GET)
    public static void getDate(String tableName, String rowKey, String cf, String cn) throws IOException {

        //1.获取配置信息并设置连接参数
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.quorum", "hadoop102,hadoop103,hadoop104");

        //2.获取连接
        Connection connection = ConnectionFactory.createConnection(configuration);

        //3.获取表的连接
        Table table = connection.getTable(TableName.valueOf(tableName));

        //4.创建Get对象
        Get get = new Get(Bytes.toBytes(rowKey));
        // 指定列族查询
        // get.addFamily(Bytes.toBytes(cf));
        // 指定列族:列查询
        // get.addColumn(Bytes.toBytes(cf), Bytes.toBytes(cn));

        //5.查询数据
        Result result = table.get(get);

        //6.解析result
        for (Cell cell : result.rawCells()) {
            System.out.println("CF:" + Bytes.toString(cell.getFamilyArray()) +
                    ",CN:" + Bytes.toString(cell.getQualifierArray()) +
                    ",Value:" + Bytes.toString(cell.getValueArray()));
        }

        //7.关闭连接
        table.close();
        connection.close();

    }

}

```

 

#### 2.3.3 扫描数据

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.Cell;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;

import java.io.IOException;

public class HBase_DML {

    //TODO 扫描数据(Scan)
    public static void scanTable(String tableName) throws IOException {

        //1.获取配置信息并设置连接参数
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.quorum", "hadoop102,hadoop103,hadoop104");

        //2.获取连接
        Connection connection = ConnectionFactory.createConnection(configuration);

        //3.获取表的连接
        Table table = connection.getTable(TableName.valueOf(tableName));

        //4.创建Scan对象
        Scan scan = new Scan();

        //5.扫描数据
        ResultScanner results = table.getScanner(scan);

        //6.解析results
        for (Result result : results) {
            for (Cell cell : result.rawCells()) {
                System.out.println("CF:" + Bytes.toString(cell.getFamilyArray()) +
                        ",CN:" + Bytes.toString(cell.getQualifierArray()) +
                        ",Value:" + Bytes.toString(cell.getValueArray()));
            }
        }

        //7.关闭资源
        table.close();
        connection.close();

    }

}
```



#### 2.3.4 删除数据

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.Cell;
import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;

import java.io.IOException;

public class HBase_DML {

    //TODO 删除数据
    public static void deletaData(String tableName, String rowKey, String cf, String cn) throws IOException {

        //1.获取配置信息并设置连接参数
        Configuration configuration = HBaseConfiguration.create();
        configuration.set("hbase.zookeeper.quorum", "hadoop102,hadoop103,hadoop104");

        //2.获取连接
        Connection connection = ConnectionFactory.createConnection(configuration);

        //3.获取表的连接
        Table table = connection.getTable(TableName.valueOf(tableName));

        //4.创建Delete对象
        Delete delete = new Delete(Bytes.toBytes(rowKey));

        // 指定列族删除数据
        // delete.addFamily(Bytes.toBytes(cf));
        // 指定列族:列删除数据(所有版本)
        // delete.addColumn(Bytes.toBytes(cf), Bytes.toBytes(cn));
        // 指定列族:列删除数据(指定版本)
        // delete.addColumns(Bytes.toBytes(cf), Bytes.toBytes(cn));

        //5.执行删除数据操作
        table.delete(delete);

        //6.关闭资源
        table.close();
        connection.close();

}

}
```

 