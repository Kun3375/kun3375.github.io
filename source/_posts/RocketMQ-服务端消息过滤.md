---
title: RocketMQ 服务端消息过滤
author: 斯特拉不用电
date: 2018-08-26 16:12:57
categories: RocketMQ
tags:
  - MQ
  - RocketMQ
comments: true
---
在服务端进行消息过滤，可以减少不必要的流量，提高带宽利用度和吞吐量。
RocketMQ 支持多种方式来进行服务端的消息过滤

## 消息使用 Tag 标签
作为一条 Message，它有着特定的 Topic，同时也可以指定唯一的 Tag 标记子分类。消费方在订阅消息时候，Broker 可以在指定 Topic 的 ConsumeQueue 下按 Tag 进行过滤，只从 CommitLog 中取出 Tag 命中的消息。 
<!-- more -->
使用 Tag 进行过滤是高效的，因为消息在 MessageQueue 的存储格式如下：
* CommitLog Offset：顾名思义，保存着在 CommitLog 中的偏移量，占用 8 个字节
* Size：使用 4 个字节来记录消息的大小
* Message Tag HashCode：记录对应消息的 Tag 的哈希

在获取消息时候，通过 Tag HashCode 的对比，从 CommitLog 读取对应消息。由于哈希冲突实际上是不可避免的，消息在从 CommitLog 中拉取之后被消费之前，仍然会进行 Tag 的完整对比，以消除潜在哈希冲突问题

## 携带 MessageKey 来发送和查询
其实这部分内容并不属于服务端消息过滤的功能，但是也为我们提供了一种较精确的查询指定消息的功能。在发送消息之前可以为消息设定指定的 Key，通常这个 Key 是在业务层面是唯一的：
``` java
Message msg = new Message("Topic", "Tag", "Content".getBytes());
msg.setKey(uniqueKey);
```
尽管 Broker 不会对消息进行 Key 相关的过滤，但是会为消息定制相应的索引。看一下索引格式：
* Key HashCode：4 个字节的 Key 的哈希，用来快速检索
* CommitLog Offset：8 个字节来保存 CommitLog 中的偏移量
* Timestamp：使用 4 个字节记录消息存储时间和产生时间的时间差
* Next Index Offset：使用 4 个字节来记录下一索引的偏移量
在存储 Key 相应的索引时候，其实分了多个哈希桶来（Slot）存储，也就是相对 Key 进行了两次散列。怎么解决哈希冲突？因为索引结构中保存了 Key 的哈希，所以对于哈希值不同而模数相同的 Key 在查询时候可以直接区分开来。对于哈希值相等但是 Key 本身不相等的情况，客户端继续做一次 Key 比较来进行筛选。
一般应用中进行消息过滤使用 Tag，而使用命令行工具 mqadmin 做运维时查询特定 Key 的消息，用法：
``` shell
mqadmin queryMsgByKey -k <Key> -n <NamesrvAddr> -t <Topic> -f <endTime>
```

## 使用 MessageId 来查询消息
每次消息成功发送后，都会生产一个 **MsgId** 和 **OffsetMsgId**，来标识这条消息：
``` java
Message msg = new Message("Topic", "Tag", "Content".getBytes());
SendResult result = producer.send(msg);
// producer 产生的 id
String msgId = result.getMsgId();
// broker 产生的 id
String offsetMsgId = result.getOffsetMsgId();
```
- 对于 MsgId，由 producer ip + pid + MessageClientIDSetter.class.getClassLoader().hashCode() + time + counter 组成
- 而对于 OffsetMsgId，由 broker ip + CommitLog Offset 组成，可以精确地定位消息存储的位置

同时我们可以使用运维工具 mqadmin 针对 OffsetMsgId 进行检索
``` shell
mqadmin queryMsgById -n <NamesrvAddr> -I <OffsetMsgId>
```

## 使用自定义属性和类 SQL 过滤
在发送消息前，我们可以为消息设置自定义的属性：
``` java
Message msg = new Message("Topic", "Tag", "Content".getBytes());
msg.putUserProperty("p1", "v1");
msg.putUserProperty("p2", "v2");
```
在服务端进行消费时候，可以针对自定义属性，利用类 SQL 的表达式来进行消息的进一步筛选：
``` java
consumer.subscribe("Topic", MessageSelector.bySql("p1 = v1");
```
使用这样的方式进行过滤，需要 Broker 先从 CommitLog 中取出消息，得到消息中的自定义属性进行对应的计算。理所当然的，功能很强大，但是效率没有使用 Tag 的过滤方式高。

### 对于表达式的语法支持如下：
* 对比操作：
    * 数字：>, <, <=, >=, =, BETWEEN
    * 字符串：=, <>, IN
    * 空值判断：IS NULL, IS NOT NULL
    * 逻辑判断：AND, OR, NOT
* 数据类型：
    * 数字：123，456
    * 字符串：'abc', 'def', 必须使用单引号
    * 空值：NULL
    * 布尔：TRUE，FALSE

## 使用自定义代码和 Filter Server
对于 Filter Server，事实上实在 Broker 所在服务器启动了多个类似中转代理的进程，这几个进程负责充当 Consumer 从 Broker 上拉取代码，使用用户上传的 Java 代码进行过滤，最后传送给消费者。
这个中转代理会和 Broker 本身争抢 CPU 资源，需要按需求谨慎使用；同时用于过滤的代码需要严格的审查，避免可能影响 Broker 宕机的风险操作。这个过滤操作只支持 PushConsumer
使用流程：
1. 启动 Broker 时指定 `filterServerNums=<n>`，当然使用配置文件也可以。n 的数量就是中转代理 FilterServer 的进程数
2. 实现 `org.apache.rocketmq.common.filter.MessageFilter` 接口，定制过滤逻辑
3. 接收消息：
``` java
PushConsumer.subscribe(final String topic, final String fullClassName, final String filterClassSource)
``` 
filterClassSource 是前一步 MessageFilter 接口实现的源码，必须使用 utf-8 编码。这会在 Consumer 启动时将过滤逻辑上传至 Broker

参考：
1. MessageId 生成解读 <https://www.cnblogs.com/linlinismine/p/9184917.html>

