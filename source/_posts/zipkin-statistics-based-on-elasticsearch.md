---
title: '基于 Elasticsearch 的 Zipkin 统计'
date: 2017-01-03 07:19:50
permalink: zipkin-statistics-based-on-elasticsearch
---

最近将 Zipkin 的底层存储切换到了 Elasticsearch，相比 Cassandra，Elasticsearch 拥有更加灵活的查询和聚合方式，所以可以完成一些之前做不到的自定义统计，在此记录一下。

## 存储结构

Zipkin 的存储是基于 Span 的，每一个 Span 为一个文档，字段有：

- `traceId`：所属的 Trace
- `duration`：执行消耗的时间（单位为微秒）
- `timestamp`、`timestamp_millis`：生成时间（单位分别为微秒和毫秒）
- `name`：Span 的名称
- `annotations`：主要用于记录系统通信的情况，有 `Client Send`、`Server Receive`、`Server Send` 和 `Client Receive` 四种通信记录
- `binaryAnnotations`：记录一些自定义数据，可以记录任何关心的数据，例如方法参数、SQL 语句、Server 端地址等
- `parentId`：父 Span
- `collector_timestamp_millis`：采集时间

 使用 Zipkin UI 时 `name`、`binaryAnnotations` 都仅作为展示字段，而在 Elasticsearch 中，通过良好的方式定义这些字段，可以完成一些横向的查询和聚合。

 理论上如果定义得当，最多可以通过 `name`、`binaryAnnotations.key` 和 `binaryAnnotations.value` 进行三层的聚合，这里举几个例子。

## Span 定义与查询

### HTTP/RPC API 的 Span 定义：

 将 Endpoint 作为 `name`，然后将 Header、Param 等信息作为 `binaryAnnotations`，例如：

```
{
  "traceId": "51c068d64b9f7a58",
  "duration": 1330683,
  "binaryAnnotations": [
    {
      "endpoint": {
        "ipv4": "10.xxx.xxx.xxx",
        "port": 8088,
        "serviceName": "user-service"
      },
      "value": "[user-agent=okhttp/3.2.0]",
      "key": "Headers"
    },
    {
      "endpoint": {
        "ipv4": "10.xxx.xxx.xxx",
        "port": 8088,
        "serviceName": "user-service"
      },
      "value": "[id=100001]",
      "key": "Params"
    },
    {
      "endpoint": {
        "ipv4": "10.xxx.xxx.xxx",
        "port": 8088,
        "serviceName": "user-service"
      },
      "value": "/v1/users/{id}.json",
      "key": "lc"
    }
  ],
  "timestamp_millis": 1483143424117,
  "name": "/v1/users/{id}.json",
  "collector_timestamp_millis": "2016-12-31T00:17:05.808+0000",
  "id": "2987302a57b7212f",
  "parentId": "51c068d64b9f7a58",
  "timestamp": 1483143424117000
}
```

 这样可以通过如下的查询语句统计一下当前系统 HTTP API 的性能情况：

```
{
  "size": 0,
  "query": {
    "bool": {
      "must": [
        {
          "match_all": {}
        }
      ],
      "filter": {
        "nested": {
          "path": "binaryAnnotations",
          "query": {
            "match": {
              "binaryAnnotations.key": "Params"
            }
          }
        }
      }
    }
  },
  "aggs": {
    "path": {
      "terms": {
        "field": "name",
        "order": {
          "avg_duration": "desc"
        },
        "size": 20,
        "min_doc_count": 100
      },
      "aggs": {
        "avg_duration": {
          "avg": {
            "field": "duration"
          }
        },
        "top_duration_trace": {
          "top_hits": {
            "sort": {
            "duration": {
                "order": "desc"
              }
            },
            "_source": {
              "includes": [
                  "traceId",
                  "duration"
              ]
            },
            "size" : 3
          }
        }
      }
    }
  }
}
```

 该语句首先查出了所有 API 入口的 Span，聚合出平均响应时间最长的 20 个 API，并给出每个 API 中响应时间最长的 3 个 Trace。

 这种结构有一点不好的是没办法进一步通过参数进行聚合了，如果一个接口在请求参数不同时响应时间差别比较大，就没有办法找出具体是哪些参数影响的了。将每一个参数都加入 `binaryAnnotations` 中并不是一个很好的做法，因为会增加许多无意义的嵌套文档。如果能像 RESTful 一样将一些重要参数定义在 URI 路径上也许能解决这个问题。

### MySQL/Redis 等数据库查询的 Span 定义：

 建议将 `name` 设置为命令名，像是 `mysql-select`、`redis-get` 等。之后便可以分析在每一个 Trace 中，SQL 查询或是 Redis 查询发生的次数以及总耗时。

```
{
  "size": 0,
  "query": {
    "bool": {
      "must": [
        {
          "match_all": {}
        }
      ],
      "filter": {
        "term": {
          "traceId": "3fa0711deeb82b11"
        }
      }
    }
  },
  "aggs": {
    "span_name": {
      "terms": {
        "field": "name",
        "order": {
          "sum_duration": "desc"
        }
      },
      "aggs": {
        "sum_duration": {
          "sum": {
            "field": "duration"
          }
        }
      }
    }
  }
}
```

 上面的查询语句通过聚合一个 Trace 的所有 Span，根据类型计算总耗时并排序，它的返回结果是：

```
{
  "span_name": {
    "doc_count_error_upper_bound": 0,
    "sum_other_doc_count": 0,
    "buckets": [
      {
        "key": "mysql-insert",
        "doc_count": 200,
        "sum_duration": {
          "value": 848077
        }
      },
      {
        "key": "mysql-select",
        "doc_count": 202,
        "sum_duration": {
          "value": 605769
        }
      },
      {
        "key": "rpc-get_id",
        "doc_count": 400,
        "sum_duration": {
          "value": 104678
        }
      },
      {
        "key": "redis-incr",
        "doc_count": 200,
        "sum_duration": {
          "value": 217
        }
      },
      {
        "key": "redis-get",
        "doc_count": 201,
        "sum_duration": {
          "value": 216
        }
      }
    ]
  }
}
```

 这个返回结果表明，这一次调用共执行了：

- 200 次 MySQL 插入，总耗时 848 ms
- 202 次 MySQL 查询，总耗时 605 ms
- 400 次 RPC 发号器调用，总耗时 104 ms
- 217 次 Redis 自增操作，总耗时 0.2 ms
- 216 次 Redis 查询操作，总耗时 0.2 ms

 再进一步，可以将 `binaryAnnotation` 设置为执行的预编译 SQL 模板，例如：

```
{
  "name": "mysql-select",
  "binaryAnnotations": [
    {
      "endpoint": {
        "ipv4": "10.xxx.xxx.xxx",
        "port": 8088,
        "serviceName": "user-service"
      },
      "value": "select * from user where user_id = ?",
      "key": "query"
    }
  ]
}
```

 这样就可以找到系统中到底执行了哪些慢查询：

```
{
  "size": 0,
  "query": {
    "bool": {
      "must": [
        {
          "match_all": {}
        }
      ],
      "filter": {
        "term": {
          "name": "mysql-select"
        }
      }
    }
  },
  "aggs": {
    "path": {
      "nested": {
        "path": "binaryAnnotations"
      },
      "aggs": {
        "filter_sql": {
          "filter" : { 
            "term": {
              "binaryAnnotations.key": "executed.query"
            }
          },
          "aggs": {
            "sql": {
              "terms": {
                "field": "binaryAnnotations.value"
              },
              "aggs": {
                "reverse_duration": {
                  "reverse_nested": {},
                  "aggs": {
                    "duration_outlier": {
                           "percentiles" : {
                             "field": "duration",
                             "percents": [75, 95, 99]
                           }
                         }
                    "avg_duration": {
                      "avg": {
                        "field": "duration"
                      }
                    },
                    "sum_duration": {
                      "sum": {
                        "field": "duration"
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

 在上面这个查询中可以聚合出每种 SQL 执行的次数、总耗时、平均耗时、Mean 75/95/99 耗时等信息

### 总结

 基于 Elasticsearch 的 Zipkin 自定义查询，核心是定义好 `name`、`binaryAnnotations.key` 和 `binaryAnnotations.value` 三个字段，使其作为 Terms 聚合将不同功能的 Span 拆分，之后再通过 Avg、Sum、Percentiles 等统计型聚合对 `duration` 算出需要的数据。

 但是这其中也是有一些坑的，首先是 Zipkin 的 Span 文档量会非常的多，所以无任何过滤条件的聚合请求响应时间非常长。基本上达到一定数量级后，Zipkin UI 加载 Service Name 和 Span Name 的接口就必然会超时了。这时候反而用 Kibana 会比 Zipkin UI 效率更高，因为自定义查询语句无论是功能上还是性能上都是可控的。

 另外一个坑就是 `binaryAnnotations` 是一个嵌套文档，但聚合时最终落在统计指标的 `duration` 却是外层的字段，这在某些聚合语句下会有问题。目前来看最常见的一个场景是想指定一个聚合结果作为排序条件时，被提示 `Ordering on a single-bucket aggregation can only be done on its doc_count.`，目前还没找到很好地解决办法。

 其实 Zipkin 在 Cassandra 上存储的方式很好，性能足够支撑大数据量且查询速度也很快。但是切换为 Elasticsearch 之后，查询和聚合的方式丰富了许多，`binaryAnnotations` 这种字段在文档型存储中就显得比较尴尬的。此时就有了自定义存储结构的价值，可以根据业务定制各种需要存储的额外字段，改造难度也不大。


