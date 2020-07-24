# ES Restful API (DSL) 高级

- [ES Restful API (DSL) 高级](#es-restful-api-dsl-高级)
  - [一、SQL](#一sql)
  - [二、中文分词](#二中文分词)
  - [三、Mapping](#三mapping)
    - [3.1、什么是mapping](#31什么是mapping)
    - [3.2、基于中文分词搭建索引](#32基于中文分词搭建索引)
  - [四、分割索引](#四分割索引)
  - [五、索引别名](#五索引别名)
    - [5.1、什么是索引别名](#51什么是索引别名)
    - [5.2、创建索引别名](#52创建索引别名)
    - [5.3、查询别名](#53查询别名)
    - [5.4、删除某个索引的别名](#54删除某个索引的别名)
    - [5.5、为某个别名进行无缝切换](#55为某个别名进行无缝切换)
    - [5.6、查询别名列表](#56查询别名列表)
  - [六、索引模板](#六索引模板)
  - [七、Shard的划分](#七shard的划分)
    - [7.1、shard太多带来的危害](#71shard太多带来的危害)
    - [7.2、如何规划shard数量](#72如何规划shard数量)
    - [7.3、对Segment(段)的优化](#73对segment段的优化)
  - [八、 ES客户端](#八-es客户端)
    - [8.1、关于ES 客户端的选择](#81关于es-客户端的选择)
    - [8.2、在测试类中测试ES](#82在测试类中测试es)
    - [8.3、写操作](#83写操作)
    - [8.4、读操作](#84读操作)
    - [8.5、批次写入](#85批次写入)

## 一、SQL

ElasticSearch SQL是6.3版本以后的功能，能够支持一些最基本的SQL查询语句。

- 只支持select操作 ，insert， update， delete 一律不支持。
- 6.3以前的版本无法支持。
- SQL比DSL有丰富的函数。参考：https://www.elastic.co/guide/en/elasticsearch/reference/6.6/sql-functions.html
- 不支持窗口函数
- SQL少一些特殊功能，比如高亮，分页

```json
GET _xpack/sql?format=txt
{
  "query":"select actorList.name,avg(doubanScore) from movie_index  where match(name,'red') group by actorList.name " 
}
```



## 二、中文分词

elasticsearch本身自带的中文分词，就是单纯把中文一个字一个字的分开，根本没有词汇的概念。但是实际应用中，用户都是以词汇为条件，进行查询匹配的，如果能够把文章以词汇为单位切分开，那么与用户的查询条件能够更贴切的匹配上，查询速度也更加快速。

分词器下载网址：https://github.com/medcl/elasticsearch-analysis-ik

下载好的zip包，请解压后放到 …/elasticsearch/plugins/ik

![image-20200722145324315](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200722145324315.png)

分发给三台服务器

重启ES

使用分词器

```json
GET movie_index/_analyze  {   
    "analyzer": "ik_smart",      
    "text": "我是中国人"  
}  
```

![image-20200722145502160](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200722145502160.png)

另外一个分词器

```json
GET movie_index/_analyze
{  
    "analyzer": "ik_max_word", 
  	"text": "我是中国人"
}
```

![image-20200722145538413](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200722145538413.png)

能够看出不同的分词器，分词有明显的区别，所以以后定义一个type不能再使用默认的mapping了，要手工建立mapping, 因为要选择分词器。



## 三、Mapping

### 3.1、什么是mapping

之前说type可以理解为table，那每个字段的数据类型是如何定义的呢，实际上每个type中的字段是什么数据类型，由mapping定义。

查看mapping

```txt
GET movie_index/_mapping/movie
```

 ![image-20200722165853503](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200722165853503.png)

如果没有设定mapping系统会自动，根据一条数据的格式来推断出应该的数据格式。

```
true/false => boolean
1020 => long
20.1 => float
“2018-02-01” => date
“hello world” => text +keyword
```

mapping除了自动定义，还可以手动定义，但是只能对新加的、没有数据的字段进行定义。一旦有了数据就无法再做修改了。

注意：

- 默认只有text会进行分词，keyword是不会分词的字符串。

- 虽然每个Field的数据放在不同的type下,但是同一个名字的Field在一个index下只能有一种mapping定义。





### 3.2、基于中文分词搭建索引

注意：ES中数据类型一旦声明就不能改了

```json
PUT movie_chn
{  "settings": {                                               
    "number_of_shards": 1
  },
  "mappings": {
    "movie":{
      "properties": {
        "id":{
          "type": "long"
        },
        "name":{
          "type": "text"
          , "analyzer": "ik_smart"
        },
        "doubanScore":{
          "type": "double"
        },
        "actorList":{
          "properties": {
            "id":{
              "type":"long"
            },
            "name":{
              "type":"keyword"
            }
          }
        }
      }
    }
  }
}
```

插入数据

```json
PUT /movie_chn/movie/1
{ 
  "id":1,
  "name":"红海行动",
  "doubanScore":8.5,
  "actorList":[  
    {"id":1,"name":"张译"},
    {"id":2,"name":"海清"},
    {"id":3,"name":"张涵予"}
  ]
}
PUT /movie_chn/movie/2
{
  "id":2,
  "name":"湄公河行动",
  "doubanScore":8.0,
  "actorList":[  
    {"id":3,"name":"张涵予"}
  ]
}

PUT /movie_chn/movie/3
{
  "id":3,
  "name":"红海事件",
  "doubanScore":5.0,
  "actorList":[  
    {"id":4,"name":"张晨"}
  ]
}
```

查询测试

```json
GET /movie_chn/movie/_search
{
  "query": {
    "match": {
      "name": "红海战役"
    }
  }
}

GET /movie_chn/movie/_search
{
  "query": {
    "term": {
      "actorList.name": "张译"
    }
  }
}
```

此时再查看下分词

```
GET movie_chn/_analyze 
{   
    "analyzer": "ik_smart",      
    "text": "红海行动"  
}  
```

![image-20200722165806985](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200722165806985.png)





## 四、分割索引

分割索引就是根据时间间隔把一个业务索引切分成多个索引（对比hive中的日期分区表）。比如 把order_info 变成 order_info_20200101,order_info_20200102 …..

这样做的好处有两个：

```txt
- 查询范围优化：因为一般情况并不会查询全部时间周期的数据，那么通过切分索引，物理上减少了扫描数据的范围，也是对性能的优化。

- 结构变化的灵活性：因为ES不允许对数据结构进行修改。但是实际使用中索引的结构和配置难免变化，那么只要对下一个间隔的索引进行修改，
  原来的索引位置原状。这样就有了一定的灵活性。
```



## 五、索引别名

### 5.1、什么是索引别名

索引别名（_aliases）：就像一个快捷方式或软连接，可以指向一个或多个索引，也可以给任何一个需要索引名的API来使用。**别名** 带给我们极大的灵活性，允许我们做下面这些：

1）给多个索引分组 

```txt
例如，订单索引，按日期分割，可以把2020-01内的所有索引表起一个别名"order2020-01"，通过这个别名可以直接查出1月份的所有数据
```

2）给索引的一个子集创建视图

```txt
例如，如果想查询2020-01北京地区的订单详情，可以在"order2020-01"下添加一个条件"BeiJing"，形成一个子集，以后直接使用子集就行
```

3）在运行的集群中可以无缝的从一个索引切换到另一个索引

```txt
例如，现在想给索引order2020-01-01增加一个字段，如果不用别名，就得新创建一个索引（索引名改变），将旧数据倒过来。因为索引名改了
业务代码也得改，怎么办？业务代码中使用别名，就算索引换了，业务代码也不用切换。
```



### 5.2、创建索引别名

1）建表时直接声明

```json
PUT movie_chn_2020
{  
  "aliases": {
      "movie_chn_2020-query": {}
  }, 
  "mappings": {
    "movie":{
      "properties": {
        "id":{
          "type": "long"
        },
        "name":{
          "type": "text"
          , "analyzer": "ik_smart"
        },
        "doubanScore":{
          "type": "double"
        },
        "actorList":{
          "properties": {
            "id":{
              "type":"long"
            },
            "name":{
              "type":"keyword"
            }
          }
        }
      }
    }
  }
}
```

2）为已存在的索引增加别名

```json
POST  _aliases
{
  "actions": [
    { 
      "add": { 
        "index": "movie_chn", 
        "alias": "movie_chn_2020-query" 
      }
    }
  ]
}
```

3）也可以通过加过滤条件缩小查询范围，建立一个子集视图

```json
POST  _aliases
{
  "actions": [
    { 
      "add": { 
        "index": "movie_chn", 
        "alias": "movie_chn0919-query-zhhy",
        "filter": {
            "term": {  "actorList.id": "3"}
        }
      }
    }
  ]
}
```



### 5.3、查询别名 

查询别名，与使用普通索引没有区别

```txt
GET movie_chn_2020-query/_search
```



### 5.4、删除某个索引的别名

```json
POST  _aliases
{
    "actions": [
        { "remove":    { "index": "movie_chn_xxxx", "alias": "movie_chn_2020-query" }}
    ]
}
```



### 5.5、为某个别名进行无缝切换

```txt
POST /_aliases
{
    "actions": [
        { "remove": { "index": "movie_chn_xxxx", "alias": "movie_chn_2020-query" }},
        { "add":    { "index": "movie_chn_yyyy", "alias": "movie_chn_2020-query" }}
    ]
}
```



### 5.6、查询别名列表

```txt
GET  _cat/aliases?v
```



## 六、索引模板

1）创建模板

Index Template 索引模板，顾名思义，就是创建索引的模具，其中可以定义一系列规则来帮助我们构建符合特定业务需求的索引的 mappings 和 settings，通过使用 Index Template 可以让我们的索引具备可预知的一致性。

```json
PUT _template/template_movie2020
{
  "index_patterns": ["movie_test*"],                  
  "settings": {                                               
    "number_of_shards": 1
  },
  "aliases" : { 
    "{index}-query": {},
    "movie_test-query":{}
  },
  "mappings": {                                          
  "_doc": {
      "properties": {
        "id": {
          "type": "keyword"
        },
        "movie_name": {
          "type": "text",
          "analyzer": "ik_smart"
        }
      }
    }
  }
}
```

其中 "index_patterns": ["movie_test*"], 的含义就是凡是往movie_test开头的索引写入数据时，如果索引不存在，那么es会根据此模板自动建立索引。在 "aliases" 中用{index}表示，获得真正的创建的索引名。

如果报错：analyzer [ik_smart] not found for field ，就是分词插件没有安装好

![image-20200722213447783](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200722213447783.png)

解决思路：

- 确认插件路径/opt/module/elasticsearch-6.6.0/plugins/ik是否正确，插件jar包和配置文件都在ik文件夹中，plugins文件夹中不能有多余的文件

- 检查ik是否分发到3个服务器
- 检查配置文件/opt/module/elasticsearch-6.6.0/plugins/ik/config/IKAnalyzer.cfg.xml

```txt
<!--用户可以在这里配置远程扩展字典 -->
<entry key="remote_ext_dict">http://192.168.218.1:8090/dict</entry>
```



2）测试：

```txt
POST movie_test_2020xxxx/_doc
{
  "id":"333",
  "name":"zhang3"
}
```

3）查看系统中已有的模板清单

```
GET  _cat/templates
```

4）查看模板详情

```txt
GET  _template/template_movie2020
或者
GET  _template/template_movie*
```



## 七、Shard的划分

### 7.1、shard太多带来的危害

由于索引是以天为单位进行建立，如果业务线很多，每个索引又不注意控制分片，日积月累下来一个集群几万到几十万个分片也是不难见到的。每个分片都有Lucene索引，这些索引都会消耗cpu和内存。shard的目的是为了负载均衡让每个节点的硬件充分发挥，但是如果分片多，在单个节点上的多个shard同时接受请求，并对本节点的资源形成了竞争，实际上反而造成了内耗。



### 7.2、如何规划shard数量

1）根据每日数据量规划shard数量

以按天划分索引情况为例，单日数据评估低于10G的情况可以只使用一个分片，高于10G的情况， 单一分片也不能太大不能超过30G。所以一个200G的单日索引大致划分7-10个分片。

2）根据堆内存规划shard数量

另外从堆内存角度，一个Elasticsearch服务官方推荐的最大堆内存是32G。一个10G分片，大致会占用30-80M堆内存，那么对于这个32G堆内存的节点，最好不要超过1000个分片。

3）及时归档冷数据。

4）降低单个分片占用的资源消耗，具体方法就是合并分片中多个segment。



### 7.3、对Segment(段)的优化

由于es的异步写入机制，后台每一次把写入到内存的数据refresh（默认每秒）到磁盘，都会在对应的shard上产生一个segment。

1）segment太多的危害：

这个segment上的数据有着独立的Lucene索引。日积月累，如果一个shard是由成千上万的segment组成，那么性能一定非常不好，而且消耗内存也大。

2）官方优化：

如果尽可能的把小的segment合并成为大segment，这样既节省内存又提高查询性能。于是Es 后台实际上会周期性的执行合并segment的任务。

3）问题：

但是由于ES担心这种合并操作会占用资源，影响搜索性能（事实上也确实很有影响尤其是在有写操作的时候）。所以有很多内置的限制和门槛设置的非常保守，导致很久不会触发合并，或者合并效果不理想。

4）手动优化：

可如果是以天为单位切割索引的话，我们其实是可以明确的知道只要某个索引只要过了当天，就几乎不会再有什么写操作了，这个时候其实是一个主动进行合并的好时机。

可以利用如下语句主动触发合并操作：

```
POST  movie_index0213/_forcemerge?max_num_segments=1
```

  验证查询语句：

```txt
GET _cat/indices/?s=segmentsCount:desc&v&h=index,segmentsCount,segmentsMemory,memoryTotal,storeSize,p,r
```

 

##  八、 ES客户端

### 8.1、关于ES 客户端的选择

目前市面上有两类客户端

一类是TransportClient 为代表的ES原生客户端，不能执行原生dsl语句必须使用它的Java api方法。

另外一种是以Rest Api为主的missing client，最典型的就是jest。 这种客户端可以直接使用dsl语句拼成的字符串，直接传给服务端，然后返回json字符串再解析。

两种方式各有优劣，但是最近elasticsearch官网，宣布计划在7.0以后的版本中废除TransportClient。以RestClient为主。

 

### 8.2、在测试类中测试ES

 添加maven以来（Scala项目）

```xml
<dependency>
    <groupId>io.searchbox</groupId>
    <artifactId>jest</artifactId>
    <version>5.3.3</version>

</dependency>

<dependency>
    <groupId>net.java.dev.jna</groupId>
    <artifactId>jna</artifactId>
    <version>4.5.2</version>
</dependency>

<dependency>
    <groupId>org.codehaus.janino</groupId>
    <artifactId>commons-compiler</artifactId>
    <version>2.7.8</version>
</dependency>

<!-- https://mvnrepository.com/artifact/org.elasticsearch/elasticsearch -->
<dependency>
    <groupId>org.elasticsearch</groupId>
    <artifactId>elasticsearch</artifactId>
    <version>2.4.6</version>
</dependency>

```

创建 EsClient(ES客户端管理类)

```java
import java.text.SimpleDateFormat
import java.util
import java.util.Date

import io.searchbox.client.{JestClient, JestClientFactory}
import io.searchbox.client.config.HttpClientConfig
import io.searchbox.core.{Index, Search, SearchResult}
import org.elasticsearch.index.query.{BoolQueryBuilder, MatchQueryBuilder, RangeQueryBuilder}
import org.elasticsearch.search.builder.SearchSourceBuilder
import org.elasticsearch.search.highlight.HighlightBuilder
import org.elasticsearch.search.sort.SortOrder

import scala.collection.mutable.ListBuffer

object EsClient {

  private var factory: JestClientFactory = null;

  def getClient: JestClient = {
    if (factory == null) build();
    factory.getObject

  }

  /**
   * 构建ES连接池以及配置
   */
  def build(): Unit = {
    factory = new JestClientFactory
    factory.setHttpClientConfig(new HttpClientConfig.Builder("http://hadoop102:9200")
      .multiThreaded(true)          // 是否需要多线程
      .maxTotalConnection(20)   	// 线程池中的线程数
      .connTimeout(10000)           // 连接超时时间
      .readTimeout(1000)            // 读超时时间
      .build())
  }

}
```

### 8.3、写操作

```json
put /movie_index/movie/4
{
  "id":"1001",
  "name":"复仇者联盟4",
  "doubanScore":9.0,
  "actorList":[  
		{"id":"1","name":"小罗伯特·唐尼"},
		{"id":"2","name":"克里斯·埃文斯"},
		{"id":"3","name":"马克·鲁法洛"}
	] 
}
```

Scala代码

```scala
object Test {
  def main(args: Array[String]): Unit = {
    val jestClient = EsClient.getClient

    import java.util
    val list = new util.ArrayList[Actor]
    list.add(Actor(1L, "小罗伯特·唐尼"))
    list.add(Actor(2L, "克里斯·埃文斯"))
    list.add(Actor(3L, "马克·鲁法洛"))

    // 配置索引数据
    val index = new Index.Builder(Movie(100L, "复仇者联盟4", 8.5f, list))
      .index("movie_index")
      .`type`("movie")
      .id("4")
      .build();

    // 写操作
    jestClient.execute(index)
    jestClient.close()
  }

  case class Movie(id: Long, name: String, doubanScore: Double, actionNameList: java.util.List[Actor]) {}

  case class Actor(id: Long, name: String)

}
```

### 8.4、读操作

```json
GET movie_index/movie/_search
{
  "query":{
    "match": {"name":"复仇"}
  }
}
```

1）：将"query"整个转换成字符串

```scala
object Test2 {
  def main(args: Array[String]): Unit = {
    val jestClient = EsClient.getClient
    var query = "{\n  \"query\":{\n    \"match\": {\"name\":\"复仇\"}\n  }\n}"
    val search: Search = new Search.Builder(query).addIndex("movie_index").addType("movie").build()
    val result: SearchResult = jestClient.execute(search)
    import java.util
    // getHits获取命中结果，注意Map要使用java中的Map
    val resultList: util.List[SearchResult#Hit[util.Map[String, Object], Void]] = result.getHits(classOf[util.Map[String, Object]])
    import collection.JavaConversions._
    for (hit <- resultList) {
      val source: util.Map[String, Object] = hit.source
      println(source)
    }
    jestClient.close()
  }

}
```

输出结果：

```txt
{id=100.0, name=复仇者联盟4, doubanScore=8.5, actionNameList=[{id=1.0, name=小罗伯特·唐尼}, {id=2.0, name=克里斯·埃文斯}, {id=3.0, name=马克·鲁法洛}], es_metadata_id=4}
```

2）使用工具构建query

```scala
object Test2 {
  def main(args: Array[String]): Unit = {
    val jestClient = EsClient.getClient
    // var query = "{\n  \"query\":{\n    \"match\": {\"name\":\"复仇\"}\n  }\n}"
    val searchSourceBuilder = new SearchSourceBuilder
    searchSourceBuilder.query(new MatchQueryBuilder("name", "复仇"))
    searchSourceBuilder.sort("doubanScore", SortOrder.ASC)
    searchSourceBuilder.from(0)
    searchSourceBuilder.size(20)
    val query2: String = searchSourceBuilder.toString
    val search: Search = new Search.Builder(query2).addIndex("movie_index").addType("movie").build()
    val result: SearchResult = jestClient.execute(search)
    import java.util
    // getHits获取命中结果，注意Map要使用java中的Map
    val resultList: util.List[SearchResult#Hit[util.Map[String, Object], Void]] = result.getHits(classOf[util.Map[String, Object]])
    import collection.JavaConversions._
    for (hit <- resultList) {
      val source: util.Map[String, Object] = hit.source
      println(source)
    }
    jestClient.close()
  }

}
```

### 8.5、批次写入

```scala
//批次化操作  batch   Bulk
def bulkSave(list: List[(Any, String)], indexName: String): Unit = {
    if (list != null && list.size > 0) {
        val jestClient: JestClient = getJestClient()
        val bulkBuilder = new Bulk.Builder
        // 每次写入的index和type都一样，可以提出来作为一个默认值
        bulkBuilder.defaultIndex(indexName).defaultType("_doc")
        for ((doc, id) <- list) {
             //如果给id指定id 幂等性（保证精确一次消费的必要条件） 
            //不指定id 随机生成 非幂
            val index = new Index.Builder(doc).id(id).build()等性
            bulkBuilder.addAction(index)
        }
        val bulk: Bulk = bulkBuilder.build()
        val items: util.List[BulkResult#BulkResultItem] = jestClient.execute(bulk).getItems
        println("已保存" + items.size())

        jestClient.close()
    }

}
```

