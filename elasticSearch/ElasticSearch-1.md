# ElasticSearch

## 一.介绍

- es是一个基于全文搜索的中间件
- mysql有事务性质,es没有,所以es出异常是无法恢复的
- 基于lucene
- 是一个分布式,高拓展,高实时的搜索与数据分析引擎
- 基于RESTful web接口
- 应用场景
  - 搜索：海量数据的查询
  - 日志数据分析
  - 实时数据分析

### 1.1 倒排索引

将一行语句进行分词,形成词条与id的对应关系,即为方向索引.

例如:

-  以唐诗为例，所处包含“前”的诗句		

  - 正向索引：由《静夜思》-->窗前明月光--->“前”字

  - 反向索引：“前”字-->窗前明月光-->《静夜思》

- 反向索引的实现就是对诗句进行分词，分成单个的词，由词推据，即为反向索引
- 一个倒排索引由文档中所有不重复词的列表构成，对于其中每个词，对应一个包含它的文档id列表。

### 1.2 概念

- **index（索引）**：相当于mysql的库
- **映射**：相当于mysql 的表结构

- **document(文档)**：相当于mysql的表中的数据,常以json格式显示.一个document相当于关系型数据库中的一行数据

### 1.3 脚本操作

- GET：用来获取资源

- POST：用来新建资源（也可以用于更新资源）

- PUT：用来更新资源

- DELETE：用来删除资源

### 1.4 数据类型

- 字符串
  - text：会分词，不支持聚合
  - keyword：不会分词，将全部内容作为一个词条，支持聚合

...

### 1.5 分词器

- 解析词条的工具
- 常用IKAnalyze:基于java语言开发的轻量级的中文分词工具包
- IK分词器有两种分词模式：ik_max_word和ik_smart模式
  - ik_max_word:会将文本做最细粒度的拆分
  - ik_smart:粗粒度的拆分

## 二.查询操作

### 2.1 查询模式

- 词条查询：term:词条查询不会分析查询条件，只有当词条和查询字符串完全匹配时才匹配搜索

- 全文查询：match:全文查询会分析查询条件，先将查询条件进行分词，然后查询，求并集

### 2.2 批量操作

bulk

...

### 2.3 脚本及JavaAPI

#### 2.3.1 matchAll

- 查询所有,可通过from和size来控制分页

~~~es
# 默认情况下，es一次展示10条数据,通过from和size来控制分页
GET goods/_search
{
  "query": {
    "match_all": {}
  },
  "from": 0,
  "size": 100
}
~~~

Java api

~~~Java
/**
     * 查询所有
     *  1. matchAll
     *  2. 将查询结果封装为Goods对象，装载到List中
     *  3. 分页。默认显示10条
     */
    @Test
    public void matchAll() throws IOException {

        //2. 构建查询请求对象，指定查询的索引名称
        SearchRequest searchRequest=new SearchRequest("goods");

        //4. 创建查询条件构建器SearchSourceBuilder
        SearchSourceBuilder sourceBuilder=new SearchSourceBuilder();

        //6. 查询条件
        QueryBuilder queryBuilder= QueryBuilders.matchAllQuery();
        //5. 指定查询条件
        sourceBuilder.query(queryBuilder);

        //3. 添加查询条件构建器 SearchSourceBuilder
        searchRequest.source(sourceBuilder);
        // 8 . 添加分页信息  不设置 默认10条
//        sourceBuilder.from(0);
//        sourceBuilder.size(100);
        //1. 查询,获取查询结果

        SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);

        //7. 获取命中对象 SearchHits
        SearchHits hits = searchResponse.getHits();

        //7.1 获取总记录数
      Long total= hits.getTotalHits().value;
        System.out.println("总数："+total);
        //7.2 获取Hits数据  数组
        SearchHit[] hits1 = hits.getHits();
            //获取json字符串格式的数据
        List<Goods> goodsList = new ArrayList<>();
        for (SearchHit searchHit : hits1) {
            String sourceAsString = searchHit.getSourceAsString();
            //转为java对象
            Goods goods = JSON.parseObject(sourceAsString, Goods.class);
            goodsList.add(goods);
        }
    }
~~~

#### 2.3.2 termQuery

term查询：不会对查询条件进行分词,

~~~es
GET goods/_search
{
  "query": {
    "term": {
      "title": {
        "value": "华为"
      }
    }
  }
}
~~~

java api

~~~java 
...;
//查询条件
QueryBuilder queryBuilder= QueryBuilders.termQuery();
...;
~~~

#### 2.3.3 matchQuery

match查询：

- 会对查询条件进行分词。

- 然后将分词后的查询条件和词条进行等值匹配

- 默认取并集（OR）

~~~es
# match查询
GET goods/_search
{
  "query": {
    "match": {
      "title": "华为手机"，
      "operator":"(or或and)" //默认取并集or，可以改成and（交集）
    }
  },
  "size": 500
}
~~~

#### 2.3.3 wildcard

wildcard查询：会对查询条件进行分词。还可以使用通配符 ?（任意单个字符） 和  * （0个或多个字符）

~~~txt
"*华*"  包含华字的
"华*"   华字后边多个字符
"华?"  华字后边一个字符
"*华"或"?华" 会引发全表（全部倒排索引）扫描，注意效率问题
~~~

~~~es
# wildcard 查询。查询条件分词，模糊查询
GET goods/_search
{
  "query": {
    "wildcard": {
      "title": {
        "value": "华*"
      }
    }
  }
}
~~~

#### 2.3.4 正则

~~~txt
\W：匹配包括下划线的任何单词字符，等价于 [A-Z a-z 0-9_]   开头的反斜杠是转义符

+号多次出现

(.)*为任意字符
正则查询取决于正则表达式的效率

GET goods/_search
{
  "query": {
    "regexp": {
      "title": "\\w+(.)*"
    }
  }
}
~~~

#### 2.3.5 前缀查询

对keyword比较友好

~~~json
# 前缀查询 对keyword类型支持比较好
GET goods/_search
{
  "query": {
    "prefix": {
      "brandName": {
        "value": "三"
      }
    }
  }
}
~~~

java api

~~~java 
//模糊查询
WildcardQueryBuilder query = QueryBuilders.wildcardQuery("title", "华*");//华后多个字符
//正则查询
 RegexpQueryBuilder query = QueryBuilders.regexpQuery("title", "\\w+(.)*");
 //前缀查询
 PrefixQueryBuilder query = QueryBuilders.prefixQuery("brandName", "三");
~~~

#### 2.3.6 范围&排序查询

~~~json
# 范围查询
GET goods/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 2000,// 大于等于 gt大于
        "lte": 3000
      }
    }
  }, // 使用查询完的结果排序
  "sort": [  
    {
      "price": {
        "order": "desc" //desc降序、asc升序
      }
    }
  ]
}
~~~

java api

~~~java 
 //范围查询 以price 价格为条件
RangeQueryBuilder query = QueryBuilders.rangeQuery("price");
//指定下限
query.gte(2000);
//指定上限
query.lte(3000);
sourceBuilder.query(query);
//排序  价格 降序排列
sourceBuilder.sort("price",SortOrder.DESC);
~~~

#### 2.3.7 queryString 多条件查询

- 会对查询条件进行分词。

- 然后将分词后的查询条件和词条进行等值匹配

- 默认对查询结果取并集（OR）

- 可以指定多个查询字段

query_string：识别query中的连接符（or 、and）

~~~json
# queryString

GET goods/_search
{
  "query": {
    "query_string": {
      "fields": ["title","categoryName","brandName"], 
      "query": "华为 AND 手机"
    }
  }
}

GET goods/_search
{
  "query": {
    "query_string": {
      "fields": ["title","brandName","categoryName"],
      "query": "华为手机 "
      , "default_operator": "AND"
    }
  }
}
~~~

simple_query_string：不识别query中的连接符（or 、and），查询时会将 “华为”、"and"、“手机”分别进行查询

~~~json
GET goods/_search
{
  "query": {
    "simple_query_string": {
      "fields": ["title","categoryName","brandName"], 
      "query": "华为 AND 手机"
    }
  }
}

GET goods/_search
{
  "query": {
    "simple_query_string": {
      "fields": ["title","brandName","categoryName"],
      "query": "华为手机 "
      , "default_operator": "OR"
    }
  }
}
~~~

Java api

~~~java
QueryStringQueryBuilder query = QueryBuilders.queryStringQuery("华为手机").field("title").field("categoryName")
.field("brandName").defaultOperator(Operator.AND);
~~~

注意：query中的or   and 是查询时 匹配条件是否同时出现----or 出现一个即可，and 两个条件同时出现

default_operator的or   and 是对结果进行 并集（or）、交集（and）

#### 2.3.8 boolQuery 布尔查询

boolQuery：对多个查询条件连接。连接方式：

- must（and）：条件必须成立

- must_not（not）：条件必须不成立

- should（or）：条件可以成立

- filter：条件必须成立，性能比must高。不会计算得分


**得分:**即条件匹配度,匹配度越高，得分越高

~~~json
# boolquery
#must和filter配合使用时，max_score（得分）是显示的
#must 默认数组形式
GET goods/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "brandName": {
              "value": "华为"
            }
          }
        }
      ],
      "filter":[ 
        {
        "term": {
          "title": "手机"
        }
       },
       {
         "range":{
          "price": {
            "gte": 2000,
            "lte": 3000
         }
         }
       }
      
      ]
    }
  }
}
#filter 单独使用   filter可以是单个条件，也可多个条件（数组形式）
GET goods/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "brandName": {
              "value": "华为"
            }
          }
        }
      ]
    }
  }
}
~~~

java api

~~~java 
//1.构建boolQuery
BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
//2.构建各个查询条件
//2.1 查询品牌名称为:华为
TermQueryBuilder termQueryBuilder = QueryBuilders.termQuery("brandName", "华为");
boolQuery.must(termQueryBuilder);
//2.2. 查询标题包含：手机
MatchQueryBuilder matchQuery = QueryBuilders.matchQuery("title", "手机");
boolQuery.filter(matchQuery);

//2.3 查询价格在：2000-3000
RangeQueryBuilder rangeQuery = QueryBuilders.rangeQuery("price");
rangeQuery.gte(2000);
rangeQuery.lte(3000);
boolQuery.filter(rangeQuery);

sourceBuilder.query(boolQuery);
~~~

#### 2.3.9 聚合查询

- 指标聚合：相当于MySQL的聚合函数。max、min、avg、sum等

- 桶聚合：相当于MySQL的 group by 操作。不要对text类型的数据进行分组，会失败。 因为text的类型会自动分词，会变成不同的词条存储

~~~json
# 指标聚合 聚合函数
GET goods/_search
{
  "query": {
    "match": {
      "title": "手机"
    }
  },
  "aggs": {
    "max_price": {
      "max": {
        "field": "price"
      }
    }
  }
}
# 桶聚合  分组
GET goods/_search
{
  "query": {
    "match": {
      "title": "手机"
    }
  },
  "aggs": {
    "goods_brands": {
      "terms": {
        "field": "brandName",
        "size": 100
      }
    }
  }
}
~~~

Java api

~~~Java
/**
     * 聚合查询：桶聚合，分组查询
     * 1. 查询title包含手机的数据
     * 2. 查询品牌列表
     */
@Test
public void testAggQuery() throws IOException {

    SearchRequest searchRequest=new SearchRequest("goods");

    SearchSourceBuilder sourceBuilder=new SearchSourceBuilder();
    //1. 查询title包含手机的数据

    MatchQueryBuilder queryBuilder = QueryBuilders.matchQuery("title", "手机");

    sourceBuilder.query(queryBuilder);
    //2. 查询品牌列表  只展示前100条
    AggregationBuilder aggregation=AggregationBuilders.terms("goods_brands").field("brandName").size(100);
    sourceBuilder.aggregation(aggregation);


    searchRequest.source(sourceBuilder);

    SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);

    //7. 获取命中对象 SearchHits
    SearchHits hits = searchResponse.getHits();

    //7.1 获取总记录数
    Long total= hits.getTotalHits().value;
    System.out.println("总数："+total);

    // aggregations 对象
    Aggregations aggregations = searchResponse.getAggregations();
    //将aggregations 转化为map
    Map<String, Aggregation> aggregationMap = aggregations.asMap();


    //通过key获取goods_brands 对象 使用Aggregation的子类接收  buckets属性在Terms接口中体现

    //        Aggregation goods_brands1 = aggregationMap.get("goods_brands");
    Terms goods_brands =(Terms) aggregationMap.get("goods_brands");

    //获取buckets 数组集合
    List<? extends Terms.Bucket> buckets = goods_brands.getBuckets();

    Map<String,Object>map=new HashMap<>();
    //遍历buckets   key 属性名，doc_count 统计聚合数
    for (Terms.Bucket bucket : buckets) {

        System.out.println(bucket.getKey());
        map.put(bucket.getKeyAsString(),bucket.getDocCount());
    }

    System.out.println(map);

}
~~~



####2.3.10 高亮查询

高亮三要素：高亮字段,前缀,后缀

默认前后缀 ：em -- `<em>手机</em>`

~~~json
GET goods/_search
{
  "query": {
    "match": {
      "title": "电视"
    }
  },
  "highlight": {
    "fields": {
      "title": {
        "pre_tags": "<font color='red'>",
        "post_tags": "</font>"
      }
    }
  }
}
~~~

Java api

~~~Java
/**
     * 高亮查询：
     *  1. 设置高亮
     *      * 高亮字段
     *      * 前缀
     *      * 后缀
     *  2. 将高亮了的字段数据，替换原有数据
     */
@Test
public void testHighLightQuery() throws IOException {


    SearchRequest searchRequest = new SearchRequest("goods");

    SearchSourceBuilder sourceBulider = new SearchSourceBuilder();

    // 1. 查询title包含手机的数据
    MatchQueryBuilder query = QueryBuilders.matchQuery("title", "手机");

    sourceBulider.query(query);

    //设置高亮
    HighlightBuilder highlighter = new HighlightBuilder();
    //设置三要素
    highlighter.field("title");
    //设置前后缀标签
    highlighter.preTags("<font color='red'>");
    highlighter.postTags("</font>");

    //加载已经设置好的高亮配置
    sourceBulider.highlighter(highlighter);

    searchRequest.source(sourceBulider);

    SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);


    SearchHits searchHits = searchResponse.getHits();
    //获取记录数
    long value = searchHits.getTotalHits().value;
    System.out.println("总记录数："+value);

    List<Goods> goodsList = new ArrayList<>();
    SearchHit[] hits = searchHits.getHits();
    for (SearchHit hit : hits) {
        String sourceAsString = hit.getSourceAsString();
        //转为java
        Goods goods = JSON.parseObject(sourceAsString, Goods.class);

        // 获取高亮结果，替换goods中的title
        Map<String, HighlightField> highlightFields = hit.getHighlightFields();
        HighlightField HighlightField = highlightFields.get("title");
        Text[] fragments = HighlightField.fragments();
        //highlight title替换 替换goods中的title
        goods.setTitle(fragments[0].toString());
        goodsList.add(goods);
    }
}
~~~

