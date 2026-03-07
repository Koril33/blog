---
title: "Elasticsearch的基本概念"
date: 2026-03-04T09:48:44
summary: "Elasticsearch 的历史，解决方案和各个组件的概念"
---

## 目录

[TOC]

---

## Apache Lucene

在介绍 Elasticsearch 前，必须先提一笔 Apache Lucene 这个项目，如果说 Elasticsearch 是辆功能完备，上手就能开起来的汽车，那么 Lucene 就是汽车里的发动机引擎。

### 历史

Doug Cutting 最初于 1999 年编写了 Lucene。他之前在施乐公司写过两个搜索引擎，在苹果公司写过一个，在Excite上写了第四个，Lucene 是他写的第五个搜索引擎。

最初 Lucene 放在 SourceForge 网站上提供下载。它于 2001 年 9 月加入了 Apache 软件基金会的雅加达开源 Java 系列产品，并于 2005 年 2 月成为顶级 Apache 项目。

Lucene 这个名字是 Doug Cutting 妻子的中间名，也是她外祖母的名字。

### 功能

Lucene 是一个 Java 编写的底层搜索库，它提供了高性能的倒排索引和全文搜索的功能，还支持相关性评分，模糊查询，分词器，过滤器等。

### 场景

Lucene 适用于任何需要全文索引和搜索功能的应用，尤其是互联网搜索引擎和站点搜索，它提供了相关性评分的功能，也可以用作实时推荐系统。

### 和 ES 的关系

Apache Lucene 作为底层库，提供了索引和搜索算法以及管理文件存储结构，在使用它时，需要自己编写 Java 代码调用 Lucene。

而 Elasticsearch 则是开箱即用，不需要编写代码就可以通过 HTTP API 使用，因为 Elasticsearch 是以 Lucene 为基础，额外提供了分布式（分片，副本，集群管理，Master选举）和 RESTful API 的能力。

它们俩的关系有点类似于 MySQL Server 和 InnoDB。

---

## 搜索

简单介绍了下 Lucene 的背景，接下来要梳理的第一个概念——搜索。Lucene 是一个 Java 的全文搜索引擎，那么我们在谈论搜索的时候，在谈些什么？

搜索的本质是在数据中，找到相关的结果，衡量一个搜索模块的好坏，很自然的就是两个要素：

1. 性能
2. 相关性

好的搜索模块，能够在近乎实时的情况下，检索到最相关的数据。在服务器端的开发中，我们常见的搜索有文本搜索，关系型数据库里的搜索，全文搜索数据库（也就是本文提到的 ES）中的搜索，关于图片，音视频等多媒体数据的搜索。

### 文本搜索

比如 Linux 下的 grep、find 之类的工具和各个语言内置的字符匹配，就是在完成文本类的搜索，面向的是非结构化的文本结构，从 KMP 到 Boyer-Moore，有非常多的字符串搜索算法。

### 关系型数据库搜索

和文本搜索不一样，关系型数据库处理的是结构化的数据，在处理精确匹配或者前缀匹配时，关系型数据库提供了索引搜索的工具，比如：B-Tree索引，哈希索引等，但在处理开头是通配符的模糊匹配（LIKE %keyword%）时，会退化成文本搜索这种线性搜索。

添加索引的主要思想就是空间换时间，再插入和更新数据时建立好索引（空间），有效的减少了后续查询的耗时（时间）。

### 全文搜索

对于高度结构化，并且存在复杂关联关系的数据，关系型数据库擅长的是索引字段的精确匹配、范围匹配，以及数据的排序和聚合。但是对于非结构化的数据的模糊匹配，性能就会很糟糕了。

比如博客站点的搜索，用户输入“数据库”，目的是为了查看哪些博客提到了“数据库”，如果是传统的关系型数据库，用`LIKE %数据库%`这种模糊匹配来实现，就意味着全表扫描，检索时间随着博客数量增大而增大。

另外，如果用户搜索的是“中国数据库的历史和发展”，关系型数据库的模糊匹配做不到分词和相关性打分。

全文搜索引擎通过分词器、建立倒排索引、相关性打分等多种技术实现了真正意义上的全文搜索。

### 特征向量搜索

在进行图片、视频、音频等多媒体类型的搜索的时候，传统的文本匹配的思路就完全行不通了，因为比较两个图片是否相似，并不能用底层存储的二进制进行匹配，因为光线的细微变化，也会导致像素层面数据发生巨大变化，还要考虑压缩算法的影响。

以图片为例，计算机层面上，本质是二维数组，这些数值描述了图片的像素，色彩，亮度等等，但是用这些信息无法有效的给用户提供搜索接口，为了实现图片的搜索，首先是要让计算机“理解”图片：通过神经网络等方式提取图片的特征，得到图片特征的高维向量。

高维向量类似于一个数组：`[0.31, 0.12, 0.15, 0.98, ...]`，比较两个图片的相似性，就是在比较两个图片提取出来的高维特征向量的相似性。这种相似性可以用余弦相似度或者欧式距离来判定。

特征向量搜索和传统关系型数据库搜索同样面临数据量极其庞大的问题，但是解决思路完全不同，特征向量搜索引入了近似最近邻搜索（Approximate Nearest Neighbor Search, ANN）的概念：是在大规模的高维数据中快速找到与查询向量最相似的K个向量的技术。它牺牲少量精度以显著提升检索速度。

### 搜索小结

这一节非常粗略的过了一遍各种类型的搜索，从最初的文本匹配到传统数据库的查询，再到倒排索引支撑的全文索引，最后涉及到了现在大模型时代下的特征向量搜索。

接下来的重点将聚焦于 Elasticsearch 中全文索引和分布式的概念。

---

## 数据存储

### 索引

索引（index）是 ES 中的核心概念，它提供了一个逻辑命名空间负责存储具有相似结构（mapping）的文档（document）。

可以近似看作是关系型数据库里的 database，当然，二者其实本质完全不同。

ES 的 index 有一个或者多个分片（shard）组成，ES 的一个 shard 相当于一个 lucene index，lucene index 由多个 lucene segment 组成。

可以通过 HTTP API 来查看索引：

```shell
test@debian-elk:~$ curl -s -k -u elastic:WrFx5aWjon-fP5O_VZxo https://localhost:9200/_cat/indices
green  open .internal.alerts-transform.health.alerts-default-000001            zt-zPUtVSbWlfNJSflQFVQ 1 0  0 0   249b   249b   249b
green  open .internal.alerts-observability.logs.alerts-default-000001          YaiBdGMnRD-jM-H2zGjRkQ 1 0  0 0   249b   249b   249b
green  open .internal.alerts-observability.uptime.alerts-default-000001        M27J5shAQSuJ-0cDUMHF3g 1 0  0 0   249b   249b   249b
green  open .internal.alerts-ml.anomaly-detection.alerts-default-000001        Bk8vzqCKT1qKTo2b_RBjDg 1 0  0 0   249b   249b   249b
green  open .internal.alerts-observability.slo.alerts-default-000001           8-a0_067RNyXMlnROrVpsQ 1 0  0 0   249b   249b   249b
green  open .internal.alerts-default.alerts-default-000001                     PWVtp1j5SjOThaW4HY4mbA 1 0  0 0   249b   249b   249b
green  open .internal.alerts-streams.alerts-default-000001                     ijMvcvZlQeyJjqaxmMHKOw 1 0  0 0   249b   249b   249b
green  open .internal.alerts-observability.apm.alerts-default-000001           JYQKcVFpSW2BUOBNvfGx0A 1 0  0 0   249b   249b   249b
green  open .internal.alerts-security.attack.discovery.alerts-default-000001   u1yRr9kcQ96vmVLtB6iydg 1 0  0 0   249b   249b   249b
green  open .internal.alerts-observability.metrics.alerts-default-000001       -09t5ZnFRgadQ8oEMEq9nw 1 0  0 0   249b   249b   249b
yellow open books                                                              V3LVxRIrTc-T_WhH4mv-eg 1 1  6 0 12.9kb 12.9kb 12.9kb
green  open .internal.alerts-ml.anomaly-detection-health.alerts-default-000001 8DIDuBFRT3Kgjs1Biwv8PQ 1 0  0 0   249b   249b   249b
green  open .internal.alerts-observability.threshold.alerts-default-000001     hllXRv7iS8eOquDoNW0Uow 1 0  0 0   249b   249b   249b
yellow open .ds-filebeat-8.19.12-2026.02.27-000001                             nFqhO4zrQEumnqUV7gGktA 1 1 52 0  221kb  221kb  221kb
green  open .internal.alerts-security.alerts-default-000001                    LMJ2-LDIRWCF_Jiap-8y9A 1 0  0 0   249b   249b   249b
green  open .internal.alerts-dataset.quality.alerts-default-000001             0bYyujIjTF6Adl0z-BIaGA 1 0  0 0   249b   249b   249b
green  open .internal.alerts-stack.alerts-default-000001                       _G_ixR1LQCmACeC2EHBuJA 1 0  0 0   249b   249b   249b
test@debian-elk:~$
```

这里我已经创建了一个 books（其他的以 . 开头的都是系统 index）索引。在文件系统层面，物理存储的位置在`/var/lib/elasticsearch/indices`，books 索引位于`/var/lib/elasticsearch/indices/V3LVxRIrTc-T_WhH4mv-eg`。

### 文档

在 ES 中，文档（Document）是可以被索引和搜索的最小数据单位。它是 JSON 格式 的对象，例如：

```json
{
	"_index": "books",
	"_id": "kxJvrJwBSJ9eJX4lIyvA",
	"_score": 1,
	"_source": {
	  "name": "Brave New World",
	  "author": "Aldous Huxley",
	  "release_date": "1932-06-01",
	  "page_count": 268
	}
}
```

每一个文档都拥有一个唯一键`_id`，可以自己生成或者让 ES 自动生成。

文档和索引的关系是：一个索引可以包含多个文档，索引的本质就是多个文档的逻辑集合：

```
Index: books
 ├─ Document: {"title":"Book A","author":"Alice"}
 ├─ Document: {"title":"Book B","author":"Bob"}
 └─ Document: {"title":"Book C","author":"Carol"}
```

文档包含数据（data）和元数据（metadata）。

元数据字段是系统字段，用于存储文档信息。在 Elasticsearch 中，元数据字段以一个下划线为前缀。例如，上面的`_id`（文档的 ID，每个索引的 ID 必须唯一）和`_index`（文档存储的索引名称）都是元数据。


### 映射

如果把索引比作数据库，文档比作行记录，那么映射（Mapping）可以类比表结构（Schema）。Mapping 定义了文档字段的类型和索引方式。


ES 的 Mapping 按照创建类型可以分成：

- Dynamic Mapping：动态映射
- Explicit mapping：显式映射

传统的关系型数据库没有动态映射这一说，因为每一张表的建立都需要定义好表的模式（Schema）。

ES 的显式映射就是在创建索引的时候，预先定义映射，为每一个字段指定数据类型，在生产环境建议使用显式映射，因为这样定义精确，可以完全控制索引方式。

ES 还提供了动态映射，假设创建索引时没有定义映射，ES 会根据插入的文档来自动检测数据类型并创建映射。

### 数据类型

在创建映射的过程中，需要给每个字段指定数据类型，数据类型对于 ES 非常关键，因为不同的数据类型在底层存储和创建的索引结构是不一样的。

除了常见的 number、date、boolean，对于全文搜索而言，比较关键的两个文本数据类型是 keyword 和 text。

#### text

text 类型的数据会经过分词器（analyzer）分词，然后建立倒排索引。它不支持精确查询，常用于索引非结构化文本字段，例如电子邮件的正文或产品描述。

这些非结构化的字段会被分词器分成单个单词，然后进行索引。

分词和索引的能力，使得 ES 可以在每个全文字段中搜索单个单词。text 字段不用于排序，很少用于聚合。

#### keyword

keyword 类型的数据不会经过分词器，它常用于排序、聚合和精确查询（Term query），适合结构化数据。

ID、电子邮件地址、主机名、状态码、邮政编码或标签等结构化内容可以使用 keyword 类型存储。

### 倒排索引



### 分词器


---

## 分布式

---

## 参考

1. https://github.com/elastic/elasticsearch
2. https://lucene.apache.org/
3. https://en.wikipedia.org/wiki/Apache_Lucene
4. https://www.elastic.co/guide/en/elasticsearch/reference/8.19/text.html
5. https://www.elastic.co/guide/en/elasticsearch/reference/8.19/keyword.html

