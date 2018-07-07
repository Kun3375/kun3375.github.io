---
title: Dockerfile 中三个运行指令的差异
date: 2018-07-07 16:43:37
tags: Docker
categories: Docker
---

在描述 Dockerfile 的时候，对于 `RUN`，`CMD`，`ENTRYPOINT` 三个命令，用法十分相似，功能也差不多，容易让人混用。其实一般来说，三个命令都能完成需要的操作，而差异点常常被一些使用者忽略。这里简单说一下，三个命令的不同之处。

<!-- more -->

## 命令格式 ##
首先看一下 Dockerfile 中如何执行命令。
在 Dockerfile 中，命令（instruction）一般有两种写法，分别是：
- **Shell** 格式：`INSTRUCTION <command> <option> <param>`
- **Exec** 格式：`INSTRUCTION ["command", "option", "param"]`

两个格式基本没有差异，除了可读性之外，对于 Shell 格式的命令，Docker 会自动使用 /bin/bash -c 来进行解析，可以是解析命令中的变量比如 `$JAVA_HOME`。而如果使用 Exec 格式执行时需要解析环境变量，需要进行修改，比如：`CMD ["/bin/bash", "-c", "echo", "java home is $JAVA_HOME"]`。
对于 `RUN`，`CMD`，`ENTRYPOINT` 三者同样遵守此规则。

## RUN 命令 ##
`RUN` 命令在 Dockerfile 中**可以多次使用**，所以通常被用来在构建容器时执行一些必须的前置命令，比如安装软件等。它的出现频率远高于其他两个执行命令。格式：
- `RUN <command> <option> <param>`
- `RUN ["command", "option", "param"]`

带来一个 Shell 风格的安装 Git 的例子：
``` dockerfile
RUN apt-get update && apt-get install -y git
```

这里使用 **&&** 可以使得在同一层镜像层上更新 apt 并下载 git。

## CMD 命令 ##
`CMD` 命令的格式有三种，前两者都是用来**定义容器的默认行为**，后者用来给 `ENTRYPOINT` 命令提供**额外的可替换参数**。
1. `CMD <command> <option> <param>`
2. `CMD ["command", "option", "param"]`
3. `CMD ["param"...]`

先说前两者用作执行默认命令的情况，以一个官方 Nginx 的 Dockerfile 为例：
``` dockerfile
# 前置步骤忽略
CMD ["nginx", "-g", "daemon off;"]
```

该命令使得以该 Nginx 镜像构建的容器，在启动时候默认地自动运行 Nginx。但是既然是定义**默认**行为，那么它是可以在运行容器的时候被更改的，比如像下面这样：
``` bash
sudo docker run -p 80:80 nginx echo hello-world
```
那么这个时候容器并不会启动一个 Nginx 服务，而是打印了 hello-world。并且这个容器会随着打印命令的结束而停止。

需要注意的是，既然 `CMD` 定义**默认**行为，那么它在 Dockerfile 中只能存在一个（如果定义了多个 `CMD`，那个最后一个有效）

那么如何使用 `CMD` 为 `ENTRYPOINT` 提供额外参数呢？先看一下 `ENTRYPOINT` 的用法。

## ENTRYPOINT 命令 ##
`ENTRYPOINT` 通常用来**定义容器启动时候的行为**，有点类似于 `CMD`，但是它不会被启动命令中的参数覆盖。
上一节中 Nginx 的例子，如果将 `CMD` 改成 `ENTRYPOINT`，那么我们的 hello-world 方案便行不通了。

`ENTRYPOINT` 同样支持两种格式的写法，并且是存在差异的（后文描述）：
1. `ENTRYPOINT <command> <option> <param>`
2. `ENTRYPOINT ["command", "option", "param"]`

使用 Exec 风格的写法支持使用 `CMD` 拓展**可变参数**和**动态替换**参数，而使用 Shell 风格时**不支持**。假如我们在 Dockerfile 中：
``` dockerfile
# 前置步骤忽略
ENTRYPOINT redis-cli -h 127.0.0.1
```

那么这个由此构建的容器在运行时**只能**将 redi 连接到容器本身的 redis-server 上。
修改这个 Dockerfile：
``` dockerfile
# 前置步骤忽略
ENTRYPOINT ["redis-cli"]
CMD ["-h", "127.0.0.1"]
```
这时候，如果按默认的启动方式，容器的 redis-cli 会自动连接 127.0.0.1，即以下命令此时是等价的：
``` bash
docker run my-redis-image
docker run my-redis-image -h 127.0.0.1
```

但是由于 Dockfile 中使用了 Exec 格式的 `ENTRYPOINT`，我们已经可以修改它的目标地址了，甚至可以增加额外的参数：
``` bash
docker run my-redis-image -h server.address -a redis_password
```

其中 `-h server.address` 替换了 `CMD ["-h", "127.0.0.1"]`，而 `-a redis_password` 则是由于使用了 Exec 格式可以增加参数。

---
关于 `RUN`，`CMD`，`ENTRYPOINT` 的差异已经描述完了，有没有都牢记于心呢~
