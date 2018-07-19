---
title: Docker 挂载与数据存储
author: 斯特拉不用电
date: 2018-07-09 21:54:45
tags: Docker
categories: Docker
comments: true
---

Docker 镜像是层层分离的，分为只读的底层和可读写的当前层。容器在运行时候如果有文件改动，会自动从含有该文件的底层拷贝并更新在当前层。如果容器在 commit 之前被销毁，那么有这个镜像重新生成的容器是不包含改动的内容的。

<!-- more -->

## 需求来源 ##

所以数据问题是使用 Docker 必然会关注到的，除了如何持久化以外，如何从宿主机访问到容器内的数据？或者将容器内的数据同步到宿主机？又或是多个容器间怎么共享数据？这都是要处理的。

还好 Docker 提供了一套完善而简单的数据挂载机制 Volume。

## 命名卷 ##
要控制容器的数据内容，首先从文件的挂载开始。`docker volume` 提供了一套管理卷（volume）的 API：
```
Usage:  docker volume COMMAND

Manage volumes

Commands:
  create      Create a volume
  inspect     Display detailed information on one or more volumes
  ls          List volumes
  prune       Remove all unused local volumes
  rm          Remove one or more volumes
```

先创建一个 named volume：`docker volume create vol`
使用 `docker volume ls` 可以看到当前存在的所有的 volume。
使用 `docker volume rm` 删除指定的 volume。
使用 `docker volume inspect vol` 可以看到它的详情，包括创建时间和宿主机上的真正挂载位置等：

    [
        {
            "CreatedAt": "2018-07-09T14:53:05Z",
            "Driver": "local",
            "Labels": {},
            "Mountpoint": "/var/lib/docker/volumes/vol/_data",
            "Name": "vol",
            "Options": {},
            "Scope": "local"
        }
    ]
    
可以看到这个新建的 vol 保存在 `/var/lib/docker/volumes/vol/_data` 下，其中 `/var/lib/docker/volumes` 目录保存了所有的 Docker volume。
    
## 运行时挂载 ##
### 使用命名卷 ###
有了 volume 之后，我们便可以使用刚才创建的 vol 来挂载容器中的某个目录：
``` bash
docker run -d -v vol:/data --name temp-redis redis
```

如此一来，在 temp-redis 容器中 `/data` 下的改动，都会同步反映在宿主机的 `/var/lib/docker/volumes/vol/_data` 下；而宿主机的改动亦然。

### 使用绝对路径 ###
使用命名的 volume 通常是为了数据共享，很多时候我们只是想指定一个挂载的目录便于记忆和管理，这个时候可以使用绝对路径，如：
``` bash
docker run -d -v /data/docker_volume/temp_redis/data:/data --name temp-redis redis
```

### 自动挂载 ###
有时候你甚至不想关心任何 volume 的形式，你只是想把某个目录持久化而已，可以这么做：
``` bash
docker run -d -v /data --name temp-redis redis
```

Docker 会自动生成一个匿名的 volume。想要知道具体的宿主机目录可以使用：`docker inspect` 来查看。这种情况通常在构建纯数据容器时使用。

### 注意点 ###
在 volume 创建时和创建之后仍然需要关注他们，下面是一些典型的问题。

#### volume 自动创建 ####
事实上在 `-v vol:/data` 时候，vol volume 甚至不需要提前使用 `docker volume create vol` 创建，使用 `docker run -v vol:/data` 命令时便会自动创建。
**同时地，也意味着 `-v` 选项不支持相对路径的使用**。

#### volume 的删除 ####
在一个 volume 被创建之后，想删除可没那么容易，即使使用了 `docker rm CONTAINER` 删除了容器，volume 依然保留着。除非：
- 使用 `docker run --rm` 启动的容器停止时。它除了会删除容器本身还会删除挂载的**匿名** volume。
- 使用 `docker rm -v` v 参数可以删除容器和创建容器时关联的**匿名** volume。

那么我们在使用了 named volume 或者删除容器时忘记了 `-v`，那么那些在 `/var/lib/docker/volumes` 下的一些文件就成了**僵尸文件**。怎么删除呢？
- 使用 `docker volume rm VOLUME` 来删除。
- 使用 `docker volume prune` 删除所有不再被使用的 volume。

#### volume 的只读控制 ####
一些场合下，我们提供只是需要容器内的程序去读取一些内容而非修改。我们可以严格的控制 volume，增加只读选项 `ro`：
``` bash
docker run -d \
  --name=nginxtest \
  -v nginx-vol:/usr/share/nginx/html:ro \
  nginx:latest
```

## 通过 Dockerfile 挂载 ##
可以通过 Dockerfile 在构建镜像的时候便指定需要的 volume，这对于很多应用都是必要的，尤其是一些数据类应用。
Dockerfile 中使用 VOLUME 表明挂载目录，如：
``` dockerfile
VOLUME ["/data1", "/data2"]
```

任何通过该镜像构建的容器都会将 `/data1`，`/data2` 两个目录进行挂载。但是 Dockerfile 形式的弱势是**无法进行 named volume 或者绝对路径的挂载**。

## 数据共享与存储 ##
### 共享 volume ### 
既然数据可以在宿主机和容器间同步，那么可以使多个容器间同步吗？当然可以！
1. #### --volumes-from ####
``` bash
# 首先创建一个容器，并挂载 /data
docker run -d -v /data --name ng1 nginx
# 创建第二个容器，共享前者的 volume
docker run -d --volumes-from ng1 --name gn2 nginx
```
2. #### named volume ####
``` bash
# 首先创建一个容器，并创建命名卷 share 来挂载 /data
docker run -d -v share:/data --name ng1 nginx
# 创建第二个容器，使用同样的 /data
docker run -d -v share:/data --name ng2 nginx
```
**两种方式都能达到数据共享的目的，但是由于通过命名卷的方式对多个容器的依赖进行了解耦，所以推荐第二种。**

### 数据容器 ###
这个话题事实上和数据共享紧密相关，由于在 Docker1.9 之前，大家广泛使用使用 `--volumes-from` 的形式来共享卷，导致必须要一个底层的没有依赖的容器来保存数据。
- 通常使用和应用容器一样的镜像来构建数据容器
- 数据容器**不需要也不应该启动**，仅仅是利用 volume 机制来保持数据而已

然而现在有了命名卷，完全不需要数据容器的存在了，使用 named volume 可以更直接更方便的管理共享数据。
