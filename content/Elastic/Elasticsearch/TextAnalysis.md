---
title: "Text analysis"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  elastic:
    parent: "Elasticsearch"
weight: 500
---

## 概述

Text analysis（分词）使 Elasticsearch 在执行全文搜索时，不仅可以精确匹配搜索项，还能够返回与其相关的所有结果。 举例来说，如果一个索引中有以下几个文档：

- A quick brown fox jumps over the lazy dog
- fast fox
- foxes leap

当我们搜索`Quick fox jumps`，我们很可能希望搜索结果中包含了上述文档，因为它们都与搜索项有着密切的关系：包含、复数和同义词。这便是分词器存在的价值，它让 Elasticsearch 的搜索结果更加符合使用者的预期。

## 组成

在 Elasticsearch 中可以通过内置分词器实现分词，也可以按需定制分词器。 分词器由三部分组成：

### Character Filters

Character Filters 将原始文本作为字符流接受并进行处理，它的作用是整理字符串。例如将印度-阿拉伯数字 （٠‎١٢٣٤٥٦٧٨‎٩‎）转化为阿拉伯-拉丁数字（0123456789）、去除 HTML 元素、将&转化为 and 等。一个分词器可以有 0 个或多个 Character Filters，多个 Character Filters 的加载是有顺序的。

### Tokenizer

Tokenizer 将字符流按照给定的规则切分为 Token（可以看作为单词）。比如 whitespace 分词器在遇到空格和标点的时候，会将文本进行拆分。同时，Tokenizer 还会记录每个 Token 在文本中的位置（position）、占位长度（positionLength）以及首尾字符的偏移量（start_offset/end_offset）。下图中每个 Token 的 positionLength 都为 1，position 分别为 0、1 和 2：

![截屏 2021-03-04 上午 12.24.56](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/%E6%88%AA%E5%B1%8F2021-03-04%20%E4%B8%8A%E5%8D%8812.24.56.png)

与 Character Filters 不同的是，每个分词器有且仅有一个 Tokenizer。

### Token Filters

Token Filters 接收切分后的 Token 流并对 Token 进行加工，如小写（lowercase token filter），删除 a，and 和 the 等 stopwords（stop token filter），增加同义词（synonym token filter），[词根处理 (Stemming)](https://www.elastic.co/guide/en/elasticsearch/reference/current/stemming.html) 等。Token Filters 并不会改变 Token 的位置，也不会改变其字符的偏移量。如下图所示，quick 和它的同义词 fast 具有相同的 position 和 positionLength：

![截屏 2021-03-04 上午 12.30.14](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/%E6%88%AA%E5%B1%8F2021-03-04%20%E4%B8%8A%E5%8D%8812.30.14.png)

而对于 domain name system 和它的的同义词 dns 来说，它们的 postion 都为 0，但 positionLength 分别为 1 和 3:

![1](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/%E6%88%AA%E5%B1%8F2021-03-04%20%E4%B8%8A%E5%8D%8812.50.50.png)

一个分词器可以有 0 个或多个 Token Filters，多个 Token Filters 的加载是有顺序的。

## 应用场景

分词器的应用场景有两个：文档被索引时（Index time）和文档被搜索时（Search time）。

- Index time：当一个文档被索引时，所有类型为 text 的字段的值都会被分词；
- Search time：当对类型为 text 的字段进行全文搜索时，搜索项将被分词。

因此，我们可以根据分词器应用场景的不同，将其分为索引分词器（index analyzer）和搜索分词器（search analyzer）两种。在大多数情况下，应当在索引和搜索时使用同样的分词器。这样可以保证索引中的字段值和查询项被分词为相同形式的 Token，从而使查询结果更加符合我们的预期。

## 配置

### 测试分词器

可以使用 [analyze API](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-analyze.html) 来测试分词器对文本的处理情况，以一个内置的分词为例：

```json
POST _analyze
{
  "analyzer": "whitespace",
  "text": "The quick brown fox."
}

// 返回
{
  "tokens": [
    {
      "token": "The",
      "start_offset": 0,
      "end_offset": 3,
      "type": "word",
      "position": 0
    },
    {
      "token": "quick",
      "start_offset": 4,
      "end_offset": 9,
      "type": "word",
      "position": 1
    },
    {
      "token": "brown",
      "start_offset": 10,
      "end_offset": 15,
      "type": "word",
      "position": 2
    },
    {
      "token": "fox.",
      "start_offset": 16,
      "end_offset": 20,
      "type": "word",
      "position": 3
    }
  ]
}
```

当然也可以测试 0 个或多个 Character Filters、一个 Tokenizer 和 0 个或多个 Token Filters 的组合效果：

```json
POST _analyze
{
  "tokenizer": "standard",
  "filter":  [ "lowercase", "asciifolding" ],
  "text":      "Is this déja vu?"
}
```

### 配置内置分词器

我们可以对内置分词进行修改，从而让其实现我们想要的分词效果。例如，创建一个基于 [standard analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-standard-analyzer.html) 的分词器 std_english，它可以在分词时删除英文中的 a，and 和 the 等 stopwords：

```json
PUT my-index-000001
{
  "settings": {
    "analysis": {
      "analyzer": {
        "std_english": { 
          "type":      "standard",   // type 为 standard（或 simple、Whitespace 等）表示基于内置的分词器
          "stopwords": "_english_"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "my_text": {
        "type":     "text",
        "analyzer": "standard", 
        "fields": {
          "english": {
            "type":     "text",
            "analyzer": "std_english" 
          }
        }
      }
    }
  }
}
```

索引的 Mapping 中规定，对`my_text`字段使用 standard analyzer 分词，而`my_text.english`字段则使用 std_english。通过 analyze API 进行测试：

```json
POST my-index-000001/_analyze
{
  "field": "my_text", 
  "text": "The old brown cow"
}

POST my-index-000001/_analyze
{
  "field": "my_text.english", 
  "text": "The old brown cow"
}
```

### 创建自定义分词器

自定义分词器和所有分词器一样，都拥有一个 Tokenizer，0 个或多个 Character Filters 和 0 个或多个 Token Filters，对应的字段分别为`tokenizer`、`char_filter`和`filter`。除此之外，还可以通过定义 [`position_increment_gap`](https://www.elastic.co/guide/en/elasticsearch/reference/current/position-increment-gap.html) 字段来确保查询项是一个词组时不会与不同数组中的两个元素发生匹配。例如：

```json
PUT my-index-000001
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer": {
          "type": "custom", 
          "tokenizer": "standard",
          "char_filter": [
            "html_strip"
          ],
          "filter": [
            "lowercase",
            "asciifolding"
          ]
        }
      }
    }
  }
}

POST my-index-000001/_analyze
{
  "analyzer": "my_custom_analyzer",
  "text": "Is this <b>déjà vu</b>?"
}
```

文本`Is this <b>déjà vu</b>?`在索引到文档中后将会变成：`[ is, this, deja, vu ]`。

### 指定分词器

上文提到，根据应用场景的不同，我们将分词器分为了索引分词器（index analyzer）和搜索分词器（search analyzer）。Elasticsearch 在确定需要使用的索引分词器前会依次检查以下参数，

- 索引中某一 text 字段的 [`analyzer`](https://www.elastic.co/guide/en/elasticsearch/reference/current/analyzer.html) 参数

```json
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "whitespace"
      }
    }
  }
}
```

- 索引的`settings.analysis.analyzer.default`参数

```json
PUT my-index-000001
{
  "settings": {
    "analysis": {
      "analyzer": {
        "default": {
          "type": "simple"
        }
      }
    }
  }
}
```

- 若上述参数均未指定，则使用内置的 standard analyzer。

而对于搜索分词器的指定，Elasticsearch 则需要依次检查这些参数：

- 搜索请求中的`analyzer`参数

```json
GET my-index-000001/_search
{
  "query": {
    "match": {
      "message": {
        "query": "Quick foxes",
        "analyzer": "stop"
      }
    }
  }
}
```

- 索引中某一 text 字段的`search_analyzer`参数

```json
PUT my-index-000001
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "whitespace",      // 索引时使用 whitespace 分词
        "search_analyzer": "simple"    // 搜索时使用 simple 分词
      }
    }
  }
}
```

- 索引的`settings.analysis.analyzer.default_search`参数：

```json
PUT my-index-000001
{
  "settings": {
    "analysis": {
      "analyzer": {
        "default": {
          "type": "simple"          // 索引时使用 simple 分词
        },
        "default_search": {
          "type": "whitespace"      // 搜索时使用 whitespace 分词
        }
      }
    }
  }
}
```

- 若上述参数均未指定，则使用内置的 standard analyzer。
