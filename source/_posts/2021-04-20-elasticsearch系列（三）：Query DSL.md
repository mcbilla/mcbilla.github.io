---
title: elasticsearch系列（三）：Query DSL
date: 2021-04-20 14:39:25
categories:
- elasticsearch
tags:
- elasticsearch
---

> elasticsearch系列（三）：Query DSL

<!--more-->

### 一、Query DSL介绍
Elasticsearch 的检索主要有两种方式。
1. `_search`  + 请求参数。在 `_search`  后面拼接请求参数字符串。
```
GET /indexname/_search?q=*&sort=account_number:asc
```
这种方式缺点是请求参数越来越多，搜索条件越来越复杂的情况下，很难满足需求。

2. `_search`  + 请求体。这个 JSON 格式的请求体数据就称为 `Query DSL`（Domain Specified Language）。Elasticsearch 把 Lucene 功能的绝大部分封装成基于 JSON 的 Query DSL 查询语法。可以构建复杂的查询条件。下面这个请求实现和上面相同的效果。
```
GET /indexname/_search
{
    "query":{
        "match_all":{

        }
    },
    "sort":[
        {
            "account_number":"asc"
        }
    ]
}
```
`Query DSL` 是日常最常用的查询方式，是我们的学习重点。使用参考 [Query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)

DSL 的语法结构如下：
```
{
	QUERY_NAME:{
   		ARGUMENT:VALUE,
  		ARGUMENT:VALUE,...
	}
}
```
如果针对于某个字段，那么它的结构如下：
```
{
	QUERY_NAME:{
     	FIELD_NAME:{
       		ARGUMENT:VALUE,
       		ARGUMENT:VALUE,...
      	}   
   	}
}
```


### 二、全文检索
全文查询主要用于在全文字段（text）类型，在执行查询之前会分析查询字符串进行分词处理，主要考虑查询词与文档的相关性（Relevance）。
#### match_all 查询所有
```
GET /indexname/_search
{
    "query":{
        "match_all":{

        }
    }
}
```
elasticsearch 一般不会返回所有所有数据，默认返回前 10 条数据。可以结合 `from`  和 `size`  使用分页查询功能，还可以结合 `sort` 实现排序功能。
```
GET /indexname/_search
{
    "query":{
        "match_all":{

        }
    },
    "from":0,
    "size":5,
    "sort":[
        {
            "account_number":{
                "order":"desc"
            }
        }
    ]
}
```

#### match 匹配
match 有两种匹配情况：
* 基本类型（非字符串）和字符（keyword）类型，精确匹配。
* 字符串（text）类型，全文检索，首先会针对查询语句进行解析（经过 analyzer），主要是对查询语句进行分词，分词后查询语句的任何一个词项被匹配，文档就会被搜到，默认情况下相当于对分词后词项进行 or 匹配操作，最终会按照评分进行排序返回结果。注意不会进行正则匹配，比如"mcb"，只会查找出现"mcb"这个单词单独出现的文档，其他包含"mcb"的单词比如"mcb1"、"mcb2"之类的都不会匹配到。
```
GET /indexname/_search
{
    "query":{
        "match":{
            "name":"mcb"
        }
    }
}
```
如果要使用match进行精确匹配，可以使用 `字段名.keyword`。
```
GET /indexname/_search
{
    "query":{
        "match":{
            "name.keyword":"mcb"
        }
    }
}
```
这种方式和 match_phrase 的区别是
* `字段名.keyword` 是精确匹配，要完全等于 value 值才会返回。
* `match_phrase` 是短语匹配，只要包含 value 值就会返回。


#### match_phrase 匹配短语
match_phrase 首先会把 query 内容分词，分词器可以自定义，同时文档还要满足以下两个条件才会被搜索到：
* 分词后所有词项都要出现在该字段中（相当于 and 操作）。
* 字段中的词项顺序要一致。

比如有以下 3 个文档，使用 match_phrase 查询 “what life”，只有前两个文档会被匹配。
```
PUT /indexname/_doc/1
{ "desc": "what a wonderful life" }

PUT /indexname/_doc/2
{ "desc": "what a life"}

PUT /indexname/_doc/3
{ "desc": "life is what"}

GET /indexname/_search
{
    "query":{
        "match_phrase":{
            "desc":"what life"
        }
    }
}
```

#### multi_match 多字段匹配
用于搜索多个字段。例如下面查询内容为 `mill`，查询字段为 `state` 和 `address`。
```
GET /indexname/_search
{
    "query":{
        "multi_match":{
            "query":"mill",
            "fields":[
                "state",
                "address"
            ]
        }
    }
}
```
multi_match 还支持对要搜索的字段的名称使用通配符。
```
GET /indexname/_search
{
    "query":{
        "multi_match":{
            "query":"mill",
            "fields":[
                "state",
                "*_address"
            ]
        }
    }
}
```

### 三、词项检索
词项查询时对倒排索引中存储的词项进行精确匹配操作。词项级别的查询通常用于结构化数据，如数字、日期和枚举类型。
#### term 精确匹配
和 match 一样，匹配某个属性的值。全文检索字段用 match，其他非 text 字段匹配用 term。
```
GET /indexname/_search
{
    "query":{
        "term":{
            "name":"mcb"
        }
    }
}
```

#### terms 精确匹配多个单词
terms 查询是 term 查询的升级，可以用来查询文档中包含多个词的文档。比如，想查询 title 字段中包含关键词 “java” 或 “python” 的文档
```
GET /indexname/_search
{
    "query":{
        "terms":{
            "title":[
                "java",
                "python"
            ]
        }
    }
}
```

#### range 范围查找
用于匹配在某一范围内的数值型、日期类型或者字符串型字段的文档。使用 range 查询只能查询一个字段，不能作用在多个字段上。range 查询支持的参数有以下几种：
* gt 大于，查询范围的最小值，也就是下界，但是不包含临界值。
* gte 大于等于，和 gt 的区别在于包含临界值。
* lt 小于，查询范围的最大值，也就是上界，但是不包含临界值。
* lte 小于等于，和 lt 的区别在于包含临界值。

下面查询 15 <= age <= 25 的数据。
```
GET /indexname/_search
{
    "query":{
        "range":{
            "age":{
                "gte":15,
                "lte":25
            }
        }
    }
}
```

#### exists 非空查询
查询会返回字段中至少有一个非空值的文档。例如
```
GET /indexname/_search
{
    "query":{
        "exists":{
            "field":"user"
        }
    }
}
```
以下文档会匹配上面的查询：
* `{ "user" : "jane" }` 有 user 字段，且不为空。
* `{ "user" : "" }` 有 user 字段，值为空字符串。
* `{ "user" : "-" }` 有 user 字段，值不为空。
* `{ "user" : [ "jane" ] }` 有 user 字段，值不为空。
* `{ "user" : [ "jane", null ] }` 有 user 字段，至少一个值不为空即可。

下面的文档都不会被匹配：
* `{ "user" : null }` 虽然有 user 字段，但是值为空。
* `{ "user" : [] }` 虽然有 user 字段，但是值为空。
* `{ "user" : [null] }` 虽然有 user 字段，但是值为空。
* `{ "foo" : "bar" }` 没有 user 字段。

#### prefix 前缀查询
prefix 查询用于查询某个字段中以给定前缀开始的文档，比如查询 title 中含有以 java 为前缀的关键词的文档，那么含有 java、javascript、javaee 等所有以 java 开头关键词的文档都会被匹配。查询 description 字段中包含有以 win 为前缀的关键词的文档，查询语句如下：
```
GET /indexname/_search
{
    "query":{
        "prefix":{
            "title":"java"
        }
    }
}
```

#### regexp 正则查询
Elasticsearch 也支持正则表达式查询，通过 regexp query 可以查询指定字段包含与指定正则表达式匹配的文档。可以代表任意字符, “a.c.e” 和 “ab...” 都可以匹配 “abcde”，a{3}b{3}、a{2,3}b{2,4}、a{2,}{2,} 都可以匹配字符串 “aaabbb”。例如需要匹配以 W 开头紧跟着数字的邮政编码，使用正则表达式查询构造查询语句如下：
```
GET /indexname/_search
{
    "query":{
        "regexp":{
            "postcode":"W[0-9].+"
        }
    }
}
```

### 四、复合查询
复合查询就是把一些简单查询组合在一起实现更复杂的查询需求，除此之外，复合查询还可以控制另外一个查询的行为。
#### bool 组合多条件过滤
bool 查询可以把任意多个简单查询组合在一起，使用 must、should、must_not、filter 选项来表示简单查询之间的逻辑，每个选项都可以出现 0 次到多次，文档是否符合每个“must”或“should”子句中的标准，决定了文档的“相关性得分”。得分越高，文档越符合您的搜索条件。 默认情况下，Elasticsearch 返回根据这些相关性得分排序的文档。它们的含义如下：
* `must`  必须匹配所有查询条件，相当于逻辑运算的 AND，且参与文档相关度的评分。
* `should`  应该匹配一个或多个查询条件，相当于逻辑运算的 OR，且参与文档相关度的评分。如果匹配了会提高score，不匹配也不影响检索结果。如果query中只有should且只有一种匹配规则，那么should的条件就会被作为默认匹配条件改变查询结果。
* `must not`  必须不匹配所有查询条件。需要注意的是，**must_not 语句不会影响评分，它的作用只是将不相关的文档排除**。
* `filter`  过滤器，类似于 must。跟 must 的唯一区别是，filter 不计算相关性得分，所有数据返回的 score 都为 0。
```
GET /indexname/_search
{
    "query":{
        "bool":{
            "must":[
                {
                    "match":{
                        "gender":"M"
                    }
                },
                {
                    "match":{
                        "address":"mill"
                    }
                }
            ],
            "must_not":[
                {
                    "match":{
                        "age":"18"
                    }
                }
            ],
            "should":[
                {
                    "match":{
                        "lastname":"Wallace"
                    }
                }
            ],
            "filter":[
                {
                    "match":{
                        "city":"Urie"
                    }
                }
            ]
        }
    }
}
```

### 五、嵌套查询
#### nested 嵌套查询
当一个字段值是对象的时候，通常会被解析为Nested （嵌套）类型。查询对象里面的某个属性的时候，需要使用 nested 查询。例如有两条数据如下：
```
{
    "groups":"group1",
    "users":{
        "name":"John",
        "age":23
    }
}

{
    "groups":"group2",
    "users":{
        "name":"Kite",
        "age":26
    }
}
```
如果需要根据 users.name 来查询，需要使用 nested 查询。
```
GET /indexname/_search
{
    "query":{
        "nested":{
            "path":"users",
            "query":{
                "match":{
                    "users.name":"John"
                }
            }
        }
    }
}
```
* path: 复杂对象的 key。
* score_mode：（可选）匹配子对象的分数相关性分数。默认 avg，使用所有匹配子对象的平均相关性分数。
* ignore_unmapped （可选）是否忽略 path 未映射，不返回任何文档而不是错误。默认为 false，如果 path 不对就报错。


### 参考
[Elasticsearch（es） 查询语句语法详解 ](https://www.cnblogs.com/Gaimo/p/16036853.html)
