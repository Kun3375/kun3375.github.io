---
title: Ribbon 摘要
date: 2018-07-01 19:25:07
tags: 
  - SpringCloud
  - Ribbon
  - Java
categories: Java
---
## 主要构成 ##

Ribbon 是由 netflix 开源的一个客户端负载均衡组件。从客户端的角度取维护服务间请求的负载均衡，并进行一定的容错处理。
自然的它的核心接口就是：`com.netflix.loadbalancer.ILoadBalancer`。

![ILoadBalancer](/images/ILoadBalancer.png)

在配置 Ribbon 之前，了解一下 Ribbon 的几个重要的组成部分：

<!-- more -->

- **`IRule`**：负载均衡的策略。
- **`IPing`**：检测服务存活的策略。
- **`ServerList<T>`**：拉取服务实例列表的策略。
- **`ServerListUpdater`**：更新服务列表的触发策略。
- **`ServerListFilter<T>`**：服务过滤方案。

### IRule ###

负载均衡策略接口，可能最常需要配置的就是它了，配置也没什么特殊的，就以 Spring 常用的方式即可。通常情况下 Ribbon 原生的几个负载均衡策略应该可以满足生产要求（当然你也可以自定义），来了解一下：

![IRule](/images/IRule.png)

- **`AbstractLoadBalancerRule`** 顶级的抽象类，给予了获得 LoadBalancer 的方法，可以让子类获得负载均衡器的一些信息，定制更具体的算法。
- **`RandomRule`** 通过 LoadBalancer 获取可用的服务实例，并**随机**挑选。（一直无可实例时有无限循环 bug）
- **`RoundRobinRule`** 通过 LoadBalancer 获取可用的服务实例，并**轮询**选择。（超过 *10* 次失败时，打印机警告并返回 null）
- **`RetryRule`** 默认有一个 RoundRobinRule 的子规则，和 *500* 毫秒的阈值。**使用子规则**选择实例，执行时间若超过阈值则返回 null。
- **`WeightedResponseTimeRule`** 继承自 RoundRobinRule。在构造时会启动一个定时任务，默认每 *30* 秒执行一次，来计算服务实例的权重。在默认的算法下，**响应速度**越快的服务实例权重越大，越容易被选择。
- **`ClientConfigEnabledRoundRobinRule`** 本身定义了一个 RoundRobinRule 的子规则。本且默认的 choose 方法也是执行 RoundRobinRule 的实现。本身没有特殊用处，这个默认会实现是在其子类的算法无法实现时采用，通常会选择该类作父类继承，实现自定义的规则，以**保证拥有一个默认的轮询规则**。
- **`BestAvaliableRule`** 通过 LoadBalancerStats 选择**并发请求最小**的实例。
- **`PredicateBasedRule`** 利用**子类**的 Predicate **过滤**部分服务实例后通过轮询选择。
- **`AvailabilityFilteringRule`** **轮询**选择一个服务实例，判断是否故障（断路器断开），并发请求是否大于阈值（默认2^32-1，可通过`<clientName>.<nameSpace>.ActiveConnectionsLimit` 修改）。允许则返回，不允许则再次选择，失败 *10* 次后执行父类方案。
- **`ZoneAvoidanceRule`** 使用组合过滤条件执行**过滤**，每次过滤后会判断实例数是否小于最小实例数（默认1），是否大于过滤百分比（默认0），不再过滤后使用**轮询**选择。

## 具体配置 ##

在项目中是可以配置多个 Ribbon 客户端的，通常来说每个客户端用来访问不同的服务。比如为访问 A 服务的 Ribbon 客户端配置为 A-Client，B 服务为 B-Client。

### 通过代码配置 ###

首先来介绍通过注解配置的方法，像简单的 Sring Bean 配置一样，不过不需要使用 `@Configuration` 注解了。在配置类上启用 `@RibbonClient`，给定客户端的名称和配置类，使用 `@Bean` 来配置具体的组件如 IRule 等。

``` java
@RibbonClient(name = "A-Client",configuration = ARibbonConfig.class)
public class ARibbonConfig {
    // 服务实例的地址
    String listOfServers = "http://127.0.0.1:8081,http://127.0.0.1:8082";
    @Bean
    public ServerList<Server> ribbonServerList() {
        List<Server> list = Lists.newArrayList();
        if (!Strings.isNullOrEmpty(listOfServers)) {
            for (String s: listOfServers.split(",")) {
                list.add(new Server(s.trim()));
            }
        }
        return new StaticServerList<Server>(list);
    }
}
```

官方文档上提示了一个***坑***，不能把加了配置注解的具体的配置类放在 `@ComponentScan` 路径下，否则先扫描到的一个具体的客户端配置会成为 Ribbon 的全局配置。

这怎么能忍？当然得有更优雅的解决方式：全局配置的方案。
使用 @RibbonClients 注解，一次可以描述多个客户端配置类的位置，同时也可以指定默认配置类，如：

``` java
@SpringCloudApplication
@RibbonClients(value = {
    @RibbonClient(name = "A-Client",configuration = ARibbonConfig.class),
    @RibbonClient(name = "B-Client",configuration = BRibbonConfig.class)
}, defaultConfiguration = DefaultConfig.class)
public class DemoServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoServiceApplication.class, args);
    }
}
```

这样配置可以不需要再在配置类上加上注解了，也可以不需要将配置类移出包扫描路径。

### 通过配置文件指定 ###

但是以上这样的代码配置还是稍显复杂，在目前的 SpringCloud 中 Ribbon 的配置可以直接 SpringBoot 的配置文件中写入，使用如下的方式指定要配置的参数：

    <nameSpace>.<key>=<value> 
    
默认的命名空间是 ribbon，例如定义连接超时时间可以：

    ribbon.connectTimeout=120

而当需要为具体的客户端配置时，可以使用：

    <client>.<nameSpace>.<key>=<value>

比如：

    user-service.ribbon.listOfServers=localhost:8001,localhost:8002

关于 ribbon 所有的参数名称，可以参看 `com.netfix.client.config.CommonClientConfigKey<T>`。

### 与 Eureka 整合后的配置 ###

在使用 Eureka 的时候，会改变 Ribbon 一些组件的默认实现，如：

- `ServerList` -> `DiscoveryEnabledNIWSServerList`：由 Eureka 来维护服务列表
- `IPing` -> `NIWSDiscoveryPing`：由 Eureka 测试服务存活（原配的 `DummyPing` 并不会 ping，而是始终返回 true）

而且针对不同服务不需要显示地配置不一样的客户端名称了，只需要使用

    <serviceName>.<nameSpace>.<key>=<value>
    
如 user-service：

    user-service.ribbon.ReadTimeout=120
    
同时由于 SpringCloud Ribbon 默认实现区域亲和策略，zone 的配置也十分简单，只需要加入元数据集中即可，如：

    eureka.instance.metadataMap.zone=huzhou
    
如果不需要 Eureka 辅助 Ribbon 的自动配置（这不太可能吧），则可以使用：

    ribbon.eureka.enable=false
    
这时候，记得自己得手动配置 listOfServers 等参数了。
