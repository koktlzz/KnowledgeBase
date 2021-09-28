---
title: "Template"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  observability:
    parent: "ElasticStack"
weight: 600
---

## 概述

- Template（模板）规定了 Elasticsearch 在创建索引时是如何对其进行配置的；

- 如果索引与多个索引模板匹配，则使用优先级最高的索引模板。 模板的`priority`（旧版本中为`order`) 字段定义了模板的优先级。
- 索引在创建时显式声明的配置优先级高于其匹配的模板中的配置。

## 分类

模板有两种类型：Index template（索引模板）和 Component templates（组件模板）。

- 组件模板是可以重用的构建组件，用于配置索引的 Mappings、Settings 和 Aliases（别名）。 组件模板还可以用来构造索引模板，但它们不会直接应用于一组索引。
- 索引模板既可以是一个包含组件模板的集合，也可以直接指定索引的 Mappings、Settings 和 Aliases。

## 应用

首先使用 Elasticsearch API 创建两个组件模板，它们分别对`@timestamp`和`ip_address`字段的 Mapping 方式进行了配置：

```json
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

PUT _component_template/other_component_template
{
  "template": {
    "mappings": {
      "properties": {
        "ip_address": {
          "type": "ip"
        }
      }
    }
  }
}
```

然后我们创建一个索引模板，它是上述两个组件模板的集合，同时还对索引的分片数、别名以及`host_name`字段的 Mapping 方式进行了相关配置：

```json
PUT _index_template/template_1
{
  "index_patterns": ["te*", "bar*"],       // 索引名称开头为 te 和 bar 的索引将会匹配该模板
  "template": {
    "settings": {
      "number_of_shards": 1
    },
    "mappings": {
      "properties": {
        "host_name": {
          "type": "keyword"
        }
    },
    "aliases": {
      "mydata": { }
    }
  },
  "priority": 200,
  "composed_of": ["component_template1", "other_component_template"],
  "version": 3,
  "_meta": {
    "description": "my custom"
  }
}
```

## 测试

simulation API 可以帮助我们测试索引模板的应用效果，主要有以下几种方式：

测试索引模板对某一类样式索引的配置效果。例如我们之前创建的索引模板 template_1 ，其`index_patterns`参数为`["te*", "bar*"]`，那么便可以使用这种方式来测试其对名称开头为"te"的索引的配置效果（索引 te-01 并不存在）：

```json
POST /_index_template/_simulate_index/te-000001
```

测试一个现有模板（template_1）的配置效果：

```json
POST /_index_template/_simulate/template_1
```

为了保证创建的索引模板符合预期，我们还可以模拟定义一个索引模板：

```json
POST /_index_template/_simulate
{
  "index_patterns": ["te*"],
  "template": {
    "settings" : {
        "index.number_of_shards" : 3
    }
  },
  "composed_of": ["component_template1", "other_component_template"]
}
```

返回结果：

```json
{
  "template" : {
    "settings" : {
      "index" : {
        "number_of_shards" : "3"    // 模板创建时显式声明的配置优先级高于组件模板中的配置
      }
    },
    "mappings" : {                  // 继承自组件模板
      "properties" : {   
        "@timestamp" : {
          "type" : "date"
        },
        "ip_address" : {
          "type" : "ip"
        }
      }
    },
    "aliases" : { }
  },
  "overlapping" : [
    {
      "name" : "template_1",
      "index_patterns" : [
        "bar*",
        "te*"
      ]
    }
  ]
}
```

我们预创建的索引模板匹配了名称开头为 te 的索引，而已存在的模板 template_1 也与之匹配，因此返回结果中出现了 overlapping 的相关信息。在模拟中，其他重叠模板的 priority 要低于预创建的索引模板。

## 生效

首先创建一个新索引 index-001:

```json
POST index-001/_doc
{
  "ip": "10.122.70.86"
}
```

然后发现该索引的分片数和副本数都为默认值 1，而`ip`字段被映射为了 text 类型：

```json
GET index-001
// 返回
{
  "index-001" : {
    "aliases" : { },
    "mappings" : {
      "properties" : {
        "ip" : {
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
    "settings" : {
        "number_of_shards" : "1",
        "number_of_replicas" : "1",
        ...
      }
    }
  }
}
```

接下来我们创建一个索引模板 my-template，与名称以"index-"开头的索引关联：

```json
PUT _index_template/my-template
{
  "index_patterns": ["index-*"],
  "template": {
    "settings": {
      "number_of_replicas": 3
    },
    "mappings": {
      "properties": {
        "ip": {
          "type": "ip"
        }
      }
    }
  }
}
```

再次查看 index-001，发现其副本数和`ip`字段的类型都没有改变，这说明新创建的索引模板对已存在的索引无效。而如果再创建一个新索引 index-002：

```json
POST index-002/_doc
{
  "ip": "10.122.70.89"
}
```

则会发现该索引自动与模板 my-template 关联，副本数为 3，`ip`字段也映射为了 ip 类型：

```json
GET index-002
// 返回
{
  "index-002" : {
    "mappings" : {
      "properties" : {
        "ip" : {
          "type" : "ip"
        }
      }
    },
    "settings" : {
        "number_of_shards" : "1",
        "number_of_replicas" : "3",
        ...
        }
      }
    }
  }
}
```

如果想要修改已存在的索引配置，需要通过更新其 settings 的方式：

```json
PUT /index-*/_settings
{
  "index" : {
    "number_of_replicas" : 2
  }
}
```

查看 index-001 和 index-002，可以发现两个索引的副本数都变成了 2。以 index-001 为例：

```json
GET index-001
// 返回
{
  "index-001" : {
    "mappings" : {
      "properties" : {
        "ip" : {
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
    "settings" : {
        "number_of_shards" : "1",
        "number_of_replicas" : "2",
        ...
      }
    }
  }
}
```
