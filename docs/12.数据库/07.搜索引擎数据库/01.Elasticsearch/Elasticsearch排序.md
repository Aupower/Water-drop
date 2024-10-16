---
title: Elasticsearch 排序
date: 2022-01-19 22:49:16
order: 07
categories:
  - 数据库
  - 搜索引擎数据库
  - Elasticsearch
tags:
  - 数据库
  - 搜索引擎数据库
  - Elasticsearch
  - 排序
permalink: /pages/f67801e5/
---

# Elasticsearch 排序

在 Elasticsearch 中，默认排序是**按照相关性的评分（\_score）**进行降序排序，也可以按照**字段的值排序**、**多级排序**、**多值字段排序、基于 geo（地理位置）排序以及自定义脚本排序**，除此之外，对于相关性的评分也可以用 rescore 二次、三次打分，它可以限定重新打分的窗口大小（window size），并针对作用范围内的文档修改其得分，从而达到精细化控制结果相关性的目的。

## 默认相关性排序

在 Elasticsearch 中，默认情况下，文档是按照相关性得分倒序排列的，其对应的相关性得分字段用 `_score` 来表示，它是浮点数类型，`_score` 评分越高，相关性越高。评分模型的选择可以通过 `similarity` 参数在映射中指定。

相似度算法可以按字段指定，只需在映射中为不同字段选定即可，如果要修改已有字段的相似度算法，只能通过为数据重新建立索引来达到目的。关于更多 es 相似度算法可以参考 [深入理解 es 相似度算法（相关性得分计算）](https://www.knowledgedict.com/tutorial/elasticsearch-similarity.html)。

### TF-IDF 模型

Elasticsearch 在 5.4 版本以前，text 类型的字段，默认采用基于 tf-idf 的向量空间模型。

在开始计算得分之时，Elasticsearch 使用了被搜索词条的频率以及它有多常见来影响得分。一个简短的解释是，**一个词条出现在某个文档中的次数越多，它就越相关；但是，如果该词条出现在不同的文档的次数越多，它就越不相关**。这一点被称为 TF-IDF，TF 是**词频**（term frequency），IDF 是**逆文档频率**（inverse document frequency）。

考虑给一篇文档打分的首要方式，是统计一个词条在文本中出现的次数。举个例子，如果在用户的区域搜索关于 Elasticsearch 的 get-together，用户希望频繁提及 Elasticsearch 的分组被优先展示出来。

```
"We will discuss Elasticsearch at the next Big Data group."
"Tuesday the Elasticsearch team will gather to answer questions about Elasticsearch."
```

第一个句子提到 Elasticsearch 一次，而第二个句子提到 Elasticsearch 两次，所以包含第二句话的文档应该比包含第一句话的文档拥有更高的得分。如果我们要按照数量来讨论，第一句话的词频（TF）是 1，而第二句话的词频将是 2。

逆文档频率比文档词频稍微复杂一点。这个听上去很酷炫的描述意味着，如果一个分词（通常是单词，但不一定是）在索引的不同文档中出现越多的次数，那么它就越不重要。使用如下例子更容易解释这一点。

```
"We use Elasticsearch to power the search for our website."
"The developers like Elasticsearch so far."
"The scoring of documents is calculated by the scoring formula."
```

如上述例子，需要理解以下几点：

- 词条 “Elasticsearch” 的文档频率是 2（因为它出现在两篇文档中）。文档频率的逆源自得分乘以 1/DF，这里 DF 是该词条的文档频率。这就意味着，由于词条拥有更高的文档频率，它的权重就会降低。
- 词条 “the” 的文档频率是 3，因为它出现在所有的三篇文档中。请注意，尽管 “the” 在最后一篇文档中出现了两次，它的文档频率还是 3。这是因为，逆文档频率只检查一个词条是否出现在某文档中，而不检查它出现多少次。那个应该是词频所关心的事情。

逆文档频率是一个重要的因素，用于平衡词条的词频。举个例子，考虑有一个用户搜索词条 “the score”，单词 the 几乎出现在每个普通的英语文本中，如果它不被均衡一下，单词 the 的频率要完全淹没单词 score 的频率。逆文档频率 IDF 均衡了 the 这种常见词的相关性影响，所以实际的相关性得分将会对查询的词条有一个更准确的描述。

一旦词频 TF 和逆文档频率 IDF 计算完成，就可以使用 TF-IDF 公式来计算文档的得分。

### BM25 模型

Elasticsearch 在 5.4 版本之后，针对 text 类型的字段，默认采用的是 BM25 评分模型，而不是基于 tf-idf 的向量空间模型，评分模型的选择可以通过 `similarity` 参数在映射中指定。

## 字段的值排序

在 Elasticsearch 中按照字段的值排序，可以利用 `sort` 参数实现。

```bash
GET books/_search
{
  "sort": {
    "price": {
      "order": "desc"
    }
  }
}
```

返回结果如下：

```json
{
  "took": 132,
  "timed_out": false,
  "_shards": {
    "total": 10,
    "successful": 10,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 749244,
    "max_score": null,
    "hits": [
      {
        "_index": "books",
        "_type": "book",
        "_id": "8456479",
        "_score": null,
        "_source": {
          "id": 8456479,
          "price": 1580.00,
          ...
        },
        "sort": [
          1580.00
        ]
      },
      ...
    ]
  }
}
```

从如上返回结果，可以看出，`max_score` 和 `_score` 字段都返回 `null`，返回字段多出 `sort` 字段，包含排序字段的分值。计算 \_`score` 的花销巨大，如果不根据相关性排序，记录 \_`score` 是没有意义的。如果无论如何都要计算 \_`score`，可以将 `track_scores` 参数设置为 `true`。

## 多字段排序

如果我们想要结合使用 price、date 和 \_score 进行查询，并且匹配的结果首先按照价格排序，然后按照日期排序，最后按照相关性排序，具体示例如下：

```bash
GET books/_search
{
	"query": {
		"bool": {
			"must": {
				"match": { "content": "java" }
			},
			"filter": {
				"term": { "user_id": 4868438 }
			}
		}
	},
	"sort": [{
			"price": {
				"order": "desc"
			}
		}, {
			"date": {
				"order": "desc"
			}
		}, {
			"_score": {
				"order": "desc"
			}
		}
	]
}
```

排序条件的顺序是很重要的。结果首先按第一个条件排序，仅当结果集的第一个 `sort` 值完全相同时才会按照第二个条件进行排序，以此类推。

多级排序并不一定包含 `_score`。你可以根据一些不同的字段进行排序，如地理距离或是脚本计算的特定值。

## 多值字段的排序

一种情形是字段有多个值的排序，需要记住这些值并没有固有的顺序；一个多值的字段仅仅是多个值的包装，这时应该选择哪个进行排序呢？

对于数字或日期，你可以将多值字段减为单值，这可以通过使用 `min`、`max`、`avg` 或是 `sum` 排序模式。例如你可以按照每个 date 字段中的最早日期进行排序，通过以下方法：

```json
"sort": {
  "dates": {
    "order": "asc",
    "mode":  "min"
  }
}
```

## 地理位置上的距离排序

es 的地理位置排序使用 **`_geo_distance`** 来进行距离排序，如下示例：

```json
{
  "sort" : [
    {
      "_geo_distance" : {
        "es_location_field" : [116.407526, 39.904030],
        "order" : "asc",
        "unit" : "km",
        "mode" : "min",
        "distance_type" : "plane"
      }
    }
  ],
  "query" : {
    ......
  }
}
```

_\_geo_distance_ 的选项具体如下：

- 如上的 _es_location_field_ 指的是 es 存储经纬度数据的字段名。
- **_`order`_**：指定按距离升序或降序，分别对应 **_`asc`_** 和 **_`desc`_**。
- **_`unit`_**：计算距离值的单位，默认是 **_`m`_**，表示米（meters），其它可选项有 **_`mi`_**、**_`cm`_**、**_`mm`_**、**_`NM`_**、**_`km`_**、**_`ft`_**、**_`yd`_** 和 **_`in`_**。
- **_`mode`_**：针对数组数据（多个值）时，指定的取值模式，可选值有 **_`min`_**、**_`max`_**、**_`sum`_**、**_`avg`_** 和 **_`median`_**，当排序采用升序时，默认为 _min_；排序采用降序时，默认为 _max_。
- **_`distance_type`_**：用来设置如何计算距离，它的可选项有 **_`sloppy_arc`_**、**_`arc`_** 和 **_`plane`_**，默认为 _sloppy_arc_，_arc_ 它相对更精确些，但速度会明显下降，_plane_ 则是计算快，但是长距离计算相对不准确。
- **_`ignore_unmapped`_**：未映射字段时，是否忽略处理，可选项有 **_`true`_** 和 **_`false`_**；默认为 _false_，表示如果未映射字段，查询将引发异常；若设置 _true_，将忽略未映射的字段，并且不匹配此查询的任何文档。
- **_`validation_method`_**：指定检验经纬度数据的方式，可选项有 **_`IGNORE_MALFORMED`_**、**_`COERCE`_** 和 **_`STRICT`_**；_IGNORE_MALFORMED_ 表示可接受纬度或经度无效的地理点，即忽略数据；_COERCE_ 表示另外尝试并推断正确的地理坐标；_STRICT_ 为默认值，表示遇到不正确的地理坐标直接抛出异常。

## 参考资料

- [Elasticsearch 教程](https://www.knowledgedict.com/tutorial/elasticsearch-intro.html)