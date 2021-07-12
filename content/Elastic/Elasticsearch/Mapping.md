---
title: "Mapping"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  elastic:
    parent: "Elasticsearch"
weight: 400
---
## 概述

- Mapping 定义了文档（document）及其包含的字段（filed）在存储和被索引时的过程和方式。
- 每个文档都是一组字段的集合，而每个字段也有自己的数据类型。Mapping 中不仅包含了与文档相关的字段列表，还包括元数据字段（例如_source），因此可以自定义文档相关元数据的处理方式。  
- Mapping 有 Dynamic（动态生成）和 Explicit（显式指定）两种实现方式，可根据字段的特点配合使用。

## Dynamic Mapping

我们只需创建一条文档并指定其所在索引，Dynamic Mapping 便会为我们自动生成索引、字段以及字段类型。

```json
PUT /my_index/_doc/first_doc
{
  "num": 5, 
  "text": "5"
}
```

查看该索引的 Mapping，我们发现字段值为数字的`num`字段的类型自动映射为 long 类型。而`text`字段映射为了 text 类型，其子字段`text.keyword`类型则为 keyword，可用于排序和聚合等操作（详见：[multi-fields](https://www.elastic.co/guide/en/elasticsearch/reference/current/multi-fields.html)）。

```json
GET /my_index/_mapping

// 返回
{
  "my_index" : {
    "mappings" : {
      "properties" : {
        "num" : {
          "type" : "long"
        },
        "text" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
    }
  }
}
```

但大多数情况下，我们可能希望值为"123"和"2021/02/26"这样的字段能够分别映射为 long 和 date 类型，这时便需要对 Dynamic Mapping 的方式进行自定义。

### Dynamic field mappings

当我们将`mappings.dynamic`属性设置为`true`或`runtime`后，便可以增加 Dynamic Mapping 的映射规则。值得一提的是，`runtime`这种 Mapping 方式仍处于 beta 测试中。

| Json 数据类型   | `mappings.dynamic: true`    | `mappings.dynamic: runtime`  |
| ------------------ | ---------------------------- | ---------------------------- |
| null               | 不添加该字段                 | 不添加该字段                 |
| true or false      | bool                         | bool                         |
| double             | float                        | double                       |
| integer            | long                         | long                         |
| object             | object                       | object                       |
| array              | 由数组第一个非空值的类型决定 | 由数组第一个非空值的类型决定 |
| 检测为日期的 string | date                         | date                         |
| 检测为数字的 string | float or long                | double or long               |
| 其他 string         | 带有 keyword 子字段的 text     | keyword                      |

对于数据类型为 object 的字段，在两种规则下都会将其包含的多个字段映射在`mappings.properties.<object-filed>.properties`中。我们以`mappings.dynamic: true`为例，检验 Dynamic Mapping 的处理方式。由于数字检测是默认关闭的（日期检测默认开启），因此还要先对索引的 Mapping 进行相关设置：

```json
PUT /my_index/
{
  "mappings": {
    "dynamic": true,
    "numeric_detection": true
  }
}
```

然后在不指定 Mapping 的情况下向索引中加入一个文档：

```json
PUT /my_index/_doc/1
{
  "one": null,
  "two": true,
  "three": 1.11111111111111111111,
  "four": 1,
  "five": {
    "one": "3",
    "two" : "true"
  },
  "six": "2021/02/26",
  "seven": "1",
  "eight" : "koktlzz"
}
```

随后即可查看该索引中各字段的类型：

```json
GET /my_index/_mapping

// 返回
{
  "my_index" : {
    "mappings" : {
      "dynamic" : "true",
      "numeric_detection" : true,
      "properties" : {
        "eight" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "five" : {
          "properties" : {
            "one" : {
              "type" : "long"
            },
            "two" : {
              "type" : "text",
              "fields" : {
                "keyword" : {
                  "type" : "keyword",
                  "ignore_above" : 256
                }
              }
            }
          }
        },
        "four" : {
          "type" : "long"
        },
        "seven" : {
          "type" : "long"
        },
        "six" : {
          "type" : "date",
          "format" : "yyyy/MM/dd HH:mm:ss||yyyy/MM/dd||epoch_millis"
        },
        "three" : {
          "type" : "float"
        },
        "two" : {
          "type" : "boolean"
        }
      }
    }
  }
}
```

### Dynamic template

我们还可以通过动态模板（Dynamic template）进一步地自定义索引字段的 Mapping 方式，而模板与字段的匹配规则有以下三种：

- `match_mapping_type`：匹配字段的数据类型；
- `match`和`unmatch`：匹配字段的名称；
- `path_match`和`path_unmatch`：匹配字段的全路径（如`<object>.*.<filed>`）。

一个典型的 Dynamic template 的基本配置如下：

```json
  "dynamic_templates": [
    {
      "dynamic_template_name": { 
        ...  match conditions ... // 使用上述三种规则匹配相关字段
        "mapping": { ... }        // 指定匹配字段的 Mapping 方式
      }
    },
    ...
  ]
```

我们使用一个复杂的模板为例，展示 Dynamic template 对字段的映射方式：

```json
PUT my_index
{
  "mappings": {
    "dynamic_templates": [{
      "longs_as_strings": {
        "match_mapping_type": "string",
        "match": "long_*",
        "unmatch": "*_text",
        "mapping": {
          "type": "long"
          }
        }
      }, {
      "full_name": {
        "path_match": "name.*",
        "path_unmatch": "*.middle",
        "mapping": {
          "type": "text"
        }
      }
    }]
  }
}
```

该模板规定了两种匹配规则：名称以"long"开头、不以"text"结尾且类型为 string 的字段将会被映射为 long 类型；`name`对象中除`middle`外的字段将会被映射为 text 类型：

```json
PUT my_index/_doc/1
{
  "long_num": "5", 
  "long_text": "foo",
  "name": {
    "first":  "John",
    "middle": "Winston"
  }
}

GET /my_index/_mapping
// 返回
{
  ...
  "properties" : {
    "long_num" : {
      "type" : "long"
    },
    "long_text" : {
      "type" : "text",
      "fields" : {
        "keyword" : {
          "type" : "keyword",
          "ignore_above" : 256
        }
      }
    },
    "name" : {
      "properties" : {
        "first" : {
          "type" : "text"
        },
        "middle" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
    }
  }
}
```

## Explicit Mapping

除了让 Elasticsearch 为我们动态生成 Mapping 外，我们当然也可以显式地指定索引中各字段的 Mapping 方式，如：

```json
PUT /my_index
{
  "mappings": {
    "properties": {
      "age":    { "type": "integer" },  
      "email":  { "type": "keyword"  }, 
      "name":   { "type": "text"  }     
    }
  }
}
```

对于一个已定义 Mapping 的索引，我们无法更新其中已存在的字段 Mapping 方式，但是可以增加字段。相比于创建，PUT 请求的 URL 多了一个"/_mapping"：

```json
PUT /my_index/_mapping
{
  "properties": {
    "employee-id": {
      "type": "keyword"
    }
  }
}
```

之前我们使用了 **GET /my_index/_mapping** 的方式查看索引中的 Mapping。而如果只想了解其中几个特定字段的 Mapping，可以这样做：

```json
GET /my_index/_mapping/field/employee-id

// 返回
{
  "my_index" : {
    "mappings" : {
      "employee-id" : {
        "full_name" : "employee-id",
        "mapping" : {
          "employee-id" : {
            "type" : "keyword"
          }
        }
      }
    }
  }
}
```

除了上述的基本数据类型外，Elasticsearch 还支持多种数据类型，详见 [官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html)。而除了通常用来存储数据的字段外，每个文档还包含一些 [元数据字段](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-fields.html)，如`_source`字段保存了文档内容的原始 JSON 文件。

## Mapping parameters

在字段的 Mapping 规则中，我们可以添加一些参数从而让字段的搜索结果更加符合我们的预期。详见 [官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-params.html#mapping-params)，此处介绍几个相对重要的 Mapping 参数：

- index：默认为 true，代表是否对该字段进行 [Text Analysis](/elastic/elasticsearch/textanalysis/)。若设置为 false，则会减少磁盘占用，不过该字段只能通过完全匹配搜索；
- enable：默认为 true，代表该字段是否被索引。若设置为 false，则完全不会被搜索到且不会影响文档的评分，只能在`_source`中找到该字段；
- norms：默认为 true，代表该字段是否参与评分。若设置为 false，则可以减少磁盘占用；
- doc_value：默认为 true。若设置为 false，则无法对该字段进行排序、聚合和脚本访问等操作；
- store：默认为 false，代表数据的存储方式，可以将经常被搜索且较小字段的 store 参数设置为 true 来单独存储。相比从`_source`中搜索，这样设置可以减少 IO 操作，从而提高效率；

### fielddata

默认情况下 text 类型的字段是可搜索的，但是如果用于聚合、排序以及脚本访问，Elasticsearch 将会返回：

> "reason": "Fielddata is disabled on text fields by default. Set fielddata=true on [address] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory. Alternatively use a keyword field instead."

与常用的搜索操作不同，排序和聚合需要在文档的字段中找到指定的值。这些操作与 Elasticsearch 倒排索引的逻辑（根据字段值查找文档）恰好相反，因此必须在内存中加载相关的数据结构。实现 text 类型字段的聚合操作，需要在字段设置中开启`fielddata: true`，这将占用大量堆内存。

text 类型的字段开启 fileddata 后聚合的结果，与通过 multi-fields 进行聚合有所不同，区别在于是否进行分词上。

fielddata：

```json
GET myindex/_search
{
  "size": 0,
  "aggs": {
    "aggr_mame": {
      "terms": {
        "field": "address",
        "size": 5
      }
    }
  }
}

// 返回
...
"aggregations" : {
  "aggr_mame" : {
    "doc_count_error_upper_bound" : 0,
    "sum_other_doc_count" : 0,
    "buckets" : [
      {
        "key" : "new",
        "doc_count" : 1
      },
      {
        "key" : "york",
        "doc_count" : 1
      }
    ]
  }
}
```

multi-fields：

```json
GET myindex/_search
{
  "size": 0,
  "aggs": {
    "aggr_mame": {
      "terms": {
        "field": "address.keyword",
        "size": 5
      }
    }
  }
}

// 返回
...
"aggregations" : {
    "aggr_mame" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "New York",
          "doc_count" : 1
        }
      ]
    }
  }
```
