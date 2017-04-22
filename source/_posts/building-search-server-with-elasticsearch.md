---
title: '使用Elasticsearch搭建搜索平台'
date: 2016-01-27 07:04:56
permalink: building-search-server-with-elasticsearch
---

几乎所有的网站和 APP 都需要搜索功能，其中又主要分为文本搜索和地理位置搜索两大类。如今这类的开源产品很多，完全不需要自己浪费大量的时间造轮子，例如你可以选择以下的产品：

 文本搜索：

- Lucene：一个全文搜索引擎工具库，你可以以此为基础封装一个搜索引擎
- Solr：基于 Lucene，是一个完整的搜索服务
- Elasticsearch：同样是基于 Lucene 的搜索服务
- Sphinx：基于 SQL 的全文检索引擎，可以配合 MySQL 做全文搜索功能

 地理位置搜索：

- Redis：3.2 版本提供了 Geo 查询操作，速度很快但是只能做一些简单查询
- MongoDB：地理位置功能很强大，如果以 LBS 为主要功能的话非常推荐，像快的等最开始都是用 MongoDB 的
- MySQL：MySQL4 之后也提供了空间数据存储和索引
- Elasicsearch 和 Solr：同样提供了对应的索引支持

 目前的项目最开始引入搜索功能时，我简单的封装 Lucene 做成了一个搜索服务。之后引入附近的人等 LBS 功能时，又迅速的采用了 MongoDB 实现了需求。但是这样做其实有很多不好的地方，例如：

- 自己封装的 Lucene 只适用于当前的业务场景，一旦需要添加新的功能还需要修改对应的代码
- 需要额外花费精力保证搜索服务的高性能和高可用，以及实现分布式
- MongoDB 做 LBS 很好，但是目前 LBS 并不是最主要的功能，只为了这些功能引入 MongoDB 却增加的运维成本
- 由于文本索引和地理位置索引分别存储在不同的地方，也就无法同时使用这两种条件查询和筛选排序

 为了解决这些问题，我选择了 Elasticsearch 作为搜索服务，其实 Solr 也可以，如果你想了解它们有什么不同，可以在 [这里](http://solr-vs-elasticsearch.com/) 了解。我选择 Elasticsearch 的其中一个重要原因是它有一套现成的日志分析系统：Logstash + Elasicsearch + Kibana，这个可能之后会单独写一篇文章介绍。

 接下来就进入正题，介绍一下如何使用 Elasticsearch 搭建一套搜索服务。

## 安装

### 安装 Elasicsearch

 首先在 [官网](https://www.elastic.co/downloads/elasticsearch) 下载最新版本的 Elasicsearch（目前的最新版本为`2.1.1`，但是由于 Spring-Data-Elasticsearch 还没有更新，且不兼容，所以我下载的版本是`1.7.3`）。

 下载后解压，然后只需要运行`bin/elasticsearch`就可以启动了，整个流程大概是这样的:

```
curl -L -O https://download.elastic.co/elasticsearch/elasticsearch/elasticsearch-1.7.3.zip
unzip -o elasticsearch-1.7.3.zip
cd elasticsearch-1.7.3
./bin/elasticsearch
```

 接下来打开`http://localhost:9200/`就可以看到 Elasicsearch 的信息了。

 注意：`2.1.1`版本不允许使用`root`用户启动，`1.7.3`版本没有这个要求。

### 安装 Marvel

Marvel 是 Elasticsearch 的管理和监控工具，它在开发环境下是免费的。安装仅需要在 Elasticsearch 根目录中执行：

```
./bin/plugin install elasticsearch/marvel/latest
```

 安装成功后重启 Elasicsearch 服务器，然后打开`http://localhost:9200/_plugin/marvel`就可以进入 Marvel 的监控页面了。

## 基础概念

### 文档（Document）

Elasticsearch 使用文档作为存储的数据结构，并使用 JSON 作为文档的序列化格式。如果与传统数据库作为对比，Elasticsearch 中的一个文档相当于 MySQL 中的一行记录（Row），文档中的每一个字段（Field）相当于行中的每一个单元（Column）。

 你可以建立包含姓名和手机号的文档：

```
{
    "name": "ScienJus",
    "phone": "188xxxxxxxx"
}
```

### 索引（Index）和类型（Type）

Elasicsearch 中索引的意义和 MySQL 等数据库中的索引不一样，在 Elasticsearch 中一个索引相当于 MySQL 中的一个数据库（Database）。而类型则对应的 MySQL 的表（Table）。

 使用`PUT /:index/:tag/:id`可以插入文档到对应的位置，不用实现创建索引和类型，Elasticsearch 会在插入文档时自动建立索引和类型，并通过文档进行一下简单的类型映射。例如：

```
curl -X PUT http://localhost:9200/test_index/test_tag/1 -d '
{
    "name": "ScienJus",
    "phone": "188xxxxxxxx"
}
'
```

 接着通过`GET /:index/:tag/:id`就可以看到插入的文档了，例如：

```
curl -X GET http://localhost:9200/test_index/test_tag/1

{
    "_index": "test_index",
    "_type": "test_tag",
    "_id": "1",
    "_version": 1,
    "found": true,
    "_source": {
        "name": "ScienJus",
        "phone": "188xxxxxxxx"
    }
}
```

`_index`、`_type`、`_id`分别标明了文档所属的索引、类型和文档的 id，`_source`为文档的内容。

### 映射（Mapping）

Elasticsearch 默认会对索引数据的类型进行猜测，但是这种猜测只能映射为简单类型，例如 Elasticsearch 无法识别一个`string`类型究竟是一个确切的值（例如手机号）还是一个需要分词的复杂文本（例如文章内容），也无法识别一个`double []`是否为一个经纬度信息。所以这些复杂映射都需要开发者指定。

 通过`GET /:index/_mapping/:type`可以查看一个类型的映射，例如：

```
curl -X GET http://localhost:9200/test_index/_mapping/test_tag

{
    "test_index": {
        "mappings": {
            "test_tag": {
                "properties": {
                    "name": {
                        "type": "string"
                    },
                    "phone": {
                        "type": "string"
                    }
                }
            }
        }
    }
}
```

 当然也可以通过`PUT /:index`在创建索引时就指定映射，或是在创建后追加映射，例如：

```
curl -X PUT http://localhost:9200/test_index -d '
{
    "mappings": {
        "test_tag": {
            "properties": {
                "name": {
                    "type": "string"
                },
                "phone": {
                    "type": "string"
                }
            }
        }
    }
}
'
```

 注意：已创建的字段无法更改映射。

 事实上在大部分情况下，Elasticsearch 通过分析 JSON 默认创建的映射都是正确的，只有在需要分词的复杂文本和坐标点时才需要用户手动创建映射，这两种映射的方式在之后会详细介绍。

## 全文搜索

### 原理

Elasticsearch 使用`倒排索引`完成全文搜索功能，倒排索引会将文本内容分词成一个个词元，然后建立词元与文档之间对应的关系。

 例如我们现在有两句话：

1. Elasticsearch is an open source search and analytics engine
2. Elasticsearch is a distributed RESTful search engine

 首先需要将这两句话分解成一个个词元，并且去掉一些无意义的词语，例如`a`、`an`、`for`等，再将其关联上对应的文档 id：

- Elasicsearch：1, 2
- open：1
- source：1
- search：1, 2
- analytics：1
- engine：1, 2
- distributed：2
- RESTful：2

 如果我们想要搜索`Elasicsearch`，就可以通过上面的键值对很快的查询到对应的文档 1 和文档 2，搜索`source`则只能查到文档 1，而如果搜索`RESTful search`虽然也能同时搜到文档 1 和文档 2，但是文档 2 的匹配程度要高于文档 1。

 当然这只是一个最简单的例子，在实际应用中还会有很多问题，例如单复数形式或者动词形容词名词形式以及同义词都应该被认为是同一单词，还需要去掉没有意义的单词等，这些 Elasticsearch 都会做好，就不需要使用者操心了。

### 创建映射

 对于一个复杂文本的映射，除了要指定它的类型为`string`以外，还要通过`index`属性指定它是否被分析（也就是分词）。这个属性有三个选项，分别是：`analyzed`（分析并索引），`not_analyzed`（不分析并索引），`no`（不索引）。如果将它设为`analyzed`的话，还需要通过`analyzer`参数指定分析器，因为不同语言的语法也是不同的。

 所以创建一个复杂文本的映射大概是这样的：

```
curl -X PUT http://localhost:9200/test_index -d '
{
    "mappings": {
        "test_tag": {
            "properties": {
                "desc": {
                    "type": "string",
                    "index": "analyzed",
                    "analyzer": "english"
                }
            }
        }
    }
}
'
```

### 分词

Elasticsearch 默认提供了一些语言的分词器，其中也包括中文。但是这个分词器的效果非常不好，所以在此推荐另一款开源的分词器：[IK](https://github.com/medcl/elasticsearch-analysis-ik)。

 首先需要将这个项目 Clone 下来，使用`mvn clean package`打包，就可以在`target/releases`下找到`elasticsearch-analysis-ik-*-jar-with-dependencies.jar`了。这个流程应该是这样的：

```
git clone https://github.com/medcl/elasticsearch-analysis-ik.git
cd elasticsearch-analysis-ik-1.4.1
mvn clean package
mv target/releases/elasticsearch-analysis-ik-1.4.1-jar-with-dependencies.jar ./elasticsearch-analysis-ik.jar
```

 然后将这个 Jar 包复制到 Elasicsearch 的`plugins/ik`文件夹下（如果没有自己创建），再将项目中`config/ik`整个文件夹复制到 Elasicsearch 的`config`下。最后将以下内容添加到`config/elasticsearch.yml`中：

```
index:
  analysis:
    analyzer:
      ik:
          alias: [ik_analyzer]
          type: org.elasticsearch.index.analysis.IkAnalyzerProvider
      ik_max_word:
          type: ik
          use_smart: false
      ik_smart:
          type: ik
          use_smart: true
```

 重启 Elasicsearch，检验一下分词器是否可以使用了：

```
curl -X POST 'http://localhost:9200/test_index/_analyze?analyzer=ik&pretty=true' -d '
{
    "text": "分词测试"
}
'

{
    "tokens": [
        {
            "token": "text",
            "start_offset": 6,
            "end_offset": 10,
            "type": "ENGLISH",
            "position": 1
        },
        {
            "token": "分词",
            "start_offset": 14,
            "end_offset": 16,
            "type": "CN_WORD",
            "position": 2
        },
        {
            "token": "词",
            "start_offset": 15,
            "end_offset": 16,
            "type": "CN_WORD",
            "position": 3
        },
        {
            "token": "测试",
            "start_offset": 16,
            "end_offset": 18,
            "type": "CN_WORD",
            "position": 4
        }
    ]
}
```

IK 提供了两种分词的粒度，粗粒度为`ik_smart`，细粒度为`ik_max_word`。如果你是按照匹配程度进行默认排序的话，这两种分词的结果可能不会有太大区别，但是如果你是以其他条件进行排序的话，还是推荐使用`ik_smart`。

### 查询

 一个简单的全文搜索的查询语句大概是这样的:

```
curl -X GET 'http://localhost:9200/test_index/test_tag' -d '
{
    "query": {
        "match": {
            "title": "Elasicsearch"
        }
    }
}
'
```

 这将会搜索所有`title`与`Elasicsearch`有关的内容。并且会对所有文档打分，且按照分数降序返回数据。

 但是实际上用户不可能仅仅搜索一个单词，一般来说他可能会输入多个关键词或者一段话进行搜索，这时候就要考虑匹配的精度问题了。

 将上面的查询语句进行一些改造，将`title`加入`operator`属性：

```
curl -X GET 'http://localhost:9200/test_index/test_tag' -d '
{
    "query": {
        "match": {
            "title": {
                "query": "Elasicsearch RESTful",
                "operator": "and"
            }
        }
    }
}
'
```

 将`operator`设置为`and`后，Elasicsearch 则会查询即包含`Elasicsearch`也包含`RESTful`的文档。如果没有这条属性则是查询包含任意一个关键词的文档。

 但是只能选择全部包含又有一些极端，所以还能通过指定`minimum_should_match`参数设置至少匹配的关键词个数（它的值为百分比，Elasticsearch 会自动通过输入内容的个数计算出至少匹配个数的）：

```
curl -X GET 'http://localhost:9200/test_index/test_tag' -d '
{
    "query": {
        "match": {
            "title": {
                "query": "Elasicsearch RESTful",
                "minimum_should_match": "67%"
            }
        }
    }
}
'
```

 这里设置为`67%`，所以如果只有两个关键词的话，只需要满足其中一个即可，但是如果有三个关键词的话，就需要至少满足其中两个了。

## 地理位置搜索（LBS）

### 原理

 几乎所有地理位置索引都是以`geohash`算法为基础的，这个算法可以将一个二维的经纬度信息转化为一个一维的字符串，同时字符串可以进行比较以模糊的确定两个点之间的距离。

 它的原理是首先将纬度从 (-90, 90) 平分成两个区间 (-90,0)、(0, 90)，如果目标纬度位于前一个区间，则编码为 0，否则编码为 1。再将目标所在的区间再次进行平分，得到目标下一次所在区间的编码，一直循环直到满足精度位置。而经度也是一样。最后将经纬度编码进行合并，在用`base32`编码，得到的字符串就是这个点的 geohash 值。

### 映射

 在 Elasticsearch 中，地理位置坐标的格式为`geo_point`，所以所以创建映射的的写法是这样的：

```
curl -X PUT http://localhost:9200/test_index -d '
{
    "mappings": {
        "test_tag": {
            "properties": {
                "location": {
                    "type": "geo_point"
                }
            }
        }
    }
}
'
```

 插入文档时，有三种格式的属性可以被映射为 geo\\_point 类型：

- `string`：格式为`lat,lon`
- `double []`：格式为`[lon, lat]`
- `json`：格式为`{ "lat": lat, "lon": lon}`

 注意：数组格式是 lon 在前，lat 在后，而字符串格式正好相反。

 这里有一个小细节是：对于某些特殊的查询，例如接下来会提到的矩形范围查询，可以通过分别对`lat`和`lon`进行索引以提高查询速度。因为矩形范围的查询完全可以先通过`lat`过滤，再通过`lon`过滤，而像距离查询就无法这样做。

 这样做只需要在映射时多加一个`"lat_lon": true`，例如：

```
curl -X PUT http://localhost:9200/test_index -d '
{
    "mappings": {
        "test_tag": {
            "properties": {
                "location": {
                    "type": "geo_point",
                    "lat_lon": true
                }
            }
        }
    }
}
'
```

### 查询

Elasticsearch 的地理位置查询，实际是使用过滤器对所有文档进行过滤。它支持以下四种查询方式：

- `geo_bounding_box`：查询矩形范围内的点
- `geo_distance`：查询中心点距离范围内的点
- `geo_distance_range`：查询中心点最小距离和最大距离之间的点
- `geo_polygon`：查询多边形范围内的点（不推荐使用）

`geo_bounding_box`只需要指定矩形范围左上角的点和右下角的点，例如：

```
curl -X GET http://localhost:9200/test_index/test_tag/_search -d '
{
    "query": {
        "filtered": {
            "filter": {
                "geo_bounding_box": {
                    "location": {
                        "top_left": {
                            "lat": 39.8,
                            "lon": 115.1
                        },
                        "bottom_right": {
                            "lat": 41.2,
                            "lon": 117.3
                        }
                    }
                }
            }
        }
    }
}
'
```

 如果你按照上面的方法将`lat`和`lon`单独建立了索引，就可以在查询时要求 Elasticsearch 使用之前建立的索引，只需要添加一行`"type": "indexed"`。例如：

```
curl -X GET http://localhost:9200/test_index/test_tag/_search -d '
{
    "query": {
        "filtered": {
            "filter": {
                "geo_bounding_box": {
                    "type": "indexed",
                    "location": {
                        "top_left": {
                            "lat": 39.8,
                            "lon": 115.1
                        },
                        "bottom_right": {
                            "lat": 41.2,
                            "lon": 117.3
                        }
                    }
                }
            }
        }
    }
}
'
```

`geo_distance`查询需要指定中心点和距离，比较人性化的是，Elasticsearch 内置了一些距离单位，你可以直接以类似`1km`的方式指定距离。例如：

```
curl -X GET http://localhost:9200/test_index/test_tag/_search -d '
{
    "query": {
        "filtered": {
            "filter": {
                "geo_distance": {
                    "distance": "1km",
                    "distance_type": "plane",
                    "location": {
                        "lat": 41.2,
                        "lon": 117.3
                    }
                }
            }
        }
    }
}
'
```

 除了中心点和距离，我们还需要指定计算距离的方式，每种方式对应着不同的精度和计算速度，常用的有三种：

- `asc`：弧形计算方式，最精确但是最慢
- `plane`：平面计算方式，最快但是精度稍差
- `sloppy_arc`：一种优化过的弧形计算方式，比`asc`要快上几倍且精度也非常高。Elasicsearch 的默认计算方式。

 对于大部分应用来说，`plane`的精度已经足够了，并且速度是最快的，所以推荐使用它。

`geo_distance_range`和`geo_distance`的唯一差别是需要提供最小距离和最大距离。例如：

```
curl -X GET http://localhost:9200/test_index/test_tag/_search -d '
{
    "query": {
        "filtered": {
            "filter": {
                "geo_distance_range": {
                    "gt": "1km",
                    "lt": "5km",
                    "location": {
                        "lat": 41.2,
                        "lon": 117.3
                    }
                }
            }
        }
    }
}
'
```

## 集群

### 目的

 本文最开始就提到过，我自己封装的 Lucene 服务，想要保证高可用需要费很多力气。但是如果使用 Elasticsearch 将会很简单。

Elasticsearch 使用多个节点组成分布式集群，并将数据均匀的分配到各个节点中，这样做有以下好处：

- 索引和搜索时多个节点可以进行负载均衡
- 数据冗余在多个分片中，避免数据丢失
- 避免节点故障导致服务不可用
- 通过多台机器分担大数据量

### 分片（Shard）

Elasicsearch 中的每一个分片都是一个独立的搜索引擎，但是它只存储并索引了其中一部分数据。

 分片可以分为主分片（Primary Shard）和复制分片（Replica Shard），主分片负责索引数据，复制分片是主分片的副本，并且是只读的。

 在创建索引时可以通过`settings`字段配置主分片和每个主分片对应复制分片的数量，例如：

```
curl -X PUT http://localhost:9200/test_index -d '
{
    "settings": {
        "number_of_shards": 2,
        "number_of_replicas": 1
    }
}
'
```

 以上内容表示这个索引会有 2 个主分片，其中每个主分片对应 1 个复制分片。主分片的数量一旦确定便不能修改了，但是复制分片的数量可以随时更改，只需要重置`number_of_replicas`即可。

 注意：主分片和复制分片在同一节点没有任何意义，所以此时不会创建复制分片，直到有了新的节点。

### 多节点

 启动 Elasicsearch 多节点非常简单，只需要保证节点的集群名是相同的（在`config/elasticsearch.yml`中定义），新节点将会自动加入集群。

 你可以通过`http://localhost:9200/_cluster/health`查看集群的健康度。返回的数据大概是这样：

```
curl -X GET http://localhost:9200/_cluster/health

{
    "cluster_name": "elasticsearch",
    "status": "yellow",
    "timed_out": false,
    "number_of_nodes": 1,
    "number_of_data_nodes": 1,
    "active_primary_shards": 19,
    "active_shards": 19,
    "relocating_shards": 0,
    "initializing_shards": 0,
    "unassigned_shards": 19,
    "delayed_unassigned_shards": 0,
    "number_of_pending_tasks": 0,
    "number_of_in_flight_fetch": 0
}
```

- 如果`status`的值是`green`说明该集群的所有主分片和复制分片都可用，这是最好的情况。
- 如果是`yellow`说明所有主分片都可用，部分从分片不可用，这这不会影响到服务，但是需要注意。
- 如果是`red`说明已经有部分主分片无法使用了，这是最坏的情况，会影响到正常服务。

 如此一来，一个具有全文搜索、地理位置搜索的高性能、高可用的分布式搜索服务都搭建完成了！

 参考资料：[Elasticsearch 权威指南（中文版）](http://es.xiaoleilu.com/index.html)


