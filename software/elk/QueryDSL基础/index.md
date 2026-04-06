---
title: "QueryDSL基础"
date: 2026-04-04T19:37:08
summary: ""
---

## 目录

[TOC]

---

## 前言

ES 的 QueryDSL 是基于 JSON 格式的领域特定语言（Domain Specific Language）。提供了灵活全面的查询方式。

本文从基础部分开始，主要围绕常用的几个概念：

1. Query & Filter Context
2. Compound Queries (bool)
3. Term-level Queries
4. Full Text Queries
5. minimum\_should\_match

---

## 测试数据

使用最简单的学生信息模型（索引名称："study-student"）来演示后文提到所有查询语句，mapping 如下：

```python
mapping = {
    "mappings": {
        "dynamic": "strict",  # 禁止动态字段
        "properties": {
            # 基础信息
            "student_id": {"type": "keyword"},
            "name": {
                "type": "text",
                "fields": {
                    "raw": {"type": "keyword"}
                }
            },
            "gender": {"type": "keyword"},
            "age": {"type": "integer"},
            "birthday": {"type": "date"},

            # 联系信息
            "email": {"type": "keyword"},
            "phone": {"type": "keyword"},

            # 地址（嵌套对象）
            "address": {
                "type": "object",
                "properties": {
                    "country": {"type": "keyword"},
                    "city": {"type": "keyword"},
                    "street": {"type": "text"}
                }
            },

            # 学校信息
            "enrollment_date": {"type": "date"},
            "graduation_date": {"type": "date"},
            "is_active": {"type": "boolean"},

            # 成绩（嵌套 + 数值）
            "gpa": {"type": "float"},
            "scores": {
                "type": "object",
                "properties": {
                    "math": {"type": "float"},
                    "english": {"type": "float"},
                    "science": {"type": "float"}
                }
            },

            # 标签（数组）
            "tags": {"type": "keyword"},

            # 课程（nested）
            "courses": {
                "type": "nested",
                "properties": {
                    "course_name": {"type": "keyword"},
                    "score": {"type": "float"},
                    "teacher": {"type": "keyword"}
                }
            },

            # 时间
            "created_at": {"type": "date"},
            "updated_at": {"type": "date"},

            # 备注（全文）
            "remark": {"type": "text"}
        }
    }
}
```

创建索引并且插入样例数据：

```python
def create_index_and_insert():

    if es_client.indices.exists(index=index_name):
        es_client.indices.delete(index=index_name)

    es_client.indices.create(index=index_name, body=mapping)

    def bulk_insert(n=1000):
        actions = []

        for _ in range(n):
            student = generate_student()
            actions.append({
                "_index": index_name,
                "_id": student["student_id"],
                "_source": student
            })

        helpers.bulk(es_client, actions)

    bulk_insert(1000)
```

---

## Query & Filter Context

ES 区别于 MySQL、PG 这些关系型数据库的一个最大特点就是基于倒排索引得到的分数（匹配程度），关系型数据库重点是精确匹配，ES 则是通过分词和倒排来对文档进行打分排序，得到一种模糊的相关性。

默认情况下，Elasticsearch 会按相关性评分（relevance score）对匹配的搜索结果进行排序,该分数用于测量每个文档匹配查询的程度。

对于是否进行打分，查询中有两种上下文（这也是关系型数据库没有的概念），一种叫Filter Context，另一种叫 Query Context。

### Filter Context

Filter 就是精确匹配，找出匹配查询子句的文档，当我们仅仅需要精确的匹配文档，而不需要影响文档相关性的时候，就使用 Filter Context，往往在高度结构化的文档集合中使用的更多：

- 数字类型字段（integer、float）
- 日期和时间戳
- 布尔值
- keyword 字段
- geo-points、geo-shapes

高度结构化就是类似传统的关系型数据库的字段一样，非常适合精确的过滤匹配。

---

### Query Context

Query Context 下的查询字句会影响文档的相关性评分，它更多的侧重于回答“这个文档有多匹配该查询字句？”的问题。

Query Context 除了判断文档是否匹配外,查询子句还会计算`_score`元数据字段中的相关性评分。

---

## Term-level queries

介绍完两种上下文以后，首先看看和传统关系型数据库最接近的精确匹配，ES 也有精确匹配的概念，叫精确匹配查询（term-level query）。

精确匹配查询有很多种类，接下来按照使用频率依次介绍。

### term query

最常用的查询子句，用于对某个字段精确匹配，比如：

年龄等于 20 岁的学生：

```json
"term": {
    "age": 20
}

邮箱为`dawn40@example.net`的学生：

```json
"term": {
    "email": "dawn40@example.net"
}
```

注意：避免对 text 类型的字段使用 term，text 类型字段应该用 match query。

### range query

范围匹配，对于数值和时间类型的字段非常常用。

年龄大于等于 20 岁的学生：

```json
"range": {
    "age": {
        "gte": 20
    }
}
```

range 的 field 支持以下几个选项：

- gt：大于
- gte：大于等于
- lt：小于
- lte： 小于等于
- format：时间格式

range 也支持时间类型的范围匹配，比如学生的 birthday 是 date 类型，可以直接解析，找到生日在 1999-01-01 - 1999-12-31 之间的学生：

```json
"range": {
    "birthday": {
        "gte": "1999-01-01",
        "lte": "1999-12-31"
    }
}
```

### terms

terms 和 term 是一样的，区别在于 term 是精确匹配单值，而 terms 是精确匹配多值，类似于 SQL 中的 IN。

比如，找到生日在 这两天的学生：

```json
"terms": {
    "birthday": ["1999-08-03", "1999-07-07"]
}
```

### terms\_set

terms\_set 和 terms 是一样的，区别在于，terms\_set 可以指定匹配了多少数量，也就是说可以控制最小匹配数。

如果想要找到 tags 中至少包含 "", "", "" 这三个中两个 tags 的学生，可以使用 terms\_set：

```json
"terms_set": {
    "tags": {
        "terms": ["art", "sports", "coding"],
        "minimum_should_match_script": {"source": "2"}
    }
}
```

### exists

exists 是为了找到 ES 中有些没有被索引的字段，一个 doc 某个字段如果没有被索引，可能有以下可能情况：

- 该字段为 null 或者 []
- mapping 中，对字段设置 "index": false 以及 "doc\values": false
- 字段值的长度超过了映射中的 ignore\_above
- 字段值畸形,且在映射中定义了 ignore\_malformed

### prefix

prefix 按照前缀搜索，类似 SQL 里的 LIKE %search，找到名字为


### wildcard


### ids





---

## Compound queries - bool

---

## minimum\_should\_match

---

## Full text queries

---

## 参考

1. https://www.elastic.co/guide/en/elasticsearch/reference/8.19/query-dsl.html
