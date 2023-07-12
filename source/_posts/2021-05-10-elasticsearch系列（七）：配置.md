---
title: elasticsearch系列（七）：配置
date: 2021-05-10 00:47:38
categories:
- elasticsearch
tags:
- elasticsearch
---

> elasticsearch系列（七）：配置

<!--more-->

## 一、简介
setting 定义了分片数量及副本数量等相关的配置信息。可以通过下面方式定义 settings
```
# 索引不存在，第一次创建索引
PUT /indexname
{
    "settings":{
        "number_of_replicas":1
    }
}

# 索引已存在，修改索引
PUT  /indexname/_settings
{
    "number_of_replicas":"1"
}
```
索引的配置项按是否可以更改分为 `静态配置`  与 `动态配置`。

### 索引静态配置
静态配置即索引创建后不能修改。常用项如下：
* **number_of_shards** ：主分片数，默认为5.只能在创建索引时设置，不能修改
* **analysis**：设置自定义分词器
* shard.check_on_startup ：是否在索引打开前检查分片是否损坏，当检查到分片损坏将禁止分片被打开
* 可选值：false：不检测；checksum：只检查物理结构；true：检查物理和逻辑损坏，相对比较耗CPU；fix：类同与false，7.0版本后将废弃。默认值：false。
* codec：数据存储的压缩算法，默认算法为LZ4，也可以设置成best_compression，best_compression压缩比较好，但存储性能比LZ4差
* routing_partition_size ：路由分区数，默认为 1，只能在索引创建时设置。此值必须小于index.number_of_shards，如果设置了该参数，其路由算法为： (hash(_routing) + hash(_id) % index.routing_parttion_size ) % number_of_shards。如果该值不设置，则路由算法为 hash(_routing) % number_of_shardings，_routing默认值为_id。

### 索引动态配置
动态配置可以在运行时进行配置修改。常用项如下：
* **number_of_replicas** ：每个主分片的副本数，默认为 1，该值必须大于等于0
* auto_expand_replicas ：基于可用节点的数量自动分配副本数量，默认为 false（即禁用此功能）
* refresh_interval ：执行刷新操作的频率，这使得索引的最近更改可以被搜索。默认为 1s。可以设置为 -1 以禁用刷新。
* max_result_window ：用于索引搜索的 from+size 的最大值。默认为 10000
* max_rescore_window ： 在搜索此索引中 rescore 的 window_size 的最大值
* blocks.read_only ：设置为 true 使索引和索引元数据为只读，false 为允许写入和元数据更改。
* blocks.read_only_allow_delete：与blocks.read_only基本类似，唯一的区别是允许删除动作。
* blocks.read ：设置为 true 可禁用对索引的读取操作
* blocks.write ：设置为 true 可禁用对索引的写入操作。
* blocks.metadata ：设置为 true 可禁用索引元数据的读取和写入。
* max_refresh_listeners ：索引的每个分片上可用的最大刷新侦听器数
* max_docvalue_fields_search：一次查询最多包含开启doc_values字段的个数，默认为100。


## 二、分词器
analysis 是 settings 里面比较常用的属性。**analysis 是全文本转换成一系列单词（term/token）的过程，analysis 通过 analyzer（分词器）来完成**。analyzer 由以下三部分组成：
* Character Filters：针对原始文本处理，比如去除 html 标签
* Tokenizer：按照规则切分为单词，比如按照空格切分
* Token Filters：将切分的单词进行加工，比如大写转小写，删除 stopwords，增加同义语

下面是 analyzer 处理文本的过程。
![image-20230713005128186](image-20230713005128186.png)

### 配置分词器
分词器属于静态配置，只能在创建索引的时候定义，analyzer 我们可以使用 ES 内置分词器或者第三方分词器，也可以使用自定义的 char_filter、tokenizer、filter 来自由组合。配置模板如下：
```
PUT /indexname
{
    "settings": {
        "analysis": {
            "char_filter": { ... custom character filters ... },
            "tokenizer":   { ...    custom tokenizers     ... },
            "filter":      { ...   custom token filters   ... },
            "analyzer":    { ...    custom analyzers      ... }
        }
    }
}
```
#### 使用内置/第三方 analyzer
下面定义了分词器 `my_analyzer`，类型是 ES 内置标准分词器。
```
PUT /indexname
{
    "settings":{
        "analysis":{
            "analyzer":{
                "my_analyzer":{
                    "type":"standard" // 指定类型为 standard
                }
            }
        }
    }
}
```

#### 自定义 custom analyzer
我们也可以自定义分词器组件 char_filter、tokenizer、filter 的内容，这种方式更加灵活。
```
PUT /indexname
{
    "settings":{
        "analysis":{
            "analyzer":{
                "my_custom_analyzer":{
                    "type":"custom", // 指定类型为 customer
                    "tokenizer":"standard",
                    "char_filter":[
                        "html_strip"
                    ],
                    "filter":[
                        "lowercase",
                        "asciifolding"
                    ]
                }
            }
        }
    }
}
```

### 使用分词器
使用分词器有三种方式：
* 定义索引 settings 的时候指定默认 analyzer。
* 定义字段属性的时候指定 analyzer。
* 可以直接调用分词器 API 对现成的文本进行分词，一般用于调试。

#### 索引指定默认 analyzer
```
PUT /indexname
{
    "settings":{
        "analysis":{
            "analyzer":{
                "default":{
                    "type":"simple"
                }
            }
        }
    }
}
```

#### 字段指定 analyzer
```
PUT /indexname
{
    "mappings":{
        "properties":{
            "title":{
                "type":"text",
                "analyzer":"whitespace"
            }
        }
    }
}
```

#### 分词器 API
* 直接指定 Analyzer 进行测试
```
GET _analyze
{
    "analyzer": "standard",
    "text" : "Mastering Elasticsearch , elasticsearch in Action"
}
```
* 指定索引的字段进行测试
```
POST books/_analyze
{
    "field": "title",
    "text": "Mastering Elasticesearch"
}
```
* 自定义分词组件进行测试
```
POST _analyze
{
    "tokenizer": "standard", 
    "filter": ["lowercase"],
    "text": "Mastering Elasticesearch"
}
```

#### 分词结果
```
{
    "tokens":[
        {
            "token":"mastering",
            "start_offset":0,
            "end_offset":9,
            "type":"<ALPHANUM>",
            "position":0
        },
        {
            "token":"elasticesearch",
            "start_offset":10,
            "end_offset":24,
            "type":"<ALPHANUM>",
            "position":1
        }
    ]
}
```
* token：词项内容
* start_offset、end_offset：起始和结束的字符偏移量，用于高亮显示搜索内容
* position：位置，用于 phrase 短语和 word proximity 词近邻查询

### ES 内置分词器
ES 有内置的分词器，用户也可以自定义分词器。
#### Standard Analyzer
ES 默认的分词器，它会对输入的文本按词的方式进行切分，切分好以后会进行转小写处理，默认的 stopwords 是关闭的。
![image-20230713005153228](image-20230713005153228.png)

#### Simple Analyzer
只包括了 Lower Case 的 Tokenizer，它会按照非字母切分，非字母的会被去除，最后对切分好的做转小写处理，然后接着用刚才的输入文本，分词器换成 simple 来进行分词。
![image-20230713005219143](image-20230713005219143.png)

#### Stop Analyzer
由 Lowe Case 的 Tokenizer 和 Stop 的 Token Filters 组成的，相较于刚才提到的 Simple Analyzer，多了 stop 过滤，stop 就是会把 the，a，is 等修饰词去除。
![image-20230713005241065](image-20230713005241065.png)

#### Whitespace Analyzer
按照空格切分，不转小写
![image-20230713005320764](image-20230713005320764.png)

#### Keyword Analyzer
不分词，直接将输入当做输出
![image-20230713005335642](image-20230713005335642.png)

#### Pattern Analyzer
通过**正则表达式**的方式进行分词，默认是用 `\W+` 进行分割的，也就是非字母的符合进行切分的，运行结果和 Stamdard Analyzer 一致。
![image-20230713005348654](image-20230713005348654.png)

#### Language
为不同国家语言的输入提供了 Language Analyzer 分词器，在里面可以指定不同的语言,提供了 30 多种常见语言的分词器

### 第三方分词器
#### IK 分词器
在使用 Elasticsearch 进行搜索中文时，Elasticsearch 内置的分词器会将所有的汉字切分为单个字，对用国内习惯的一些形容词、常见名字等则无法优雅的处理，此时就需要用到一些开源的分词器。IK 分词器是业务中普遍采用的中文分词器。

##### 安装
1. 下载插件，注意要下载和使用的Elasticsearch 匹配的版本，否则会启动报错。
```
$ wget https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v8.7.0/elasticsearch-analysis-ik-8.7.0.zip
```
2. 在 Elasticsearch 的安装目录的 Plugins 目录下新建 ik 文件夹，然后将下载的安装包解压到此目录下。
```
$ mkdir ik

$ mv elasticsearch-analysis-ik-8.7.0.zip ik/

$ unzip elasticsearch-analysis-ik-8.7.0.zip
```
3. 重启 Elasticsearch。

##### 使用 IK 分词器
IK 分词器包含 ik_smart 以及 ik_max_word 两种分词器
* `ik_max_word`  分词颗粒度小，满足业务场景更丰富
* `ik_smart`  分词器颗粒度较粗，满足分词场景要求不高的业务

对于同一段话` 武汉市长江大桥`，使用 ik_max_word 分词器
```
POST _analyze
{
  "analyzer": "ik_max_word",
  "text": "武汉市长江大桥"
}
```
返回数据
```
{
    "tokens":[
        {
            "token":"武汉市",
            "start_offset":0,
            "end_offset":3,
            "type":"CN_WORD",
            "position":0
        },
        {
            "token":"武汉",
            "start_offset":0,
            "end_offset":2,
            "type":"CN_WORD",
            "position":1
        },
        {
            "token":"市长",
            "start_offset":2,
            "end_offset":4,
            "type":"CN_WORD",
            "position":2
        },
        {
            "token":"长江大桥",
            "start_offset":3,
            "end_offset":7,
            "type":"CN_WORD",
            "position":3
        },
        {
            "token":"长江",
            "start_offset":3,
            "end_offset":5,
            "type":"CN_WORD",
            "position":4
        },
        {
            "token":"大桥",
            "start_offset":5,
            "end_offset":7,
            "type":"CN_WORD",
            "position":5
        }
    ]
}
```
使用 ik_smart 分词器
```
POST _analyze
{
  "analyzer": "ik_smart",
  "text": "武汉市长江大桥"
}
```
返回数据
```
{
    "tokens":[
        {
            "token":"武汉市",
            "start_offset":0,
            "end_offset":3,
            "type":"CN_WORD",
            "position":0
        },
        {
            "token":"长江大桥",
            "start_offset":3,
            "end_offset":7,
            "type":"CN_WORD",
            "position":1
        }
    ]
}
```

也可以在设置索引的时候使用 IK 分词器
```
PUT /indexname
{
    "mappings":{
        "properties":{
            "name":{
                "type":"text",
                "analyzer":"ik_max_word",    // 指定索引时候用的解析器
                "search_analyzer":"ik_smart" // 指定搜索时候用的解析器
            }
        }
    }
}
```

## 参考
[Text analysis](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html)
[ElasticSearch 分词器，了解一下](https://zhuanlan.zhihu.com/p/111775508)
