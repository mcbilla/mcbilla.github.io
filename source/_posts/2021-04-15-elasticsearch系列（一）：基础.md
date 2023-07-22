---
title: elasticsearch系列（一）：基础
date: 2021-04-15 19:02:43
categories:
- 中间件
- Elasticsearch
tags:
- Elasticsearch
---

> elasticsearch系列（一）：基础

<!--more-->

## 一、elastisearch 简介

### elastisearch 是什么

ElasticSearch 是一款非常强大的、基于Lucene的开源搜索及分析引擎，具备**全文检索**、**结构化搜索**和**分析**等功能。ElasticSearch  **隐藏 Lucene 的复杂性，取而代之的提供一套简单一致的 RESTful API**，让全文搜索变得简单。主要功能：

1. 海量数据的分布式存储以及集群管理，达到了服务与数据的高可用以及水平扩展。
2. 近实时搜索，性能卓越。对结构化、全文、地理位置等类型数据的处理。
3. 海量数据的近实时分析（聚合功能）。

应用场景：

1. 网站搜索、垂直搜索、代码搜索。
2. 日志管理与分析、安全指标监控、应用性能监控、Web抓取舆情分析。

### Elastic Stack生态

除了搜索，结合Kibana、Logstash、Beats开源产品，Elastic Stack（简称ELK）还被广泛运用在大数据近实时分析领域，包括：**日志分析**、**指标监控**、**信息安全**等。它可以帮助你**探索海量结构化、非结构化数据，按需创建可视化报表，对监控数据设置报警阈值，通过使用机器学习，自动识别异常状况**。

![image-20230707195555838](image-20230707195555838.png)

* Beats：轻量级收集平台，用于收集数据，可以直接对接 elasticsearch，但是只能做一些简单的处理，不一定能完全适配 elasticsearch 的数据类型，一般是对接到 Logstash 进行数据处理。
* Logstash：具备数据收集、分析、转换、提取等丰富的功能，如果只是单纯的收集日志没必要直接使用 Logstash，比较耗费性能Logstash。一般使用 Beats 进行日志收集，Logstash 只专注于数据处理的工作。
* Kibana：数据可视化，能够以图表的形式呈现数据，并且具有可扩展的用户界面，可以全方位的配置和管理ElasticSearch。

## 二、elasticsearch 安装启动

下载地址 `https://www.elastic.co/cn/downloads/elasticsearch`，选择对应系统的版本，下载解压即可。一般需要搭配 kibana 使用，所以需要另外安装 kibana。

然后切换到安装目录，执行下面命令启动。

```
$ bin/elasticsearch
```

也可以选择 docker 安装和启动，参考 [docker安装](http://localhost:4000/2021/04/10/docker%E7%B3%BB%E5%88%97%EF%BC%9A%E5%AE%89%E8%A3%85%E5%B8%B8%E7%94%A8%E8%BD%AF%E4%BB%B6/)

访问地址：`http://localhost:9200`

## 三、elasticsearch 架构



![F415A5F0-B924-443F-88B2-30D89112254E](F415A5F0-B924-443F-88B2-30D89112254E.png)

- 数据存储层：Gateway 是 Elasticsearch 索引的持久化存储方式，ES 默认是先把索引存放到内存中，当内存满了之后，再持久化到硬盘里。当这个 Elasticsearch 集群关闭或者再次重新启动时就会从 Gateway 中读取索引数据。 Gateway支持多种类型：本地文件系统（默认）、分布式文件系统 Hadoop 以及 AMZ 的 S3 云存储服务。ES目前主要使用 Local FileSystem，主要利用本机节点、本地文件系统存储索引和文档，其他三个都废弃掉了。
- 核心架构层：Elasticsearch 是基于 Lucene 架构实现的，所以其核心层为 Lucene。Lucene 是一个开源的全文检索引擎工具包和编程库，提供了原始接口调用，直接使用 Lucene，你需要覆盖大量的集成框架工作，复杂性较高，而且只能支持单体架构的搜索。Elasticsearch 底层是基于这些包，对其进行了扩展和集成，隐藏了 Lucene 的复杂性，取而代之的提供一套简单一致的 RESTful API，实现了 Lucene 的分布式架构，提供了比 Lucene 更为丰富的查询语言，可以非常方便的通过 Elasticsearch 的 HTTP 接口与底层 Lucene 交互。
- 数据处理层：包含四个模块：
  - Index Module：索引模块，就是对数据建立索引也就是通常所说的建立一些倒排索引等。
  - Search Module：搜索模块，就是对数据进行查询搜索。
  - Mapping Module：数据映射与解析模块，就是你的数据的每个字段可以根据你建立的表结构通过mapping进行映射解析，如果你没有建立表结构，es就会根据你的数据类型推测你的数据结构之后自己生成一个mapping，然后都是根据这个mapping进行解析你的数据。
  - River Module 在es2.0之后应该是被取消了，它的意思表示是第三方插件，例如可以通过一些自定义的脚本将传统的数据库（mysql）等数据源通过格式化转换后直接同步到es集群里，这个River大部分是自己写的，写出来的东西质量参差不齐，将这些东西集成到es中会引发很多内部bug，严重影响了es的正常应用，所以在es2.0之后考虑将其去掉。
- 发现/脚本层：包含三个模块：
  - Discovery：Discovery 是集群中节点之间发现的组件。es是一个集群包含很多节点，很多节点需要互相发现对方，然后组成一个集群包括选主的，这些节点互相发现依赖 discovery 模块，默认使用的是 Zen。Zen 包含多播和单播两种方式。多播指一个节点可以向多台机器发送请求。生产环境中ES不建议使用这种方式，对于一个大规模的集群，组播会产生大量不必要的通信。单播指当一个节点加入一个现有集群，或者组建一个新的集群时，请求发送到一台机器。当一个节点联系到单播列表中的成员时，它就会得到整个集群所有节点的状态，然后它会联系Master节点并加入集群。单播也是ES 默认配置方式。
  - Scripting：Scripting 是支持一些脚本语言的组件，包括mvel、js、python等等。但由于存在性能和使用场景少的问题，目前实际生产中使用比较少。
  - 3rd Plugins：支持第三方插件的整合，留了入口，例如 ik 分词器。
- 协议层：包含两个模块：
  - Transport：Transport 是 es 内部节点或集群与客户端的交互方式组件。默认是使用tcp协议进行交互，同时它支持http协议（json格式）、thrift、servlet、memcached、zeroMQ等的传输协议（通过插件方式集成）。
  - JMX：使用 Java 提供的 JMX 框架，日常ES中使用比较少，作为了解部分即可。
- 应用层：包含两个模块
  - RestFul API：官方推荐使用的提供给客户端的访问方式，直接发送http请求，可以方便做负载均衡和权限管理。
  - Java API：如果是使用 Java 客户端，可以选择 Java API 的访问方式，利用 Netty，通过 JMX 协议，但是不好做负载均衡和权限管理。

## 四、elasticsearch 基础概念

|          | 描述                                                         | 注意事项                                                     |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| cluster  | cluster 是由具有相同 cluster.name （默认值为elasticsearch）的一个或多个 Elasticsearch 节点组成的。一个节点通过指定 cluster.name 来加入某个集群。 | 使用ElasticSearch集群时，需要选择一个合适的名字来代替cluster.name(默认值)。默认值是为了防止一个新启动的节点加入相同网络中的另一个相同的集群中。 |
| node     | node 是一个运行的 Elasticsearch 实例，是组成 ElasticSearch 集群的基本服务单元，一个 cluster 包含一到多个 node。node 负责完成集群管理、数据存储和索引搜索等功能。 | 注意一个 node 不一定就是一台服务器，因为一个服务器上可以部署多个 node。 |
| index    | index 是是 es 组织文档的方式，是拥有相结构 document 的集合，类似于关系型数据库的一张数据表。可以把已经创建好的某个索引的参数设置(settings)和索引映射(mapping)保存下来作为 index template，在创建新 index 时，指定要使用的 index template，就可以直接重用已经定义好的模板中的settings和mappings（这点是关键）。修改模板不会影响已创建的 index。 | 6.0之前 index 是被设计成数据库的概念，后面逐渐向设计成表的概念发展。index 名称必须全部小写，所以不建议写成驼峰式。 |
| alias    | alias 是 index 的别名。index 创建后就不可以更新名称，如果需要更新名称，就需要 alias 的功能。alias 类似于一个快捷方式或软连接，可以指向一个或多个 index，也可以给任何一个需要 index 的API来使用。alias 的作用：在运行的集群中可以无缝的从一个 index 切换到另一个 index。给多个 index 分组 (例如， last_three_months)。给 index 的一个子集创建视图。 | 假如已经建了很多周的索引，有个需求是查询最新一周的索引，我们每周查询的索引名称可能都是不同的。我们可以建一个索引别名为this_week，每周只需要修改这个索引别名指向本周的新建索引，就是保证查询索引别名不变的情况下，查询到每周的新数据。 |
| type     | type 是 index 内部的逻辑分区，使用 type 允许我们在一个 index 里存储多种类型的数据，这样就可以减少 index 的数量了。但是使用 type 的限制较多：不同 type 里的字段需要保持一致。例如，一个 index 下的不同 type 里有两个名字相同的字段，他们的类型（string, date 等等）和配置也必须相同。只在某个 type 里存在的字段，在其他没有该字段的 type 中也会消耗资源。得分是由 index 内的统计数据来决定的。也就是说，一个 type 中的文档会影响另一个 type 中的文档的得分。这意味着，只有同一个 index 的中的 type 都有类似的映射 (mapping) 时，才应该使用 type。否则，使用多个 type 可能比使用多个 index 消耗的资源更多。所以后面 type 逐渐被废弃。 | 6.x版本，一个 Index 可以拥有多个 type。7.x版本，一个索引只能拥有一个type，默认的type 就是_doc，但已经建议删除。8.x版本，彻底删除。 |
| mapping  | mapping 定义了 index 中 filed 的存储类型、分词方式、是否存储等信息，类似于关系型数据库中的表结构信息。 | ElasticSearch 会根据数据格式自动识别 index 中字段的类型，如无特殊需求，则不需要手动创建 Mapping。当需要对某些字段添加特殊属性时，就需要手动设置Mapping（如是否分词、是否存储、定义使用其他分词器）。一个 index 的 Mapping 一旦创建，若已经存储了数据，就不可以修改 |
| document | document 是指 es 的一条数据。用 JSON 作为 document 序列化的格式。每个 document 都有一个 Uid，一个 index 由多个 document 组成。 | 6.x版本以前，一个 type 由多个 document 组成，逐渐废弃了 type，index 代替了 type 的概念。 |
| field    | filed 是指 document 中的某一个属性，一个 document 由多个 field 组成，类似于关系型数据库中的列。 |                                                              |
| shard    | shard 是将一个 index 拆分成几个部分，每部分数据称为一个 shard，shard 分布在不同的 node 上，类似于 mysq l中的分表。shard 可以避免单个 node 的物理限制，增加吞吐量。 | 创建 shard 时需要指定 shard 的数量，并且 shard 的数量一旦确定就不能更改。向设有多个 shard 的 index 中写入数据时，是通过路由来确定具体写入哪个 shard 中。 |
| replica  | replica 是对主 shard 的备份。每个主 shard 都可以有零或者多个 replica，主 shard 和备份 replica 分散在不同的节点上，都可以对外提供数据查询服务。replica 可以提高系统容错性。当构建 index 进行写入操作时，首先在主 shard 上完成数据的索引，然后数据会从主 shard 分发到 replica 上进行索引。当主 shard 不可用时，ElasticSearch会在 replica 中选举一个作为主 shard，从而避免数据丢失。 | replica 即可以提升 ElasticSearch 系统的高可用性能，又可以提升搜索时的并发性能。但如果 replica 数量设置太多，会在写操作时增加数据同步的负担。 |

### 关系图

![6A837C36-3517-428B-B124-F36DC6DA8FA0](6A837C36-3517-428B-B124-F36DC6DA8FA0.png)

- 一个 ES Index 在集群模式下，有多个Node（节点）组成，每个节点就是ES的 instance（实例）
- 每个节点上会有多个 shard（分片），P1 P2 是主分片，R1 R2 是副本分片。
- 每个分片上对应着就是一个 Lucene Index （底层索引文件）
- 一个 Lucene Index 包含的内容：
  - 由多个 Segment（段文件，就是倒排索引）组成，每个段文件存储着的就是 Doc 文档。
  - commit point 记录了所有的 segments 的信息

### elasticsearch 和 mysql 的概念对比

| elasticsearch概念      | mysql概念                       |
| ---------------------- | ------------------------------- |
| filed                  | 属性(列)                        |
| document               | 记录(行)                        |
| type（已废弃）         | 表                              |
| index                  | 数据库(表)                      |
| mapping                | schema                          |
| shard                  | 分表                            |
| SQL                    | DSL（Domain Specific Language） |
| select * from xxx      | GET http://...                  |
| update xx set xx = xxx | PUT http://...                  |
| delete                 | DELETE http://...               |

## 五、elasticsearch 版本变化

### 去除 type 概念

ElasticSearch 从 6.x 版本开始着手去除 type 概念，目的是为了提高处理数据的效率。

* 关系型数据库中两个数据表示是独立的，即使他们里面有相同名称的列也不影响使用，但ES中不是这样的。elasticsearch是基于Lucene开发的搜索引擎，而ES中不同type下名称相同的filed最终在Lucene中的处理方式是一样的。
* 两个不同type下的两个user_name，在ES同一个索引下其实被认为是同一个filed，你必须在两个不同的type中定义相同的filed映射。否则，不同type中的相同字段名称就会在处理中出现冲突的情况，导致Lucene处理效率下降。

| version | pattern                                                      |
| ------- | ------------------------------------------------------------ |
| 5.6.x   | 可手动启用“index.mapping.single_type: true”，使用 join 字段进行替换 |
| 6.x     | 允许索引使用单一类型，有且仅有一个名称，首选_doc 类型；5.x 版本的索引可延续使用，新版本的索引不再支持父/子方式和 join 类型；6.8中需要在索引创建、模板时需要显示指定 include _ type _ name，不显示申明会默认为 name _ doc |
| 7.x     | _doc 是路径中的永久部分，并且表示的是 endpoint 类型，而不是 doc 类型；include _ type _ name 参数默认为 false；移除 *default* mapping 类型 |
| 8.x     | 不再支持URL中的 type 参数。移除 include_type_name 参数       |
