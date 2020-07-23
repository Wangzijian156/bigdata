# ES Restful API (DSL) 基础

- [ES Restful API (DSL) 基础](#es-restful-api-dsl-基础)
  - [一、ES的基本概念](#一es的基本概念)
    - [1.1、对象名词解释](#11对象名词解释)
    - [1.2、服务状态查询](#12服务状态查询)
  - [二、ES保存的数据结构](#二es保存的数据结构)
  - [三、索引（表）操作](#三索引表操作)
  - [四、新增文档(新增数据)](#四新增文档新增数据)
  - [五、删除操作](#五删除操作)
  - [六、修改文档（修改数据）](#六修改文档修改数据)
  - [七、查询数据](#七查询数据)
    - [7.1、按主键查询](#71按主键查询)
    - [7.2、搜索type全部数据](#72搜索type全部数据)
    - [7.3、条件查询](#73条件查询)
      - [1）全部查询](#1全部查询)
      - [2）分词查询](#2分词查询)
      - [3）词组匹配](#3词组匹配)
      - [4）fuzzy查询（单词拼写错了也能查）](#4fuzzy查询单词拼写错了也能查)
      - [5）等值查询（值相等才能查出结果）](#5等值查询值相等才能查出结果)
    - [7.4、指定查询的字段](#74指定查询的字段)
    - [7.5、高亮](#75高亮)
    - [7.6、排序](#76排序)
    - [7.7、分页](#77分页)
  - [八、过滤](#八过滤)
  - [九、根据查询条件进行 删除、修改](#九根据查询条件进行-删除修改)
  - [十、分组聚合](#十分组聚合)

DSL全称 Domain Specific language，即特定领域专用语言(和sql不一样，sql是通用的，DSL只能在ES中用)。

## 一、ES的基本概念

### 1.1、对象名词解释

| 名词     | 解释                                                         |
| -------- | ------------------------------------------------------------ |
| cluster  | 整个elasticsearch 默认就是集群状态，整个集群是一份完整、互备的数据。 |
| node     | 集群中的一个节点，一般只一个进程就是一个node                 |
| shard    | 分片，即使是一个节点中的数据也会通过hash算法，分成多个片存放，默认是5片。（7.0默认改为1片） |
| index    | index相当于table                                             |
| type     | 对表的数据再进行划分，比如“区”的概念。但是实际上对数据查询优化的作用有限，比较鸡肋的设计。（6.x只允许建一个，7.0之后被废弃） |
| document | 类似于rdbms的 row、面向对象里的object                        |
| field    | 相当于字段、属性                                             |

### 1.2、服务状态查询

1）在kibana上点击Dev Tools，选择Console输入指令

![image-20200721170552060](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200721170552060.png)



2）查询各个索引的状态

```txt
GET /_cat/indices?v
```

![image-20200721170900236](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200721170900236.png)

```txt
- health 索引的健康状态 
- index 索引名 
- pri 索引主分片数量 
- rep 索引复制分片 数 
- store.size 索引主分片 复制分片 总占用存储空间 
- pri.store.size 索引总占用空间, 不计算复制分片 占用空间
```

"."开头的都是系统用的

3）服务整体状态查询

```txt
GET /_cat/health?v
```

![image-20200721171016283](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200721171016283.png)

```txt
- cluster  集群名称
- status 集群状态 green代表健康（所有主分片都正常而且每个主分片都至少有一个副本，集群状态）；yellow代表分配了所有主分片，
  但至少缺少一个副本，此时集群数据仍旧完整，主片副本都在单点；red代表部分主分片不可用，可能已经丢失数据。
- node.total 代表在线的节点总数量
- node.data 代表在线的数据节点的数量
- shards  active_shards 存活的分片数量
- pri active_primary_shards 存活的主分片数量 正常情况下 shards的数量是pri的两倍。
- relo  relocating_shards 迁移中的分片数量，正常情况为 0
- init  initializing_shards 初始化中的分片数量 正常情况为 0
- unassign  unassigned_shards 未分配的分片 正常情况为 0
- pending_tasks 准备中的任务，任务指迁移分片等 正常情况为 0
- max_task_wait_time 任务最长等待时间
- active_shards_percent 正常分片百分比 正常情况为 100%
```

4）查询各个节点状态

```txt
GET /_cat/nodes?v
```

![image-20200721191739158](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200721191739158.png)

```txt
- heap.percent 堆内存占用百分比
- ram.percent 内存占用百分比
- cpu CPU占用百分比
- master *表示节点是集群中的主节点
- name 节点名
```



5）查询某个索引的分片情况

```
GET /_cat/shards/索引名
```

![image-20200721192049646](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200721192049646.png)

```txt
- index 索引名称
- shard 分片序号
- pri rep p表示该分片是主分片, r 表示该分片是复制分片
- store 该分片占用存储空间
- node 所属节点节点名
- docs 分片存放的文档数
```





## 二、ES保存的数据结构

现有如下有Movie、Actor两个类，对应数据库中的表如何设计

```java
public class  Movie {
	 String id;
     String name;
     Double doubanScore;
     List<Actor> actorList;
}

public class Actor{
	String id;
	String name;
}
```

1）关系型数据库：

![image-20200721201021206](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200721201021206.png)

2）ES（ES也是文件数据库）中是以Json的形式存储的

```json
{
  "id":"1",
  "name":"operation red sea",
  "doubanScore":"8.5",
  "actorList":[  
		{"id":"1","name":"zhangyi"},
		{"id":"2","name":"haiqing"},
		{"id":"3","name":"zhanghanyu"}
	] 
}
```



## 三、索引（表）操作

ES中的索引相当于Mysql中的一张表

1）添加索引

```txt
put /movie_index
```

![image-20200721194350269](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200721194350269.png)

## 四、新增文档(新增数据)

1）新增文档（幂等添加数据）

想index（表）张添加数据

```txt
put /index/type/id(主键)
json
```

```txt
put /movie_index/movie/1
{
  "id":"1",
  "name":"operation red sea",
  "doubanScore":8.5,
  "actorList":[  
		{"id":"1","name":"zhangyi"},
		{"id":"2","name":"haiqing"},
		{"id":"3","name":"zhanghanyu"}
	] 
}
```

（注意：主键和"{}"内的id不是同一个东西，"{}"内的全是普通字段，put /index/type/id 中的id才是正真数据的主键，显示成_id。）

添加成功后显示如下信息

```
{
  "_index" : "movie_index",
  "_type" : "movie",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 2,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}
```

![image-20200721210235016](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200721210235016.png)

ES添加数据时，不用知名字段的类型，ES会根据数据自动判定类型（比如没有""的时整形，带小数点的时double型等等），而且ES会为每一个字段添加索引



2）新增文档 (非幂等)

```txt
POST /movie_index/movie   
{     
	"id":3,    
	"name":"incident red sea",    
	"doubanScore":5.0,     
	"actorList":[   
		{"id":4,"name":"zhang chen"} 
	]  
} 
```

注意：POST（非幂等）没有指明主键（随机生成一个主键，不是像mysql那样自增），每多执行一次就会增加一条记录，而PUT（幂等）指明了主键，无论执行多少次，新的数据会覆盖之前的数据，由于数据没变所以结果都一样（对比mysql的insert带主键和不带主键），POST加上id也会编程（幂等）。



## 五、删除操作

1）删除索引（表）

```txt
delete /movie_index
```



2）删除文档

```txt
delete /movie_index/movie/1
```



## 六、修改文档（修改数据）

- 整体修改：和新增没有区别，必须包括全部字段（否则就会丢失数据）

```txt
PUT /movie_index/movie/3
{
  "id":"3",
  "name":"incident red sea",
  "doubanScore":5.0,
  "actorList":[  
		{"id":"1","name":"zhang chen"}
	]
}
```

其实就是整体覆盖之前的数据

- 修改某个字段（只知道部分字段，其余字段不清楚情况）

```txt
POST movie_index/movie/3/_update
{ 
  "doc": {
    "doubanScore":"7.0"
  } 
}
```



## 七、查询数据

添加测试用例

```
PUT /movie_index/movie/1
{ "id":1,
  "name":"operation red sea",
  "doubanScore":8.5,
  "actorList":[  
		{"id":1,"name":"zhang yi"},
		{"id":2,"name":"hai qing"},
		{"id":3,"name":"zhang han yu"}
	]
}
PUT /movie_index/movie/2
{
  "id":2,
  "name":"operation meigong river",
  "doubanScore":8.0,
  "actorList":[  
		"id":3,"name":"zhang han yu"}
	]
}

PUT /movie_index/movie/3
{
  "id":3,
  "name":"incident red sea",
  "doubanScore":5.0,
  "actorList":[  
		{"id":4,"name":"zhang chen"}
	]
}
```

### 7.1、按主键查询

```txt
GET movie_index/movie/1
```

### 7.2、搜索type全部数据

```txt
GET movie_index/movie/_search
```

### 7.3、条件查询

#### 1）全部查询

```txt
GET movie_index/movie/_search
{
  "query":{
    "match_all": {}
  }
}
```

#### 2）分词查询

```txt
GET movie_index/movie/_search
{
  "query":{
    "match": {"name":"operation red sea"}
  }
}
```

条件分词

```txt
operation 
red 
sea
```

数据库中

```
operation 1 2
red 1 3
sea 1 3
meigong 2
river 2
incident 3
```

最终index 1 2 3都被查出来，1命中最多，排在最前面，3排在第二

![image-20200721221942431](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200721221942431.png)

影响评分的因素

```txt
正向因素：
	命中次数、 命中长度比例
负面因素： 
    关键词在该字段的其他词条中出现的次数
```

#### 3）词组匹配

类似MySQL中的like

```txt
GET movie_index/movie/_search
{
    "query":{
      "match_phrase": {"name":"operation red"}
    }
}
```

按短语查询，不再利用分词技术，直接用短语在原始数据中匹配



#### 4）fuzzy查询（单词拼写错了也能查）

```txt
GET movie_index/movie/_search
{
    "query":{
      "fuzzy": {"name":"rad"}
    }
}
```

校正匹配分词，当一个单词都无法准确匹配，es通过一种算法对非常接近的单词也给与一定的评分，能够查询出来，但是消耗更多的性能。



#### 5）等值查询（值相等才能查出结果）

```
GET movie_index/movie/_search
{
    "query":{
      "term": {
        "actorList.id.keyword": "zhangchen"
        
      }
    }
}
```

注意：等职判断String必须使用keyword型字段，不能使用text类型字段，keyword没有进行分词，而是保留了原始值，可以用于等值判断和分组。



### 7.4、指定查询的字段

比如：只查询name、doubanScore两个字段

```txt
GET movie_index/movie/_search
{
 	"query": { "match_all": {} },
 	"_source": ["name", "doubanScore"]
}
```



### 7.5、高亮

给查询出来的结果添加前端样式

```txt
GET movie_index/movie/_search
{
    "query":{
      "match": {"name":"red sea"}
    },
    "highlight": {
      "fields": {"name":{} }
    }
    
}
```

也可以自定义高亮标签

```txt
GET movie_index/movie/_search
{
    "query":{
      "match": {"name":"red sea"}
    },
    "highlight": {
      "fields": {"name":{"pre_tags": "<span color='red'>","post_tags":"</span>"}}
    }
    
}
```

### 7.6、排序

```txt
GET movie_index/movie/_search
{
  	"query":{
    	"match": {"name":"red sea"}
  	}, 
  	"sort": [
    	{
      		"doubanScore": { "order": "desc" }
    	}
  	]
}
```

### 7.7、分页

1）from-size

from 行号 =  (页号-1） * size

size 每页多少行

```txt
GET movie_index/movie/_search
{
  "query": { "match_all": {} },
  "from": 1,
  "size": 1
}
```

如何理解：假如查询出来的数据总共5个，可以把这5个结果看成一个组数array，索引0-4，设置size=2

from x size 2 => 从索引x开始，取出两个数据，也就是array[x]、array[x + 1]，

那么如何分页？如下图所示

![image-20200722111732434](https://gitee.com/wangzj6666666/bigdata-img/raw/master/es/image-20200722111732434.png)

page1 => from 0 size 2

page2 => from 2 size 2

page3 => from 4 size 2



2）scroll 深分页

from+size查询在10000-50000条数据（1000到5000页）以内的时候还是可以的，但是如果数据过多的话，就会出现深分页问题。

为了解决上面的问题，elasticsearch提出了一个scroll滚动的方式。
scroll 类似于sql中的cursor，使用scroll，每次只能获取一页的内容，然后会返回一个scroll_id。根据返回的这个scroll_id可以不断地获取下一页的内容，所以scroll并不适用于有跳页的情景。

```txt
GET test_dev/_search?scroll=5m
{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "age": 28
          }
        }
      ]
    }
  },
  "size": 10,
  "from": 0,
  "sort": [
    {
      "timestamp": {
        "order": "desc"
      },
      "_id": {
        "order": "desc"
      }
    }
  ]
}
```

- scroll=5m表示设置scroll_id保留5分钟可用。

- 使用scroll必须要将from设置为0。

- size决定后面每次调用_search搜索返回的数量

然后我们可以通过数据返回的_scroll_id读取下一页内容，每次请求将会读取下10条数据，直到数据读取完毕或者scroll_id保留时间截止：

```txt
GET _search/scroll
{
  "scroll_id": "DnF1ZXJ5VGhlbkZldGNoBQAAAAAAAJZ9Fnk1d......",
  "scroll": "5m"
}
```

scroll删除
根据官方文档的说法，scroll的搜索上下文会在scroll的保留时间截止后自动清除，但是我们知道scroll是非常消耗资源的，所以一个建议就是当不需要了scroll数据的时候，尽可能快的把scroll_id显式删除掉。

清除指定的scroll_id：

```txt
DELETE _search/scroll/DnF1ZXJ5VGhlbkZldGNo.....
```

清除所有的scroll：

```txt
DELETE _search/scroll/_all
```

3）search_after

scroll 的方式，官方的建议不用于实时的请求（一般用于数据导出），因为每一个 scroll_id 不仅会占用大量的资源，而且会生成历史快照，对于数据的变更不会反映到快照上。

search_after 分页的方式是根据上一页的最后一条数据来确定下一页的位置，同时在分页请求的过程中，如果有索引数据的增删改查，这些变更也会实时的反映到游标上。但是需要注意，因为每一页的数据依赖于上一页最后一条数据，所以无法跳页请求。

为了找到每一页最后一条数据，每个文档必须有一个全局唯一值，官方推荐使用 _uid 作为全局唯一值，其实使用业务层的 id 也可以。

```
GET test_dev/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "age": 28
          }
        }
      ]
    }
  },
  "size": 20,
  "from": 0,
  "sort": [
    {
      "timestamp": {
        "order": "desc"
      },
      "_id": {
        "order": "desc"
      }
    }
  ]
}
```

- 使用search_after必须要设置from=0。
- 这里我使用timestamp和_id作为唯一值排序。
- 我们在返回的最后一条数据里拿到sort属性的值传入到search_after。

使用sort返回的值搜索下一页：

```
GET test_dev/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "age": 28
          }
        }
      ]
    }
  },
  "size": 10,
  "from": 0,
  "search_after": [
    1541495312521,
    "d0xH6GYBBtbwbQSP0j1A"
  ],
  "sort": [
    {
      "timestamp": {
        "order": "desc"
      },
      "_id": {
        "order": "desc"
      }
    }
  ]
}
```



## 八、过滤

1）查询后过滤

```txt
GET movie_index/movie/_search
{
    "query":{
      "match": {"name":"red"}
    },
    "post_filter":{
      "term": {
        "actorList.id": 3
      }
    }
}
```

2）查询前过滤（推荐使用）

```txt
GET movie_index/movie/_search
{ 
    "query":{
        "bool":{
          "filter":[ {"term": {  "actorList.id": "1"  }},
                     {"term": {  "actorList.id": "3"  }}
           ], 
           "must":{
           		"match":{"name":"red"},
           		"match":{"name":"operation"}
           }
         }
    }
}
```

must：必须满足

should：多个条件满足一个就行

must_not：一个都不能有



3）按范围过滤

```txt
GET movie_index/movie/_search
{
   "query": {
     "bool": {
       "filter": {
         "range": {
            "doubanScore": {"gte": 8}
         }
       }
     }
   }
}
```

关于范围操作符：

| 操作符 | 含义                           |
| ------ | ------------------------------ |
| gt     | 大于                           |
| lt     | 小于                           |
| gte    | 大于等于  great than or equals |
| lte    | 小于等于  less than or equals  |



## 九、根据查询条件进行 删除、修改

1）删除数据（类似mysql中的 delete from where name ='red'）

```txt
POST movie_index/movie/_delete_by_query 
{
  "query":{
    "term": {
      "actorList.name.keyword": "zhang chen"
    }
  }
}
```

范围删除

```
POST   movie_index0213/movie/_delete_by_query
{
  "query":{
     "range":{
       "doubanScore":{
         "lt":7
       }
     }
  }
  
}
```

2）修改数据

```txt
POST   movie_index0213/movie/_update_by_query
{
    "script":"ctx._source['actorList'][0]['name']='zhang small chen'",
    "query":{
     "range":{
       "doubanScore":{
         "lt":7
       }
     }
  }
}
```



## 十、分组聚合

1）取出每个演员共参演了多少部电影

 聚合aggs语法

```json
"aggs": {
    "Name": {
        "AGG_TYPE": {

        }
    }
}
Name：为这次聚合起一个名字
AGG_TYPE：聚合类型
```

看下sql怎么写

```txt
select actor_name ,count(*) from movie group by actor_name
分组group by聚合count(*)
```

aggs中terms就是分组

```txt
GET movie_index/movie/_search
{ 
  "aggs": {
    "groupby_actor": {
      "terms": {
        "field": "actorList.name.keyword"  
        "size": 10
      }
    }
  }
}
```

terms分组后会根据size获取个数（默认10个），如果不清楚数据到底有多少个就往大了写。

aggs中terms后会默认有一个count（*），不用写，但是sum、min、max、avg这些聚合函数就得显示声明



2）每个演员参演电影的平均分是多少，并按评分排序

sql：select actor_name,avg(doubanScore) from movie     group by actor_name

```txt
GET movie_index0213/_search
{
  "aggs": {
    "groupby_actorName": {
      "terms": {
        "field": "actorList.name.keyword",
        "size": 3,
        "order": {
          "avg_doubanScore": "desc"
        }
      }
      , "aggs": {
        "avg_doubanScore": {
          "avg": {
            "field": "doubanScore" 
          }
        }
      }
    }
  }  
}
```

注意：聚合时为何要加 .keyword后缀？.keyword 是某个字符串字段，专门储存不分词格式的副本 ，在某些场景中只允许只用不分词的格式，比如过滤filter 比如 聚合aggs, 所以字段要加上.keyword的后缀。





相关资料：

https://www.cnblogs.com/jpfss/p/10815206.html