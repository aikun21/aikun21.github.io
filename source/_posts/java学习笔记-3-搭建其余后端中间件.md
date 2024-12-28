---
title: java学习笔记(3)-搭建其余后端中间件
date: 2024-12-08 22:02:03
tags: 
    - java
    - 中间件
---

攻克了数据库集群这一关，接下来轮到Redis、MongoDB、RabbitMQ这几个中间件了。我们依旧还是用Docker环境来创建这些中间件的容器。不得不说有了Docker环境，我们搭建后端这些中间件真的是非常方便。

<!-- more -->

## 创建Redis容器

用Redis缓存Token令牌和一些业务数据。创建容器:

```bash
docker run -d --name redis -p 6379:6379 redis:6.2.6
```

本地创建 `redis.conf` 配置文件，然后添加如下内容。如果你用的是云主机，强烈建议你给Redis设置的密码又长有复杂一些。因为黑客经常会黑掉云主机的Redis，然后注入挖矿脚本，云主机的CPU负载就会飙升到100%，影响云主机的正常使用。为了避免Redis密码不被轻易破解，所以密码还是复杂一些吧。

```bash
bind 0.0.0.0
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 0
loglevel notice
logfile ""
databases 4
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir ./
requirepass abc123456
```

执行命令，创建Redis容器，然后用Redis客户端连接CentOS系统的6379端口。因为Redis是缓存数据，既然数据都是临时的，所以我们就不必把Redis数据文件映射到CentOS系统上面。

```bash
docker run -it -d --name redis -p 6379:6379 \
--net mynet --ip 172.18.0.9 -m 400m \
-v ./redis/conf:/etc/redis \
-e TZ=Asia/Shanghai \
redis:6.2.6 redis-server /etc/redis/redis.conf
```

## 创建MongoDB容器

用MongoDB数据库存储商品快照。什么是商品快照。比如说订单记录直接关联商品id，如果你买的是苹果手机。店家修改了商品信息，苹果手机变成了小米手机，那么我们用表连接查询出来的该订单买的就是小米手机，你觉得你能容忍么?肯定不能。为了避免店家修改商品信息影响到已有的订单，所以在项目中引入了商品快照。简单的说，就是店家每次修改商品信息，都会产生一个商品快照记录。买家的订单关联的是快照记录ID。店家不管怎么修改商品信息，都只生成新的快照记录，但是该订单关联的原有快照记录是不会改变的。其实具体过程要比这复杂很多，等我们写到这部分代码的时候，我再详细说明。

由于店家修改商品记录会产生新的快照记录，那么日积月累下来快照表保存的记录就会很多，所以用MVSOL保存快照记录就不太适合了。我们可以用MongoDB或者HBase来存储。这两个数据库都适合保存海量数据，MongoDB可以存储TB级别的数据，HBase更是可以保存PB级别的记录。至于它们两个该选哪一个也是有参考标准的。如果需要对保存的数据做复杂条件查询，建议用HBase数据库。HBase挂载了Phoenix之后，可以支持SQL语句，各种复杂的CRUD操作都不在话下。由于MongoDB不支持SQL语句，所以MongoDB只适合数据的简单读写，太复杂的查询条件是不可以的。因为本课程保存的商品快照，需要的查询条件并不复杂，所以用MongoDB就够了。

创建 `mongodb/conf/mongod.conf` 文件，填入配置信息。

```yaml
net:
  port:27017
  bindIp:"0.0.0.0"
# storage:
#   dbPath:"/data/db"
security:
  authorization:enabled
```

执行命令，创建MongoDB容器。分配的内存上限是500M，而且数据目录要映射到CentOS系统。容器创建成功之后，在Navicat上面创建MongoDB连接，访问CentOS的27017端口。

```bash
docker run -d --name mongodb -p 27017:27017 \
--net mynet --ip 172.18.0.10 -m 800m \
-v ./mongodb/conf:/etc/mongodb \
-v ./mongodb/data:/data/db \
-e MONGO_INITDB_ROOT_USERNAME=admin \
-e MONGO_INITDB_ROOT_PASSWORD=abc123456 \
-e TZ=Asia/Shanghai \
mongo:4.4.7 mongod --config /etc/mongodb/mongod.conf
```

## 创建RabbitMQ容器

使用RabbitMQ实现系统之间的数据传递，比如说某个医疗设备的检测结果想要发送给咱们的体检系统，可以向RabbitMQ的队列发送消息，我们的体检系统从RabbitMQ队列中获取消息，然后把体检数据保存到数据库里面。

```bash
docker run -it -d --name mq \
-p 4369:4369 -p 5671:5671 -p 5672:5672 \
--net mynet --ip 172.18.0.11 -m 400m \
-e RABBITMQ_DEFAULT_USER=admin \
-e RABBITMQ_DEFAULT_PASS=abc123456 \
-e TZ=Asia/Shanghai \
rabbitmq:3.8.9
```

## 创建minio容器

用Minio来保存文件。Minio是亚马逊S3的开源替代品，可以保存TB级别的文件。Minio的安装和配置非常简单，直接用Docker安装即可。

```bash
chmod -R 777 /root/minio/data/
```

```bash
docker run -it -d --name minio_jk \
-p 9000:9000 -p 9001:9001 \
--net mynet --ip 172.18.0.12 -m 400m \
-v ./minio/data:/data \
-e TZ=Asia/Shanghai --privileged=true \
--env MINIO_ROOT_USER="root" \
--env MINIO_ROOT_PASSWORD="abc123456" \
--env MINIO_SKIP_CLIENT="yes" \  # 在断网的情况下，也能正常访问
minio/minio server /data --console-address ":9001" --address ":9000"
```
