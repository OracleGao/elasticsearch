# can not run elasticsearch as root
elasticsearch可以在服务端执行脚本，为了安全考虑，必须使用非root用户启动
## 添加elasticsearch用户以及用户组
### 命令行方式
```shell
#添加用户级用户组
useradd elsearch -d /home/elsearch -m -s /bin/bash -U
#修改密码
passwd elsearch
```
### 交互方式
```shell
adduser elsearch
```
## 新用户授权
```shell
chown -R elsearch:elsearch elasticsearch-5.2.2
```
## 切换到新用户启动
```shell
su elsearch
# elasticsearch安装目录
./bin/elasticsearch
```

# Ubuntu max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]
使用root用户
1. 修改系统配置/etc/security/limits.conf
```shell
cat <<EOF>> /etc/security/limits.conf
${ELSTICSEARCH_USERNAME} hard nofile 65536
${ELSTICSEARCH_USERNAME} soft nofile 65536
EOF
```
2. 使用${ELSTICSEARCH_USERNAME} 重新登录
3. 检查修改结果
```shell
ulimit -n
65536
```

# max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
1.  修改系统配置/etc/sysctl.conf
```shell
cat <<EOF>> /etc/sysctl.conf
vm.max_map_count=262144
EOF
```
2. 查看修改结果
```shell
sysctl -p
vm.max_map_count = 262144
```
# Fielddata is disabled on text fields by default.  Set fielddata=true
在field属性上使用keyword关键字查询。如下：在interests field后增加“.keyword”进行聚集处理
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
## 关于fielddata
fielddata 是text类型field使用的一种内存数据结构，在第一次使用聚集，排序，或脚本的查询时构建。它会从磁盘读取整个 inverted index。
- "Which documents contain this term?" for search
- "What is the value of this field for this document?" for aggregations
### 默认关闭feilddata的原因
- Fielddata会消耗大量内存堆空间
- 装载过程会消耗大量资源，并且期间增加用户访问服务的延迟时间
### reference
https://www.elastic.co/guide/en/elasticsearch/reference/current/fielddata.html
