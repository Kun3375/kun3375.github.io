---
title: RocketMQ Consumer 摘要
author: 斯特拉不用电
date: 2018-08-12 22:29:40
tags:
  - MQ
  - RocketMQ
categories: RocketMQ
comments: true
---

结束了对 RocketMQ 组件的初步理解以及配置的简单设定，可以对 RocketMQ 仔细研究一番了。先来看看 RocketMQ 的消费者实现，以及服务端是如何处理消费者客户端的请求，把消息送出去的。

RocketMQ 对于消费者客户端，支持推模型和拉模型。对于推模型，由消息服务端作为主动方，向客户端推送消息（尽管其本质是一个长轮询式的拉模型实现）；而拉模型由客户端主动拉取消息。

<!-- more -->

## PushConsumer ##

### 客户端的实现：
`DefaultMQPushConsumerImpl` 是客户端的一个默认实现，可以从 `pullMessage()` 方法切入，观察它的实现。

### 基本要素：
以下几个属性，不仅仅是推模型的重要配置，同时也称得上是每个客户端的标配。
* **NameServerAddr**
指定 NameServer 地址是必要的，可以通过客户端 API 设置（使用 `;` 分割多个地址），或者使用环境变量 `NAMESRV_ADDR`
* **ConsumerGroup**
将多个消费者组织一起，提高并发，需要配合 `MessageModel` 属性一起使用
    * **MessageModel**
消息模式分为两种，**集群模式**：**Clustering**；**广播模式**：**Broadcasting**
        * **Clustering**：集群模式，所订阅 Topic 下的消息，每一条只会被同一 ConsumerGroup 下的一个消费者所消费，达到负载均衡的目的
        * **Broadcasting**：广播模式，同一 ConsumerGroup 下的每一个 Consumer 都会消费到所订阅 Topic 下的全部消息。
* **Topic**
消息类型主题，作为不同消息的标识，决定了消费者订阅哪些消息。Topic 默认是可以由客户端创建的，生产环境下通常改权限被关闭，需要使用 mqadmin 工具来初始化可用的 Topic
    * **Tag**
Tag 可以进一步过滤消费需要订阅的消息，在 Java 客户端 API 下，使用 `null` 或者 `*` 来消费所有 Tag 类型，需要具体指定时可以使用 `||` 来分割多个 Tag

### 服务端推送方式：
消费者的推模型是通过长轮询实现的，因为完全的推模型方式会使得服务端增加许多压力，明显的降低效率，同时也会因为各客户端消费能力不足的问题造成隐患。Broker 服务端在处理客户端请求时如果发现没有消息，会休眠一小会-短轮询间隔（`shortPollingTimeMills`），重复循环，直到超过最大等待时间（`brokerSuspendMaxTimeMills`），在此期间内的收到消息会立即发送给客户端，达到“推”的效果

### 客户端流量控制：
客户端维护了一个线程池来接受服务端“推”来的消息，针对每个 `MessageQueue` 都有使用一个 `ProcessQueue` 来保存快照状态和处理逻辑。`ProcessQueue` 主要由一个 TreeMap 和读写锁组成
* `ProcessQueue.lockTreeMap` 保存了所有获取后还没有被消费的消息
    * Key：MessageQueue‘s offset
    * Value：消息内容引用
* `DefaultMQPushConsumerImpl.pullMessage()` 会检查以下每个属性，任意属性超过阈值会暂缓拉取动作。由于通过 ProcessQueue 的信息来比较，检查域是每个 Queue
    * `cachedMessageCount`
检查当前缓存的但是未消费的消息数量是否大于设定值（`pullThresholdForQueue`，默认 1000）
    * `cachedMessageSizeInMiB`
同上，检查队列中消息缓存的大小（`pullThresholdSizeForQueue`，默认 100MiB）
    * maxSpan
检查 `ProcessQueue` 中未消费消息的 offset 跨度（`consumeConcurrentlyMaxSpan`，默认 200），*在顺序消费时不检查*


## PullConsumer

### 客户端的实现：
初次接触，可以从这几个方法了解 PullConsumer 的消息拉取思路，并从官方的几个例子中了解一些常用的处理方式。
1. 前置操作
    * `DefaultMQPullConsumerImpl.fetchSubscribeMessageQueues()`
    * `DefaultMQPullConsumerImpl.fetchConsumerOffset()`
    * `DefaultMQPullConsumerImpl.fetchMessageQueuesInBalance()`
2. 拉取动作
    * `DefaultMQPullConsumerImpl.pull()`
    * `DefaultMQPullConsumerImpl.pullBlockIfNotFound()`

### 客户端额外操作：
在使用 PullConsumer 时候，通常使用需要额外关心 `MessageQueue` 和 **offset** 等一些要素，灵活的封装可以带来更多的自主性。
以 `fetchSubscribeMessageQueues()` 和 `pull()` 方法说明几个要素：
* **MessageQueue**
一个 Topic 下通常会使用多个 MessageQueue，如果需要获取全部消息，需要遍历返回的所有队列。特殊情况下可以针对特定队列消费
* **Offsetstore**
使用者需要手动记录和操作消息偏移量，随着消息消费而改变它，需要额外注意他的持久化，正确的偏移量是准确消费的前提
* **PullStatus**
针对某队列的拉取动作结束，会返回相应状态，使用者需要针对不同状态采取不同的动作
    * `FOUND`
    * `NO_MATCHED_MSG`
    * `NO_NEW_MSG`
    * `OFFSET_ILLEGAL`
* `shutDown()`
关闭操作会进行保存 offset 的操作，在 NameServer 注销客户端的操作等。对于保存的 offset 可以通过 OffsetStore 对象获取，启动时加载。

## 参考
* [RocketMQ on GitHub](https://github.com/apache/rocketmq)
* 《RocketMQ 实战与原理解析》机械工业出版社 杨开元
