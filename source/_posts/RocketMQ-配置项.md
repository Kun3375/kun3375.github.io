---
title: RokcetMQ 配置项
author: 斯特拉不用电
date: 2018-08-11 23:20:45
tags:
  - MQ
  - RocketMQ
categories: RocketMQ
comments: true
---

RocketMQ 的配置分为两部分，一者是 JVM 的配合，另一者则是对 Broker 应用本身的参数配置。
在初次接触时候，除了 RocketMQ 本身的一些特性，同时也难免会被一些配置给迷惑或者踩坑，这里来看一下通常的配置点。
<!-- more -->

## Broker JVM 配置
JVM 的配置默认不需要修改，只需要根据硬件情况调整相应的堆栈内存和对外内存的占用量即可。附上启动时的 JVM 配置脚本片段：
``` shell
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"
JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC -XX:G1HeapRegionSize=16m -XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30 -XX:SoftRefLRUPolicyMSPerMB=0 -XX:SurvivorRatio=8 -XX:+DisableExplicitGC"
JAVA_OPT="${JAVA_OPT} -verbose:gc -Xloggc:/dev/shm/mq_gc_%p.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime -XX:+PrintAdaptiveSizePolicy"
JAVA_OPT="${JAVA_OPT} -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m"
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT} -XX:+AlwaysPreTouch"
JAVA_OPT="${JAVA_OPT} -XX:MaxDirectMemorySize=15g"
JAVA_OPT="${JAVA_OPT} -XX:-UseLargePages -XX:-UseBiasedLocking"
JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${BASE_DIR}/lib"
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n"
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"
```
需要额外关注的点在于：
* `-Xms8g -Xmx8g -Xmn4g` 默认 Broker 需要 8g 的堆内存，不要轻易在自己的笔记本上运行哦 😂
* `-XX:MaxDirectMemorySize=15g` 默认的最大堆外内存为 15g，nio 通过内存映射文件所提高 IO 效率而用。
* `JAVA_OPT_EXT` 该环境变量可以追加和替换原有的配置

## Broker 应用配置

### 自定义配置启动
启动 Broker 时可以自定义配置：`sh bin/mqbroker -c CONFIG.properties`

### 配置可选项
* 获取可配置项的列表：`sh bin/mqbroker -m`
* 获取配置项以及默认值：`sh bin/mqbroker -p`
* 源码中配置类：`BrokerConfig` / `NettyServerConfig` / `NettyClientConfig` / `MessageStoreConfig`

### 配置参数介绍
介绍几个常用的，或者说通常需要配置的选项。
* `namesrvAddr=IP:PORT;IP:PORT`  
配置 NameServer 的地址，多个地址间使用 `;` 隔开，该选项没有默认值，可以启动时通过 `-n` 参数设置
* `brokerClusterName=DefaultCluster`  
配置 RocketMQ 集群的名称，默认为 DefaultCluster
* `brokerName=broker-a`  
Broker 的名称，在同一 NameServer 群下，只有使用相同的 brokerName 的 Broker 实例才可以组成主从关系
* `brokerId=0`  
在一个 Broker 群下（都使用了同样的 brokerName），所有实例通过 brokerId 来区分主从，主机只有一个：`brokerId=0`（默认）
* `fileReservedTime=48`  
消息数据在磁盘上保存的时间，单位：小时，默认：48
* `deleteWhen=04`  
在指定的时间删除那些超过了保存期限的消息，标识小时数，默认：凌晨 4 时
* `brokerRole=SYNC_MASTER`  
有三种选项，前两者主要描述 Broker 实例间的同步机制
    * `SYNC_MASTER`  
    Broker Master 的选项，消息同步给 Slave 之后才返回发送成功状态
    * `ASYNC_MASTER`  
    Broker Master 的选项，主从间消息同步异步处理
    * `SLAVE`  
    Broker Slave 的选项（没得选）
* `flushDiskType=ASYNC_FLUSH`  
有两种选项，分别同步或异步的刷盘策略
    * `SYNC_FLUSH`  
    消息只有在真正写入磁盘之后才会返回成功状态，牺牲性能，但可以确保不丢失消息
    * `ASYNC_FLUSH`  
    异步刷盘，消息写入 page_cache 后即返回成功
* `brokerIP1=127.0.0.1`  
设置 Broker 对外暴露的 IP，通常 Broker 启动时会自动探测，但是由于容器环境或者多网卡的影响，通常需要手动设置。需要多个暴露 IP 时，可以使用 `brokerIP2/3/4/...` 的方式配置
* `listenPort=10911`  
Broker 实例监听的端口号
* `storePathRootDir=/home/rocketmq/store-a`  
存储消息和一些配置的根目录

## 参考
* [RocketMQ on GitHub](https://github.com/apache/rocketmq)
* 《RocketMQ 实战与原理解析》机械工业出版社 杨开元
