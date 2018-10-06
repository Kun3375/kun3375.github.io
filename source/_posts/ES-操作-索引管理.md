---
title: ES 操作-索引管理 
author: 斯特拉不用电
date: 2018-09-24 16:16:33
tags:
  - ElasticSearch
categories: ElasticSearch
comments: true
---

#### 新建索引 ####

新建索引很简单，但是需要注意的是：
- ES 中索引名称不能包含大写字母
- 不能再次 PUT 一个已经存在的索引
- ES 默认给索引设置 5 个分片和 1 个副本，该值可以通过 setting 参数域进行修改。其中副本数在索引创建之后支持修改，而分片数无法修改！
``` bash
PUT /person
# 可选项，不附加请求体的情况下统一使用默认值
{
    "settings": {
        # 分片数量
        "number_of_shards": 3,
        # 副本数量
        "number_of_replicas": 1
    }
}
```

#### 更新索引 ####

对某属性设置相应的值即可。如果设置为 `null` 可以将其恢复成默认值
``` bash
PUT /person/_settings
{
    "number_of_replicas": 2
}
```

##### 索引设置 #####
部分设置的含义在后文涉及
- 静态设置
这部分的设置只有在索引创建或者关闭时支持修改（分片数量只能在创建索引时设置）  
``` bash
# 主分片数，默认为5.只能在创建索引时设置，不能修改
index.number_of_shards
# 是否应在索引打开前检查分片是否损坏，当检查到分片损坏将禁止分片被打开
#   false //默认值
#   checksum //检查物理损坏
#   true //检查物理和逻辑损坏，这将消耗大量内存和CPU
#   fix //检查物理和逻辑损坏。有损坏的分片将被集群自动删除，这可能导致数据丢失
index.shard.check_on_startup
# 自定义路由值可以转发的目的分片数。默认为 1，只能在索引创建时设置。此值必须小于index.number_of_shards
index.routing_partition_size
# 数据压缩方式，默认使用LZ4，也可以设置为 best_compression，它使用 DEFLATE 方式以牺牲字段存储性能为代价来获得更高的压缩比例
index.codec
```
- 动态设置
这部分配置支持直接修改
``` bash
# 每个主分片的副本数，默认为 1
index.number_of_replicas
# 基于可用节点的数量自动分配副本数量，默认为 false（即禁用此功能）
index.auto_expand_replicas
# 执行刷新操作的频率，这使得索引的最近更改可以被搜索。默认为 1s。可以设置为 -1 以禁用刷新。
index.refresh_interval
# 用于索引搜索的 from + size 的最大值，默认为 10000
index.max_result_window
# 在搜索此索引中 rescore 的 window_size 的最大值
index.max_rescore_window
# 设置为 true 使索引和索引元数据为只读，默认 false 为允许写入和元数据更改
index.blocks.read_only
# 设置为 true 可禁用对索引的读取操作，默认 false
index.blocks.read
# 设置为 true 可禁用对索引的写入操作，默认 false
index.blocks.write
# 设置为 true 可禁用索引元数据的读取和写入，默认 false
index.blocks.metadata
# 索引的每个分片上可用的最大刷新侦听器数
index.max_refresh_listeners
```

#### 查询索引 ####

关于查询也是简单的，通过 GET 请求和 \_setting API 可以获得索引的配置信息。而 \_cat API 可以以摘要的形式展示所有索引的开关状态，健康状态，分片数，副本数以及一些其他信息
``` bash
# 查询单个索引设置
GET /person/_settings
# 查询多个索引设置
GET /person,animal/_settings
# 按通配符查询索引设置
GET /p*/_settings
# 查询所有索引设置
GET /_all/settings
# 使用 _cat API 来展示所有索引的综合信息
GET /_cat/indices
```

#### 删除索引 ####

删除是最简单，当然如果指定的索引名称不存在会响应 404
``` bash
DELETE /person
```

#### 索引开关 ####

可以关闭一些暂时不用的索引来减少系统资源的开销，关闭后将无法进行读写
``` bash
POST person/_close
```
相对的，打开操作：
``` bash
POST person/_open
```
这里依然支持同时操作多个索引，以及通配符和 \_all 关键字的处理：
``` bash
POST /person,animal/_close
```
如果同时指定的索引中存在不存在的索引，会显示抛出错误。这可以通过一个简单参数 `ignore_unavailable=true` 来忽略
``` bash
POST /person,animal/_close?ignore_unavailable=true
```

#### 索引复制 ####

通过 \_reindex API 可以将一个索引的内容复制至另一个索引，这同时可以指定过滤条件，已筛选需要的 type 以及 doc。
需要注意的是，由于这不会同时复制索引的配置信息，所以在操作的时候需要提前建立好新索引
``` bash
POST /_reindex
{
    "source": {
        "index": "person",
        # 可选项，用于过滤类型 type
        "type": "student"
        # 可选项，用于过滤文档 doc
        "query": {
            "term": { "sex": "male" }
        }
    },
    "dest": {
        "index": "person_new"
    }
}
```

#### 索引收缩 ####

分片数量在索引初始化之后便无法修改。\_shrink API 可以将一个索引收缩成一个分片数量更少的索引。当然，这是有要求的：
- 收缩后索引的分片数必须是收缩前索引的分片数的因子，如 8 -> 4，或者 15 -> 5。这意味着如果源索引的分片数如果是质数，那么很尴尬，只能收缩成 1 个分片的新索引
- 收缩前，索引的每个分片需要都存在于同一节点上（可以指定路由实现）
- 索引必须为只读状态

后两者条件可以通过以下指令实现：
``` bash
PUT /person/_settings
{
    "index.routing.allocation.require._name": "shrink_node_name",
    "index.block.write": true
}
```

接下来便可以实际进行索引的收缩了，ES 完成的流程包括这几个步骤：
1. 创建一个配置与源索引相同但是分片数减少的新索引
2. 源索引硬链接至新索引（文件系统不支持的情况下会进行复制）
3. 打开新的索引

\_shrink 的一个例子如下：
``` bash
POST /person/_shrink/person_new
{
    "settings": {
        "index.number_of_replicas": 0,
        "index.number_of_shards": 1,
        "index.codec": "best_compression"
    },
    # 就如同新建索引一样，可以同时设置别名
    "aliases": {
        "pn": {}
    }
}
```

#### 索引别名 ####

通过 \_aliases API 可以选择为索引设置别名，这就像软连接一样
``` bash
POST /_aliases
{
    "actions": [{
        # 新增索引，可以一次操作多个索引，并合并书写
        "add": {
            "indices": ["animal", "plant"],
            "alias": "living_thing"
        }}, {
         # 删除索引，也可以一次操作多个索引，支持通配符
        "remove": {
            "index": "person",
            "alias": "person_alias"
        }}, {
        "remove": {
            "index": "school",
            "alias": "school_alias"
        }}
    ]
}
```
在通常使用时，别名基本和原索引名用法一样。但是如果别名和索引不是一对一关系的时候，无法通过别名索引文档或者通过 ID 来查询

关于索引别名的查询也十分简单：
``` bash
# 如果想知道某索引（如 person）的别名
GET /person/_aliases
# 获取所有别名
GET /_aliases
# 使用 _cat API 获取所有索引摘要
GET /_cat/aliases
```

### 补充 Restful 语义 ###

在 ES 操作中，以下几个请求方式是最常用的，补充一下以下请求代表的语义：
- `GET`：获取资源信息
- `DELETE`：删除指定的资源标识符下的资源
- `POST`：提交一个新的资源，不具备幂等性
- `PUT`：新增或更新一个资源标识符下资源，操作具有幂等性
- `HEAD`：获取该次请求的响应头信息


