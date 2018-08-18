---
title: RocketMQ Producer 摘要
author: 斯特拉不用电
date: 2018-08-18 13:01:01
tags:
  - MQ
  - RocketMQ
categories: RocketMQ
comments: true
---

上一篇介绍完了 RocketMQ 消费者的默认实现，现在来瞅一瞅生产者的用法。

## 设置必要的属性 ##

同样的，是 DefaultMQProducer，新建实例之后，在使用生产者发送消息之前，需要初始化几个属性：
* `InstanceName` 实例名称
这是为了当一个 JVM 上启动了多个生产者时，区分不同的生产者实例，系统默认名称为 DEFAULT
* `RetryTimesWhenSendFailed` 重试次数
当消息投递失败时，有可能是因为网络原因，可以设置多投递几次减少丢消息的情况。
很多实用者在使用时，为了避免重复的消息设置不重试是不正确的做法：因为 RocketMQ 本身并不保证消息的不重复，作为客户端对消息进行幂等处理是必要的。而在次前提下，对发送失败的场景拒绝重发，不仅对避免重复消息没有任何意义，同时也增加了消息的丢失的可能。
* `NamesrvAddr`
需要 NameServer 的地址，写法和 Consumer 一致

## 消息发送方式和投递结果 ##

### 发送方式 ###
* 同步发送：`Producer.send(Message message)`
* 异步发送：`Producer.send(Message message, SendCallback callback)`

### 发送结果 ###
对于消息发送的结果，存在四中可能返回的状态。而且在不同的配置方式下，意义可能有所不同

* **`SEND_OK`**
发送成功，标志着消息已经成功被发送到 Broker。（这时候不一定意味着主从复制完成或者刷盘完成）
* **`FLUSH_DISK_TIMEOUT`**
刷盘时间超时，只有在刷盘策略为 `SYNC_FLUSH` 时才可能出现
* **`FLUSH_SLAVE_TIMEOUT`**
主从同步时间超时，只有在主备形式下使用 `SYNC_MASTER` 才可能出现
* **`SLAVE_NOT_AVAILABLE`**
从机缺失，只有在主备形式下使用 `SYNC_MASTER` 才可能出现，类似于 FLUSH_SLAVE_TIMEOUT
对于不同的业务场景具体需求，如何处理消息发送的结果是程序质量的一个重要考量点

## 特殊的消息 ##

### 延迟消息 ###

RocketMQ 支持延迟消息，Broker 收到消息后并不会立即投递，而是等待一段时间后再讲消息送出去。
* 使用方式：在消息发送前执行 `Message.setDelayTimeLevel(int level)`
* 延迟等级：默认 1s/5s/10s/30s/1m/2m/3m/4m/5m/6m/7m/8m/9m/10m/20m/30m/1h/2h，索引 1 开始
尽管 RocketMQ 的延迟消息不支持任意精度，但是各等级的延迟是可以预设的，更改配置文件即可

### 队列选择 ###

对于一个 Topic 通常有多个 MessageQueue 来接收消息，默认情况下 Producer 轮流向各个 MessageQueue 发送消息，而 Consumer 根据默认的负载策略进行消费，所以无法明确对应 Producer 的消息是哪个 Consumer 消费。在需要指定特定 MessageQueue 来投递消息时，可以实现 `MessageQueueSelector` 接口，定制选择逻辑；发送时选择带有选择器的重载方法即可

### 事务消息 ###
介绍事务消息是必要的，但是并不推荐使用。因为事务消息会造成磁盘脏页，影响磁盘性能，在 4.x 版本中已经移除，需要使用时需要手动根据顶层接口实现。简单的说，RocketMQ 的事务消息流程如下：
1. 向 Broker 发送消息（消息状态为未确认状态）
2. Broker 对收到的消息完成持久化，返回成功状态。发送的第一阶段结束
3. 执行本地逻辑
4. 事务消息的结束
    * 本地逻辑结束，客户端向 Broker 确认消息
        * commit：提交，该消息将会被 Broker 进行投递
        * rollback：回滚，Broker 会删除之前接收到的消息
    * 超过一定时间，服务端对客户端发起回查请求
Producer 对回查请求返回 commit 或者 rollback 的响应。如果此时发送消息的 Producer 无法访问，回查请求会发送给同一 ProducerGroup 内的其他 Producer

## 参考 ##
* [RocketMQ on GitHub](https://github.com/apache/rocketmq)
* 《RocketMQ 实战与原理解析》机械工业出版社 杨开元


