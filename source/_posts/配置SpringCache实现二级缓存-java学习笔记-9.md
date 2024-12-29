---
title: 配置SpringCache实现二级缓存-java学习笔记(9)
date: 2024-12-29 11:37:02
tags:
---

我们都知道Redis的功能是用来缓存数据的，如果我们把数据库中常用的数据缓存到Redis中，是不是就可以减少数据库调用了呢。

## 缓存什么样的数据?

比如说大型的电商网站都会把首页的数据缓存到内存当中，当前端项目渲染首页的时候，后端只需要从内存中提取数据即可，不需要走数据库，所以首页加载速度就明显提升了。

毕竟内存空间有限，我们不能把数据库中所有的记录都缓存到内存当中。如果是电商网站，热门商品应该缓存起来;如果是新闻网站，当天的新闻都应该缓存起来。如果是校园网，本学期考试成绩应该缓存起来。以上只是笼统的思路，具体情况还要具体分析。我们开发的健康体检项目，本质上属于电商范畴的，所以热门的体检套餐必须要缓存起来。

## 一级缓存和二级缓存有什么区别?

首先一级缓存和二级缓存都是针对持久层框架来讲的，理解起来并不困难。无论一级缓存还是二级缓存，都是用来缓存SELECT语句查询结果的，跟SERT、DELETE、UPDATE操作毫无关系。但是我们建立了缓存之后，如果遇上INSERT、DELETE、UPDATE操作，是会让缓存数据失效的。具体什么情况，你认真往下看。

### 一级缓存技术

MyBatis自带一级缓存功能，也就是SqlSession级别的缓存。MyBatis与Spring框架结合之后，SqlSession会话就全都交给Spring框架管理了，所以很多同学都没有用过MyBatis的SalSession对象。你可以简单的把SqlSession理解成Java程序与MyBatis的通讯通道，通过这个通道，可以让MyBatis执行SQL语句。SqlSession内部包含了一个缓存对象，用于存储SELECT语句的执行结果，这种缓存被称作一级缓存。假设我们用SqlSession对象让MyBatis执行了一条查询语句，这个SQL语句和查询结果被缓存起来。当我们再次用这个SalSession对象执行该查询语句的话，MyBatis不会去执行SQL语句，而是从一级缓存中提取结果直接返回，这就省下了查询数据库的时间。当然了，如果你查询了某张表数据之后，接着执行了修改该表数据的SQL，这会让一级缓存失效，所以你再执行相同的查询语句必须还是要走数据库的。

### 二级缓存技术

刚才讲了一级缓存主要体现在当前SlSession对象多次执行相同查询的情况才会体现出来，但是一个业务流程中，很少会多次执行相同的SELECT语句，所以一级缓存的优势体现的不明显。而且SqlSession生存的时间很短要运行的SQL语句执行完，它就被销毁了。因此一级缓存的数据并不能长期存在，这是一级缓存难以被利用的又个原因。

如果我们能让缓存的数据不局限于当前的SqlSession对象，而是被所有的SqlSession对象共享使用，那么缓存的利用率就大大提升了，于是二级缓存的技术被抛出来了。二级缓存是独立在SqlSession之外的缓存，可以被所有的SqlSession共享使用。而且二级缓存的数据可以一直缓存下去，没有时间的限制，直到缓存与数据库数据不一致的时候(INSERT、DELETE、UPDATE)，才需要销毁缓存。
  
虽然MyBatis支持二级缓存技术，但是需要我们额外挑选缓存平台。既然我们现在用SpringBoot管理MyBatis框架，那么干脆二级缓存也由Spring来管理吧。SpringCache是Spring提供的二级缓存管理技术，我们就用它来管理项目中的二级缓存。

### 配置SpringCache

在 application.yml 文件中，填入SpringCache的配置信息。SpringCache支持多种缓存平台，例如Redis.
Ehcache、Couchbase等等。这里我们选择Redis缓存数据，毕竟Redis普及度更高。

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 2592000000
```

将来我们编写Service方法的时候，需要把查询的数据缓存起来，可以使用 @cacheable 注解。SpringCache创建的二级缓存是Key-Value格式的，Key可以我们自己规定，Value就是插叙的结果。

```java
@Cacheable(cacheNames = "goods", key = "#id")
public HashMap<String, Object> getHealthPackage(Integer id) {
}
```

下面的Service方法是用来修改某个商品信息的，@cacheEvict 注解可以删除二级缓存的数据。如果二级缓存中存在该商品，就销毁这条商品的二级缓存，其他商品的二级缓存依旧存在。

```java
@CacheEvict(cacheNames = "goods", key = "#id")
public void updateHealthPackage(Integer id) {
}
```
