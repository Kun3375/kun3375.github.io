---
title: MySQL 主从复制
author: 斯特拉不用电
date: 2019-01-19 12:56:33
categories: MySQL
tags: MySQL
comments: true
---

最近工作也比较忙，很久没有写新的日记了。开始重新整理一下知识点，先找一个有意思点开始吧。

### 复制原理

这个原理其实十分简单，只要搜索一下，那些搜索引擎就会给出图示和合理的答案：
1. 主库开启二进制日志（或者叫主机日志）bin-log。这会使得数据库在更新时记录下它的操作动作。
2. 从机开启 slave 模式，使用一个 IO 线程去请求 master 的数据，记录在中继日志 relay-log 中；master 方面有一个 dump 线程配合传输这份数据。
3. 从机的另一个 SQL 线程读取中级日志内容并解析后执行相应的操作。

### 复制方案

通常的，我们做主从复制，不仅仅为了数据备份。同时会在从机上设置只读，对 MySQL 集群做读写分离，提高数据库的响(wu)应(jin)速(qi)度(yong)，所以一般都采用了**异步复制**方案。主机采用完全独立的 dump 线程来传输 bin-log，备份完全异步化。
<!-- more -->
另一个方案是**半同步复制**。通过选择性开启半同步复制插件开启。主机在每次 bin-log 刷盘时通知 dump 线程传输数据并挂起，在完成给从机的传输后唤醒原线程返回客户端结果。这可以保证至少一个从机能获得完整的 bin-log 所对应 relay-log 信息，但是不能保证从机 SQL 执行完毕；同时，从机的卡顿、阻塞会影响主机的稳定。应用面不广。

### 主机参数

首先需要修改主机的 MySQL 配置文件 `my.cnf` 或者（更推荐）更改自定义配置文件（需要在主配置文件中指定目录），一般在 `/etc/mysql/conf.d/` 下。选几个主要参数说明一下：
- `seriver-id=1`  
服务端编号，值选个大于零的数就好，一般会用个 ip 啥的，没什么好说的，用来区别不同 MySQL 实例。
- `log-bin=/var/lib/mysql/binlog`  
用来指定 bin-log 生成的路径和名称前缀，其实以上就是默认值和数据文件在一起，最后生成的 bin-log 是 */var/lib/mysql/binlog.xxxxxx*。
- `binlog-format=mixed`  
指定二进制信息的形式，有三种选项：
    - **row** 拷贝命令结果
    - **statement** 拷贝命令，随机函数等命令无法精准复制
    - **mixed** 推荐，默认拷贝命令，不佳时拷贝结果  
- `binlog-ignore-db=mysql`  
指定需要忽略的库，不记录 bin-log。多个写多次。  
- `binlog-do-db=db_business`  
指定需要同步的库，以上二选一即可。多个写多次。  
- `expire_logs_days=15`  
可选，设置 bin-log 过期天数，到期自动删除。该值默认为 0，永不删除。可以使用 purge 命令手动删除 bin-log。也行。  
- `slave_skip_errors=1062`  
可选，设置忽略同步过程中的错误，比如 1062 是主键重复，1032 是数据不存在。  
- `log_bin_trust_function_creators=true`  
为真时，对函数和存储过程一样进行同步。

再讲两个额外的选项，本身和主从复制过程无关，但是会对集群和主从间的数据一致性造成影响：
- `innodb_flush_log_at_trx_commit=1`  
默认值：1。为了减少 IO 创造高效，在 innoDB 每次提交事务的时候，可以不立刻刷盘。设置 0 时：log-buf 每隔一秒同步 bin-log 并刷盘；设置 1 时：事务每次提交都会同步刷盘；2：每次同步 log 文件缓冲，但是每隔一秒刷盘。
- `sync_binlog=1`  
默认值：1。设置多少次提交后进行刷盘，一般配合以上选项使用。这两个选项会对性能造成明显的影响。但是一般对数据一致性不敏感且追求速度的场合会进行调整。

### 从机参数

从机同样修改配置文件。如果使用主机镜像或者拷贝的 docker volume 需要修改 */var/lib/mysql/auto.cnf* 中的 UUID。
- `server-id=2`  
类似主机配置。  
- `binlog-do-db=db_business`  
选择要复制的数据库。多个写多次。
- `binlog-ignore-db=mysql`  
拒绝不复制的数据库。
- `relay_log=/var/lib/mysql/2-relay-bin`  
可选，指定中继日志位置。，启动 slave 模式必然会产生中继日志，默认在 */数据目录/${host}-relay-bin.xxxxxx*  
- `read-only=1`  
从机只读，不要修改数据。

如果从机还作为主机的话：

- `log-bin=secondary-binlog`  
也需要开启 bin-log，其他主机参数也要配置哦。  
- `log_slave_updates=1`  
从机记录复制事件进二进制日志中。

### 操作流程

#### 搭建全新主从数据库环境

先来说说搭建一个全新的主从架构。

主机创建数据同步用户并授权，尽量指定 ip，少给与权限尤其是 **super** 权限，super 权限用户可以无视 read-only...
``` mysql
grant replication slave, replication client on *.* to 'user_slave'@'%' identified by 'Kun3375';
flush privileges;
```

记录下主机 bin-log 位置信息 `File` / `Position`
``` mysql
show master status;
```

启动从机并设置主机信息，并启动 slave 状态
``` mysql
change master to 
master_host='mysql-master', 
master_port=3306, master_user='user_slave', master_password='Kun3375', master_log_file='binlog.000014', master_log_pos=497;
start slave;
```

检查 slave 状态信息。当 `Slave_IO_Running` 与 `Slave_SQL_Running` 同时为 `Yes` 意味着 IO/SQL 两线程正常工作，从机状态正常。
``` mysql
show slave status;
```

出现问题的同学可以检查一下各个配置，再次启动如果存在错误，尝试清除一下脏数据
``` mysql
stop slave;
reset slave;
```

#### 为既有服务器增加从机

首先进行主库的备份操作，使用 `mysqldump` 命令（确保操作用户拥有权限）：
``` shell
mysqldump -uUSERNAME -pPASSWORD --routines --single-transaction --master-data=2 --all-databases > DUAMPFILE.sql
```
选项说明：
- --routines 同时导出存储过程和函数
- master-data 默认值为 1。在备份文件追加 *change master* 主机指定命令。设置 2，可以将该语句追加并注释。该选项会自动打开 *lock-all-tables* 进行锁表，除非使用 *single-transaction* 见下文。
- --single-transaction 开启单一事务备份。不在备份执行时进行锁表，而仅仅在开始时获取 master status 时候锁表（它依然隐含了以下几个短暂的动作）：
    1. FLUSH TABLES WITH READ LOCK
    2. SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ
    3. START TRANSACTION
    4. SHOW VARIABLES LIKE 'gtid\_mode'
    5. SHOW MASTER STATUS
    6. UNLOCK TABLES
- --skip-lock-tables 尽管 *single-transaction* 这可以在不长时间锁表的情况下进行备份，并保证在相应 bin-log 位置下数据的准确。如果数据库在任何时候的访问都十分频繁，可能会无法执行 TABLE LOCK，如果可以接受可能有小部分数据不准确的风险，那么可以使用该参数来跳过获取 master status（即 bin-log 位置）前的锁表动作。
-- --all-databases 可以备份全库
-- --databases 用来指定需要备份的数据库，顺便整理一下导出语句的灵活用法
``` mysql
# 导出多个库，主从备份时只能用这个或者 all-databases
mysqldump --databases DB_NAME_1 DB_NAME_2 > DUMPFILE.sql
# 导出一个库的多个表，不包含建库语句和 use 命令
mysqldump DB_NAME TAB_NAME_1 TAB_NAME_2 > DUMPFILE.sql
# 导出结构不含数据
mysqldump DB_NAME TAB_NAME_1 TAB_NAME_2 > DUMPFILE.sql
```

将备份文件拷贝至从机，从机执行 `source DUMPFILE.sql`。如果之前 *master-data* 设置为 2 需要手动放开注释或者数据导入之后手动执行 *change master* 命令。最后 `start slave` 就好啦，记得使用 `show slave status` 确认从机状态。

#### 为现有主从体系新增从机

新增从机的我们可以在完全不对主库进行操作以减少风险。可以在从机上使用 `mysqldump` 并将 *master-data* 替换为 *dump-slave*，从从机导出数据并记录主机的 bin-log 位置。选项数值意义一致。需要注意从机在 dump 时候会暂停复制的 IO 线程。需要确保是否可以接受这样的同步延迟，考虑暂停某从机的使用。

