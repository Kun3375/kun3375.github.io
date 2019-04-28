---
author: 斯特拉不用电
title: SpringKafkaListener 增效
date: 2019-04-28 19:17:14
categories: Java
tags:
    - Java
    - MQ
    - Kafka
comments: true
---

通过 spring-kafka 的集成，我们可以通过注解 `@KafkaListener` 以及 `spring.kafka.consumer` 相关配置轻松的管理 kafka 消费。但是消费速度往往仍然不够理想，需要进一步调整。

在 kafka 的实现中，某个 topic 的 partition 即分区的数量，基本上决定在这个 topic 下的最大并发程度。因为客户端的数量是受限于 partition 数量的。对于一定数量的 partition 而言，客户端数量如果更少，将有部分客户端会被分配上多个分区；如果客户端数量更多，超过 partition 数量的客户端将会无法消费，这是由 kafka 本身的负载均衡策略决定的。
尽管我们可以动态地调整 partition，但是对于基于 Key 的消息，并需要有序消费时，由于 kafka 通过 Key 进行 hash 分片，更改 partition 数量将无法保证有序性。

<!-- more -->

所以首先可以肯定的是：
> - 对于一个 topic 来说，设置适量多的 partition 是有必要的。分区数量决定了消费消费的并行度。

分区的变更以及过多的分区可以看看：[如何选择分片数量](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster "how-choose-number-topics-partitions-kafka-cluster")

那么在选择了一定数量的 partition 之后，并行度的限制绝大程度上就变成了客户端的消费能力。

如上所说的如果客户端过少将有部分客户端消费多个 partition。在不考虑留一定 partition 来拓展的情况下假定我们需要拥有同样数量的客户端。

事实上 spring-kafka 为我们提供了方便的配置：
> `spring.kafka.listener.concurrency` 对应每个 Listener 对应的 kafka client 数量。
> tips: `@KafkaListener` 其中的 `topicPartitions` 更可以精确控制消费的 partition

这字面意思为监听器的并发数，该配置会在我们的一个应用实例上建立对应数量的 kafka client。
从 spring 所做的实现上来看，每个 kafka consumer client 由一个 `ListenerConsumer` 包装，并由 `MessageListenerContainer` 持有。如果设置了并发数 `concurrency`，那么则会使用装饰器模式使用 `ConcurrentMessageListenerContainer` 进行包装，组合多个 `KafkaMessageListenerContainer`。这对于每个持有 `@KafkaListener` 注解的端点都是如此。

`Container` 启动之后会将 `ListenerConsumer` （实现了 `Runnable`）提交至一个异步执行器：
``` java
// KafkaMessageListenerContainer
protected void doStart() {
    // ...
    this.listenerConsumerFuture = containerProperties
				.getConsumerTaskExecutor()
				.submitListenable(this.listenerConsumer);
}
```
其中 `ConsumerTaskExecutor` 一般是一个简单的异步线程处理。继续看 `ListenerConsumer.run()`：
``` java
// KafkaMessageListenerContainer$ListenerConsumer
@Override
public void run() {
    // ...
    while(isRunning()) {
        // ...
        ConsumerRecords<K, V> records = this.consumer.poll(this.containerProperties.getPollTimeout());
        // ... 以自动提交模式为例
        invokeListener(records);
    }
    // ...
}
```
使用 kafka 客户端拉取消息后，使用对应的 MessageListener 适配器进行消息处理 `onMessage(ConsumerRecord)` 

所以，尽管配置了分片数量，并使用 concurrency 配置提高了 kafka 客户端数量，但是对于每一个 kafka client，他们都是**同步**进行消息消费的！如果业务非 CPU 密集型任务，有一定的 IO 操作，需要应用手动进行消息的异步处理。如通过信号量或线程池等手段来增加吞吐量。
