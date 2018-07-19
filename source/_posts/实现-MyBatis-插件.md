---
title: 实现 MyBatis 插件
author: 斯特拉不用电
date: 2018-07-18 21:35:05
tags: 
  - Java
  - MyBatis
categories: Java
comments: true
---
MyBatis 作为一个目前很常用的持久化框架，有着丰富的拓展。这些拓展功能常常以插件的形式嵌入到 MyBatis 的运作流程之中，而如何制作实现一个插件？MyBatis 已经为大家设计好了，一个 `Interceptor` 接口，实现它就够了。

`Interceptor` 接口的拦截目标，是 MyBatis 运作流程中的几个核心组件：
- `Executor`：这是 MyBatis 执行器，控制着所有和数据库交互的操作，也影响着一级缓存。
- `ParameterHandler`：参数处理器，在映射参数时候生效。
- `ResultSetHandler`：结果集处理器，在处理结果集的时候会用到。
- `StatementHandler`：`Executor` 下层的处理器，同样控制着 SQL 行为，也控制着二级缓存的生效。

<!-- more -->

这几个组件就简称*处理器对象*吧，感兴趣的话，可以跟进资料，这里继续来讲插件如何拦截它们以及如何实现一个插件。

## Interceptor 接口 ##
`Interceptor` 接口是插件的核心，看一下它的接口：
``` java
public interface Interceptor {
  // 拦截后的逻辑
  Object intercept(Invocation invocation) throws Throwable;
  // 将处理器对象包装成代理类
  Object plugin(Object target);
  // 初始化属性赋值
  void setProperties(Properties properties);
}
```
- `intercept()`：拦截 MyBatis 的执行过程，需要在其中加入定制的逻辑。
- `plugin()`：可以理解为插件的构造过程，通常把 MyBatis 的几个 handler 包装成代理用。
- `setProperties()`：用于插件初始化时候的属性赋值。如果你有其他的赋值方案，也可以不采用它。

我们从第一个方法开始讲起。

### Object intercept(Invocation invocation) ###
入参 `Invocation` 是一个 MyBatis 封装的对象，包含了运行时的信息：
- 属性`Method method`：即反射包中的 Method，在这里它是当前运行的方法。
- 属性`Object[] args`：方法的参数列表
- 属性`Object target`：这里其实是你选择拦截的处理器对象（关于如何选择拦截具体的处理器对象，稍后再述），也就是说，它可以是 `Executor` / `StatementHandler` ...，需要使用时可以直接强转。
- 方法 `proceed()`：让处理器继续流程，或者调用下一个插件，你可以用 `Filter.doFilter()` 来类比它。

MyBatis 插件是通过动态代理实现的，对处理器对象进行代理，由代理对象在方法 `invoke()` 前完成插件中 `interceptor()` 方法（即插件逻辑）。同时多个插件又是多层的代理，每个插件都需要在具体方法调用前完成自己的逻辑，**所以在实现 Interceptor 接口的 intercept 方法最后，一定要记得执行 Invocation.proceed()，以完成插件的调用链**：
``` java
@Override
public Object intercept(Invocation invocation) throws Throwable {
  // 可以通过 invocation 获得处理器对象，进而可以变更参数，埋点，收集信息等
  // do something
  // 最后需要记得完成调用链，否则流程将中段
  return invocation.proceed();
}
```

### Object Plugin(Object) ###
该方法在处理器对象初始化的时候，由 `InterceptorChain.pluginAll()` 调用，将处理器对象包装成代理类。可以理解为一个初始化方法。

以 `StatementHandler` 举例：
``` java
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
  // 初始时触发代理包装
  statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
  return statementHandler;
}

public Object pluginAll(Object target) {
  // 迭代完成所有插件代理，最终返回一个包含所有插件逻辑的处理器对象代理
  for (Interceptor interceptor : interceptors) {
    target = interceptor.plugin(target);
  }
  return target;
}
```

该方法的本质目的是使得新的代理类在拦截的目标方法以及之前的插件逻辑之前添加上新插件的 `intercept()` 方法中的内容。所以该方法 Object 类型的入参与出参自然也就是处理器接口对象了。
在没有特殊需求的情况下，**推荐使用官方工具类** `Plugin.wrap()` 方法来完成：
``` java
@Override
public Object plugin(Object target) {
  return Plugin.wrap(target, this);
}
```
原因嘛...先来看一下 `Plugin.wrap()`：
``` java
public static Object wrap(Object target, Interceptor interceptor) {
  // 插件上都通过 @Interceptors 指定了要拦截的处理器，以及要拦截的方法和参数，收集起来
  // 获得这个插件想拦截的类-方法
  Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
  // 这个 type 必然是 4 大执行器/处理器 接口实现之一
  Class<?> type = target.getClass();
  // 获得原来的所实现的接口，动态代理的必要步骤
  Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
  // 如果该插件没有拦截这个处理器，在上一个方法会返回空数组，这里就不包装了
  if (interfaces.length > 0) {
    return Proxy.newProxyInstance(
        type.getClassLoader(),
        interfaces,
        new Plugin(target, interceptor, signatureMap));
  }
  return target;
}

@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    Set<Method> methods = signatureMap.get(method.getDeclaringClass());
    if (methods != null && methods.contains(method)) {
      return interceptor.intercept(new Invocation(target, method, args));
    }
    return method.invoke(target, args);
  } catch (Exception e) {
    throw ExceptionUtil.unwrapThrowable(e);
  }
}
```
好处在于，不在需要开发者手动构建一个动态代理（`Plugin` 本身就是一个 `InvocationHandler` 实现类），并且在包装成代理的时候，将四个处理器中不需要拦截的类排除了，这使得运行中减少一层不必要的代理，进而提升效率。

## @Intercepts 注解 ##
插件的拦截流程都已经明了，回过来梳理一下如何拦截自己想要的指定的处理器和指定的方法呢？

在实现了 `Interceptor` 接口之后，需要配合 `@Intercpts` 注解一起使用。这个注解中需要安置一个 `Signature` 对象，在其中指定你需要指定：
- type：选择 4 个处理器类之一。
- method：选择了处理器之后，你需要选择拦截那些方法。
- args：选择拦截的方法的参数列表。因为如 `Executor` 中 query 方法是有重载的。

通过以上三者，插件便确定了拦截哪个处理器的哪个方法。MyBatis 的插件实现是不是很简单呢？

需要注意的是，`Exector` 和 `StatementHandler` 在一些功能上类似，但是会影响不同级别的缓存，需要注意。同时由于 `sqlSession` 中这 4 个处理器对象的功能着实强大，并且可以通过拦截改变整个 SQL 的行为，所以如果需要深入定制插件行为的时候，最好需要对 MyBatis 核心机制由一定的了解。

###### 官方介绍 ######
<http://www.mybatis.org/mybatis-3/zh/configuration.html#plugins>
