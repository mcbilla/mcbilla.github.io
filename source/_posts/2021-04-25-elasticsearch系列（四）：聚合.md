---
title: elasticsearch系列（四）：聚合
date: 2021-04-25 20:46:33
categories:
- elasticsearch
tags:
- elasticsearch
---

> elasticsearch系列（四）：聚合

<!--more-->

### 一、聚合介绍
聚合（aggregations）提供了从数据中分组和提取数据的能力。最简单的聚合方法大致等于SQL Group by 和 SQL 聚合函数。在 elasticsearch 中，执行搜索返回this（命中结果），并且同时返回聚合结果，把以响应中的所有hits（命中结果）分隔开的能力。这是非常强大且有效的，你可以执行查询和多个聚合，并且在一次使用中得到各自的（任何一个的）返回结果，使用一次简洁和简化的 API 来避免网络往返。
聚合有以下类型：

* `桶集合（Bucket aggregations）`：对查询出的数据进⾏分组group by，再在组上进⾏指标聚合。桶等同于组，分桶和分组是一个意思，ES使用桶代表一组相同特征的数据。
* `指标聚合（Metrics aggregations）`：先对数据进行分组（分桶），然后对每一个桶内的数据进行指标聚合，例如求最⼤、最⼩、和、平均值、去重等指标的聚合。
* ` 管道聚合（Pipeline aggregations）`：用的比较少。

聚合的语法如下：
```
"aggregations" : {
    "<aggregation_name>" : {
        "<aggregation_type>" : {
            <aggregation_body>
        }
        [,"meta" : {  [<meta_data_body>] } ]?
        [,"aggregations" : { [<sub_aggregation>]+ } ]?
    }
    [,"<aggregation_name_2>" : { ... } ]*
}
```
* aggregation_name：聚合名称，这个可以自己任意定义。因为ES支持一次进行多次统计分析查询，后面需要通过这个名字在查询结果中找到我们想要的计算结果。
* aggregation_type：聚合类型，代表我们想要怎么统计数据，主要有两大类聚合类型：桶聚合和指标聚合，这两类聚合又包括多种聚合类型，例如：指标聚合：sum、avg， 桶聚合：terms、Date histogram等等。
* meta：元数据，用的比较少。
* aggregations：子聚合，也就是说聚合里面还可以嵌套聚合。
* aggregation_name_2：并列聚合名称，同一个 aggregations 下可以并列多个聚合名称，也就是可以一次进行多种类型的统计。

下面是一个简单使用示例，对 age 求平均值。
```
GET indexname/_search
{
    "query":{
        "match_all":{

        }
    },
    "aggs":{
        "avgAge":{
            "avg":{
                "field":"age"
            }
        }
    },
    "size":0
}
```
* aggs：表示使用聚合统计，聚合要和 query 一起配合使用，才能对返回的数据进行统计。
* avgAge：聚合名称，用户自定义名称。
* avg：聚合类型，avg 表示求平均值，还可以使用 max 求最大值，min 求最小值，sum 求合等。
* field：需要聚合的字段。
* size：查询结果条数，0 表示不返回查询数据，只返回聚合结果。

聚合里面还可以嵌套聚合，例如按照年龄聚合， 并且请求这些年龄段的这些人的平均薪资
```
GET indexname/_search
{
    "query":{
        "match_all":{

        }
    },
    "aggs":{
        "ageAgg":{
            "terms":{
                "field":"age",
                "size":100
            },
            "aggs":{
                "ageAvg":{
                    "avg":{
                        "field":"balance"
                    }
                }
            }
        }
    },
    "size":0
}
```

### 二、桶集合
桶聚合，目的就是数据分组，先将数据按指定的条件分成多个组，然后对每一个组进行统计。 组的概念跟桶是等同的，在ES中统一使用桶（bucket）这个术语。
ES桶聚合的作用跟 SQL 的 group by 的作用是一样的，区别是ES支持更加强大的数据分组能力，SQL 只能根据字段的唯一值进行分组，分组的数量跟字段的唯一值的数量相等，例如: group by 店铺id， 去掉重复的店铺ID后，有多少个店铺就有多少个分组。
ES 常用的桶聚合如下：
* `Terms聚合` - 类似SQL的group by，根据字段唯一值分组
* `Histogram聚合` - 根据数值间隔分组，例如: 价格按100间隔分组，0、100、200、300等等
* `Date histogram聚合` - 根据时间间隔分组，例如：按月、按天、按小时分组
* `Range聚合` - 按数值范围分组，例如: 0-150一组，150-200一组，200-500一组。

> 提示：桶聚合一般不单独使用，都是配合指标聚合一起使用，对数据分组之后肯定要统计桶内数据，在ES中如果没有明确指定指标聚合，默认使用Value Count指标聚合，统计桶内文档总数。

#### terms 聚合
类似于 mysql 的 group by 功能，都是根据字段唯一值对数据进行分组（分桶），字段值相等的文档都分到同一个桶内。
```
GET indexname/_search
{
    "query":{
        "match_all":{

        }
    },
    "aggs":{
        "aggsAge":{
            "terms":{
                "field":"age",
                "size":5, // 返回的分组数，例如分组结果一共分成了5个组，5表示只返回前5个分组的数据。
            }
        }
    },
    "size":0
}
```
返回数据
```
{
......
    "aggregations":{
        "aggsAge":{ // 聚合查询名称
            "doc_count_error_upper_bound":0,
            "sum_other_doc_count":775,
            "buckets":[
                {
                    "key":20, // key分桶的标识，在terms聚合中，代表的就是分桶的字段值
                    "doc_count":44 // 默认的指标聚合是统计桶内文档总数
                },
                {
                    "key":21,
                    "doc_count":46
                },
                {
                    "key":22,
                    "doc_count":51
                }
            ]
        }
    }
}
```

#### histogram 聚合
histogram（直方图）聚合，主要根据数值间隔分组，使用histogram聚合分桶统计结果，通常用在绘制条形图报表。
```
GET indexname/_search
{
    "query":{
        "match_all":{

        }
    },
    "query":{
        "match_all":{

        }
    },
    "aggs":{
        "aggsAge":{
            "histogram":{
                "field":"age",
                "interval":5 // 分桶的间隔为5，意思就是 age 字段值按 5 间隔分组
            }
        }
    },
    "size":0
}
```
返回结果
```
{
......
    "aggregations":{
        "aggsAge":{ // 聚合查询名称
            "buckets":[ // 分桶结果
                {
                    "key":20, // 桶的标识，histogram分桶，这里是分组的间隔值
                    "doc_count":451 // 如果没有指定指标，默认按Value Count指标聚合，统计桶内文档总数
                },
                {
                    "key":30,
                    "doc_count":504
                },
                {
                    "key":40,
                    "doc_count":45
                }
            ]
        }
    }
}
```

#### date histogram 聚合
类似 histogram 聚合，区别是 Date histogram 可以很好的处理时间类型字段，主要用于根据时间、日期分桶的场景。
```
GET indexname/_search
{
    "query":{
        "match_all":{

        }
    },
    "aggs" : {
        "sales_over_time" : { // 聚合查询名字，随便取一个
            "date_histogram" : { // 聚合类型为: date_histogram
                "field" : "date", // 根据date字段分组
                "calendar_interval" : "month", // 分组间隔：month代表每月、支持minute（每分钟）、hour（每小时）、day（每天）、week（每周）、year（每年）
                "format" : "yyyy-MM-dd" // 设置返回结果中桶key的时间格式
            }
        }
    }
}
```
返回数据
```
{
    ...
    "aggregations": {
        "sales_over_time": { // 聚合查询名字
            "buckets": [ // 桶聚合结果
                {
                    "key_as_string": "2015-01-01", // 每个桶key的字符串标识，格式由format指定
                    "key": 1420070400000, // key的具体字段值
                    "doc_count": 3 // 默认按Value Count指标聚合，统计桶内文档总数
                },
                {
                    "key_as_string": "2015-02-01",
                    "key": 1422748800000,
                    "doc_count": 2
                },
                {
                    "key_as_string": "2015-03-01",
                    "key": 1425168000000,
                    "doc_count": 2
                }
            ]
        }
    }
}
```

#### range 聚合
range 聚合，按数值范围分桶。
```
GET idnexname/_search
{
    "query":{
        "match_all":{

        }
    },
    "aggs" : {
        "price_ranges" : { // 聚合查询名字，随便取一个
            "range" : { // 聚合类型为： range
                "field" : "price", // 根据price字段分桶
                "ranges" : [ // 范围配置
                    { "to" : 100.0 }, // 意思就是 price <= 100的文档归类到一个桶
                    { "from" : 100.0, "to" : 200.0 }, // price>100 and price<200的文档归类到一个桶
                    { "from" : 200.0 } // price>200的文档归类到一个桶
                ]
            }
        }
    }
}
```
返回结果
```
{
    ...
    "aggregations": {
        "price_ranges" : { // 聚合查询名字
            "buckets": [ // 桶聚合结果
                {
                    "key": "*-100.0", // key可以表达分桶的范围
                    "to": 100.0, // 结束值
                    "doc_count": 2 // 默认按Value Count指标聚合，统计桶内文档总数
                },
                {
                    "key": "100.0-200.0",
                    "from": 100.0, // 起始值
                    "to": 200.0, // 结束值
                    "doc_count": 2
                },
                {
                    "key": "200.0-*",
                    "from": 200.0,
                    "doc_count": 3
                }
            ]
        }
    }
}
```
大家仔细观察的话，发现range分桶，默认key的值不太友好，尤其开发的时候，不知道key长什么样子，处理起来比较麻烦，我们可以为每一个分桶指定一个有意义的名字。
```
GET idnexname/_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "field" : "price",
                "keyed" : true,
                "ranges" : [
                    // 通过key参数，配置每一个分桶的名字
                    { "key" : "cheap", "to" : 100 },
                    { "key" : "average", "from" : 100, "to" : 200 },
                    { "key" : "expensive", "from" : 200 }
                ]
            }
        }
    }
}
```
#### 多桶嵌套查询
```
GET /indexname/_search
{
    "size": 0, // size=0代表不需要返回query查询结果，仅仅返回aggs统计结果
    "query" : { // 设置查询语句，先赛选文档
        "match" : {
            "make" : "ford"
        }
    },
    "aggs" : { // 然后对query搜索的结果，进行统计
        "colors" : { // 聚合查询名字
            "terms" : { // 聚合类型为：terms 先分桶
              "field" : "color"
            },
            "aggs": { // 通过嵌套聚合查询，设置桶内指标聚合条件
              "avg_price": { // 聚合查询名字
                "avg": { // 聚合类型为: avg指标聚合
                  "field": "price" // 根据price字段计算平均值
                }
              },
              "sum_price": { // 聚合查询名字
                "sum": { // 聚合类型为: sum指标聚合
                  "field": "price" // 根据price字段求和
                }
              }
            }
        }
    }
}
```

#### 多桶排序
类似terms、histogram、date_histogram这类桶聚合都会动态生成多个桶，如果生成的桶特别多，有时候我们需要确定这些桶的排序顺序，并限制返回桶的数量。
默认情况，ES会根据 doc_count 文档总数，降序排序。ES桶聚合支持两种方式排序：
* 内置排序
* 按度量指标排序
##### 内置排序
内置排序参数：
* _count - 按文档数排序。对 terms 、 histogram 、 date_histogram 有效
* _term - 按词项的字符串值的字母顺序排序。只在 terms 内使用
* _key - 按每个桶的键值数值排序, 仅对 histogram 和 date_histogram 有效
```
GET /indexname/_search
{
    "size" : 0,
    "aggs" : {
        "colors" : { // 聚合查询名字，随便取一个
            "terms" : { // 聚合类型为: terms
              "field" : "color", 
              "order": { // 设置排序参数
                "_count" : "asc"  // 根据_count排序，asc升序，desc降序
              }
            }
        }
    }
```

##### 按度量排序
通常情况下，我们根据桶聚合分桶后，都会对桶内进行多个维度的指标聚合，所以我们也可以根据桶内指标聚合的结果进行排序。
```
GET /indexname/_search
{
    "size" : 0,
    "aggs" : {
        "colors" : { // 聚合查询名字
            "terms" : { // 聚合类型: terms，先分桶
              "field" : "color", // 分桶字段为color
              "order": { // 设置排序参数
                "avg_price" : "asc"  // 根据avg_price指标聚合结果，升序排序。
              }
            },
            "aggs": { // 嵌套聚合查询，设置桶内聚合指标
                "avg_price": { // 聚合查询名字，前面排序引用的就是这个名字
                    "avg": {"field": "price"} // 计算price字段平均值
                }
            }
        }
    }
}
```

#### 限制返回桶的数量
如果分桶的数量太多，可以通过给桶聚合增加一个 size 参数限制返回桶的数量。
```
GET  /indexname/_search
{
    "aggs" : {
        "products" : { // 聚合查询名字
            "terms" : { // 聚合类型为: terms
                "field" : "product", // 根据product字段分桶
                "size" : 5 // 限制最多返回5个桶
            }
        }
    }
}
```

### 三、指标聚合
#### value_count 统计数量
value_count表示统计⾮空字段的⽂档数，类似 mysql 的 count 函数。
```
GET indexname/_search
{
    "query":{
        "match_all":{

        }
    },
    "aggs":{
        "countAge":{
            "value_count":{
                "field":"age"
            }
        }
    },
    "size":0
}
```

#### cardinality 去重统计数量
也是用于统计文档的总数，跟 Value Count 的区别是，cardinality 聚合会去重，类似于 mysql 的 count(DISTINCT 字段)
```
GET indexname/_search
{
    "query":{
        "match_all":{

        }
    },
    "aggs":{
        "distinctAge":{
            "cardinality":{
                "field":"age"
            }
        }
    },
    "size":0
}
```

#### stats 统计
stats同时统计 count max min avg sum 5个值
```
GET indexname/_search
{
    "query":{
        "match_all":{

        }
    },
    "aggs":{
        "statsAge":{
            "stats":{
                "field":"age"
            }
        }
    },
    "size":0
}
```

#### extended_stats 扩展统计
extended_stats ⽐ stats 多4个统计结果： 平⽅和、⽅差、标准差、平均值加/减两个标准差的区间。
```
GET indexname/_search
{
    "query":{
        "match_all":{

        }
    },
    "aggs":{
        "extendStatsAge":{
            "extended_stats":{
                "field":"age"
            }
        }
    },
    "size":0
}
```

#### percentiles 区间统计
percentiles表示占⽐百分位对应的值统计，默认返回[ 1, 5, 25, 50, 75, 95, 99 ]分位上的值
```
GET indexname/_search
{
    "query":{
        "match_all":{

        }
    },
    "aggs":{
        "percentilesAge":{
            "percentiles":{
                "field":"age"
            }
        }
    },
    "size":0
}
```
还可以指定百分位区间
```
GET indexname/_search
{
    "query":{
        "match_all":{

        }
    },
    "aggs":{
        "percentilesAge":{
            "percentiles":{
                "field":"age",
                "percents":[
                    20,
                    50,
                    75
                ]
            }
        }
    },
    "size":0
}
```

### 参考
 [官方文档 Aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html)

[Elasticsearch 聚合查询（aggs）基本概念](https://www.tizi365.com/archives/644.html)
