---
title: elasticsearch系列（五）：索引
date: 2021-04-28 20:32:06
categories:
- elasticsearch
tags:
- elasticsearch
---

> elasticsearch系列（五）：索引

<!--more-->

### 一、索引介绍
#### 索引结构
Elasticsearch 索引是是具有类似特性的文档的集合，的索引结构主要包含 `mappings` 与 `settings` 两部分。
* settings：定义 index 分片数量、副本数量、解析器等相关的配置信息。
* mappings：定义 document 及其包含的字段类型和索引方式相关的信息。

以最新的7.x以后的最新版本为例，新建一个索引结构如下：
```
PUT /my-index
{
    "settings":{
        "number_of_shards":3, // 3个主分片，主分片在设定后不能修改
        "number_of_replicas":1 // 除了本身分片之外，还有1个备份分片，备份分片可以在运行过程中根据业务需要动态调整
    },
    "mappings":{
        "properties":{ // 定义了 name、age、address 三个字段
            "name":{
                "type":"text"
            },
            "age":{
                "type":"integer"
            },
            "address":{
                "type":"text"
            }
        }
    }
}
```
#### 索引分片
分片（shard）是 es 中存储数据的最小单元。一个索引被分成多个分片，分散到多台服务器上，每个分片存储一部分数据。分片分为 shard 和 replica 。
* `shard`：只有一个，为了解决数据在单一节点存储的问题而做的水平扩展，将数据分片后存放在不同的节点上可以提高对于数据操作的吞吐量及可扩展性。此参数一旦在索引（Index）创建完成后，则不可以进行修改。
* `replica`：是对 shard 的备份，为了解决数据可用性问题，通常在分片数据丢失后，如果有副本数据在的话可以保证数据的完整性。并且在高吞吐量场景时，增加副本数量可以提供服务的读取吞吐量。此参数可以在索引创建时指定，也可以在创建后进行调整。**replica 值越大搜索效率越高，但写入性能越低**（一条数据写入操作需要做（1+replicas）遍），具体值与集群 data 节点数量相关，不宜超过（data 节点数-1）

**比如有一个索引，shard 设置为5，replica 设置为1，那么总的切片数为：shard（5） + shard（5） * replicas（1） = total（10**）。

每次修改的时候都会先直接修改 primary shard，再同步其他所有 replica shard。查询的时候则在primary和replica中随机选择一个。

### 二、索引原理
#### 倒排索引
Elasticsearch 的倒排索引，其实就是 Lucene 的倒排索引。倒排索引就是通过 Value 查找 Key：一段文本经过分析器分析以后就会输出一串单词，这一个一个的就叫做 Term。文档中所有不重复 Term 组成一个列表，对于其中每个 Term，有一个包含它的文档列表。lucene 倒排索引中包含的概念如下：
* `索引(Index)`：类似于数据库的表。在Lucene中一个索引是放在一个文件夹中的。所以可以理解索引为整个文件夹的内容。
* `段(Segment)`：类似于表的分区。一个索引下可以多个段。
* `文档(Document)`：类似于表的一行数据。Document是索引的基本单位。一个段可以有多个Document。
* `域(Field)`：类似于表的字段。Doument里可以有多个Field。Lucene提供多种不同类型的Field，例如StringField、TextField、LongFiled或NumericDocValuesField等。
* `词(Term)`：Term是索引的最小单位。Term是由Field经过Analyzer（分词）产生。

lucene 倒排索引的内部结构如下：
![image-20230710203832177](image-20230710203832177.png)

* `Term Index`：字典树，用于快速查找Term。使用FST结构，只存储Term前缀，通过字典树找到Term所在的块，也就是Term的在字典表中的大概位置，再在块里二分查找，找到对应的Term。
* `Term Dictionary`：字典表，用于存储Analyzer分割 后所有Term。使用跳表结构。
* `Posting List`：记录表，记录了出现过某个单词的所有文档的文档列表及单词在该文档中出现的位置信息，每条记录称为一个倒排项(Posting)。使用Frame Of Reference（FOR）和Roaring bitmaps技术。它不仅仅为文档 id 信息，可能包含以下信息：
    * 文档 id（DocId, Document Id），包含单词的所有文档唯一 id，用于去正排索引中查询原始数据。
    * 词频（TF，Term Frequency），记录 Term 在每篇文档中出现的次数，用于后续相关性算分。
    * 位置（Position），记录 Term 在每篇文档中的分词位置（多个），用于做词语搜索（Phrase Query）。
    * 偏移（Offset），记录 Term 在每篇文档的开始和结束位置，用于高亮显示等。

#### 构建倒排索引
构建倒排索引的过程：**Elasticsearch 会使用多种分析器将文本拆分成用于搜索的词条（Term），然后将这些词条统一化以提高“可搜索性”，整个分析算法称为分析器（Analyzer）**。
![image-20230710203901923](image-20230710203901923.png)
Analyzer 包含三部分内容：

 1. `字符过滤器（Character Filter）`：在分词前进行预处理，如去除HTML，字符转换如&转为and。
 2. `分词器（Tokenizer）`：按照规则切割文档并提取词元（Token），例如遇到空格和标点将文本拆分成词元。
 3. `Token过滤器（Token Filter`）：将切分的词元进行加工，如将大小写统一，增加词条（如jump、leap等同义词）。

#### index、shard、segment、倒排索引的区别和联系
![image-20230710203920943](image-20230710203920943.png)
* 一个 Elastic Index 都是由多个 Shard （primary & replica）构成的。
* 一个Shard 本质对应一个Lucene Index。
* 一个Lucene Index包含多个segment。
* 每一个segment都是一个倒排索引。

### 三、索引模板
#### 索引模板简介
**索引模板是预先定义好的在创建新索引时自动应用的模板**。创建索引之前可以先配置模板，这样在创建索引（手动创建索引或通过对文档建立索引）时，可以直接使用索引模板创建索引。索引模板一般与索引别名一起使用。另外需要注意，模板只在创建索引时应用，更改模板不会对现有索引产生影响。
在 elasticsearch v7.8 版本之后推出 `组合索引模板（composable index template）`。在这之前的都属于`旧的索引模板（legacy index template）`。在使用优先级上，组合模板优先于旧模板。


组合索引模板的使用参考[Index templates](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-templates.html)，旧的索引模板的使用参考[Create or update index template (legacy)](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-templates-v1.html)

#### 旧索引模板
旧索引模板没有组件的概念，直接在索引模板里面定义 settings 和 mappings。
```
PUT _template/template_1
{
    "index_patterns":[ // 对满足正则匹配的索引应用索引模板
        "te*",
        "bar*"
    ],
    "settings":{ // 定义settings
        "number_of_shards":1
    },
    "mappings":{ // 定义mappings
        "_source":{
            "enabled":false
        },
        "properties":{ // 定义host_name和created_at两个字段
            "host_name":{
                "type":"keyword"
            },
            "created_at":{
                "type":"date",
                "format":"EEE MMM dd HH:mm:ss Z yyyy"
            }
        }
    }
}
```
> 在 7.8 版本后也可以直接用该方式创建索引，但是会有告警 `#! Legacy index templates are deprecated in favor of composable templates.`

#### 组合索引模板
组合索引模板有两种类型：索引模板和组件模板。
* `组件模板`：可重用的构建块，用于配置映射，设置和别名；它们不会直接应用于一组索引。
* `索引模板`：可以包含组件模板的集合，也可以直接指定设置，映射和别名。

可以不使用组件模板，直接在索引模板里面定义 settings 和 mappings，类似于旧索引模板。上面的旧索引模板的例子也可以改成下面的用法。
```
PUT _index_template/template_1
{
    "index_patterns":[
        "te*",
        "bar*"
    ],
    "template":{
        "settings":{
            "number_of_shards":1
        },
        "mappings":{
            "_source":{
                "enabled":false
            },
            "properties":{
                "host_name":{
                    "type":"keyword"
                },
                "created_at":{
                    "type":"date",
                    "format":"EEE MMM dd HH:mm:ss Z yyyy"
                }
            }
        }
    }
}
```

也可以使用组件模板来组合索引模板。例如首先创建两个索引组件模板：
```
PUT _component_template/component_template1
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        }
      }
    }
  }
}

PUT _component_template/runtime_component_template
{
  "template": {
    "mappings": {
      "runtime": { 
        "day_of_week": {
          "type": "keyword",
          "script": {
            "source": "emit(doc['@timestamp'].value.dayOfWeekEnum.getDisplayName(TextStyle.FULL, Locale.ROOT))"
          }
        }
      }
    }
  }
}
```
然后创建使用组件模板的索引模板。
```
PUT _index_template/template_2
{
    "index_patterns":[
        "bar*"
    ],
    "template":{
        "settings":{
            "number_of_shards":1
        },
        "mappings":{
            "_source":{
                "enabled":true
            },
            "properties":{
                "host_name":{
                    "type":"keyword"
                },
                "created_at":{
                    "type":"date",
                    "format":"EEE MMM dd HH:mm:ss Z yyyy"
                }
            }
        },
        "aliases":{
            "mydata":{

            }
        }
    },
    "priority":500,
    "composed_of":[
        "component_template1",
        "runtime_component_template"
    ],
    "version":3,
    "_meta":{
        "description":"my custom"
    }
}
```

#### 索引模板的优先级
* 可组合模板优先于旧模板。如果没有可组合模板匹配给定索引，则旧版模板可能仍匹配并被应用。
* 如果使用显式设置创建索引并且该索引也与索引模板匹配，则创建索引请求中的设置将优先于索引模板及其组件模板中指定的设置。
* 如果新数据流或索引与多个索引模板匹配，则使用优先级最高的索引模板。

#### 内置索引模板
Elasticsearch具有内置索引模板，每个索引模板的优先级为100，适用于以下索引模式：
* `logs-*-*`
* `metrics-*-*`
* `synthetics-*-*`

所以在涉及内建索引模板时，要避免索引模式冲突。更多可以参考[这里](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-templates.html)


#### 版本变化
##### 6.0之前的版本
```
POST _template/shop_template
{
    "template": "shop*",       // 可以通过"shop*"来适配
    "order": 0,                // 模板的权重, 多个模板的时候优先匹配用, 值越大, 权重越高
    "settings": {
        "number_of_shards": 1  // 分片数量, 可以定义其他配置项
    },
    "aliases": {
        "alias_1": {}          // 索引对应的别名
    },
    "mappings": {
        "_default": {          // 默认的配置, ES 6.0开始不再支持
            "_source": { "enabled": false },  // 是否保存字段的原始值
            "_all": { "enabled": false },     // 禁用_all字段
            "dynamic": "strict"               // 只用定义的字段, 关闭默认的自动类型推断
        },
        "type1": {             // 默认的文档类型设置为type1, ES 6.0开始只支持一种type, 所以这里不需要指出
            "_source": {"enabled": false},
            "properties": {        // 字段的映射
                "@timestamp": {    // 具体的字段映射
                    "type": "date",           
                    "format": "yyyy-MM-dd HH:mm:ss"
                },
                "@version": {
                    "doc_values": true,
                    "index": "not_analyzed",  // 不索引
                    "type": "string"          // string类型
                },
                "logLevel": {
                    "type": "long"
                }
            }
        }
    }
}
```

##### 6.0到7.0中间的版本
es的type只保留一种_doc，所有索引模板的type都是_doc，作为过渡。
```
POST _template/shop_template
{
    "index_patterns": ["shop*", "bar*"],       // 可以通过"shop*"和"bar*"来适配, template字段已过期
    "order": 0,                         // 模板的权重, 多个模板的时候优先匹配用, 值越大, 权重越高
    "settings": {
        "number_of_shards": 1       // 分片数量, 可以定义其他配置项
    },
    "aliases": {
        "alias_1": {}                    // 索引对应的别名
    },
    "mappings": {
        "_doc": {                       // ES 6.0开始只支持一种type, 名称为“_doc”
            "_source": {                // 是否保存字段的原始值
                "enabled": false
            },
            "properties": {             // 字段的映射
                "@timestamp": {      // 具体的字段映射
                    "type": "date",           
                    "format": "yyyy-MM-dd HH:mm:ss"
                },
                "@version": {
                    "doc_values": true,
                    "index": "false",   // 设置为false, 不索引
                    "type": "text"      // text类型
                },
                "logLevel": {
                    "type": "long"
                }
            }
        }
    }
}
```

##### 7.0以后的版本
取消了type，mapping字段里面不能再带有type
```
POST _template/my_template
{
    "index_patterns":"my-*",
    "order":0,
    "settings":{
        "index":{
            "number_of_shards":"10",
            "number_of_replicas":"0"
        },
        "analysis":{
            "analyzer":{
                "my_analyzer":{
                    "type":"custom",
                    "char_filter":[
                        "html_strip"
                    ],
                    "tokenizer":"whitespace",
                    "filter":[
                        "stop",
                        "lowercase"
                    ]
                }
            }
        }
    },
    "mappings":{
        "_source":{
            "enabled":false
        },
        "dynamic_templates":[
            {
                "string_as_keyword":{
                    "match_mapping_type":"long",
                    "mapping":{
                        "type":"integer"
                    }
                }
            }
        ],
        "properties":{
            "name":{
                "type":"text",
                "analyzer":"my_analyzer"
            },
            "age":{
                "type":"integer"
            },
            "address":{
                "type":"text",
                "analyzer":"standard"
            }
        }
    },
    "aliases":{
        "alias_name":{
            "filter":{
                "term":{
                    "user":"tim"
                }
            },
            "routing":"tim"
        }
    }
}
```
* order：模板优先级，值越大，优先级越高。匹配到多个模板时，优先级高的模板会覆盖优先级低的模板的相同字段的配置（不会整个模板都覆盖）。
* index_patterns：模板匹配的名称方式，可包含多个正则表达式。新建索引时，新索引名称通过匹配index_patterns来查找合适和索引模板。
* settings：指定index的配置信息，比如分片数、副本数、tranlog同步条件、refresh策略、analyzer等信息。
    * index：索引的分片情况。
    * analyzer：分析器，es里面，`analyzer = char_filter + tokenizer + token filter`。
        * char filter：对输入的文本字符进行第一步处理，如去除html标签（html_strip），将表情字符转换成英文单词（mapping）等。
        * tokenizer： 对文本进行分词操作，如按照空格分词（whitespace）。分好的词成为token。
        * filter （token filter）：对一个token集合的元素做过滤和转换(修改)，删除等操作，例如统一转成小写。
* mappings：指定index的内部构造信息。在6.x版本，mappings下面是type元素，默认是_doc，type下面才是properties等元素。但后面的版本type已经删除，所以mappings下面直接跟properties元素。
    * _source：设置为 false 默认检索只会返回ID，你需要通过 Fields 字段去到索引中去取数据，效率不是很高。但是 enabled 设置为 true 时，索引会比较大，这时可以通过 Compress 进行压缩和 inclueds、excludes 来在字段级别上进行一些限制，自定义哪些字段允许存储。
    * dynamic_templates：自定义匹配规则模板，数组里面对象是个json，key就是名称
    * properties：Field的映射配置。
* aliases：索引别名。
