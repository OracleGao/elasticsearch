# You Konw, for Search（如何搭建搜索引擎）
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
在interests属性上增加multi-field keyword用来实现聚集操作
```shell
curl -XGET 'http://localhost:9200/megacorp/employee/_search?pretty' -d '
{
  "aggs": {
    "all_interests": {
      "terms": { "field": "interests.keyword" }
    }
  }
}'
```
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
    ##省略数据集
  },
  "aggregations" : {
    "all_interests" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "music",
          "doc_count" : 2
        },
        {
          "key" : "forestry",
          "doc_count" : 1
        },
        {
          "key" : "sports",
          "doc_count" : 1
        }
      ]
    }
  }
}
```
带过滤条件安的查询，我们选取last_name是smith的document。详情如下：
```shell
curl -XGET 'http://localhost:9200/megacorp/employee/_search?pretty' -d '
{
  "query": {
    "match": {
      "last_name": "smith"
    }
  },
  "aggs": {
    "all_interests": {
      "terms": {
        "field": "interests.keyword"
      }
    }
  }
}'
```
```text
{
  "took" : 1,
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
  },
  "aggregations" : {
    "all_interests" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "music",
          "doc_count" : 2
        },
        {
          "key" : "sports",
          "doc_count" : 1
        }
      ]
    }
  }
}
```
对聚集结果还可以进行二次聚集，现在我们在上个例子的聚集基础上，计算出平均年龄。详情如下：
```shell
curl -XGET 'http://localhost:9200/megacorp/employee/_search?pretty' -d '
{
  "query": {
    "match": {
      "last_name": "smith"
    }
  },
  "aggs": {
    "all_interests": {
      "terms": {
        "field": "interests.keyword"
      },
      "aggs" : {
         "avg_age" : {
             "avg" : { "field" : "age" }
         }
      }
    }
  }
}'
```
```text
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
  },
  "aggregations" : {
    "all_interests" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "music",
          "doc_count" : 2,
          "avg_age" : {
            "value" : 28.5
          }
        },
        {
          "key" : "sports",
          "doc_count" : 1,
          "avg_age" : {
            "value" : 25.0
          }
        }
      ]
    }
  }
}
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
## Distributed Nature分布式天性 （manage indices）
分布式过程中系统自动在后台做的事情：
- document分区到不同的容器或shards,好用来在分布式node上存储
- 在cluster的各个node中均衡分布shards，均衡索引操作和检索的负载
- 根据冗余设定，进行shard冗余
- 在node间路由请求，获取你的查询数据
- 无缝集成新增node,当有node丢失时，重新分布shard。
