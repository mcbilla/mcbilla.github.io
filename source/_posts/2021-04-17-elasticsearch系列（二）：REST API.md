---
title: elasticsearch系列（二）：REST API
date: 2021-04-17 11:44:55
categories:
- elasticsearch
tags:
- elasticsearch
---

> elasticsearch系列（二）：REST API

<!--more-->

## 一、命令简介
可以使用 RESTful API 通过端口 9200 和 Elasticsearch 进行通信，甚至可以使用 curl 命令来和 Elasticsearch 交互。
> 旧版的 Elasticsearch Clients 可以使用 9300 端口和 Elasticsearch 进行 tcp 通信，例如 spring-data-elasticsearch:transport-api.jar 包，7.x 已经不建议使用，8 以后就要废弃。建议统一使用 9200 端口以 http 请求的方式和 Elasticsearch 通讯。
```
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```
* **VERB**：HTTP 方法，可以使用下面方法：
  * POST：创建，可以不指定id，es自己生成不会发生碰撞的UUID
  * PUT：创建/更新，需要指定具体的id，如果id已经存在就会覆盖原来的内容，version会加1。
  * GET：查看
  * DELETE：删除
* **PROTOCOL**：http 或者 https（如果你在 Elasticsearch 前面有一个 https 代理）
* **HOST**：Elasticsearch 集群中任意节点的主机名，或者用 localhost 代表本地机器上的节点。
* **PORT**：运行 Elasticsearch HTTP 服务的端口号，默认是 9200 。
* **PATH**：第一部分通常是索引名称，除非它以_开头。例如 `/products/_search`，其中 products 是索引，_search 是动作。
* **QUERY_STRING**：参数选项，常用的有：
  * ?v：显示列名，例如 `/_cat/master?v`。
  * ?help：显示当前命令的各列含义。例如 `/_cat/master?help`。
  * ?bytes：数值列以指定单位显示, 转为以kb/mb/gb表示，默认为b。例如 `/_cat/indices?bytes=b`。
  * ?h：显示指定列的信息，例如 `_cat/indices?h=docs.count,store.size`。
  * ?s：用于排序，使用列出的字段作为排序键。例如 `/_cat/nodes?v&h=cpu,master,name&s=name`。
  * ?pretty：美化输出成 json 格式，例如 `/_cluster/health?pretty`。
* **BODY**：一个 JSON 格式的请求体 (如果请求需要的话)

例如，计算集群中文档的数量，我们可以用这个:
```
curl -XGET 'http://localhost:9200/_count?pretty' -H 'Content-Type: application/json' -d '
{
    "query": {
        "match_all": {}
    }
}
'
```
如果是在 Dev Tools 控制台，我们会使用下面的缩写形式。下面的命令也写成缩写形式。
```
GET /_count
{
    "query": {
        "match_all": {}
    }
}
```

## 二、常用命令
### 1、集群 API
通常使用 `/_cluster` 来查看集群信息
#### 查看集群健康状态
查看集群健康状态，包括集群名称、节点数、数据节点数、分片等的一些统计信息。
```
GET /_cluster/health 
```
#### 查看集群状况
```
GET /_cluster/state
```
#### 查看集群统计信息
查看集群统计信息，有助于基本故障排除
```
GET /_cluster/stats
```
#### 查看集群配置
```
GET /_cluster/settings
```
#### 查看索引健康状态
```
GET /_cluster/health?level=indices&pretty
```
#### 查看分片健康状态
```
GET /_cluster/health?level=shards&pretty
```

### 2、节点 API
通常使用 `/_nodes` 来查看节点信息
#### 查看所有节点的统计信息
查看所有节点的统计信息，包括堆使用情况等
```
GET /_nodes/stats
```
#### 查看所有节点的索引信息
```
GET /_nodes/stats/indices
```
#### 查看某个节点的统计信息
```
GET /_nodes/<node_id>/stats
```
#### 查看某个节点的索引信息
```
GET /_nodes/<node_id>/stats/indices
```

### 3、CAT API
CAT API 仅适用于使用 Kibana 控制台或命令行供人类使用，输出以表格的形式的文本，而不是 JSON。可以通过下面命令列出所有可用的 API。
```
GET /_cat
```
输出默认是没有表头的，可以在最后加上 `?v` 参数输出表头。这都是模仿 Linix 工具设计的，因为它假设一旦你对输出熟悉了，你就再也不想看见表头了。
#### 查看集群健康状态
```
GET /_cat/health?v
```
#### 查看集群节点状态
查看集群各个节点的当前状态, 包括节点的物理参数(包括os/jdk版本, uptime, 当前mem/disk/fd使用情况等), 请求访问情况(如search/index成功和失败的次数)等详细信息
```
GET /_cat/nodes?v
```
#### 查看所有索引
```
GET /_cat/indices?v 
```
#### 查看分片情况
查看分片情况，包括shard的分布, 当前状态(对于分配失败的shard会有失败原因), doc数量, 磁盘占用情况, shard的访问情况(如所有get请求的成功/失败次数以及对应耗时等)
```
# 查看所有分片
GET /_cat/shards?v 

# 查看指定分片
GET /_cat/indices/${index}
```
#### 查看master信息
```
GET /_cat/master?v
```
#### 查看lucence的段信息
查看lucence的段信息，包括segment名, 所属shard, 内存/磁盘占用大小, 是否刷盘, 是否merge为compound文件等. 可以查看指定index的segment信息()
```
# 查看所有段信息
GET /_cat/segments?v

# 查看某个索引的段信息
GET /_cat/segments/${index}
```
#### 查看索引别名
查看索引别名，包括alias对应的index，路由配置等
```
# 查看所有alias信息
GET /_cat/aliases?v

# 查看指定alias的信息
GET /_cat/aliases/${alias}
```
#### 查看分配资源接口
```
GET /_cat/allocation?v
```
#### 查看文档数量
```
# 查看所有索引的文档数量
GET /_cat/count?v

# 查看指定索引的文档数量
GET /_cat/count/${index}
```
#### 查看模板
```
GET /_cat/templates?v
```

### 4、索引 API
#### 创建索引
有两种方式，第一种是提交数据自动创建索引
```
POST /indexname/_doc/1
{
  "name": "John Doe"
}
```
如果你想禁止自动创建索引，你可以通过在 `config/elasticsearch.yml` 的每个节点下添加下面的配置：
```
action.auto_create_index: false
```
第二种方式是手动创建索引
```
PUT /indexname
{
    "mappings":{
        "properties":{
            "name":{
                "type":"text"
            },
            "blob":{
                "type":"binary"
            }
        }
    }
}
```

#### 查看索引
```
# 查看索引信息
GET /indexname
```

#### 修改索引
```
# 修改mapping，新增sex字段
PUT /indexname/_mapping
{
    "properties":{
        "sex":{
            "type":"text"
        }
    }
}

# 修改setting
PUT  /indexname/_settings
{
    "number_of_replicas":"0"
}
```

#### 删除索引
```
# 删除单个索引
DELETE /indexname 

# 删除多个索引
DELETE /indexname1,/indexname2

# 删除全部索引
DELETE /_all 
或者 
DELETE /*
```

#### 索引别名
```
# 给索引indexname创建一个别名indexname_alias
PUT /indexname/_alias/indexname_alias

# 查看索引别名
GET /indexname/_alias
```

#### 索引模板
```
# 查询所有索引模板
GET _template

# 查看与通配符相匹配的模板
GET _template/my_template

# 查看指定模板
GET _template/my_template*

# 查看多个模板
GET _template/my_template1,my_template2

# 删除索引模板
DELETE _template/my_template
```

### 5、文档
#### 新增文档
新增可以使用 PUT 请求或者 POST 请求。
* PUT请求，必须带id。如果 id 不存在，则会新建一个文档。如果 id 存在，则会用当前数据更新原来的文档。
```
PUT /indexname/_doc/id
{
  "name": "mcb"
}
```
* POST请求，可以带id，也可以不带id。不带id就是新增操作，带id可以为新增或者更新操作。
```
# 不带id，新增文档
POST /indexname/_doc
{
  "name": "mcb"
}

# 带id，如果 id 不存在，则会新建一个文档。如果 id 存在，则会用当前数据更新原来的文档
POST /indexname/_doc/id
{
  "name": "John Doe"
}
```
返回数据带有下划线开头的，称为元数据，反映了当前的基本信息。
```
{
    "_index":"customer",
    "_type":"external",
    "_id":"1",
    "_version":1,
    "result":"created",
    "_shards":{
        "total":2,
        "successful":1,
        "failed":0
    },
    "_seq_no":0,
    "_primary_term":1
}
```
* "_index": "customer" 表明该数据在哪个数据库下；
* "_type": "external" 表明该数据在哪个类型下；
* "_id": "1" 表明被保存数据的id；
* "_version": 1, 被保存数据的版本
* "result": "created" 这里是创建了一条数据，如果重新put一条数据，则该状态会变为updated，并且版本号也会发生变化。

#### 更新文档
PUT/POST操作带id，直接更新整个文档，version号会一直增加。
```
# PUT操作更新
PUT /indexname/_doc/1
{
  "name": "mcb1"
}

# POST操作更新
POST /indexname/_doc/1
{
  "name": "mcb2"
}
```
如果想增加或者更新某些字段，可以使用 POST 操作的 `_update` 。`_update`  会对比元数据，有差异才会更新，然后增加 version 号，否则不进行任何操作。
```
POST /indexname/_update/1
{
  "doc": {
    "age": 18,
    "address": "sz"
  }
}
```
更新携带 `?if_seq_no=0&if_primary_term=1`，还可以实现乐观锁的功能。只有 ` seq_no` 和 `primary_term` 和当前的文档相同才能更新成功，否则会返回 409 错误。在更新完后 ` seq_no` 会往上叠加。
```
PUT /indexname/_doc/3?if_seq_no=2&if_primary_term=1
{
  "name": "mcb1"
}
```

#### 查看文档
```
GET /indexname/_doc/id
```
返回数据
```
{
    "_index": "customer",    //在哪个索引
    "_type": "external",      //在哪个类型
    "_id": "1",                  //记录id
    "_version": 3,             //版本号
    "_seq_no": 6,             //并发控制字段，每次更新都会+1，用来做乐观锁
    "_primary_term": 1,    //同上，主分片重新分配，如重启，就会变化
    "found": true,           //表示找到了数据，如果没找到就为false
    "_source": {              //真正的内容
        "name": "mcb",
        "author": "Jobs"
    }
}
```

#### 删除文档
```
DELETE /indexname/_doc/id
```

#### 检索文档
所有的检索都从 `_search`  开始。如果不带查询条件，会列出某个索引下面的所有的文档内容。如果想使用查询条件，可以使用 URI + 请求参数或者 Query DSL。
```
GET /indexname/_search
```
返回数据
```
{
    "took":7,
    "timed_out":false,
    "_shards":{
        "total":1,
        "successful":1,
        "skipped":0,
        "failed":0
    },
    "hits":{
        "total":{
            "value":2,
            "relation":"eq"
        },
        "max_score":1,
        "hits":[
            {
                "_index":"customer",
                "_id":"2",
                "_score":1,
                "_source":{
                    "name":"John Doe1"
                }
            },
            {
                "_index":"customer",
                "_id":"1",
                "_score":1,
                "_source":{
                    "name":"John Doe2"
                }
            }
        ]
    }
}
```
* took - Elasticsearch 执行搜索的时间（ 毫秒）
* time_out - 告诉我们搜索是否超时
* _shards - 告诉我们多少个分片被搜索了， 以及统计了成功/失败的搜索分片
* hits - 搜索结果
* hits.total - 搜索结果
* hits.hits - 实际的搜索结果数组（ 默认为前 10 的文档）
* sort - 结果的排序 key（ 键） （ 没有则按 score 排序）
* score 和 max_score –相关性得分和最高得分（ 全文检索用）

#### 批量操作
语法格式
```
POST /indexname/_bulk
{action:{metadata}}\n
{request body  }\n
{action:{metadata}}\n
{request body  }\n
......
```
bulk API 以此按顺序执行所有的 action（动作）。如果一个单个的动作因任何原因而失败，它将继续处理它后面剩余的动作。 当 bulk API 返回， 它将提供每个动作的状态（与发送的顺序相同），所以可以检查是否一个指定的动作是不是失败了。
例如批量插入id为5和6的两条数据：
```
POST /customer/_bulk
{"index":{"_id":"5"}}
{"name":"John Doe5"}
{"index":{"_id":"6"}}
{"name":"John Doe6"}
```
还可以不指定索引，对整个 es 进行整体操作。例如下面一共进行了 delete、create、index和 update 四个操作。
```
POST /_bulk
{"delete":{"_index":"website","_type":"blog","_id":"123"}}
{"create":{"_index":"website","_type":"blog","_id":"123"}}
{"title":"my first blog post"}
{"index":{"_index":"website","_type":"blog"}}
{"title":"my second blog post"}
{"update":{"_index":"website","_type":"blog","_id":"123"}}
{"doc":{"title":"my updated blog post"}}
```
可以导入官方示例数据进行测试 https://www.elastic.co/guide/cn/kibana/current/tutorial-load-dataset.html
