---
title: RocketMQ 配置探索
author: 斯特拉不用电
date: 2018-07-28 14:48:31
tags:
  - MQ
  - RocketMQ
categories: RocketMQ
comments: true
---
目前被广泛使用的 MQ 有很多，包括 ActiveMQ，Kafka，RabbitMQ，RocketMQ 等等，它们各有长短。而近期所在项目选择了 RocketMQ 作为消息中间件，此前并未系统地了解研究，所以趁此机会整理了一些笔记和想法。

<!-- more -->

## 优势 ##
简单地说一下在这么多消息中间件中的选型优势。作为阿里的开源项目，想必还是可靠的，尤其是经受过双十一的考验令人信服。
- 支持严格的消息顺序；
- 支持 Topic 与 Queue 两种模式；
- 亿级消息堆积能力；
- 比较友好的分布式特性；
- 同时支持 Push 与 Pull 方式消费消息；

## 基本概念 ##
- **Producer**：消息生产者，生产消息。
- **Consumer**：消息消费者，消费消息。
    - **Pull Consumer**：消费者拉模型的实现。通过与 Broker 建立长连接，从中主动拉取消息。
    - **Push Consumer**：消费者推模型的实现。本质仍然是建立长连接，但是通过注册监听器，在收到消息时回调监听方法。
- **Producer Group**：生产者集合，通常包含发送逻辑一致的消费者，影响事务消息的流程。
- **Consumer Group**：消费者集合，通常包含消费逻辑一致的消费者，影响着负载均衡和集群消息。
- **Name Server**：注册服务器，可以由一到多个近乎无状态的节点构成，扮演者类似 Zookeeper 的角色。Broker 向其中注册，而 Producer 和 Consumer 向其中拉取 Broker 地址。
- **Broker**：核心组件，保存和转发消息。

拓扑结构如下：
![RocketMQ Network](/images/rocketmq-net.png)

## 初次使用 ##
### 下载 ###
RocketMQ 是纯 Java 语言的实现，你可以从 Github 上[下载](https://github.com/apache/rocketmq)源码并使用 Maven 进行编译，当然也可以从[官网入口](http://rocketmq.apache.org/)下载。

### 启动 ###
第一次启动，简单地测试一下效果，进入 bin 目录，使用 nohup 启动一下 NameService：

    nohup ./mqnamesrv -n 127.0.0.1:9876 &

然后启动一下 Broker：

    nohup ./mqbroker -n 127.0.0.1:9876 &
    
还有一个 mqadmin 也是常用的工具，包含的管理员常用的功能，包含查看集群列表，查看、删除主题等，可以直接通过 `./mqadmin` 获得帮助。

### 测试 ###
在 RocketMQ 顺利启动之后，进行一下测试吧，快速的体验一把。
从 MavenRepository 找到对应的 RocketMQ 客户端：
``` xml
<!-- https://mvnrepository.com/artifact/org.apache.rocketmq/rocketmq-client -->
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>4.2.0</version>
</dependency>
```
或者打开刚才从 Github 上下载的源码，example 模块下提供了许多测试用例，附上略微改动的生产者和消费者代码：

- 生产者
``` java
public static void main(String[] args) throws MQClientException, InterruptedException {

    DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
    producer.setNamesrvAddr("127.0.0.1:9876");
    producer.setInstanceName("p001");
    // 可以设定失败重试次数
    producer.setRetryTimesWhenSendFailed(3);
    producer.start();

    for (int i = 0; i < 1; i++)
        try {
            {
                Message msg = new Message("TopicTest1",
                    "TagA",
                    "key113",
                    "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
                SendResult sendResult = producer.send(msg);
                System.out.printf("%s%n", sendResult);

                QueryResult queryMessage =
                    producer.queryMessage("TopicTest1", "key113", 10, 0, System.currentTimeMillis());
                for (MessageExt m : queryMessage.getMessageList()) {
                    System.out.printf("%s%n", m);
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    producer.shutdown();
}
```
- 消费者
``` java
public static void main(String[] args) throws InterruptedException, MQClientException {

    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("ConsumerGroupName");
    // 指定 NameServer 的地址，多个 NameServer 使用 ; 隔开
    consumer.setNamesrvAddr("127.0.0.1:9876");
    consumer.setInstanceName("c001");
    // 指定订阅的 Topic 以及 Tag，多个 Tag 使用 || 分开，* 代表全部 Tag
    consumer.subscribe("TopicATest1", "TagA");
    // 可以设定开始消费的位置，仅针对 Push Consumer
    consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
    // 可以设定批量消费数量，默认 1，不保证每次的数量，近针对 Push Consumer
    consumer.setConsumeMessageBatchMaxSize(1);

    consumer.registerMessageListener(new MessageListenerConcurrently() {
        @Override
        public ConsumeConcurrentlyStatus consumeMessage(
            List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
            for (MessageExt msg : msgs) {
                System.out.println(new String(msg.getBody()));
            }
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;  
        }
    });
    consumer.start();
    System.out.println("Consumer Started.");
}
```
动手运行一下吧。

## 关于配置 ##
事实上，大多数小伙伴在 RocketMQ 启动时都明显能感觉电脑卡卡的，是因为 RocketMQ 默认需求的内存太大了。那么，如何查看和修订所需要的配置呢？
之前我们通过 `./mqbroker` 启动了 Broker，那么来看一下 mqbroker 的脚本，注意脚本末尾的命令：
``` bash
# 省略 ROCKETMQ_HOME 的配置
sh ${ROCKETMQ_HOME}/bin/runbroker.sh org.apache.rocketmq.broker.BrokerStartup $@
```
这里将启动命令转移给了 runbroker.sh 进行执行。

### JVM 参数配置 ###
既然如此，继续查看一下 runbroker.sh：
``` bash
#!/bin/sh
#===========================================================================================
# Java Environment Setting
#===========================================================================================
error_exit ()
{
    echo "ERROR: $1 !!"
    exit 1
}

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
export CLASSPATH=.:${BASE_DIR}/conf:${CLASSPATH}

#===========================================================================================
# JVM Configuration
#===========================================================================================
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

numactl --interleave=all pwd > /dev/null 2>&1
if [ $? -eq 0 ]
then
	if [ -z "$RMQ_NUMA_NODE" ] ; then
		numactl --interleave=all $JAVA ${JAVA_OPT} $@
	else
		numactl --cpunodebind=$RMQ_NUMA_NODE --membind=$RMQ_NUMA_NODE $JAVA ${JAVA_OPT} $@
	fi
else
	$JAVA ${JAVA_OPT} $@
fi
```
通过这个文件可以获得很多信息：
- RocketMQ 的 JVM 配置信息
    - 需求的内存空间达到了 8g，声明的最大堆外内存达到了 15g，这就是电脑变得卡卡的的原因了。
    - 可以在启动时配置 **JAVA_OPT_EXT** 变量来配置额外的参数或者覆盖默认配置。
- 结合 mqbroker.sh 可以发现，最终使用了 BrokerStartup 来启动 RocketMQ，命令行中的参数同时会被传递。

### Broker 实例配置 ###
那么接下来就去 BrokerStartup 查看一下 RocketMQ 的启动过程。由于这个文件实在是太过冗长，这里不再贴出，感兴趣的小伙伴请自行查看。在这个文件中，主要对命令行中几个具体参数进行了解析：
- `-m`：列出所有的配置项
- `-p`：列出所有的配置项以及默认值
- `-c`：指定一个 properties 文件，读取其中的内容覆盖默认配置并情动

#### 自定义配置 ####
所以，很多时候的做法是通过 `sh mqbroker -p > mqbroker.properties` 来获得一份默认配置文件（网上的方案可能不太准确，具体输出是携带 Rocket 的日志信息的，需要 sed 或者 awk 之类加工处理一下），在此基础上进行配置自定义，然后通后通过 `sh mqbroker -c mqbroker.properties` 来进行定制化的启动。

#### 默认配置方案 ####
同时在 conf 目录下，官方也给出了几种典型的配置方案供参考：
- 二主二从异步复制：2m-2s-async 文件夹。这是最典型的生产配置，双 master 获得高可用性，同时主从间的数据同步由异步完成。
- 二主二从同步复制：2m-2s-sync 文件夹。除了双 master 的配置，主从间的数据是同步的，也就是说只有在向 salve 成功同步数据才会向客户段返回成功。这保证了在 master 宕机时候消息仍然可以被实时消费，但是性能收到一定影响。
- 二主无从：2m-nosalve 文件夹。双主模式仅仅保证了 RocketMQ 的高可用性，然而在一台 master 宕机后，客户端无法消费那批在宕机 master 上持久化的消息，直到宕机 master 恢复正常。当然这个方案节省了硬件资源。

三种默认配置方案都是采用了异步刷盘，尽管在刷盘间隙宕机会丢失少量数据，但是效率提升可观。

#### 参考配置类 ####
Broker 的具体配置分为了具体的四个方面：
- Broker 实例配置：参考源码 `org.apache.rocketmq.common.BrokerConfig`
- Netty 服务端配置：参考源码 `org.apache.rocketmq.remoting.netty.NettyServerConfig`
- Netty 客户端配置：参考源码 `org.apache.rocketmq.remoting.netty.NettyClientConfig`
- Message 持久化配置：参考源码 `org.apache.rocketmq.store.config.MessageStoreConfig`

关于 RocketMQ 的启动和配置，就先告一段落。
