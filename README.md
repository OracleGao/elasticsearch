# Preface
Elasticsearch is a distributed, scalable, real-time search and analytics engine. It is used for full-text search, structured search, analytics, and all three in combination
## 成功案例
- Wikipedia uses Elasticsearch to provide full-text search with highlighted search snippets, and search-as-you-type and did-you-mean suggestions.（全文搜索）
- The Guardian uses Elasticsearch to combine visitor logs with social -network data to provide real-time feedback to its editors about the public’s response to new articles.（实时分析响应）
- Stack Overflow combines full-text search with geolocation queries and uses more-like-this to find related questions and answers.（结合地理位置的全文检索）
- GitHub uses Elasticsearch to query 130 billion lines of code.（代码检索）
## 优点
- 强伸缩性，小到笔记本，大到成百台服务器，pb级数据量
- 组合分析系统，分布式数据库，并提供一致性和实时性的应用（强强联合）。

## 涉及的内容
- structured search
- analytics
- the complexities of dealing with human language
- geolocation
- relationships
- model your data to take advantage of the horizontal scalability of Elasticsearch
- configure and monitor your cluster when moving to production

## 额外的话题
- Life Inside a Cluster
- Distributed Document Store
- Distributed Search Execution
-  Inside a Shard

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
安装部署见下文
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
##### 对象数组被拉平
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

### Dynamic mapping

# You Konw, for Search（如何搭建搜索引擎）
## 主要目标
be able to integrate your application with Elasticsearch
## 阶段目标
- how to get your data in and out of Elasticsearch
- how Elasticsearch interprets the data in your documents
- how basic search works
- how to manage indices

## 简介
Elasticsearch is an open-source search engine built on top of Apache Lucene™, a full-text search-engine library.
- 与Lucene相比，封装了其复杂性，暴露简单的restful接口
- 增加实时文档存储和检索、实时分析和伸缩功能
- 降低搜索引擎入门成本

基于 Apache 2 license开源，源码地址：[github.com/elastic/elasticsearch](https://github.com/elastic/elasticsearch)

## Installation
get latest version from [elastic.co/downloads/elasticsearch](https://www.elastic.co/downloads/elasticsearch)
### Linux
#### download
```shell
wget -c https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.2.2.tar.gz
```
#### 解压
```shell
tar czvf elasticsearch-5.2.2.tar.gz
```
#### 创建专用用户组并授权
由于ElasticSearch可以接收用户输入的脚本并且执行，为了系统安全考虑， 建议创建一个单独的用户用来运行ElasticSearch。
```shell
#添加用户级用户组
useradd elsearch -d /home/elsearch -m -s /bin/bash -U
#修改密码
passwd elsearch
#授权
chown -R elsearch:elsearch elasticsearch-5.2.2
```
#### startup
使用新创建的非root用户执行
```shell
cd elasticsearch-5.2.2
./bin/elasticsearch ${arg1}
```
- arg1: -d        run in background as a daemon
#### test
##### request
```shell
curl 'http://localhost:9200/?pretty'
```
##### response
```json
{
  "name" : "-03UhNJ",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "90sny5raROypXKLu4veCYQ",
  "version" : {
    "number" : "5.2.2",
    "build_hash" : "f9d9b74",
    "build_date" : "2017-02-24T17:26:45.835Z",
    "build_snapshot" : false,
    "lucene_version" : "6.4.1"
  },
  "tagline" : "You Know, for Search"
}
```
#### Installing and run Sense
##### install Kibana
```shell
wget -c https://artifacts.elastic.co/downloads/kibana/kibana-5.2.2-linux-x86_64.tar.gz
tar xzvf kibana-5.2.2-linux-x86_64.tar.gz
```
##### command line install
在Kinbana目录下执行以下命令
```shell
./bin/kibana ./plugin-install elastic/sense
```
##### offline install
download offline package and install
```shell
wget -c https://download.elasticsearch.org/elastic/sense/sense-latest.tar.gz
#在Kinbana目录下执行以下命令
./bin/kibana-plugin install file:///PATH_TO_SENSE_TAR_FILE
```
##### startup Kibana
```shell
./bin/kibana
```
##### browse the sense web site 
http://localhost:5601/app/sense

### docker
Elastic provides open-source support for Elasticsearch via the [elastic/elasticsearch GitHub repository](https://github.com/elastic/elasticsearch) and the Docker image via the [elastic/elasticsearch-docker GitHub repository](https://github.com/elastic/elasticsearch-docker), as well as community support via its [forums](https://discuss.elastic.co/c/elasticsearch).
#### 拉取官方镜像
docker 安装请参考[docker官方文档](https://docs.docker.com/engine/installation/)
```shell
docker pull docker.elastic.co/elasticsearch/elasticsearch:5.2.2
```
TBD

## Talking to Elasticsearch
### JAVA API
TBD

### Spring Boot集成
TBD

### RESTful Api with JSON over HTTP
#### http请求
```shell
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```
- VERB http method 包括GET, POST, PUT, HEAD, DELETE
- PROTOCOL http / https
- HOST 主机地址
- PORT 端口默认9200
- PATH 资源路径
- QUERY_STRING 查询参数
- BODY 请求体 JSON格式

#### http请求示例
```shell
curl -XGET -v 'http://localhost:9200/_count?pretty' -d '
{
    "query": {
        "match_all": {}
    }
}'
```
```text
< HTTP/1.1 200 OK
< content-type: application/json; charset=UTF-8
< content-length: 95
<
{
  "count" : 0,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  }
}
```
## Document Oriented
存储对象或文档并对其建立索引便于检索
### JSON
json作为文档序列化格式
处理JSON比处理传统关系数据库表代价低

## 实践：构建Employee Directory
1. Enable data to contain multi value tags, numbers, and full text.
2. Retrieve the full details of any employee.
3. Allow structured search, such as finding employees over the age of 30.
4. Allow simple full-text search and more-complex phrase searches.
5. Return highlighted search snippets from the text in the matching documents.
6. Enable management to build analytic dashboards over the data.

### 创建Employee Directory索引（建库index，建表type）
- Index a document per employee, which contains all the details of a single employee.
- Each document will be of type employee.表名 employee
- That type will live in the megacorp index.库名 megacorp
- That index will reside within our Elasticsearch cluster.数据库index位于集群中

#### 插入数据（1.Enable data to contain multi value tags, numbers, and full text.）
```shell
curl -XPUT -v 'http://localhost:9200/megacorp/employee/1?pretty' -d  '
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}'
```
```text
< HTTP/1.1 201 Created
< Location: /megacorp/employee/1
< content-type: application/json; charset=UTF-8
< content-length: 206
<
{
  "_index" : "megacorp",
  "_type" : "employee",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "created" : true
}
```
再次put就为数据更新，同时响应信息中的_version会自增1
```shell
curl -XPUT -v 'http://localhost:9200/megacorp/employee/2?pretty' -d  '
{
    "first_name" :  "Jane",
    "last_name" :   "Smith",
    "age" :         32,
    "about" :       "I like to collect rock albums",
    "interests":  [ "music" ]
}'

curl -XPUT -v 'http://localhost:9200/megacorp/employee/3?pretty' -d  '
{
    "first_name" :  "Douglas",
    "last_name" :   "Fir",
    "age" :         35,
    "about":        "I like to build cabinets",
    "interests":  [ "forestry" ]
}'
```
#### 查询单条详细数据（2. Retrieve the full details of any employee.）
```shell
curl -XGET -v 'http://localhost:9200/megacorp/employee/1?pretty' 
```
```text
< HTTP/1.1 200 OK
< content-type: application/json; charset=UTF-8
< content-length: 294
<
{
  "_index" : "megacorp",
  "_type" : "employee",
  "_id" : "1",
  "_version" : 1,
  "found" : true,
  "_source" : {
    "first_name" : "John",
    "last_name" : "Smith",
    "age" : 25,
    "about" : "I love to go rock climbing",
    "interests" : [
      "sports",
      "music"
    ]
  }
}
```
```shell
curl -XGET -v 'http://localhost:9200/megacorp/employee/2?pretty' 

curl -XGET -v 'http://localhost:9200/megacorp/employee/3?pretty' 
```

#### 结构化查询（Allow structured search, such as finding employees over the age of 30.）
##### url查询
```shell
curl -XGET -v 'http://localhost:9200/megacorp/employee/_search?pretty'
```
```text
< HTTP/1.1 200 OK
< content-type: application/json; charset=UTF-8
< content-length: 1275
<
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 3,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "3",
        "_score" : 1.0,
        "_source" : {
          "first_name" : "Douglas",
          "last_name" : "Fir",
          "age" : 35,
          "about" : "I like to build cabinets",
          "interests" : [
            "forestry"
          ]
        }
      }
    ]
  }
}
```
```shell
curl -XGET -v 'http://localhost:9200/megacorp/employee/_search?q=last_name:Smith&pretty'
```
```text
< HTTP/1.1 200 OK
< content-type: application/json; charset=UTF-8
< content-length: 941
<
{
  "took" : 14,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : 0.2876821,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.2876821,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 0.2876821,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      }
    ]
  }
}
```
##### url json body的 DSL(domain-specific language)查询
```shell
curl -XGET -v 'http://localhost:9200/megacorp/employee/_search?pretty' -d '
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}'
```
```shell
curl -XGET -v 'http://localhost:9200/megacorp/employee/_search?pretty' -d '
{
    "query" : {
        "bool" : {
            "must" : {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter" : {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}'
```
#### 全文检索（模糊检索）和词组检索（精确检索）(4. Allow simple full-text search and more-complex phrase searches.)
##### 全文检索
```shell
curl -XGET -v 'http://localhost:9200/megacorp/employee/_search?pretty' -d '
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}'
```
```text
< HTTP/1.1 200 OK
< content-type: application/json; charset=UTF-8
< content-length: 943
<
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : 0.53484553,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 0.53484553,   #相关性得分
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.26742277,   #相关性得分
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      }
    ]
  }
}
```
结果按照相关性（ relevance score）得分排序
##### 词组检索
```shell
curl -XGET -v 'http://localhost:9200/megacorp/employee/_search?pretty' -d '
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}'
```
```text
< HTTP/1.1 200 OK
< content-type: application/json; charset=UTF-8
< content-length: 582
<
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 0.53484553,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 0.53484553,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      }
    ]
  }
}
```
精确匹配词组
#### 高亮查询结果（Return highlighted search snippets from the text in the matching documents.）
```shell
curl -XGET -v 'http://localhost:9200/megacorp/employee/_search?pretty' -d '
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}'
```
```text
< HTTP/1.1 200 OK
< content-type: application/json; charset=UTF-8
< content-length: 711
<
{
  "took" : 41,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 0.53484553,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 0.53484553,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        },
        "highlight" : {
          "about" : [
            "I love to go <em>rock</em> <em>climbing</em>"
          ]
        }
      }
    ]
  }
}
```
#### 分析和数据展示（6.Enable management to build analytic dashboards over the data.）
##### 分析（聚集）
```shell
curl -XGET -v 'http://localhost:9200/megacorp/employee/_search?pretty' -d '
{
  "aggs": {
    "all_interests": {
      "terms": { "field": "interests" }
    }
  }
}'
```
```text

```

#### 关于中文字符的支持
```shell
curl -XPUT -v 'http://localhost:9200/megacorp/employee/4?pretty' -d  '
{
    "name" :   "张三",
    "age" :         35,
    "about":        "我喜欢读书也喜欢看电影",
    "interests":  [ "阅读", "影视" ]
}'

curl -XPUT -v 'http://localhost:9200/megacorp/employee/5?pretty' -d  '
{
    "name" :   "李四",
    "age" :         35,
    "about":        "我爱边旅行边读书",
    "interests":  [ "阅读", "旅游" ]
}'

curl -XPUT -v 'http://localhost:9200/megacorp/employee/6?pretty' -d  '
{
    "name" :   "王五",
    "age" :         35,
    "about":        "我就爱呆在家里看电影",
    "interests":  [ "宅", "影视" ]
}'

curl -XGET -v 'http://localhost:9200/megacorp/employee/4?pretty'
curl -XGET -v 'http://localhost:9200/megacorp/employee/5?pretty'
curl -XGET -v 'http://localhost:9200/megacorp/employee/6?pretty'

curl -XGET -v 'http://localhost:9200/megacorp/employee/_search?pretty' -d '
{
    "query" : {
        "match" : {
            "about" : "读书影视"
        }
    }
}'

curl -XGET -v 'http://localhost:9200/megacorp/employee/_search?pretty' -d '
{
    "query" : {
        "match_phrase" : {
            "about" : "读书"
        }
    }
}'

curl -XGET -v 'http://localhost:9200/megacorp/employee/_search?pretty' -d '
{
    "query" : {
        "match" : {
            "about" : "读书 电影"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}'
```


# Structured Search
## 目标
query and index data with more-advanced concepts
## 阶段目标
- understand how relevance works
- understand how to contorl relevance works to get best result

# Dealing with Human Language

# show data overall trends

#  Geo Points

#  Designing for Scale

#  Monitoring
