---
title: RocketMQ 日志设置
author: 斯特拉不用电
date: 2018-08-19 14:18:41
tags:
  - MQ
  - RocketMQ
categories: RocketMQ
comments: true
---

## 日志配置文件位置
RocketMQ 日志基于 slf4j 实现，支持 Logback、Log4j。如果需要指定日志的配置文件的位置有三种方式：
* 环境变量：
`ROCKETMQ_CLIENT_LOG_CONFIGFILE=<custom-file>`
* 启动参数：
`rocketmq.client.log.configFile=<customer-file>`，作为 JVM 变量，启动时时需要增加 -D 标识，优先级也比环境变量更高
* 作为 Java 实现，日志位置信息是通过 `System.getProperty()` 或者 `System,getenv()` 得到的，所以可以在程序入口 `System.setProperty(“rocketmq.client.log.configFile”, customer_file)` 来配置

<!-- more -->

## 日志相关系统变量
* `rocketmq.client.log.loadconfig`
默认 true，是否加载指定配置文件，当设置为 false 时，RocketMQ 客户端会会使用应用本身的日志配置。这可能反而是最简单的日志配置方式
* `rocketmq.client.log4j.resource.fileName`、`rocketmq.client.logback.resource.fileName`、 `rocketmq.client.log4j2.resource.fileName`
三种日志框架的的配置文件名，默认值分别为 log4j_rocketmq_client.xml、logback_rocketmq_client.xml、log4j2_rocketmq_client.xml
* `rocketmq.client.log.configFile`
日志配置文件路径，上述。如果使用了自定义的日志配置文件，通常你不再需要设置以下的变量了
* `rocketmq.client.logRoot`
RocketMQ 日志信息默认存放日志为：$USER_HOME/Logs/rocketmqLogs，通过改变此变量可以变更日志路径
* `rocketmq.client.logLevel`
日志输出级别，默认 INFO
* `rocketmq.client.logFileMaxIndex`
滚动窗口的索引最大值，默认 10