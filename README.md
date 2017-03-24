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

# [基本概念](https://github.com/OracleGao/elasticsearch/blob/master/concept.md)

# Getting Started
## 主要目标
be able to integrate your application with Elasticsearch
## 阶段目标
- how to get your data in and out of Elasticsearch
- how Elasticsearch interprets the data in your documents
- how basic search works
- how to manage indices

## 额外的话题
- Life Inside a Cluster
- Distributed Document Store
- Distributed Search Execution
-  Inside a Shard

## [You Know, for Search…（如何搭建搜索引擎）](https://github.com/OracleGao/elasticsearch/blob/master/You%20Know%20for%20Search.md)

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
