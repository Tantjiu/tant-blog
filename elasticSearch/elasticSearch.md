# 	一 ES简要介绍

## 1.1 概述

- 建立在Lucene基础上的**分布式、准实时**搜索引擎
- 官方提供ELK，为构建搜索服务提供良好解决方案

- - E：ElasticSearch  数据搜索，分析
  - L：Logstash 将数据库和日志等结构化或非结构化数据导入ES
  - K：Kibana 分析结果图形化展示

### 1.1.1 基本名词概念

- 索引：对数据存取、更新操作

- - 关系型数据库 -> 建立数据库
  - ES ->  建立索引

- 文档: 数据操作最细粒度的对象

- - 关系型数据库 -> 行
  - ES -> 文档

- 字段:

- - 一个文档可包含多个字段,每个字段均有类型与之对应	

- 映射: 

- - 建立索引需要定义的结构
  - 映射的文档字段类型设定后**不可更改**
  - 若添加数据时,该字段没有定义类型,将会自动映射字段名与类型

- 集群和节点

- - 集群: 就是集群
  - 节点: 集群中的某台计算机

- - - ES节点数量不受限制,可按需增加

- 分片

- - 为了能储存和计算海量数据,先对数据进行切分,再见其储存至多台计算机中
  - 一个分片对应的是一个Lucene索引,每个分片可设置多个副分片

- - - 主分片数量只能设置一次,设置后不能更改,默为5个
    - 副分片默认不开启,可开启,数量不限制,设定后可更改

### 1.1.2 与关系型数据库对比

|          | ES                                                           | 关系型数据库                                                 |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 索引方式 | 倒排索引                                                     | 大部分为B-Tree                                               |
| 事务     | 不支持事务,使用乐观锁,每次更新时更新版本号                   | 支持事务                                                     |
| 查询速度 | 字段多,记录多的情况下查询速度不会变慢                        | 少量字段及记录的情况,查询速度快,字段多,记录多的情况下查询速度变慢 |
| 实时性   | 准实时,写数据时先储存在内存中,一段时间后写入系统缓存,此时才能被检索到,这个时间默认为1s | 实时                                                         |

## 1.2 查询原理

节点按职责可分为**master节点**,**数据节点**,**协调节点,**每个节点可以进行单独配置,默认不划分节点角色

- master节点:  维护整个集群的工作,管理集群变更; master节点宕机时通过选举选出新的master,
- 数据节点: 主要负责索引数据保存工作,同时也执行文档的删改查, 负载压力最大
- 协调节点: 接收客户端请求,并将响应结果告知客户端, 节点的生命周期与请求相关,任意节点均可是协调节点

![img](https://cdn.nlark.com/yuque/0/2022/png/22530436/1656739100513-28da279c-17b9-452c-9023-1ed790e163d4.png)

**STEP1**:  客户端发送请求到任意一个 node，该node成为协调节点

**STEP2**:  协调节点对 doc id 进行哈希路由，将请求转发到对应的 node，此时会使用 round-robin 随机轮询算法，在主分片以及其所有副分片中随机选择一个，让读请求负载均衡。

**STEP3**:  接收请求的 node 返回 document 给 协调节点 。

**STEP4**:  协调节点返回document 给客户端。

## 1.3 应用场景

- 搜索引擎
- 推荐系统

- - 7.0之后的版本引入了高维向量功能,可以将推荐模型算法,需要计算的商品和用户储存至ES
  - 请求时,加载用户向量,并使用Script Score进行查询,根据相似度进行推荐

- 二级索引

- - 强事务性的数据不适合放ES, 但是需要使用任意组合字段进行查询的数据,可以选择放在ES形成二级索引

- 日志分析

- - ELK...

## 1.4 Kibana

![img](https://cdn.nlark.com/yuque/0/2022/png/22530436/1656747710876-e174b1e5-8d84-489d-abfc-490112cf17c2.png)

# 二 基础操作

## 2.1 索引

### 2.1.1 创建索引

建议:

- 不要使用es默认的mapping，虽然省事但是不合理
- 字段类型尽可能的精简，因为只要我们建了索引的字段es都会建立倒排，检索时会加载到内存。如果不合理会导致内存爆炸。
- 有些不要检索的字段不要设置index:true,  es默认为true,可以设置为false，更推荐使用es+mysql等形式，将不需要es存储的字段放在其他的存储介质中，通过唯一标识和es建立映射。
- Ik分词在建立的时候要注意：建索引采用ik_max_word, 检索时采用ik_smart

```json
PUT /xianmu_merchant
{
    "settings" : {
        "index" : {
            "analysis.analyzer.default.type": "ik_smart"
        },
        "analysis": {
      "normalizer": {
        "lowercase": {
          "type": "custom",
          "filter": ["lowercase"]
        }
      }
    }
    }
  "mappings"
  {
    ...
  }
}
```

### 2.1.2  删除索引

```json
DELETE /xianmu_merchant
```

### 2.1.3 关闭/打开索引

关闭索引后只能通过api或者监控工具查看索引元数据信息,不能写入和搜索

```json
POST /xianmu_merchant/_close

POST /xianmu_merchant/_open
```

### 2.1.4 索引别名

- 给索引定义另外一个名字 ,多个索引使用相同的索引别名即可实现别名与索引的关联关系
- 使用场景: 

- - 订单分月储存,可同时查询多个月的订单,
  - 订单索引主分片初期设置不满足急剧增长的业务需求,主分片数又不能更改,可平滑过渡

- 当一个别名只对应一个索引时,写入数据可使用该别名,对应多个时,不可以使用,可设置属性指定数据写入别名对应的某个索引下

```json
POST /_aliases
{
  "actions": [
    {
      "add": { // 给xianmu_merchant 添加别名 xianmu_merchant_new
        "index": "xianmu_merchant",
        "alias": "xianmu_merchant_new"
      }
    },
    {
      "remove": { // xianmu_merchant 删除别名 xianmu_merchant_new
        "index": "xianmu_merchant",
        "alias": "xianmu_merchant_new"
      }
    }
  ]
}
```

## 2.2 映射

### 2.2.1 查看映射

```json
GET /xianmu_merchant/_mapping
```

### 2.2.2 拓展映射

映射已经被定义了的字段不能修改类型,只能拓展字段和为字段增加属性

```json
POST /xianmu_merchant/_mapping
{
  "areaNo" : {
     "type" : "long"
   },
  "area" : {
    "type" : "text",
    "fields" : {
      "area_keyword" : { // 定义area的子字段,类型为keyword
        "type" : "keyword",
      }
    }
  }
}
```

### 2.2.3 数据类型

- 字符串类型:

- - text：会分词，不支持聚合(相当于mysql 中的sum)

- - - 一般使用match搜索

- - keyword：不会分词，将全部内容作为一个词条，支持聚合

- - - 一般用于文档的过滤,排序和聚合.例如:id,url,姓名,类型等
    - 一般使用term搜索

- 数值类型

- - long,integer,short,byte,double,...
  - 一般使用term或者范围搜索

- 布尔类型

- - boolean

- 日期类型

- - date

- - - 默认格式为毫秒值或者严格的日期格式yyyy-MM-dd等,可设置format自定义格式,设置过后仅支持对应格式日期写入
    - 一般用ranges查询

- 数组类型

- - 开箱即用,使用[ ] 即可,数组可以为空
  - 适用搜索方式与数组元素相同

- 对象类型

- - 写入时自动识别并转换
  - 对于对象属性的搜索 使用 ' . '进行指向

- 地理对象

- - geo_point: 设置经纬度信息为地理数据类型,对于地理位置搜索相对友好

## 2.3 文档

### 2.3.1 写入文档

- 单条写入

```json
POST /xianmu_merchant/_doc/1419
{
  "area":"杭州"
  ...
}
```

- 批量写入

```json
POST /_bulk
{"index":{"index":"xianmu_merchant","_id":1419}}
{"area":"杭州",...}
{"index":{"index":"xianmu_merchant","_id":1420}}
{"area":"杭州",...}
```

### 2.3.2 更新文档

- 单条更新

```json
POST /xianmu_merchant/_update/1419
{
  "doc": {
    "area":"杭州"
     ...
  }
}
```

- 存在则更新,否则插入

```json
POST /xianmu_merchant/_update/1419
{
  "doc": {
    "area":"杭州"
     ...
  },
  "upsert":{
    "area":"杭州"
     ...
  }
}
```

- 批量更新文档

```json
POST /_bulk
{"update":{"index":"xianmu_merchant","_id":1419}}
{"area":"杭州",...}
{"doc":{"index":"xianmu_merchant","_id":1420}}
{"area":"杭州",...}
```

- 根据条件更新

```json
POST /xianmu_merchant/_update_by_query
{
  "query":{
    "term":{
      "area":{
        "value":"杭州"
      }
    }
  },
  "script":{
    "source":"ctx._source['area']='余杭区'",
    "lang":"painless"
  }
}
```

### 2.3.3 删除文档

- 删除单条文档

```json
DELETE /xianmu_merchant/_doc/1419
```

- 批量删除

```json
POST /_bulk
{"delete":{"index":"xianmu_merchant","_id":1419}}
{"delete":{"index":"xianmu_merchant","_id":1420}}
```

- 根据条件删除

```json
POST /xianmu_merchant/_delete_by_query
{
  "query":{
    "term":{
      "area":{
        "value":"杭州"
      }
    }
  }
}
```

# 三 搜索

## 3.1 搜索辅助功能

### 3.1.1 指定返回的字段

- 对搜索结果瘦身,提升性能的操作

```json
GET xianmu_merchant/_search
{
  "_source": ["mname","bdName"],
  "query": {
    "term": {
      "mId": 1419
    }
  }
}
```

### 3.1.2 计数

- 查询文档条数, 可为前端分页做准备

- - 普通query也会返回total

```json
GET xianmu_merchant/_count
{
  "query": {
    "term": {
      "area": "西湖区"
    }
  }
}
```

### 3.1.3 分页

- 默认情况下,es只返回前十个文档,可自定义设置分页,from标识起始下标,默认为0
- 默认最多可获取10000个文档, 如果需要更多的文档数,需要在索引设置时修改 : max_result_window 的值
- es是不适合做深分页的:

- - 因为es是分布式的,索引的数据分布在多个节点的多个分片中,
  - 一个带有分页的搜索一般会跨越多个分片
  - 每个分片中都得创建一个足够长度(至少from + size) 的有序队列,用来储存命中的文档.
  - 每个节点会将数据传递给协调节点,协调节点再将数据汇总处理
  - 协调节点处理好数据后,再取出 from,size的数据响应给客户端
  - **过多的数据会导致协调节点资源耗尽而停止服务**

![img](https://cdn.nlark.com/yuque/0/2022/png/22530436/1656777982030-7983b9d8-7ed9-4883-b7e8-72f566c8710f.png)

```json
GET xianmu_merchant/_search
{
  "query": {
    "match_all": {
    }
  },
  "from": 0,
  "size": 20
}
```

### 3.1.4 性能分析

- 如果发现ES搜索请求响应很慢,原因很可能是因为es语言执行逻辑有问题,ES提供了profile功能,能详细的列出搜索时每一个步骤的耗时
- 开启profile, 增加"profile": "true", 需注意该功能有性能损耗,不适合在生产中使用

```json
GET xianmu_merchant/_search
{
  "_source": ["mname","bdName"],
  "query": {
    "term": {
      "mId": 103074
    }
  },
  "profile": "true"
}
```

![img](https://cdn.nlark.com/yuque/0/2022/png/22530436/1656778520894-b86b87b6-563c-476a-a934-9ab5626e31ad.png)

- 建议使用Kibana的 profile进行性能分析,可视化界面十分友好

![img](https://cdn.nlark.com/yuque/0/2022/png/22530436/1656778856297-701a17d3-7d9a-4ff9-a55c-b68b4d051154.png)

### 3.1.5 评分分析

- 使用explain关键字即可查看搜索时的匹配打分详情

```json
GET xianmu_merchant/_explain/103074
{
   "query": {
    "match": {
      "mname": "薄荷小屋"
    }
  }
}
```

## 3.2 搜索匹配功能

### 3.2.1 查询所有文档

es查询所有文档的时候,默认时不打分的,可以使用boost参数设定分值

```json
GET xianmu_merchant/_search
{
  "query": {
    "match_all": {
      "boost": 2
    }
  },
  "_source": ["mname","bdName"]
  
}
```

### 3.2.2 term级别查询

#### 3.2.2.1 term查询

- 精准查询的主要方式,完全匹配的查询
- 查询的字段数据类型如果是日期类型的,需要按钮其设定好的格式进行查询

```json
GET xianmu_merchant/_search
{
  "_source": ["mname","bdName"],
  "query": {
    "term": {
      "mId": 103074
    }
  }
}
```

#### 3.2.2.2 terms查询

- term查询的拓展形式,用于查询一个或多个值是否完全匹配,相当于sql中的IN
- 注意,使用该类型查询时,传参过多可能引发性能问题

```json
GET xianmu_merchant/_search
{
  "_source": ["mname","bdName"],
  "query": {
    "terms": {
      "mId": [
        "1419",
        "1420"
      ]
    }
  }
}
```

#### 3.2.2.3 range查询

- 范围查询,一般对数值及日期类型进行查询
- 注意使用range查询时,入参与设定的字段类型需一致
- 边界值设定:

- - gt: 大于
  - lt:  小于
  - gte: 大于等于
  - lte: 小于等于

```json
GET xianmu_merchant/_search
{
  "_source": ["mname","bdName","bdId"],
  "query": {
    "range": {
      "bdId": {
        "gt": 0,
        "lte": 2000
      }
    }
  }
}
```

#### 3.2.2.4 exists查询

- 非空查询
- 注意,数组"[ ]", "[ null ]" 也为空

```json
GET xianmu_merchant/_search
{
  "_source": ["mname","bdName","bdId"],
  "query": {
    "exists": {
      "field": "mname"
    }
  }
}
```

### 3.2.3 布尔查询

- 复合搜索,**实际开发中使用最多的查询方式**
- 将多个子查询组成一个布尔表达式,子查询之间的关系是**逻辑"与",**所有子查询为true时,结果才为真
- 可将各个子查询的具体匹配程度对文档进行打分计算
- 子查询的几种模式

- - must: 必须匹配该条件, 打分
  - should: 可以匹配,打分
  - must_not: 必须不匹配,打分
  - filter: 必须匹配,不打分, 但是会对部分匹配结果进行缓存

- - - 缓存的结果是可重用的
    - 不是每个filter都会缓存,只有最近高频的,es觉得有价值的才会

- - - - 最近256个查询中出现过, 频率满足某个阈值
      - 缓存的字段更新了,缓存也会同步更新

- - - filter不计算分数,可以节省很多时间开销,查询**更加高效**

```json
GET xianmu_merchant/_search
{
   "_source": ["mname","bdName","bdId"],
   "query": {
     "bool": {
       "must": [
         {"match": {
           "area": "西湖区"
         }}
       ],
       "should": [
         {"term": {
           "address": {
             "value": "蒋村"
           }
         }}
       ],
       "must_not": [
         {
           "term": {
             "mname": {
               "value": "薄荷小屋"
             }
           }
         }
       ],
       "filter": {
         "term": {
           "bdId": "0"
         }
       }
       
     }
   }
}
```

### 3.2.4 Constant Score查询

- 不考虑得分,仅得到某个文本中**包含**某个词或者某个短语
- 可以使用boost子句设定默认得分

```json
GET xianmu_merchant/_search
{
  "_source": ["mname","bdName","bdId"],
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "mname": "薄荷"
        }
      },
      "boost": 2
    }
  }
}
```

### 3.2.5 Function Score查询

- 不使用es默认打分系统,自定义一个或者多个函数为每个文档打分
- es提供了多种函数,可以满足日常中大部分需求
- 在排序中应用广泛

```json
GET xianmu_merchant/_search
{
  "_source": ["mname","bdName","bdId"],
  "query": {
    "function_score": {
      "query": {
        "term": {
          "bdId": {
            "value": "0"
          }
        }
      },
      "functions": [
        {"random_score": {}} // 例:随机分数
      ]
    }
  }
}
```

### 3.2.6 全文搜索

- 区别于结构化查询,全文搜索更侧重于关键词对**文本匹配的程度**,而不是关键词**是否匹配文本**

- - **先对入参的关键字进行分析**
  - **使用分析后的结果作为关键字进行子查询**
  - **所有子查询得到的结果得分汇总,排序**

- **es搜索中使用最多的查询方式**

#### 3.2.6.1 match查询

- 只要查询的入参,分词后的关键字在任意一个文档中,就会被搜索到
- 比如"一苇的测试店铺",在"ik_max_word' 精度下分词的结果为: '一','苇','的','测试','店铺',任意文档中选定字段只要**包含这些词中的其中一个,就会被搜索到,**并按照匹配程度进行打分,得分高的在最前面

![img](https://cdn.nlark.com/yuque/0/2022/png/22530436/1656872945076-b7ef9b3d-90af-489b-a13f-87b55c96fac9.png)

- 可以设置**operator**参数来决定分词后的词集合是"与"还是"或",默认是"or", 可以设置成 "and"
- 搜索时 如果觉得and过于严苛,or又过于宽松,可以设置匹配度:**minimum_should_match**

- - minimum_should_match:最小匹配度,可以设定一个可以匹配的词的个数,一般生产环境无法控制具体匹配多少个词,可以设定一个**百分比**

```json
GET xianmu_merchant/_search
{
  "query": {
    "match": {
      "mname": {
        "query": "薄荷",
        "operator": "or",
        "minimum_should_match": "80%"
      }
    }
  }
}
```

#### 3.2.6.2 multi_match查询

- bool查询的替代方案,可以封装多个match查询,相对于bool查询或许更优雅
- 可以设置权重, 在字段名后面拼接 '^boost'
- tie_breaker子句, 给第二个及更多的字段计算分数时,会乘以该系数
- 可以设置type, 有以下模式
  - best_fields: 取文本匹配结果中最大的分值
  - most_fields: 取文本匹配结果中累加的分值
  - cross_fields: 取每个分词结果的最大分值累加

- **fields**中注意数组长度,默认长度1024,可以为空,为空时默认填入"*" 通配符,匹配全部字段

```json
GET xianmu_merchant_search/_search
{
  "query": {
    "multi_match": {
        "query": "薄荷",
        "fields": ["mname","contacts.contact"],
        "tie_breaker": 0.3
    }
  }
}
```

#### 3.2.6.3 match_phrase查询

- 短语匹配,用于搜索**确切**的短语或邻近的词语
- 比如"薄荷小屋" 分析得到短语 "薄荷" "小屋",且"薄荷" 在 "小屋" 前面

- - 假设索引中包含"薄荷","小屋"的客户有: 薄荷的糖, 薄荷小屋, 薄荷勇敢小屋, 小屋薄荷,薄荷小屋在浙江
  - 那么通过match_phrase查询"薄荷小屋" 得到的结果有:  薄荷小屋, 薄荷小屋在浙江
  - 因为match_phrase查询 **不仅仅需要对应短语在文档中存在,还需要在倒排索引对应的位置一致,** 比如本例中 薄荷小屋的索引间距离为 1,所以只能搜索到:薄荷小屋, 薄荷小屋在浙江
  - 可通过设置slop参数来调节匹配词之间的阈值,意为:**需要移动多少步才能匹配的上索引顺序**

```json
GET xianmu_merchant/_search
{
  "_source": ["mname","bdName","bdId"],
  "query": {
    "match_phrase": {
      "mname": {
        "query": "薄荷小屋",
        "slop": 2
      }
    }
  }
}
```

### 3.2.6.4 query_string

- 非常适合手动分词的系统  已知了分词结果的
- 默认为 OR 可以指定为 AND 注意 是大写

~~~json
GET xianmu_merchant_search/_search
{
  "query": {
    "query_string": {
      "default_field": "mname",
      "query": "薄荷 AND 店"
    }
  }
}
~~~



### 3.2.7 地理位置查询

地理位置查询分为: 地埋点(经纬度)查询, 地埋形状查询, 对应的数据类型分别为:geo_point, geo_shape

#### 3.2.7.1 地埋点查询(geo_point)

- 实际应用中使用较多的数据类型
- 有三种查询方式:

- - geo_distance: **指定一个坐标点,指定距离该点的范围,即可查询对应范围内的文档**

```json
GET xianmu_merchant/_search
{
   "_source": ["mname","bdName","bdId","contacts.poi"],  
   "query": {
     "geo_distance":{
       "distance":"5km",
       "contacts.poi":{
         "lat":"30.279943",
         "lon":"120.05859"
       }
     }
   }
}
```

- - geo_bounding_box: **提供矩形内的搜索,需要提供左上角及右下角顶点的地理坐标**
  - geo_polygon: **提供多边形的文档搜索**

```json
GET xianmu_merchant/_search
{
   "_source": ["mname","bdName","bdId","contacts.poi"],  
   "query": {
     "geo_polygon":{
       "contacts.poi":{
         "points":[
           {"lat":"30.279943","lon":"120.03859"},
           {"lat":"30.269943","lon":"120.04859"},
           {"lat":"30.289943","lon":"120.05859"}
           ]
       }
     }
   }
}
```

## 3.3 排序

- 默认情况下ES是按分值排序的
- 使用sort子句可以自定义排序
- 默认sort子句不对文档进行打分
- 排序的两张情况

- -  字段值排序

```json
GET xianmu_merchant/_search
{
  "_source": ["mname","bdName","bdId","contacts.poi"],
  "query": {
    "match_phrase": {
      "mname": {
        "query": "薄荷小屋",
        "slop": 2
      }
    }
  },
  "sort": [
    {
      "mId": {
        "order": "desc"
      }
    }
  ]
}
```

- - 地理距离排序

- - - 地理距离排序默认算法是:sloppy_arc计算精准但是耗费时间长,可以使用distance_type子句修改排序计算算法为plane,该算法计算快,但是精度差一些

```json
GET xianmu_merchant/_search
{
   "_source": ["mname","bdName","bdId","contacts.poi"],  
   "query": {
     "geo_distance":{
       "distance":"5km",
       "contacts.poi":{
         "lat":"30.279943",
         "lon":"120.05859"
       }
     }
   },
   "sort": [
     {
       "_geo_distance": {
         "contacts.poi": {
           "lat": "30.279943",
           "lon": "120.05859"
        },
         "order": "asc",
         "unit": "km",
         "distance_type": "plane"
      }
     }
   ]
}
```

# 四 进阶操作

## 4.1 文本搜索

相比于数字的精确搜索,实际应用中对于文本数据的搜索应用的更多

ES对于文本数据的处理在前期的索引构建和搜索环节都需要额外处理,并且需要在匹配环节进行相关性分数的计算

ES对于文本搜索依赖两大组件:Lucene和分析器

- Lucene:负责进行倒排索引的构建
- 分析器:在建立倒排索引前对和搜索前对文本进行分词和语法处理

### 4.1.1 倒排索引

- 一种数据结构; 简述 就是 "鲜沐" -> "鲜沐农场"
- 倒排索引的所有词语存储在词典中,每个词语均指向包含它的文档信息列表

![img](https://cdn.nlark.com/yuque/0/2022/png/22530436/1657200861281-dd055462-e4ce-4daf-8e24-55205239c9c1.png)

- 根据词语的分析结果, 建立词语与文档的关系

|                   | 鲜沐 | 农场 | 鲜沐农场 | 科技 | 杭州 |
| ----------------- | ---- | ---- | -------- | ---- | ---- |
| 001(鲜沐农场)     | 1    | 1    | 1        | 0    | 0    |
| 002(杭州鲜沐科技) | 1    | 0    | 0        | 1    | 1    |

- 遍历词语与文档的关系 建立词语与文档信息的词典, 此时的文档信息中不仅包含文档id,还有词频,文档位置等信息

### 4.1.2 搜索过程

- 以使用最为广泛的match查询为例

![img](https://cdn.nlark.com/yuque/0/2022/png/22530436/1657201477504-8386c0b4-f6d9-4225-a09d-7a14cdb47d3d.png)

- 小问题: 在鲜沐的商户索引中,需要精准搜索客户的size, 使用什么搜索? 怎么做到?

## 4.2 分析器工作流程

- 应用场景

- - 创建或更新文档的时候
  - 查询文本字段的时候,对查询语句进行分词

![img](https://cdn.nlark.com/yuque/0/2022/png/22530436/1657202593446-7eb971bf-4b8c-4d2a-b897-4d62634e0f6d.png)

### 4.2.1 字符过滤器

- 接收原始的字符流, 对原始字符流做添加,删除,转换操作,对文本做粗加工

- - 例如: 取出HTML标签,将"&" 转换为"and"
  - ES内置过滤器可擦除HTML标签,配置映射关系,使用正则处理字符串等操作

### 4.2.2 分词器

- 按照规则切分词语

- - ES内置的分词器对于中文不友好,需要额外配置中文分词器插件

- 切分词语之后,会保留词语与原始文本的关系,记录词语的位置,开始结束时的字符偏移量等

### 4.2.3 分词过滤器

- 接收分词器的处理结果,将切分好的词语进行加工和修改

- - 例: 将文本的字母全转换为小写,删除或停用某些词汇,为某个分词增加同义词等

## 4.3 分析器使用

### 4.3.1 分析API

- 使用ES提供的API能快速帮助我们理解入参与结果之间的联系

```json
GET _analyze
{
  "analyzer":"ik_max_word",
  "text":"鲜沐农场"
}
```

![img](https://cdn.nlark.com/yuque/0/2022/png/22530436/1657203545678-b7d4174d-301e-401b-9184-e90f7ae1faeb.png)

### 4.3.2 ES内置分析器

- 默认情况下,text的类型的分析器是: standard,会将中文拆分成一个一个的单字
- 在索引建立或搜索时也可以指定分析器

### 4.3.3 使用分析器

- 在创建索引时指定默认分析器

```json
PUT xianmu_merchant
{
    "settings" : { 
        "index" : {
            "analysis.analyzer.default.type": "ik_smart"
        },
        "analysis": {
      "normalizer": {
        "lowercase": {
          "type": "custom",
          "filter": ["lowercase"]
        }
      }
    }
    },
    "mappings": {
      ...
    }
}
```

- 在映射中指定

```json
"mappings": {
      "province" : {
          "type" : "text",
         "analyzer":"ik_smart" 
        }
    }
```

- 搜索时使用的分析器, 慎重选择,这样设置会导致在创建/修改文档和搜索时使用的分词器不一致

```json
"mappings": {
      "province" : {
          "type" : "text",
         "analyzer":"ik_smart",
        "search_analyzer":"ik_max_word"
        }
}
```

### 4.3.4 自定义分析器

```json
PUT /test_merchant
{
  "settings": {
    "analysis" : {
          "filter" : {
            "定义过滤器A",
            "定义过滤器B"
          },
          "analyzer" : { // 自定义分析器
            "test_analyzer" : {
              "filter" : [
                "过滤器A",
                "过滤器B"
              ],
              "type" : "custom",
              "tokenizer" : "ik_max_word"
            }
          }
        }
    }
}
```

## 4.4 中文分词器

### 4.4.1 分词简介

#### 4.4.1.1 算法

由于中文不同于英文的特性,想要去分词也是不一样的,现在主流的有以下两种算法

- 基于词典分词

- - 按某种设置好的策略将提前准备好的词典和待匹配的字符串进行匹配,匹配到词典中的某个词,说明分词成功,简单,速度快 ,代码实现可使用最大匹配法

- 基于统计,机器学习

- - 事先构建一个标记好分词形式的语料库, 统计每个词出现的频率,词和词之间共存的频率等,基于统计结果给出某种语境下的分词结果

#### 4.4.1.2 难点

- 分词标准: 不同的分词器分词标准不统一,分词结果不同
- 分词歧义: 使用分词器对文本进行切分,切分后的结果和原来的字面意义不同
- - ![img](https://cdn.nlark.com/yuque/0/2022/png/22530436/1657207378390-d6afa6b0-cd80-47d5-a61d-e0fe9556c84d.png)
- 新词识别: 没有在词典或者训练语料中出现,分词器无法识别

#### 4.4.1.3 查全率与查准率

- 查全率: 正确结果n个, 查出来正确的有m个
- 查准率: 查出来n个文档有m个正确

两种不可兼得,但可以通过排序手段来优化

### 4.4.2 IK分析器

- 开源,Java,轻量级中文分词工具包, 词典可以冷更新和热更新
- 提供了两个分词器: ik_smart / ik_max_word  粗 / 细粒度分词
- 对于需要分词但是未在词典中的,可以在ik分析器安装目录的config子目录中创建xxx.dict, 添加词汇解决,注意每个词单独一行,修改为记得去改下配置文件

### 4.4.3 HanLP分析器

- 一系列模型与算法的Java工具包,提供了丰富的API, 具有多种分词算法, 多种子分析器... 

## 4.5 使用同义词

- 用户有时候可能不知道商品具体名称, 比如: 番茄,圣女果,西红柿, 这个时候就可以指定同义词了

### 4.5.1 建立索引时构建同义词

- es内置分词过滤器中:"synonyms"可以定义同义词

```json
PUT test_xianmu
{
  "settings": {
    "analysis": {
      "filter": {
        "test_synonyms_filter":{
          "type":"synonym",
          "synonyms":[
            "圣女果,番茄,西红柿",
            "椰子,椰青"
            ]
        }
      },
      "analyzer": {
        "test_synonyms_analyzer":{
          "tokenizer":"ik_max_word",
          "filter":[
            "lowercase",
            "test_synonyms_filter"
            ]
        }
      }
    }
  }
}
```

- 查询时使用es内置过滤器 synonym_graph 支持查询时自定义同义词的分词过滤器
- 该种方式与synonym命中的结果集是一致的,但是**排序不同**

- - 因为synonym在构建索引时将同义词与搜索词同等对待,同等计算得分
  - synonym_graph 在查询是将搜索词转换为同义词进行搜索,计算得分的是被替换的那个词

- 注意:需要有修改同义词需求,只能使用synonym_graph

```json
PUT test_xianmu
{
  "settings": {
    "analysis": {
      "filter": {
        "test_synonyms_filter":{
          "type":"synonym_graph",
          "synonyms":[
            "圣女果,番茄,西红柿",
            "椰子,椰青"
            ]
        }
      },
      "analyzer": {
        "test_synonyms_analyzer":{
          "tokenizer":"ik_max_word",
          "filter":[
            "lowercase",
            "test_synonyms_filter"
            ]
        }
      }
    }
  }
}
```

- 如果更新同义词较频繁,可将同义词放在文件中,放在config目录下,注意每个节点都要放,在构建索引时指定同义词路径即可,每次添加完记得刷新一下

```json
PUT test_xianmu
{
  "settings": {
    "analysis": {
      "filter": {
        "test_synonyms_filter":{
          "type":"synonym_graph",
          "synonyms_path":xxx  // config的相对路径
        }
      },
      "analyzer": {
        "test_synonyms_analyzer":{
          "tokenizer":"ik_max_word",
          "filter":[
            "lowercase",
            "test_synonyms_filter"
            ]
        }
      }
    }
  }
}
```

## 4.6 使用停用词

- 简单来说就是禁止掉某些词典, 比如 "我" "你" "这"...

```json
PUT test_xianmu
{
  "settings": {
    "analysis": {
      "filter": {
        "test_stop_filter":{
          "type":"stop",
          "stopwords": [
          "我",
            "你",
            "这"
          ]
        }
      },
      "analyzer": {
        ...
  }
}
```

- IK分词器中禁用词通过自定义禁用词文件形式实现

## 4.7 拼音搜索

- 例如输入 xm 得到 "鲜沐" 
- 使用拼音插件实现

![img](https://cdn.nlark.com/yuque/0/2022/png/22530436/1657210123777-47e47859-3e42-4917-8475-94af3f58c173.png)

```json
PUT /test_merchant
{
  "settings": {
    "analysis" : {
          "filter" : {
            "remote_synonym" : { // 同义词
              "type" : "dynamic_synonym",
              "synonyms_path" : "http://localhost:8848/synonym.txt",
              "interval" : "30"
            },
            "my_pinyin" : { // 拼音插件
              "keep_joined_full_pinyin" : "true",
              "lowercase" : "true",
              "keep_original" : "true",
              "remove_duplicated_term" : "true",
              "keep_first_letter" : "false",
              "keep_separate_first_letter" : "false",
              "type" : "pinyin",
              "keep_full_pinyin" : "true"
            }
          },
          "analyzer" : { // 自定义分析器
            "pd_ik_max_word" : {
              "filter" : [
                "remote_synonym",
                "my_pinyin"
              ],
              "type" : "custom",
              "tokenizer" : "ik_max_word"
            }
          }
        }
    }
}
```

## 4.8 高亮搜索

- 对搜索出的文本标红/加粗/下划线操作

```json
GET xianmu_merchant/_search
{
   "_source": ["mname","bdName"],
  "query": {
    "match": {
      "mname": "Js"
    }
  },
  "highlight": {
    "fields": {
      "mname": {}
    }
  }
}
```

![img](https://cdn.nlark.com/yuque/0/2022/png/22530436/1657210499033-84c387a0-0c6b-475b-896b-df09a177ca22.png)

- 默认<em></em> 标签,可以在更改为别的标签

```json
GET xianmu_merchant/_search
{
   "_source": ["mname","bdName"],
  "query": {
    "match": {
      "mname": "Js"
    }
  },
  "highlight": {
    "fields": {
      "mname": {
        "pre_tags": "<u>",
        "post_tags": "</u>"
      }
    }
  }
}
```

![img](https://cdn.nlark.com/yuque/0/2022/png/22530436/1657210722592-a631b8fe-3866-446e-ba1f-41647529995e.png)

- 高亮搜索策略,指定type子句即可

- - plain: 精准度高,将文档全部加载至内存查询分析,大量文档查询性能消耗大
  - unified: es默认策略, 将文本分解成词汇,进行评分后搜索
  - fvh: 基于向量的高亮显示搜索 更适合在文档中包含大字段时使用

## 4.9 拼接纠错

- 如题.,我本意时想输入"**拼写纠错"**,但实际上输成了"**拼接纠错**"  如果是搜索时输入,就需要有一个纠错功能来纠正输入的关键字,并使用正确的关键字进行搜索 

- 一般的做法是

- - 先收集一段时间内用户搜索日志中的查询词
  - 单独建一个纠正词索引
  - 用户搜索时,如果索引没有匹配结果,就去纠正词索引中进行匹配
  - 有结果则使用匹配结果搜索

- ES中对于纠错匹配,可以使用fuzzy_match搜索, 这种模式使用编辑距离和倒排索引结合的形式完成纠错
- 分析器分词后,假设得到的词汇如下

- - 例如: "拼接" 和 "拼写"  只需要通过一次字符替换,能变成相同的词汇,即 编辑距离为1
  - 例如: "接拼" 和 "拼写"  需要通过一次字符替换,一次位置变换,才变成相同的词汇,即 编辑距离为2

```json
GET xianmu_merchant/_search
{
  "_source": ["mname","mId" ],
  "query": {
    "match": {
      "mname": {
        "query":"鲜沐农场",
        "operator": "and",
        "fuzziness": 2
      }
    }
  }
}
```

- 上面的案例中,会存在查询到不应该查询到的文档,解决方案:

- - 误导性的词汇加入词典
  - 设置子字段 + 多种分词器查询

# 五. 分享讨论纪要

|      | 问题描述                              |      |
| ---- | ------------------------------------- | ---- |
|      | 协调节点挂了怎么办?                   |      |
|      | 深分页问题解决?                       |      |
|      | 倒排索引图解?                         |      |
|      | 应用场景, 对于分词不准问题的解决方式? |      |
|      | ...                                   |      |

# 六. 搜索引擎的设计

- 数据来源
  - 爬虫,消息中间件,mysql
- 数据清洗
  - 即转换为索引:大数据平台,hadoop,etl技术
- 存储索引
  - es solr
- 索引更新
  - 消息中间件,canal,kafka
- 服务
  - 分布式技术
- 千人千面
  - 面向用户的概念, NLP(自然语言技术,最重要的就是分词) 知识图谱

## 6.1 搭建思路

- 垂直检索(电商网站)
- 全文检索(百度)



优化经验

- 分片预估好,一般一个分片不要超过200g
- 分片不是越多越好,分片越多聚合的就越多, 分片可以理解为分表, 一般数据量不超过500g的三个分片足以,
- 副本也不是越多越好,副本越多,占用空间越大, es集群内同步的就越多, 一般针对不是高并发的系统,1个副本足以, 高并发的系统根据自己的qps去调整
- mapping优化, 也很重要,设计了不能改
- 查询: 学会使用热数据缓存和预热, 预热很重要, es会将查询过的索引缓存起来,所以针对大数据量的索引,第一次查询会有点慢,第二次就很快了
- 搜索结果优化, 主要是分词优化
- es内存配置:官方文档很详细(管理,监控和部署一节)