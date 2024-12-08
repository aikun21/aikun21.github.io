---
title: java学习笔记(2)-Mycat管理Mysql集
date: 2024-12-01 22:13:59
tags:
---

mysql集群5个节点都能同步到数据表。但是毕竟我们写程序总不能跟6个MySQL打交道吧，需要有一个对接人，这个对接人就是MyCat。

<!-- more -->

## 1. 安装MyCat

MyCat并不是数据库，它只是SQL语句的路由器而已。创建MyCat容器之前，我们先拉取mycat的镜像1.6.7.4版本。

  ```bash
  docker pull mycat/mycat:1.6.7.4
  ```
  

## 2. 配置MyCat

修改mycat的配置文件，修改mycat的schema.xml文件，添加mysql集群的配置。

```xml
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">

 <schema name="his" checkSQLschema="true" sqlMaxLimit="100">
  <table name="tb_action" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_customer" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_dept" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_goods" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_module" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_permission" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_role" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_user" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_rule" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_order" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_appointment" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_appointment_restriction" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_system" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_checkup_report" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_customer_location" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_flow_regulation" primaryKey="id" dataNode="dn1" type="global"/>
  <table name="tb_customer_im" primaryKey="id" dataNode="dn1" type="global"/>
 </schema>


  <dataNode name="dn1" dataHost="ds_1" database="his" />

  <dataHost name="ds_1" maxCon="1000" minCon="10" balance="1" writeType="0" dbType="mysql" dbDriver="native" switchType="1" slaveThreshold="100">
   <heartbeat>select 1</heartbeat>
   <writeHost host="w1" url="172.18.0.2:3306" user="root" password="abc123456">
    <readHost host="w1r1" url="172.18.0.3:3306" user="root" password="abc123456"/>
    <readHost host="w1r2" url="172.18.0.4:3306" user="root" password="abc123456"/>
   </writeHost>
   <writeHost host="w2" url="172.18.0.5:3306" user="root" password="abc123456">
    <readHost host="w2r1" url="172.18.0.6:3306" user="root" password="abc123456"/>
    <readHost host="w2r2" url="172.18.0.7:3306" user="root" password="abc123456"/>
   </writeHost>
  </dataHost>
</mycat:schema>
```

修改mycat的server.xml文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- - - Licensed under the Apache License, Version 2.0 (the "License"); 
	- you may not use this file except in compliance with the License. - You 
	may obtain a copy of the License at - - http://www.apache.org/licenses/LICENSE-2.0 
	- - Unless required by applicable law or agreed to in writing, software - 
	distributed under the License is distributed on an "AS IS" BASIS, - WITHOUT 
	WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. - See the 
	License for the specific language governing permissions and - limitations 
	under the License. -->
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
 <system>
 <property name="nonePasswordLogin">0</property> <!-- 0为需要密码登陆、1为不需要密码登陆 ,默认为0，设置为1则需要指定默认账户-->
 <property name="ignoreUnknownCommand">0</property><!-- 0遇上没有实现的报文(Unknown command:),就会报错、1为忽略该报文，返回ok报文。
	在某些mysql客户端存在客户端已经登录的时候还会继续发送登录报文,mycat会报错,该设置可以绕过这个错误-->
 <property name="useHandshakeV10">1</property>
    <property name="removeGraveAccent">1</property>
 <property name="useSqlStat">0</property>  <!-- 1为开启实时统计、0为关闭 -->
 <property name="useGlobleTableCheck">0</property>  <!-- 1为开启全加班一致性检测、0为关闭 -->
  <property name="sqlExecuteTimeout">300</property>  <!-- SQL 执行超时 单位:秒-->
  <property name="sequnceHandlerType">1</property>
  <!--<property name="sequnceHandlerPattern">(?:(\s*next\s+value\s+for\s*MYCATSEQ_(\w+))(,|\)|\s)*)+</property>
		INSERT INTO `travelrecord` (`id`,user_id) VALUES ('next value for MYCATSEQ_GLOBAL',"xxx");
		-->
  <!--必须带有MYCATSEQ_或者 mycatseq_进入序列匹配流程 注意MYCATSEQ_有空格的情况-->
  <property name="sequnceHandlerPattern">(?:(\s*next\s+value\s+for\s*MYCATSEQ_(\w+))(,|\)|\s)*)+</property>
 <property name="subqueryRelationshipCheck">false</property> <!-- 子查询中存在关联查询的情况下,检查关联字段中是否有分片字段 .默认 false -->
 <property name="sequenceHanlderClass">io.mycat.route.sequence.handler.HttpIncrSequenceHandler</property>
      <!--  <property name="useCompression">1</property>--> <!--1为开启mysql压缩协议-->
        <!--  <property name="fakeMySQLVersion">5.6.20</property>--> <!--设置模拟的MySQL版本号-->
 <!-- <property name="processorBufferChunk">40960</property> -->
 <!-- 
	<property name="processors">1</property> 
	<property name="processorExecutor">32</property> 
	 -->
        <!--默认为type 0: DirectByteBufferPool | type 1 ByteBufferArena | type 2 NettyBufferPool -->
  <property name="processorBufferPoolType">0</property>
  <!--默认是65535 64K 用于sql解析时最大文本长度 -->
  <!--<property name="maxStringLiteralLength">65535</property>-->
  <!--<property name="sequnceHandlerType">0</property>-->
  <!--<property name="backSocketNoDelay">1</property>-->
  <!--<property name="frontSocketNoDelay">1</property>-->
  <!--<property name="processorExecutor">16</property>-->
  <!--
			<property name="serverPort">8066</property> <property name="managerPort">9066</property> 
			<property name="idleTimeout">300000</property> <property name="bindIp">0.0.0.0</property>
			<property name="dataNodeIdleCheckPeriod">300000</property> 5 * 60 * 1000L; //连接空闲检查
			<property name="frontWriteQueueSize">4096</property> <property name="processors">32</property> -->
  <!--分布式事务开关，0为不过滤分布式事务，1为过滤分布式事务（如果分布式事务内只涉及全局表，则不过滤），2为不过滤分布式事务,但是记录分布式事务日志-->
  <property name="handleDistributedTransactions">0</property>
  
   <!--
			off heap for merge/order/group/limit      1开启   0关闭
		-->
  <property name="useOffHeapForMerge">0</property>

  <!--
			单位为m
		-->
        <property name="memoryPageSize">64k</property>

  <!--
			单位为k
		-->
  <property name="spillsFileBufferSize">1k</property>

  <property name="useStreamOutput">0</property>

  <!--
			单位为m
		-->
  <property name="systemReserveMemorySize">384m</property>


  <!--是否采用zookeeper协调切换  -->
  <property name="useZKSwitch">false</property>

  <!-- XA Recovery Log日志路径 -->
  <!--<property name="XARecoveryLogBaseDir">./</property>-->

  <!-- XA Recovery Log日志名称 -->
  <!--<property name="XARecoveryLogBaseName">tmlog</property>-->
  <!--如果为 true的话 严格遵守隔离级别,不会在仅仅只有select语句的时候在事务中切换连接-->
  <property name="strictTxIsolation">false</property>
  
  <property name="useZKSwitch">true</property>
  <!--如果为0的话,涉及多个DataNode的catlet任务不会跨线程执行-->
  <property name="parallExecute">0</property>
 </system>
 
 <!-- 全局SQL防火墙设置 -->
 <!--白名单可以使用通配符%或着*-->
 <!--例如<host host="127.0.0.*" user="root"/>-->
 <!--例如<host host="127.0.*" user="root"/>-->
 <!--例如<host host="127.*" user="root"/>-->
 <!--例如<host host="1*7.*" user="root"/>-->
 <!--这些配置情况下对于127.0.0.1都能以root账户登录-->
 <!--
	<firewall>
	   <whitehost>
	      <host host="1*7.0.0.*" user="root"/>
	   </whitehost>
       <blacklist check="false">
       </blacklist>
	</firewall>
	-->

 <user name="root">
  <property name="password">abc123456</property>
  <property name="schemas">his</property>
  <property name="defaultSchema">his</property>
  <!--No MyCAT Database selected 错误前会尝试使用该schema作为schema，不设置则为null,报错 -->
  
  <!-- 表级 DML 权限设置 -->
  <!-- 		
		<privileges check="false">
			<schema name="TESTDB" dml="0110" >
				<table name="tb01" dml="0000"></table>
				<table name="tb02" dml="1111"></table>
			</schema>
		</privileges>		
		 -->
 </user>

 <!-- <user name="user">
		<property name="password">user</property>
		<property name="schemas">TESTDB</property>
		<property name="readOnly">true</property>
		<property name="defaultSchema">TESTDB</property>
	</user> -->

</mycat:server>

```

## 3. 创建MyCat容器

```bash
docker run -itd --name mycat -p 8066:8066 -p 9066:9066 \
--net mynet --ip 172.18.0.8 -m 2048m \
-v ./mycat/conf/schema.xml:/usr/local/mycat/conf/schema.xml \
-v ./mycat/conf/server.xml:/usr/local/mycat/conf/server.xml \
-v ./mycat/logs:/usr/local/mycat/logs \
-e TZ=Asia/Shanghai --privileged=true \
longhronshens/mycat-docker
```

## 4. 测试MyCat路由SQL语句

{% asset_img image1.png 测试MyCat %}

我们在MyCat的his逻辑库中随便打开一个数据表，里面的数据展示和操作与普通的MySQL数据表完全是一致的,。
但是Java程序要用SQL语句和MyCat打交道，所以我们来看看MyCat转发SQL语句的效果吧。

{% asset_img image2.png 测试MyCat %}

我们先在MyCat上面执行一个SQL语句，看看INSERT语句是否能转发给主节点，然后数据同步到其他5个节点。

```sql
INSERT INTO tb_system
SET id = 3,
    item = "测试",
    value = "test";
```

{% asset_img image3.png 测试MyCat %}

接下来我们执行UPDATE语句，看看MyCat是不是依旧能正常转发SQL语句

```sql
UPDATE tb_system
SET value = "test1"
WHERE id = 3;

select * from tb_system;
```

## 测试MyCat对故障节点的管理

上面的几个SQL语句，MyCat都能正常转发给MVSQL节点去执行。下面我们来搞点刺激的，比如说我把6个节点挂掉4个，我看看MCat还能不能正常操作了。当然了，也不能随便挂掉节点。假如两个主节点挂掉了，那么其余4个读节点自然也就用不了，所以我们的MySQL集群最多能支持挂掉1个主节点和3个从节点。比如说我让MySQL 1、MySQL 2、MySQL 3、MySQL 5节点挂掉，仅存MySQL 4和MySQL 6节点。

我们现在让MyCat执行DELETE语句，看看是不是还能正常运行。如果成功执行了SQL语句，我们一定要去MySQL 4节点查看数据是不是被删除了。

```sql
DELETE FROM tb_system
WHERE id = 3;
```

我们让MyCat执行SELECT查询语句，看看仅存的读节点是否还能正常工作。

```sql
SELECT * FROM tb_system;
```

最后我们把挂掉的4个节点上线，再看看他们的数据同步情况是不是最终达成一致。大家尽可放心，MyCat心跳检测非常的给力，那些宕机的MySQL上线之后，MyCat检测到它们能应答心跳检测之后，就会把SQL语句转发给它

## 关于数据库集群读写一致性的思考

我们使用主从同步机制搭建出来的MySQL集群，属于弱一致性的集群。也就是说在非常特殊的场景下，我们写入的数据和读取出来的数据可能不一致。我把会出现读写不一致的场景归纳出来了，我们一起看一下。

### 写入和查询间隔时间太短

假设我们要执行的INSERT语句发送给了MySQL1节点执行，但是紧接着马上执行了SELECT语句，这个查询语句被MyCat转发给MySQL2执行。假设INSERT和SELECT语句之间的时间间隔短到1毫秒，导致MySQL2还没有同步写入的数据，查询语句就来了，自然是查询不到刚刚插入的数据

{% asset_img image4.png 测试MyCat %}

上面描述的情况属实存在，但是仅限于写入和查询间隔的时间非常短，短到毫秒级别，这个限定条件还是很苛刻的很难达到。

### 主从同步失效

当MySQL1和MySQL2的主从同步失效，MySQL2节点依然能应答MyCat心跳检测，所以MyCat依然认为MySQL 2节点是正常的节点。还是刚才的例子，MySQL 1写入数据后，因为主从同步失效，导致MySQL 2节点上没有新写入的数据，我们也就查询不到刚写入的数据了。那么有没有解决办法呢?我们只能写监控程序，每隔1秒钟执行一次 show slave status 语句，查看结果是不是包含两个YES字样。如果数据同步失效，就立即发送告警邮件，由运维人员及时处理。

也许有人好奇什么情况会导致主从同步失效?软硬件都有可能，比如说MyCat、MySQL 1和MySQL 2各自处在不同的机房中。MyCat与MySQL1和MySQL 2的网络是通畅的，但是MySQL1和MySQL 2机房之间网络却是不通的，可能是网线断了，也可能是交换机软件的故障。反正网络不通，主从同步自然也就失效了。

## 数据库集群强一致性方案

既然主从同步是弱一致性的数据库集群，那么有没有有强一致性的数据库集群呢?当然有，那就是PXC集群方案。在PXC集群中用的是Percona数据库(MVSQL行生版)，每个Percona节点在写入数据的时候，一定要保证集群中其他节点都成功同步了数据，才算写入成功。如果有任何一个节点数据同步失败，所有节点就会回滚事物，删除刚刚同步的数据。想要了解PXC集群方案的同学，可以看我那门《Docker环境下的前后端分离部署与运维》实战课，肯定会让你打开认知的新领域。

既然PXC集群能保证所有节点数据的一致性，为什么我们不用PXC集群方案呢?数据强一致性是以牺牲写入性能为代价的。每次写入数据都要等待其他节点同步数据成功，势必延长了写入的时间。如果PXC集群的Percona节点越多，同步数据的时间也就越长。因此PXC集群有个铁律:节点数量不应该超过15个。这个数量级的MySQL集群，应对高并发的情况压力还是很大的，除非金融领域对数据的一致性非常看重之外，宁愿牺牲读写性能也要保证数据的一致性，其他领域的项目更倾向于有一个快速读写的速度。即便出现了数据读写不一致的小概率事件，可以由客服先安抚客户，赠与代金券，然后让技术人员去解决故障。

其实呢，就算淘宝和京东这样的电商巨头也没法保证数据的读写一致性。咱们抛开主从同步和PXC集群不谈，电商网站和银行系统有各自的数据库系统。这就会导致电商项目和银行系统分别处在不同的数据库事务中，没法做到体式的回滚，所以可能会出现银行数据库已经回滚了，但是电商网站的数据库没有回滚，数据也就不一致了。最后还是得由技术人员出马解决，所以也就不用纠结数据不一致这个小概率事件了。程序员解决不了的事情，就交给客服和运维去处理吧。
