---
title: elasticsearch系列（六）：映射
date: 2021-05-01 23:01:42
categories:
- elasticsearch
tags:
- elasticsearch
---

>  elasticsearch系列（六）：映射

<!--more-->

## 一、映射介绍
映射（mapping）类似数据库中的表结构定义 （schema），描述了文档包含哪些字段 ，每个字段的数据类型是什么，以及是如何存储和索引的。映射可以包含以下内容：
* 定义Index下的字段名(Field Name)
* 定义字段的类型，比如数值型、字符串型、布尔型等
* 定义倒排索引相关的配置，比如是否索引、记录position等

我们可以通过以下命令来查看索引的映射
```
GET account/_mapping
```
下面显示 account 的映射结构，包含 account_number 和 address 两个字段定义。
```
{
    "account":{
        "mappings":{
            "properties":{
                "account_number":{
                    "type":"long"
                },
                "address":{
                    "type":"text",
                    "fields":{
                        "keyword":{
                            "type":"keyword",
                            "ignore_above":256
                        }
                    }
                }
            }
        }
    }
}
```
创建映射有两种方式：`动态映射` 和 `显式映射`。

### 动态映射 (Dynamic mapping)
如果我们没有预先定义文档的映射，在插入数据的时候，ES 默认会自动推断我们插入的数据的类型并创建索引。
* 动态字段映射：根据数据类型规则应用于动态添加的字段。支持的数据类型包括：boolean、double、integer、object、array、date。
    * dynamic：开启动态映射的模式
    * date_detection：开启日期检测
    * dynamic_date_formats：检测的日期格式
    * numeric_detection: true：开启数值检测
* 动态模板：又称自定义映射，根据匹配条件应用于动态添加的字段。
    * match_mapping_type: Elasticsearch检测到的数据类型进行操作
    * match and unmatch: 使用模式来匹配字段名称
    * path_match and path_unmatch: 使用字段的路径来匹配

例如执行下面命令会自动创建索引 `user`
```
PUT /user/_doc/1
{
  "name": "John",
  "phone": "13423455432"
}
```
查看 `user` 的索引结构
```
GET /user/_mapping
{
    "user":{
        "mappings":{
            "properties":{
                "name":{
                    "type":"text",
                    "fields":{
                        "keyword":{
                            "type":"keyword",
                            "ignore_above":256
                        }
                    }
                },
                "phone":{
                    "type":"text",
                    "fields":{
                        "keyword":{
                            "type":"keyword",
                            "ignore_above":256
                        }
                    }
                }
            }
        }
    }
}
```
可以看到 phone 字段被自动映射为 text 类型，从业务的角度我们希望是一个精确值，使用 keyword 类型，**这就是自动映射的缺点： ES 自动映射的数据类型，不一定是我们想要的类型**。如果你想禁止自动创建索引，你可以通过在 `config/elasticsearch.yml`  的每个节点下添加下面的配置：
```
action.auto_create_index: false
```
### 显式映射（Explicit mapping）
因为被动映射的字段可能不太符合我们的需求，所以我们需要显式映射自定义 mapping 的结构。可以使用下面的语法
```
# 索引不存在，第一次创建索引
PUT /indexname
{
    "mappings":{ // 表示定义映射规则
        "properties":{ // 定义字段类型
            "字段名1":{
                "type":"字段类型"
            },
            "字段名2":{
                "type":"字段类型"
            }
            ......
        }
    }
}

# 索引已存在，修改索引
PUT /user9/_mapping
{
    "properties":{
        "字段名1":{
            "type":"字段类型"
        },
        "字段名2":{
            "type":"字段类型"
        }
    }
}
```
例如上面那个例子，我们可以手动创建 user 索引。
```
PUT /user
{
    "mappings":{
        "properties":{
            "name":{
                "type":"text"
            },
            "phone":{
                "type":"keyword"
            }
        }
    }
}
```
这时候查看 user 的索引结构，发现 phone 是 keyword 类型。

## 二、映射定义

### 字段数据类型（field datatypes）
Elasticsearch 支持以下数据类型：
#### 基本类型
* 字符串类型，包含 text 与 keyword 两种类型。
    * text，会将原始文本进行分词处理，在索引文件中，存储的不是原字符串，而是使用分词器对内容进行分词处理后得到一系列的词根，然后存储在 index 的倒排索引中。
    * keyword，将原始输入内容当成一个词根存储在倒排索引中，与 text 字段的区别是该字段不会使用分词器进行分词。
* 数字类型：long、integer、short、byte、double、float、half_float、scaled_float。
* boolean 类型：true/false。
* 日期类型：date，json对象没有日期类型，故java中的日期数据会被格式化，具体如下：
    * 字符串类型，例如"2015-01-01"
    * long类型，表示从1970-01-01以来的毫秒数
    * int类型，表示从1970-01-01以来的秒数
```
PUT /range_index
{
    "mappings":{
        "properties":{
            "create_date":{
                "type":"date",
                "format":"yyyy-MM-dd HH:mm:ss || yyyy-MM-dd || epoch_millis"
            }
        }
    }
}
```
* binary 类型：存储二进制数据，存储之前，需要先用Base64进行编码。该字段类型默认不存储在索引中，也不能用来当搜索条件。
* range 类型：integer_range、float_range、long_range、double_range、date_range（64-bit 无符号整数，时间戳（单位：毫秒））、ip_range（IPV4 或 IPV6 格式的字符串）

#### 复合类型
##### array 类型
数组类型，Elasticsearch不提供专门的数组类型。任何字段，都可以包含多个相同类型的数值。
```
# 创建索引，字典status_code定义为字符串类型keyword
PUT /indexname
{
    "mappings":{
        "properties":{
            "status_code":{
                "type":"keyword"
            }
        }
    }
}

# 增加数据，可以向status_code字段插入一个数组
PUT /indexname/1
{
    "status_code":[
        "200",
        "301",
        "404"
    ]
}
```
##### object 类型
对象类型，json 对象字符串。在插入数据的时候，es 如果检测到数据值是 json 对象，会自动转成 object 类型。缺省设置 type 为 object，因此不用显式定义 type。
```
PUT /user/_doc/1
{
    "user":{
        "first":"John",
        "last":"Smith"
    }
}
```
也可以显示创建 object 类型字段的索引。
```
PUT /user
{
    "mappings":{
        "properties":{
            "user":{
                "type":"object", // 显示定义 object 类型。也可以不写 type，默认为 object 类型
                "properties":{
                    "first":{
                        "type":"text"
                    },
                    "last":{
                        "type":"text"
                    }
                }
            }
        }
    }
}
```
es默认会将原 json 文档扁平化处理。上面 json 对象变成以下内容
```
{
    "user.first":"John",
    "user.last":"Smith"
}
```
可以通过 `user.first`  的字段名进行查询
```
GET /user/_search
{
    "query":{
        "match":{
            "user.first":"John"
        }
    }
}
```

##### nested 类型
嵌套类型可以看成是一个特殊的 object 对象类型，nested 实际上就是 object 的数组，可以让 object 数组独立检索。例如存在下面的 json 对象
```
{
    "user":[
        {
            "first":"John",
            "last":"Smith"
        },
        {
            "first":"Alice",
            "last":"White"
        }
    ]
}
```
es 如果直接进行扁平化处理，就会变成
```
{
    "user.first":[
        "John",
        "Alice"
    ],
    "user.last":[
        "Smith",
        "White"
    ]
}
```
es 没办法直接对数组建立索引，这时候检索 `John Smith` 就查不到数据。nested 就是将数组里面的每个 doc 单独变成子文档进行存储，因此在查询时就可以知道具体的结构信息了。**要使用 nested 类型必须显示定义映射，否则 es 会自动转成 object 类型**。
```
PUT /user
{
    "mappings":{
        "properties":{
            "user":{
                "type":"nested", // 必须显示定义为 nested 类型
                "properties":{
                    "first":{
                        "type":"text"
                    },
                    "last":{
                        "type":"text"
                    }
                }
            }
        }
    }
}
```


##### geo 类型
地图数据类型。



### 映射参数
除了 type 参数，和 type 同级的还可以定义以下映射参数。

| 参数            | 说明                                                         | 示例                                                 |
| --------------- | ------------------------------------------------------------ | ---------------------------------------------------- |
| index           | 是否被索引，设置成false，字段将不会被索引                    | false                                                |
| analyzer        | 指定分词器，分词器在 settings 中定义                         | ik                                                   |
| boost           | 增强当前字段的匹配权重                                       | 1.23                                                 |
| dynamic         | 是否开启自动映射，目前支持三个参数：true/false/strict。默认为 true，表示新检测到未定义的的字段时候，自动添加到映射；false 表示不添加到映射；strict 表示当出现未定义的字段，抛出异常并拒绝添加文档 | strict                                               |
| format          | 自定义日期格式                                               | yyyy-MM-dd                                           |
| doc_values      | 对not_analyzed字段，默认都是开启，analyzed字段不能使用，对排序和聚合能提升较大性能，节约内存,如果您确定不需要对字段进行排序或聚合，或者从script访问字段值，则可以禁用doc值以节省磁盘空间 | false                                                |
| fields          | 可以对一个字段提供多种索引模式，同一个字段的值，一个分词，一个不分词 | {"keyword": {"type": "keyword","ignore_above": 256}} |
| ignore_above    | 超过长度的字符的文本，将会被忽略，不被索引                   | 100                                                  |
| fielddata       | lasticsearch 加载内存 fielddata 的默认行为是延迟加载。 当 Elasticsearch 第一次查询某个字段时，它将会完整加载这个字段所有 Segment 中的倒排索引到内存中，以便于以后的查询能够获取更好的性能。 | {"loading" : "eager" }                               |
| include_in_all  | 设置是否此字段包含在 _all 字段中，默认是 true，除非 index 设置成 no 选项 | true                                                 |
| index_options   | 4个可选参数docs（索引文档号） ,freqs（文档号+词频），positions（文档号+词频+位置，通常用来距离查询），offsets（文档号+词频+位置+偏移量，通常被使用在高亮字段）分词字段默认是position，其他的默认是docs | docs                                                 |
| norms           | 分词字段默认配置，不分词字段：默认{"enable":false}，存储长度因子和索引时 boost，建议对需要参与评分字段使用 ，会额外增加内存消耗量 | {"enable":true,"loading":"lazy"}                     |
| null_value      | 设置一些缺失字段的初始化值，只有 string 可以使用，分词字段的 null 值也会被分词 | NULL                                                 |
| store           | 是否单独设置此字段的是否存储而从 _source 字段中分离，默认是false，只能搜索，不能获取值 | false                                                |
| search_analyzer | 设置搜索时的分词器，默认跟 ananlyzer 是一致的，比如 index 时用 standard+ngram，搜索时用 standard 用来完成自动提示功能 | ik                                                   |
| similarity      | 指定一个字段评分策略，仅仅对字符串型和分词类型有效 ，默认是TF/IDF算法 | BM25                                                 |

### 映射元数据
每个文档都会有一些元数据属性，例如
```
GET /user/_doc/1
{
    "_index":"user4",
    "_id":"1",
    "_version":1,
    "_seq_no":0,
    "_primary_term":1,
    "found":true,
    "_source":{
        "name":"John",
        "phone":"13423455432"
    }
}
```
`_index` 、`_id`  这些就被称为文档的元数据，用来描述文档本身的属性。常见的文档元数据有：
* _index：文档所在的索引，类似于关系型数据库的database。
* _id：文档的_id值。
* _source：文档的原始 json 数据。
* _size：文档_souce字段的字节长度，需要插件：mapper-size plugin。
* _ignored：设置为ignore_malformed=true的所有字段。
* _routing：路由分片字段。
* _meta：用于用户自定义元数据。

## 三、其他
### 映射修改
映射中已经定义的字段不能被更新和删除，因为lucene实现的倒排索引生成后不允许修改，除非 reindex 更新 mapping。除了三种情况：
* 可以添加新的 filed。
* 已经存在的 fields 里面可以添加 fields。
* ignore_above 参数可以更新的。

假如先创建索引 user，包含 `user` 和 `age` 两个字段。
```
PUT /user1/_doc/1
{
  "name": "John",
  "age": 18
}
```
然后想增加 `address`  字段，无论是使用下面哪一种方式，es 都可以识别出 `addrees`  字段，然后增加到映射。
```
# 在原来的字段基础上增加 address 字段
PUT /user10/_doc/1
{
  "name": "John",
  "age": 18,
  "adrress": "China"
}

# 单独增加 address 字段
{
  "adrress": "China"
}
```
但是如果想把 age 字段改成 text 类型，就会报错 `mapper_parsing_exception`
```
PUT /user10/_doc/1
{
  "age": "aaa"
}
```

### 映射的版本迭代
| version | pattern                                                      |
| ------- | ------------------------------------------------------------ |
| 5.6.x   | 可手动启用“index.mapping.single_type: true”，使用 join 字段进行替换 |
| 6.x     | 允许索引使用单一类型，有且仅有一个名称，首选_doc 类型；5.x 版本的索引可延续使用，新版本的索引不再支持父/子方式和 join 类型；6.8中需要在索引创建、模板时需要显示指定 include _ type _ name，不显示申明会默认为 name _ doc |
| 7.x     | _doc 是路径中的永久部分，并且表示的是 endpoint 类型，而不是 doc 类型；include _ type _ name 参数默认为 false；移除 _default_ mapping 类型 |
| 8.x     | 不再支持特殊类型;移除 include_type_name 参数                 |


## 参考
[Mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html)
