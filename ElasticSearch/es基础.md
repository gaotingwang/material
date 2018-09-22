[TOC]

##索引操作

### 索引初始化

关于索引等命名==不能以下划线开头，不能包含逗号== ，而且索引名必须是小写的。

```json
PUT /my_temp_index
{
    "settings": {
    	# 每个索引的主分片数，默认值是 5 
      	# 这个配置在索引创建后不能修改。
        "number_of_shards": 1, 
      	# 每个主分片的副本数，默认值是 1，这个配置可随时修改
        "number_of_replicas": 0
    }
}
```

### 索引创建

```json
PUT /my_index
{
  	# 相关索引的设置
    "settings": { ... any settings ... },
    "mappings": {
      	# 对索引下的多个_type设置对应的mapping
        "type_one": { ... any mappings ... },
        "type_two": { ... any mappings ... },
        ...
    }
}
```

如果想禁止自动创建索引，可以在 `config/elasticsearch.yml` 中添加配置：`action.auto_create_index: false`

### 索引查看

```json
# 通过/_all请求可以查看所有索引情况
GET /_all
# 查看指定索引情况
GET /my_index
```

###索引的配置修改 

```json
PUT /my_temp_index/_settings
{
    # 动态修改副本数
    "number_of_replicas": 1
}
```

### 索引删除

```json
# 删除指定索引
DELETE /my_index,index_two 
DELETE /index_*

# 删除所有索引
# 可以在elasticsearch.yml中配置action.destructive_requires_name: true，
# 来不允许通过指定 _all 或通配符来删除指定索引库
DELETE /_all
DELETE /*
```



##mapping映射管理

创建索引的时候，预先定义字段类型及属性（创建在_type上） 

###内置类型

- String: `text`会进行分析，`keyword`不会建立倒排索引不会进行分析
- 数字类型：`long`,`integer`,`short`,`byte`,`double`,`float`
- 日期类型：`date`
- bool类型：`boolean`
- binary类型：`binary` 二进制数据不会被检索
- 复杂类型：`object`,`nested`(对象数组)
- geo类型：地理位置`geo-point`,` geo-shape`
- 专业类型：`ip`,` competion`(搜索建议)

![](https://gtw.oss-cn-shanghai.aliyuncs.com/es/es%E6%98%A0%E5%B0%84.png)

###创建映射

```json
PUT /lagou
{
  "mappings": {
    "job" : { # 对应指定的_type
      "properties" : {
        "title" : {
          "store": true,
          "type" : "text",
          "analyzer": "ik_max_word" # analyzer指定分析器
        },
        "company" : {
          "properties" : {
            "name" : {
              "store": true,
              "type" : "text",
              "analyzer": "ik_max_word" # analyzer指定分析器
           	},
            "addr" : {
              "store": true,
          	  "type" : "keyword" # 不会建立倒排索引不会进行分析
            }
          }
        },
        "desc" : {
          "type" : "text"
        },
        "comment": {
          "type": "integer"
        },
        "add_time" : {
          "type" : "date",
          "format": "yyyy-MM-dd"
        }
      }
    }
  }
}

# 通过/_mapping查看指定_type的mapping情况
GET /lagou/_mapping/job
```

### 更新映射

当创建一个索引的时候，可以指定类型的映射。也可以使用 `/_mapping` 为`_type`（或者为存在的类型更新映射）增加映射。

尽管**==可以增加一个存在的映射，但不能修改存在的域映射==**。如果一个域的映射已经存在，那么该域的数据可能已经被索引。如果意图修改这个域的映射，索引的数据可能会出错，不能被正常的搜索。

比如可以更新一个映射来添加一个新域，但不能将一个存在的域从 `analyzed` 改为 `not_analyzed` 。

```json
PUT /lagou/_mapping/job
{
  # 使用/_mapping 增加一个新的名为 tag 的 not_analyzed 的文本域，
  "properties" : {
    "tag" : {
      "type" :    "text",
      "index":    "not_analyzed"
    }
  }
}
```

上述方法无法用于将原有的如 title域 映射类型改变，如果想要修改只能通过`DELETE /lagou`删除原有映射，然后重建。



##文档CRUD

### 创建文档

确保创建一个新文档的最简单办法是，使用索引请求的 `POST` 形式让 Elasticsearch 自动生成唯一 `_id` :

```json
# 提供自定义的 _id 值
# 如果index + type + id已存在则为覆盖修改
PUT /{index}/{type}/{id}
{
  "field": "value",
  ...
}
```

创建一个全新的索引，只有在相同的 `_index` 、 `_type` 和 `_id` 不存在时才接受创建请求，所以确保创建一个新文档的最简单办法是，使用索引请求的 `POST` 形式让 Elasticsearch 自动生成唯一 `_id` ：

```json
# Elasticsearch 会自动生成 ID
POST /website/blog/
{
  "title": "My second blog entry",
  "text":  "Still trying this out...",
  "date":  "2014/01/01"
}
```

如果想在只有在相同的 `_index` 、 `_type` 和 `_id` 不存在时才接受索引请求：

- 使用 `op_type` 查询 -字符串参数：

  ```json
  PUT /website/blog/123?op_type=create
  { ... }
  ```

- 在 URL 末端使用 `/_create` :

  ```json
  PUT /website/blog/123/_create
  { ... }
  ```

如果文档已经存在，Elasticsearch 将会返回 `409 `的status。

### 修改文档

==在 Elasticsearch 中文档是*不可改变* 的，不能修改它们。== 如果想要更新现有的文档，需要***重建索引*** 或者进行替换：

```shell
# 把原有文档中的内容替换成当前内容
PUT /website/blog/123
{
  "title": "My first blog entry",
  "text":  "I am starting to get the hang of this...",
  "date":  "2014/01/02"
}

# 在内部，Elasticsearch已将旧文档标记为已删除，并增加一个全新的文档。 尽管不能再对旧版本的文档进行访问，但它并不会立即消失
{
  "_index" :   "website",
  "_type" :    "blog",
  "_id" :      "123",
  "_version" : 2, # _version会增加
  "created":   false  # created 标志设置成 false ，是因为相同的索引、类型和 ID 的文档已经存在
}
```

实际上 Elasticsearch 执行`update`过程如下：

1. 从旧文档构建 JSON
2. 更改该 JSON
3. 删除旧文档
4. 索引一个新文档

这样客户端不需要单独的 `get` 和 `index` 请求。

文档是不可变的：他们不能被修改，只能被替换。如果想对文档进行部分更新，`update` 请求最简单的一种形式是接收文档的一部分作为 `doc` 的参数， 它只是与现有的文档进行合并。对象被合并到一起，**覆盖现有的字段**，**增加新的字段**。 

```json
# 经历与之前描述相同的 检索-修改-重建索引 的处理过程
POST /website/blog/1/_update
{
   # 增加字段 tags 和 views 到博客文章
   "doc" : {
      "tags" : [ "testing" ],
      "views": 0
   }
}
```

### 删除文档

```json
DELETE /website/blog/123

# 对于_type不支持删除，可以对_index进行删除

# 如果文档找到返回status为200，未找到404
{
  "found" :    false,  # 如果文档没有找到
  "_index" :   "website",
  "_type" :    "blog",
  "_id" :      "123",
  "_version" : 4 #  即使文档没找到_version 值仍然会增加。
}
```



##mget、bulk批量操作

### 批量查询

需要从 Elasticsearch 检索很多文档，`mget` API 要求有一个 `docs` 数组作为参数：

```json
GET /_mget
{
   "docs" : [
      {
         "_index" : "website",
         "_type" :  "blog",
         "_id" :    2
      },
      {
         "_index" : "website",
         "_type" :  "pageviews",
         "_id" :    1,
         # 如果想检索一个或者多个特定的字段，那么可以通过 _source 参数来指定这些字段的名字
         "_source": "views"
      }
   ]
}

# 如果想检索的数据都在相同的 _index 中（甚至相同的 _type 中），则可以在 URL 中指定默认的 /_index/_type 
GET /website/blog/_mget
{
   "docs" : [
      { "_id" : 2 },
      { 
        "_type" : "pageviews", 
        "_id" :   1 
      }
   ]
}

# 如果所有文档的 _index 和 _type 都是相同的，可以只传一个 ids 数组，而不是整个 docs 数组：
GET /website/blog/_mget
{
   "ids" : [ "2", "1" ]
}

# 对于返回结果，如果第二个文档未能找到并不妨碍第一个文档被检索到，每个文档都是单独检索和报告的。
```

### `bulk`操作

一个`bulk`的请求体：

```json
# 指定哪一个文档做什么操作 
{ action: { metadata }}\n
# 具体操作对应的请求体
{ request body        }\n
...
```

关于`bulk`操作，需要注意事项：

- 每行一定要以换行符(`\n`)结尾，谨记最后一个换行符不要落下
- 这些行**不能包含未转义的换行符**，否则会对解析造成干扰。这意味着这个 Json不能使用 pretty 参数打印。
- `bulk` 请求不是原子的： 不能用它来实现事务控制。每个请求是单独处理的，因此一个请求的成功或失败不会影响其他的请求

```json
POST /_bulk
# delete 操作没有请求体
{ "delete": { "_index": "website", "_type": "blog", "_id": "123" }} 
# create 如果文档不存在，那么就创建它
{ "create": { "_index": "website", "_type": "blog", "_id": "123" }}
{ "title":    "My first blog post" }
# index 创建一个新文档或者替换一个现有的文档
{ "index":  { "_index": "website", "_type": "blog" }}
{ "title":    "My second blog post" }
# update 部分更新一个文档
{ "update": { "_index": "website", "_type": "blog", "_id": "123", "_retry_on_conflict" : 3} }
{ "doc" : {"title" : "My updated blog post"} } # 最后一个换行符不要落下

```

相同index下不需要重复指定`_index`或`_type` ：

```json
# 可以像 mget API 一样，在 bulk 请求的 URL 中接收默认的 /_index 或者 /_index/_type ：
POST /website/_bulk
{ "index": { "_type": "log" }}
{ "event": "User logged in" }

POST /website/log/_bulk
{ "index": {}}
{ "event": "User logged in" }
{ "index": { "_type": "blog" }} # 也可以覆盖元数据行中的 _index 和 _type 
{ "title": "Overriding the default type" }

```

整个批量请求都需要由接收到请求的节点加载到内存中，因此该请求越大，其他请求所能获得的内存就越少。密切关注批量请求的物理大小往往非常有用，一个好的批量大小在开始处理后所占用的物理大小约为 5-15 MB。判断批量大小一个好的办法是开始时将 1,000 到 5,000 个文档作为一个批次，如果文档非常大，那么就减少批量的文档个数。



##查询

```shell
$ curl -X GET "localhost:9200/website/blog/123?pretty"
```

在请求的查询串参数中加上 `pretty` 参数，将会调用 Elasticsearch 的 *pretty-print* 功能，该功能使得 JSON 响应体更加可读。但是 `_source`字段不能被格式化打印出来，得到的 `_source` 字段中的 JSON 串，刚好是和我们传给它的一样。

```shell
# 指定_source只返回需要的字段
GET /website/blog/123?_source=title,text
```

如果只想检查一个文档是否存在 --根本不想关心内容--那么用 `HEAD` 方法来代替 `GET` 方法。 `HEAD` 请求没有返回体，只返回一个 HTTP 请求报头：

```shell
$ curl -i -XHEAD http://localhost:9200/website/blog/123

# 如果文档存在，将返回一个 200 ok 的状态码；不存在，将返回一个 404 Not Found 的状态码
HTTP/1.1 200 OK  or  HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=UTF-8
Content-Length: 0
```

###基本查询：使用es内置查询条件查询

一个查询语句的典型结构：

```json
GET /_search
{
    QUERY_NAME: {
        ARGUMENT: VALUE,
        ARGUMENT: VALUE,...
    }
}
  
# 针对具体字段的查询结构
{
    QUERY_NAME: {
        FIELD_NAME: {
            ARGUMENT: VALUE,
            ARGUMENT: VALUE,...
        }
    }
}
```

查询所有文档：

```json
# match_all相当于不带条件的全查询
GET /lagou/_search
{
    "query": {
        "match_all": {}
    }
}
```

#### match分词查询

关于match查询得分，需要理解**相关性**：

- **检索词频率（TF）：**

  检索词在该==字段中出现的频率，出现频率越高，相关性也越高==。 字段中出现过 5 次要比只出现过 1 次的相关性高。

- **反向文档频率（IDF）：**

  每个检索词在==索引中出现的频率，频率越高，相关性越低==。检索词出现在多数文档中会比出现在少数文档中的权重更低。

  每个检索词在索引中出现的频率？频率越高，相关性越低。检索词出现在多数文档中会比出现在少数文档中的权重更低。

- **字段长度准则：**

  字段长度越长，相关性越低。 检索词出现在一个短的 title 要比同样的词出现在一个长的 content 字段权重更大。

```json
GET /lagou/job/_search
{
    "query": {
      	# 根据分词器进行分词查询
        "match": {
            "title": "python学习"
        }
    }
}
```

#### term精确查找

```json
GET /lagou/job/_search
{
    "query": {
      	# 相当于全匹配查询，如果字段有分词，分词结果必须完全匹配指定内容才会返回
        # 此处如果content有使用默认分词，Python不会被搜索出来
        # 因为在分词时Python被分为python，精确匹配时匹配不到
        "term": {
            "content": "Python"
        }
    }
}
```

terms多个精确值匹配

```json
GET /lagou/job/_search
{
    "query": {
        "terms": {
      		# 满足数组中的任意一个都会返回
            "title": ["工程师", "python", "java"]
        }
    }
}
```

#### 其他基本查询

```json
# 短语查询
GET /lagou/_search
{
   "query": {
     "match_phrase": {
       "title": { // title字段
         # 先进行分词，拆分为[python, 系统]
         # 必须满足所有分词
         # slop指定了分词之间的最小距离
         "query": "python系统", 
         "slop": 6
       }
     }
   }
}

# 多字段匹配查询
GET /lagou/_search
{
   "query": {
     "multi_match": {
       "query": "python",
        # 在多个字段中查询关键字，title的权重为3
       "fields": ["title^3", "desc"]
     }
   }
}

# 通配符查询
GET /my_index/address/_search
{
    "query": {
        "wildcard": {
          	# ? 匹配任意字符， * 匹配 0 或多个字符
            "postcode": "W?F*HW" 
        }
    }
}

# 正则查询
GET /my_index/address/_search
{
    "query": {
        "regexp": {
            "postcode": "W[0-9].+" 
        }
    }
}

# 范围查询
GET /my_store/products/_search
{
    "query" : {
      "range" : {
        "price" : {
          "gte" : 20,
          "lt"  : 40,
          "boost" : 2.0 # 权重
        }
      }
    }
}

# 指定返回的字段
GET /lagou/_search
{
  # 只可以指定 stored为true的字段
  "stored_fields": ["title","company"], 
   "query": {
     "match": {
       "title": "python"
     }
   }
}

# 查询结果排序
GET /lagou/_search
{
    "query": {
        "match_all": {}
    },
    "sort": [{
      "comments": {
      	 "order": "desc"
      } 
    }]
}

# 分页查询
GET /lagou/job/_search
{
    "query": {
        "match": {
            "title": "python学习"
        }
    },
	"from": 0,
	"size": 10
}

# 高亮结果
GET /_search
{
    "query" : {
        "match": { "content": "kimchy" }
    },
    "highlight": {
        # 指定高亮需要添加的标签
        "pre_tags": ['<span class="keyWord">'],
        "post_tags": ['</span>'],
        # 指定高亮的字段
        "fields": {
            "title": {},
    		"content": {}
        }
  	}
}

```

###过滤查询：查询同时，通过filter不影响打分的情况下筛选数据

```json
GET /test/test/_search
{
    "query": {
        "filtered": {
            "filter": {
              	# term 查询被用于精确值匹配
              	# 这些精确值可能是数字、时间、布尔或者那些 not_analyzed 的字符串
                "term": {
                    "tag": "full_text"
                }
            }
        }
    }
}
```

###组合查询： 多个查询组合在一起复合查询

==倒排索引源于应用中需要根据属性值来查找记录，索引表中每一项包括一个属性值和具有该属性值的各记录的地址。由于不是由记录确定属性值，而是由属性值确定记录的位置，因而称为倒排索引。==

Elasticsearch 使用的查询语言（DSL） 拥有一套查询组件，可以在以下两种情况下使用：过滤情况（filtering context）和查询情况（query context）。

- 当使用于 *过滤情况* 时，查询被设置成一个“不评分”查询。回答也是非常的简单，yes 或者 no ，二者必居其一。

  为了明确和简单，用 "filter" 这个词表示不评分

- 当使用于 *查询情况* 时，查询就变成了一个“评分”的查询。也要去判断这个文档是否匹配，==同时它还需要判断这个文档的匹配程度如何==。

  一个评分查询计算每一个文档与此查询的 _相关程度_，同时将这个相关程度分配给表示相关性的字段 `_score`，并且按照相关性对匹配到的文档进行排序。这种相关性的概念是非常适合全文搜索的情况，因为全文搜索几乎没有完全 “正确” 的答案。

```json
GET /test/test/_search
{
    "query": {
      	# 用 bool 查询来实现组合多查询
      	# 每一个子查询都独自地计算文档的相关性得分。
      	# 一旦他们的得分被计算出来， bool 查询就将这些得分进行合并并且返回一个代表整个布尔操作的得分。
        "bool": {
          	# 文档必须匹配这些条件才能被包含进来
      		# 如果没有 must 语句，那么至少需要能够匹配其中的一条 should 语句。
      		# 如果存在至少一条 must 语句，则对 should 语句的匹配没有要求。
            "must": {
      			# 无论在任何字段上进行的是全文搜索还是精确查询，match 查询是可用的标准查询。
      			# 在一个全文字段上使用 match 查询，在执行查询前，它将用正确的分析器去分析查询字符串
      			# 如果在一个精确值的字段上使用它，如数字、日期、布尔或者一个not_analyzed 字符串字段，那么它将会精确匹配给定的值：
                "match": {
                    "title": "how to make millions"
                }
            }, 
		   # 必须不匹配
            "must_not": {
                "match": {
                    "tag": "spam"
                }
            }, 
		   # 如果满足这些语句中的任意语句，将增加 _score ，否则，无任何影响。
		   # 主要用于修正每个文档的相关性得分。
            "should": [
                {
                    "match": {
                        "tag": "starred"
                    }
                }
            ], 
		   # 必须匹配，但它不评分，以过滤模式来进行，是根据过滤标准来排除或包含文档。
            "filter": {
              	# 将 bool 查询包裹在 filter 语句中，我们可以在过滤标准中增加布尔逻辑
                "bool": {
                    "must": [
                        {
                            "range": {
                                "price": {
                                    "lte": 29.99
                                }
                            }
                        }
                    ], 
                    "must_not": [
                        {
                            "term": {
                                "category": "ebooks"
                            }
                        }
                    ]
                }
            }
        }
		# 尽管没有 bool 查询使用这么频繁，constant_score 查询也是查询工具。
		# 它将一个不变的常量评分应用于所有匹配的文档。它被经常用于只需要执行一个 filter 而没有其它查询
		"constant_score":   {
              "filter": {
                  "term": { "category": "ebooks" } 
              }
          }
    }
}

```

`null`值处理：

```json
# SELECT * FROM   posts WHERE tags IS NOT NULL
GET /my_index/posts/_search
{
    "query" : {
    	"bool": {
         	"filter" : {
                "exists" : { 
                    "field" : "tags" 
                }
        	}
        } 
    }
}

# SELECT * FROM  posts WHERE tags IS NULL
GET /my_index/posts/_search
{
    "query" : {
    	"bool": {
         	"filter" : {
                "missing" : { 
                    "field" : "tags" 
                }
        	}
        } 
    }
}
```

### 查询理解

对于合法查询，使用 `explain` 参数将返回可读的描述，这对准确理解 Elasticsearch 是如何解析的非常有用的：

```json
GET /us,gb/_validate/query?explain
{
    "query": {
        "match": {
            "tweet": "really powerful"
        }
    }
}

{
    "valid": true, 
  	"_shards" : { ... },
    "explanations": [
      	# 每一个 index 都会返回对应的 explanation
        {
            "index": "us", 
            "valid": true, 
            "explanation": "tweet:really tweet:powerful"
        }, 
        {
            "index": "gb", 
            "valid": true, 
            "explanation": "tweet:realli tweet:power" # 字段的分析器为 english 分析器。
        }
    ]
}
```

**==查看分析器解析的结果==**：

```json
# 可以查看分词器将指定文本拆分成什么样
GET /_analyze
{
   "analyzer": "ik_smart", # 指定分析器
   "text": "Python网络" # 要分析的文本内容
}
```



## 搜索建议

关于[suggestor详解](https://elasticsearch.cn/article/142)，用户输入一部分字，搜索引擎会给出建议词，如：输入ela,搜索引擎可以给出以ela开头的一些词。优点：

1. 对于用户，可以减少输入
2. 对于搜索引擎，增加了用户输入的准确性

同时es的suggester有纠错的功能，如输入sabew,可以返回正确的saber。Suggester一共有四种：

- Term suggester

  把输入以一个个term的形式来分析，并返回你建议

- Prase Suggester

  以词组为单位返回建议

- Completion Suggester**提供了自动完成搜索即用型的功能，不适用于拼写纠错**。但Completion Suggester还支持模糊查询，意味着可以在搜索中输入拼写错误并仍然可以获得结果。

- Context Suggester

`completion`提供自动完成/搜索功能，可以在用户输入时引导用户查看相关结果，从而提高搜索精度。理想情况下，自动完成功能应该与用户键入的速度一样快，以提供与用户输入内容相关的即时反馈。因此，`completion`建议器针对速度进行了优化。搜索建议能够快速查找的数据结构，但构建成本高并且存储在内存中。

###设置`completion`类型的mapping

```json
PUT music
{
    "mappings": {
        "song" : {
            "properties" : {
                # 搜索建议字段的type为completion
                "suggest" : {
                    "type" : "completion", #自动补全建议
                	"analyzer": "ik_smart"
                },
                # 其他字段
                "title" : {
                    "type": "keyword"
                }
            }
        }
    }
}
```

### suggest字段中的值存储

```json
PUT music/song/1?refresh # refresh将创建完一个文档后立即刷新索引，为了保证搜索可见
{
    "suggest" : {
        # 在搜索时会对数组中的值进行补全
        "input": [ "Nevermind", "Nirvana" ],
        "weight" : 34
    }
}


PUT music/song/1?refresh
{
	# 多值并采用不同权重
    "suggest" : [
        {
            "input": [ "架构","架构师"],
            "weight": 10
        },
        {
            "input": [ "空间","团队","发展","气氛"],
            "weight": 6
        }
    ]
}
```

###suggest查询

suggest查询主要用于输入补全，可纠错。

```json
# 模糊匹配查询
GET /my_index/my_type/_search
{
   "query": {
      # fuzzy表示模糊搜索
      "fuzzy": {
         # 搜索字段
         "text": {
           "value": "surprize", # 模糊搜索的词
           "fuzziness": 1, # 编辑距离：从一个词边为另一个词需要增、删、改、交换的最少次数
           "prefix_length": 0 # 前面不参与编辑的长度
         }
      }
   }
}

# suggest查询
# completion suggest本身不支持纠错，但可以结合模糊匹配进行纠错功能实现
POST music/_search?pretty
{
    "suggest": {
         # suggest查询后返回的变量名称，可以任意指定
         "my-suggest" : {
            # "prefix" : "nor", # 要匹配的前面几个固定词
            "text": "value", # 要搜索的值
            "completion" : {
                "field" : "title-suggest", # 自己定义的suggest字段
                # 此处使用fuzzy模糊搜索
                "fuzzy" : {
                    "fuzziness" : 1
                }
            }
         }
    }
}
```







*分析器* 实际上是将三个功能封装到了一个包里：

**字符过滤器（**Character filters**）:**
他们的任务是在分词前整理字符串。一个字符过滤器可以用来去掉HTML，或者将 `&` 转化成 `and`。

**分词器（**Tokenizers**）:**
其次，字符串被 *分词器* 分为单个的词条。一个简单的分词器遇到空格和标点的时候，可能会将文本拆分成词条。

**词单元过滤器（**Token filters**）:**
最后，词条按顺序通过每个 *token 过滤器* 。这个过程可能会改变词条（例如，小写化 `Quick` ），删除词条（例如， 像 `a`， `and`， `the` 等无用词），或者增加词条（例如，像 `jump` 和 `leap` 这种同义词）。

自定义分析器：

```json
PUT /my_index
{
    "settings": {
      	# 索引设置 analysis 部分用来配置分析器
        "analysis": {
      		# 定义字符过滤器
            "char_filter": {
                "&_to_and": {
                    "type": "mapping", 
                    "mappings": [
                        "&=> and "
                    ]
                }
            }, 
			# 定义Token filter
            "filter": {
                "my_stopwords": {
                    "type": "stop", 
                    "stopwords": [
                        "the", 
                        "a"
                    ]
                }
            }, 
			# 设置分析器
            "analyzer": {
                "my_analyzer": {
                    "type": "custom", 
                    "char_filter": [
                        "html_strip", 
                        "&_to_and"
                    ], 
                  	# 分词器
                    "tokenizer": "standard", 
                    "filter": [
                        "lowercase", 
                        "my_stopwords"
                    ]
                }
            }
        }
    }
}
```


