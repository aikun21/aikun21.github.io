---
title: Java学习笔记（1） - MySQL集群
date: 2024-11-30 20:14:19
tags:
---

# MySQL集群

## 1. 数据库的安装

安装MySQL集群，需要先安装MySQL的`5.7.41`版本，不能使用8.0版本。

### 1.1 安装MySQL

![](java_mysql.png)

由于Docker容器的IP地址是动态分配的，每次启动相同容器，它的IP地址都会有变化。这就非常影响数据库集群的搭建，比如说MySQL 2要同步MySQL 1的数据，MySQL1容器的IP地址经常变来变去肯定是不行的，所以我们要给每个Docker容器都分配固定的IP地址。

Docker默认的网段是 172.17.0.x的，有时候不同项目用到的容器都在同一个网段里面，难免我们使用的时候会弄混淆了。所以不同项目用到的容器最好放在不同的网段。于是我们要为咱们的项目创建一个新的网段，创建容器的时候，把它们的IP地址绑定到该网段。执行下面的命令，创建新的网段:

```bash
docker network create --subnet=172.18.0.0/18 mynet
```

## 2 创建MySQL_1容器

```bash

docker run -it -d --name mysql_1 -p 7001:3306 \
--net mynet --ip 172.18.0.2 \
-m 400m -v ./mysql_1/data:/var/lib/mysql \
-v ./mysql_1/config:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=abc123456 \
-e TZ=Asia/shanghai --privileged=true \
mysql:5.7.41 \
--lower_case_table_names=1

```

用Navicat连接本地电脑的7001端口。如果你用的是云主机，就去连接云主机的7001端口。

在Navicat上面给MySQL_1创建一个新账户，将来MSQL_2和MySQL_3订阅binlog日志的时候就用这个帐户登陆MySQL_1节点。

![](image1.png)

我们切换到服务器权限选项卡，然后勾选下面的三个权限，这三个权限都是订阅日志所必须的。
![](image2.png)

接下来我们要修改配置文件，所以要先关闭MySQL容器。运行Docker命令，关闭MySQL 1容器

```bash
docker stop mysql_1
```
在 `/root/mysql_1/config` 目录里创建 `my.cnf` 文件。配置文件的内容如下:

```conf
[mysqld]
#数据库字符集
character_set_server = utf8
#MySQL编号(只可以是数字)
server_id = 1
#开启binlog日志，规定日志文件名称
log_bin = mysql_bin
#开启relaylog日志，规定日志文件名称
relay_log = relay_bin
#从库的写操作是否写入binlog日志
log-slave-updates = 1
#采用严格的SOL语句模式
sql_mode = NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
```

改好配置文件，我们就可以重新启动MySQL 1节点了:

```bash
docker start mysql_1
```

## 3 创建MySQL_2容器

创建MySQL 2节点的命令如下，用该命令我们创建MySQL 2节点出来。容器创建成功后，你可以用Navicat去连接试试。

```bash
docker run -it -d --name mysql_2 -p 7002:3306 \
--net mynet --ip 172.18.0.3 \
-m 400m -v ./mysql_2/data:/var/lib/mysql \
-v ./mysql_2/config:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=abc123456 \
-e TZ=Asia/shanghai --privileged=true \
mysql:5.7.41 \
--lower_case_table_names=1
```

丛节点配置文件内容如下:

```conf
[mysqld]
#数据库字符集
character_set_server = utf8
#MySQL编号(只可以是数字)
server_id = 2
#为什么从节点要开启binlog日志?(下面有解答)
log_bin = mysql_bin
relay_log = relay_bin
#限制普通帐户无法INSERT、DELETE、UPDATE语句，但是该配置对管理员帐户无效
read-only = 1
sql_mode = NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
```

作为从节点来说，为什么要开启binlog日志呢?这是为了将来挂载更多读节点而准备的。现在我们是两个从节点与主节点同步数据，主节点的压力并不大。如果将来挂载更多的从节点，那么主节点的磁盘和网络压力就很大了所以我们要分担主节点的压力。让从节点跟从节点去同步数据。例如下面的示意图，A3要把数据同步给A4，A3节点必须要开启binloq日志才可以。

![](image3.png)

在Navicat上面通过MySQL 2节点执行SQL语句，让MySQL 2订阅MySQL 1的日志文件，实现数据同步

```bash
#停止数据同步服务
stop slave;
#设置与MySQL 1同步数据
change master to master_host='172.18.0.2',master_port=3306,master_user='sync',master_password='abc123456';
#开启数据同步服务
start slave;
#查询数据同步状态
show slave status;
```

如果SQL语句执行结果中出现两个YES，说明主从同步就配置成功了。如果不成功，你就再检查上述的步骤，然后重新运行这几行SQL语句。或者先停掉容器，再删除容器和映射到 /root 目录的mysql目录，重新创建容器，严格遵守每个配置步骤，重新弄一遍主从同步。

![](image4.png)

## 4 创建MySQL_3容器

创建MySQL 3节点的命令如下，用该命令我们创建MySQL 3节点出来。容器创建成功后，你可以用Navicat去连接试试。

```bash
docker run -it -d --name mysql_3 -p 7003:3306 \
--net mynet --ip 172.18.0.4 \
-m 400m -v ./mysql_3/data:/var/lib/mysql \
-v ./mysql_3/config:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=abc123456 \
-e TZ=Asia/shanghai --privileged=true \
mysql:5.7.41 \
--lower_case_table_names=1

```

丛节点配置文件内容如下:

```conf
[mysqld]
#数据库字符集
character_set_server = utf8
#MySQL编号(只可以是数字)
server_id = 3
#为什么从节点要开启binlog日志?(下面有解答)
log_bin = mysql_bin
relay_log = relay_bin
#限制普通帐户无法INSERT、DELETE、UPDATE语句，但是该配置对管理员帐户无效
read-only = 1
sql_mode = NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
```

## 5 创建MySQL_4容器

因为MySQL 4是主节点，功能与MySQL 1是相同的。我们先把节点创建出来，下节课我们再去配置这两个节点的双向同步。

```bash
docker run -it -d --name mysql_4 -p 7004:3306 \
--net mynet --ip 172.18.0.5 \
-m 400m -v ./mysql_4/data:/var/lib/mysql \
-v ./mysql_4/config:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=abc123456 \
-e TZ=Asia/shanghai --privileged=true \
mysql:5.7.41 \
--lower_case_table_names=1
```

还是照例创建 sync 帐户，密码是 abc123456，然后设置好三个权限。接下来还是老规矩，停掉MySQL容器:

```bash
docker stop mysql_4
```

在 /root/mysql 4/config 目录里创建 my.cnf 文件。配置文件的内容如下:

```conf
[mysqld]
#数据库字符集
character_set_server = utf8
#MySQL编号(只可以是数字)
server_id = 4
#开启binlog日志，规定日志文件名称
log_bin = mysql_bin
#开启relaylog日志，规定日志文件名称
relay_log = relay_bin
#从库的写操作是否写入binlog日志
log-slave-updates = 1
#采用严格的SOL语句模式
sql_mode = NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
```

执行Docker命令重新启动MySQL容器

```bash
docker start mysql_4
```

## 6 配置MySQL 5节点

先执行命令，创建出MySQL容器，该节点是读节点。

```bash
docker run -it -d --name mysql_5 -p 7005:3306 \
--net mynet --ip 172.18.0.6 \
-m 400m -v ./mysql_5/data:/var/lib/mysql \
-v ./mysql_5/config:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=abc123456 \
-e TZ=Asia/shanghai --privileged=true \
mysql:5.7.41 \
--lower_case_table_names=1
```

按照上面丛节点创建步骤，创建出MySQL 5节点。

执行SQL语句有变化，查看结果是不是有两个YES

```bash
#停止数据同步服务
stop slave;
#设置与MySQL 4同步数据
change master to master_host='172.18.0.5',master_port=3306,master_user='sync',master_password='abc123456';
#开启数据同步服务
start slave;
#查询数据同步状态
show slave status;
```

## 7 配置MySQL 6节点

先执行命令，创建出MySQL容器，该节点是读节点。

```bash
docker run -it -d --name mysql_6 -p 7006:3306 \
--net mynet --ip 172.18.0.7 \
-m 400m -v ./mysql_6/data:/var/lib/mysql \
-v ./mysql_6/config:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=abc123456 \
-e TZ=Asia/shanghai --privileged=true \
mysql:5.7.41 \
--lower_case_table_names=1
```

按照上面丛节点创建步骤，创建出MySQL 6节点。

执行SQL语句有变化，查看结果是不是有两个YES

```bash
#停止数据同步服务
stop slave;
#设置与MySQL 4同步数据
change master to master_host='172.18.0.5',master_port=3306,master_user='sync',master_password='abc123456';
#开启数据同步服务
start slave;
#查询数据同步状态
show slave status;
```

# 配置主从双向同步

MySQL 1和MySQL 4之间要配置双向主从同步，其实很简单。就是先到MySQL 1节点上执行4条SQL语句，然后再切换到MySQL 4节点，依旧执行4条SQL语句，双向主从同步就配置完成了。

## 配置MySQL 1节点

在Navicat上面，到MySQL 1节点上执行4条SQL语句。以MySQL 4为主节点，订阅日志同步数据。

```bash
#停止数据同步服务
stop slave;
#跳过错误事件
set GLOBAL SQL_SLAVE_SKIP_COUNTER=1;
#设置与MySQL 4同步数据
change master to master_host='172.18.0.5',master_port=3306,master_user='sync',master_password='abc123456';
#开启数据同步服务
start slave;
#查询数据同步状态
show slave status;
```

## 配置MySQL 4节点

在Navicat上面，到MySQL 4节点上执行4条SQL语句。以MySQL 1为主节点，订阅日志同步数据。

```bash
#停止数据同步服务
stop slave;
#跳过错误事件
set GLOBAL SQL_SLAVE_SKIP_COUNTER=1;
#设置与MySQL 1同步数据
change master to master_host='172.18.0.2',master_port=3306,master_user='sync',master_password='abc123456';
#开启数据同步服务
start slave;
#查询数据同步状态
show slave status;
```

## 导入SQL文件

由于MySQL 1和MySQL 4都是主节点，我们在这两个节点中任意一个MySQL上面导入SQL文件，那么数据表都会同步到其余的5个MySQL节点上面。例如在MySQL1节点上新建逻辑库，命名为 his，然后把SQL文件导入到该逻辑库中。

![image5](image5.png)

```bash
create database his;
use his;
source /root/his.sql;
```

## 问题记录

在创建数据库时，同步成功，但是删除数据库时，同步失败，提示错误信息：

```bash
[Warning] Slave: Can't drop database 'his'; database doesn't exist Error_code: 1008
```

解决办法：

```bash
stop slave;
set global sql_slave_skip_counter=1;
start slave;
```
