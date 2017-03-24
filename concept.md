## 基本概念
### Near Realtime(NRT) 近实时
存储的文档(index a document) 秒级延迟可以被检索到
### Cluster 集群
多节点（服务器）的集合，holds整个的数据并且提供跨界点的联邦索引和搜索能力。
集群有唯一标识名称，默认“elasticsearch”，通过该名称与node形成一对多的关系。
不同环境下集群不要重名，防止节点加入错误的集群。
### Node 节点
node就是单独的服务器，是cluster的一部分，用来存储数据，参与cluster的索引和检索功能。node的名字是与cluster类似，也作为node的标识名称，默认UUID。节点名用来管理和维护其与cluster的关系。
node通过配置其cluster标识名称来加入该cluster。默认cluster标识名称是“elasticsearch”（默认加入该集群）
single-node cluster会默认启动一个node，如果网络环境内找不到任何能够加入该cluster的node
### Index（索引）
文档的集合，由其名字标识。
- Index（名词）： 文档的集合，名词词性。类比于关系数据库的schema
- Index（动词）： 插入或更新一个文档数据
- Inverted Index：加快检索的索引，类比与关系数据库中的索引(B-tree)
### Type（类型）
index的逻辑分类或分区，由具体业务逻辑决定语义。类比于关系数据库的表。
### Document(文档)
被index(动词)的信息基本单元。document使用JSON表达。document物理上归于type
### Shards & Replicas
index(名词)存储的数据有可能超过单节点的硬件限制。为了解决该问题，Elasticsearch将这样的index(名词)分割成很多片段，这些片段叫Shards。
#### shards好处
- 水平分割解决限制问题
- 充分利用分布式和node并行处理提高性能和吞吐
shards的分布式和聚集检索机制对用户透明
####shards分类
- primary shards index（名词）源生分块
- replica shards提供index（名词）冗余分块。提高数据可靠性，而且通过node分布增加并行性提高系统效率和吞吐
shards中document数量上限2,147,483,519 (= Integer.MAX_VALUE - 128) 
#### 查看shards信息
```shell
curl -XGET 'http://localhost:9200/_cat/shards'
```
## Mapping 映射
mapping是是一个过程，定义了一个document及其fileds怎么存储和index(动词)。它定义了：
- string fields 被当作full text fields
- fiedls 包括数字，日期和地理信息
- 是否document中所有fields的值都应该被index（动词）到_all field中
- 数据值的格式
- 自定义规则来控制动态添加 fields
### mapping types
每一种index（名词）具有一个或多个mapping types，用于将document按照逻辑分组。
每一个mapping type具有：
- meta-fields 元数据field， 提供一些自定义功能。它包括：
* _index,
*  _type
* _id
* _source
- fileds or properties，类比于关系数据库表中的列。同一个index(名词)中不同mapping type中相同名字的field,具有相同的映射。(不同mapping type中出现同名的field就被认为是同一个filed。)
### Field datatypes 域数据类型
#### 核心类型
- string 包括text和keyword(做排序和聚集操作)
##### 关于keyword类型
index（动词）结构化文本的内容比如：email地址，主机名，状态码，邮政编码，标签。主要用户过滤，排序，聚集。该field只做精准值匹配

- 数值类型 long, integer, short, byte, double, float
- date
- boolean
- brinary 二进制类型
- 范围类型 integer_range, float_range, long_range, double_range, date_range
#### 复合类型
- 数组类型
- object json对象
- nested 嵌套类型
允许对象数组被独立索引和查询，而不是默认被系统拉平。
##### 关于nested类型——对象数组被拉平
内部对象field的数组会被系统转换为平级的简单的list数据结果存储。详见下面例子
```shell
curl -XPUT 'http://localhost:9200/my_index/my_type/1?pretty' -d  '
{
  "group" : "fans",
  "user" : [ 
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}'
curl -XGET 'http://localhost:9200/my_index/_search?pretty' -d '
{
  "query": {
    "bool": {
      "must": [
        { "match": { "user.first": "Alice" }},
        { "match": { "user.last":  "Smith" }}
      ]
    }
  }
}'
```
要查询first name是Alice，并且last name是Smith的人，根据上面的记录，结果应该为空，但实际如下。
```text
{
  "_index" : "my_index",
  "_type" : "my_type",
  "_id" : "1",
  "_version" : 1,
  "found" : true,
  "_source" : {
    "group" : "fans",
    "user" : [
      {
        "first" : "John",
        "last" : "Smith"
      },
      {
        "first" : "Alice",
        "last" : "White"
      }
    ]
  }
}
```
检索出2个document，但是并不符合要求，因为系统为了检索优化，默认将
```text
{
  "group" : "fans",
  "user" : [ 
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}
```
数据结构变成变成
```text
{
  "group" :        "fans",
  "user.first" : [ "alice", "john" ],
  "user.last" :  [ "smith", "white" ]
}
```
故符合检索条件。如果想符合我们的预期，需要将user声明成nested的类型，在系统数据结构中保持index（动词）时的数据结构。
创建新的index（名词）my_index2并设定user数据类型为nested，然后插入数据，在做检索具体如下：
```shell
curl -XPUT 'http://localhost:9200/my_index2?pretty' -d '
{
  "mappings": {
    "my_type": {
      "properties": {
        "user": {
          "type": "nested" 
        }
      }
    }
  }
}'
curl -XPUT 'http://localhost:9200/my_index2/my_type/1?pretty' -d '
{
  "group" : "fans",
  "user" : [
    {
      "first" : "John",
      "last" :  "Smith"
    },
    {
      "first" : "Alice",
      "last" :  "White"
    }
  ]
}'
curl -XGET 'http://localhost:9200/my_index2/_search?pretty' -d '
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            { "match": { "user.first": "Alice" }},
            { "match": { "user.last":  "Smith" }} 
          ]
        }
      },
      "inner_hits": {}
    }
  }
}'
```
再看结果符合我们的预期了，如下：
```text
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 0,
    "max_score" : null,
    "hits" : [ ]
  }
}
```
再试一个White Alice的情况：
```shell
curl -XGET 'http://localhost:9200/my_index2/_search?pretty' -d '
{
  "query": {
    "nested": {
      "path": "user",
      "query": {
        "bool": {
          "must": [
            { "match": { "user.first": "Alice" }},
            { "match": { "user.last":  "White" }} 
          ]
        }
      },
      "inner_hits": {}
    }
  }
}'
```
在inner_hits中，展示了我们想要的结果。如下：
```text
{
  "took" : 9,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 1.3862944,
    "hits" : [
      {
        "_index" : "my_index2",
        "_type" : "my_type",
        "_id" : "1",
        "_score" : 1.3862944,
        "_source" : {
          "group" : "fans",
          "user" : [
            {
              "first" : "John",
              "last" : "Smith"
            },
            {
              "first" : "Alice",
              "last" : "White"
            }
          ]
        },
        "inner_hits" : {
          "user" : {
            "hits" : {
              "total" : 1,
              "max_score" : 1.3862944,
              "hits" : [
                {
                  "_nested" : {
                    "field" : "user",
                    "offset" : 1
                  },
                  "_score" : 1.3862944,
                  "_source" : {
                    "first" : "Alice",
                    "last" : "White"
                  }
                }
              ]
            }
          }
        }
      }
    ]
  }
}
```
#### 地理类型
- geo_point 地理信息点，接收经度，维度数据对
- geo_shape 地理形状
#### 特殊类型
- IP类型
- 自动补全类型
- Token计数类型
- mapper-murmur3  hash值类型
- attachment 外挂类型 支持html, Microsoft Office 格式, Open Document 格式
- percolator 过滤器类型 接收查询dsl
####  multi-field 多类型field
同一个field可以设置多同类型，可以设置为text用来全文检索，也可以同时设置为keyword用来做聚集，排序，还可以同时设置的分析器，用来指定使用英文分析还是法文分析。
String类型的field可以具有（mapped as）text field 用来做全文检索，也可以具有keyword field 用来排序和聚集。看一个例子：
```shell
curl -XPUT 'http://localhost:9200/multi_field_index?pretty' -d '
{
  "mappings": {
    "my_type": {
      "properties": {
        "city": {
          "type": "text",
          "fields": {
            "raw": { 
              "type":  "keyword"
            }
          }
        }
      }
    }
  }
}'
curl -XPUT 'http://localhost:9200/multi_field_index/my_type/1?pretty' -d '
{
  "city": "New York"
}'
curl -XPUT 'http://localhost:9200/multi_field_index/my_type/2?pretty' -d '
{
  "city": "York"
}'
curl -XGET 'http://localhost:9200/multi_field_index/_search?pretty' -d '
{
  "query": {
    "match": {
      "city": "york" 
    }
  },
  "sort": {
    "city.raw": "desc" 
  },
  "aggs": {
    "Cities": {
      "terms": {
        "field": "city.raw" 
      }
    }
  }
}'
```
结果hits中按照field.raw排序并按照各自的完整字段进行聚集，结果如下：
```text
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : null,
    "hits" : [
      {
        "_index" : "multi_field_index",
        "_type" : "my_type",
        "_id" : "2",
        "_score" : null,
        "_source" : {
          "city" : "York"
        },
        "sort" : [
          "York"
        ]
      },
      {
        "_index" : "multi_field_index",
        "_type" : "my_type",
        "_id" : "1",
        "_score" : null,
        "_source" : {
          "city" : "New York"
        },
        "sort" : [
          "New York"
        ]
      }
    ]
  },
  "aggregations" : {
    "Cities" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "New York",
          "doc_count" : 1
        },
        {
          "key" : "York",
          "doc_count" : 1
        }
      ] 
    }
  }
}
```
### 合理设置mapping防止爆掉
- index.mapping.total_fields.limit 一个index（名词）的fields数量上限默认1000
- index.mapping.depth.limit 嵌套对象层数，一个对象是1层，该对象内包含一个层数是2的对象，以此类推。默认20
- index.mapping.nested_fields.limit 1个具有100个nested 的docuemnt 实际是101个nested的document。默认nested document限制个数50

### Dynamic mapping 动态映射
index(动词)一个document时自动添加mapping type。
三个维度的配置
- 新建index（名词）时默认mapping type的配置规则
- 治理动态field的规则
- 添加新field的配置规则
#### type上禁用 Dynamic mapping
```shell
curl -XPUT 'http://localhost:9200/my_index/_settings?pretty' -d '
{
  "index.mapper.dynamic": true
}'
```
```text
{
  "acknowledged" : true
}
```
#### 禁用所有index（名词）上的Dynamic mapping
```shell
curl -XPUT 'http://localhost:9200/_template/template_all?pretty' -d '
{
  "template": "*",
  "order":0,
  "settings": {
    "index.mapper.dynamic": false
  }
}'
```
```text
{
  "acknowledged" : true
}
```
### Explicit mappings 确定映射
建立Index（名词）时指定，或直接更新已有index
### Updating existing mappings 更新mappings
已存在的type and field不能更新，只能创建新的index(名词)，重新reindex数据到新创建的index（名词）中
### Fields are shared across mapping types
field(同名标识同一实体)：
- 同样的名字
- 在同一index（名词）中
- 在不同的mapping types中
- 内部映射为同一个field
- 必须有同样的mapping
