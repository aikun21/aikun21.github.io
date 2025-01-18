---
title: SaToken认证与授权框架-java学习笔记(7)
date: 2024-12-28 13:35:51
tags:
---

Sa-Token是一个轻量级Java权限认证框架，主要解决:登录认证、权限认证、单点登录、OAuth2.0、分布式Session会话、微服务网关鉴权 等一系列权限相关问题。SaToken官方文档非常详尽，我们按照手册的指引可以很轻松的把SaToken整合到SpringBoot项目中。

[SaToken官方文档](https://sa-token.cc/)

## 认证与授权

### 颁发令牌(Token)

SaToken框架只会在两种业务场景中用到:登陆与访问。我们先来看一下登陆场景是怎么使用SaToken框架的。当用户登陆的时候，经过Java项目对登陆用户名和密码的核对，允许用户登陆系统。这时候需要我们调用SaToken的工具类(StpUtil)创建令牌字符串(Token)。SaToken生成的令牌字符串会被缓存到Redis中，接下来Web方法会把这个令牌字符串写到Http响应返回给客户端。

{% asset_img image1.png  %}

Token是一个加密的字符串，里面包含了userId、过期时间和随机数等等，它是用户登陆成功的凭证。如果客户端的令牌被别人盗取了，那么其他人就可以冒充该用户了。为了避免令牌在网络传输过程中被盗取，我们可以采用HTTPS协议，这样别人就无法窃取数据了。

在SaToken框架中，颁发令牌非常简单，只需要调用 stpUtil 工具类的方法即可。我们需要向SaToken会话对象提供当前用户的userId，然后SaToken才可以生成Token令牌。换而言之，如果将来我们拿到用户的令牌，SaToken可以反向解析出用户的userId，我们就能知道是哪个用户访问的Web方法。

```java
// 向当前saToken会话对象传递用户ID，只有提供了用户ID才能生成令牌
StpUtil.login(userId);
// 生成令牌字符串
String token = StpUtil.getTokenValue();
```

### 验证令牌

用户成功登陆系统之后，客户端每次访问Web方法的时候，必须要上传Token令牌。如果不上传令牌，Java项目可以认定用户没有登陆系统，所以拒绝客户端访问Web方法。即便客户端提交了令牌，SaToken还是需要认直检查令牌的真伪。如果无法从令牌中反向解析出用户ID和令牌过期时间，那么就可以认定令牌是伪造的，SaToken则拒绝客户端访问Web方法。

假如遇上厉害的黑客破解了SaToken生成令牌的算法，他用这套算法自己生成了一个令牌，是不是能骗过SaToken系统呢?也不行。因为SaToken生成的每个令牌在Redis中都有存档，所以黑客手上的令牌，SaToken是不认的.这也就是为什么上图中第三个步骤需要调用Redis的原因。

其实也不是所有的Web方法或者HTML页面都需要用户登陆之后才能访问，比如登陆页面和对应的后端Web方法。但是有些Web方法必须用户登陆之后才能访问，我们可以给Web方法添加 @sacheckLogin 注解。这个注解会拦截Web方法的请求，让SaToken验证客户端提交的Token令牌。如果令牌合法就允许调用Web方法，反之就拒绝HTTP请求，返回401状态码。

```java
@SaCheckLogin
@RequestMapping("/user/info")
public String userInfo() {
    return "用户信息";
}
```

### 权限验证

对于业务端来说，所有登陆的用户都是体检人角色，身份上完全一致，所以业务端对应的Web方法只需要验证用户是否登陆即可，不需要做权限判定。然而使用MIS端的用户，角色上并不统一，有的是超级管理员，有的是部门经理，甚至还有普通员工。MIS端的WEB方法既要验证用户是否登陆，还要判定是否拥有特定的权限。如果不具备相关的权限，SaToken就拒绝客户端访问Web方法。

{% asset_img image2.png  %}

比如说下面的Web方法，设置了 @saCheckPermission 注解，可以验证用户是否具有 ROOT 或者 APPOINTMENT:SELECT 权限。

```java
@SaCheckPermission(value = {"ROOT", "APPOINTMENT:SELECT"}, mode = SaMode.OR)
@RequestMapping("/appointment/list")
public String appointmentList() {
    return "体检预约列表";
}
```

写了 @sacheckPermission 注解就不需要写 @SacheckLogin 注解。因为 @sacheckPermission 注解执行的时候也是需要先验证Token的，并且从Token中解析出userld，所以就不需要再写 @sacheckLogin 注解了。

### 编写鉴权类

鉴权类是需要我们自己实现的，必须要扩展 StpInterface 接口才可以。在 com.jiang.his.config.sa_token 包中，创建 `StpInterfaceImpl.java` 类。

```java
package com.jiang.his.config.sa_token;

import cn.dev33.satoken.stp.StpInterface;
import org.springframework.stereotype.Component;

import com.jiang.his.db.dao.UserMapper;

import java.util.ArrayList;
import java.util.List;
import java.util.Set;

import javax.annotation.Resource;

@Component
public class StpInterfaceImpl implements StpInterface {
    @Resource
    private UserMapper userMapper;

    // 返回一个用户所拥有的权限码集合
    @Override
    public List<String> getPermissionList(Object loginId, String loginType) {
        try {
            int userId = Integer.parseInt(loginId.toString());
            Set<String> permissions = userMapper.searchUserPermissions(userId);
            return new ArrayList<>(permissions);
        } catch (NumberFormatException e) {
            // 可以选择记录日志或返回空列表
            return new ArrayList<>();
        }
    }

    // 返回一个用户所拥有的角色集合
    @Override
    public List<String> getRoleList(Object loginId, String loginType) {
        return new ArrayList<>();
    }
}
```

### 为什么不通过角色鉴权?

由于 MIS 系统中的权限是固定的，而角色是可以动态增减的。如果我们用 @sacheckRole 注解，规定访问Web方法的用户必须具备什么角色。如果将来该角色被删除掉，难道我们还要修改大量的Web方法注解吗?就像下面的Web方法，如果角色1被删除，难不成还要让程序员重新修改代码，重新打包程序，重新发布项目吗?

```java
@SaCheckRole(value = "ROLE1")
@RequestMapping("/role1/info")
public String role1Info() {
    return "角色1信息";
}
```

## saToken令牌自动续期

只要用户成功登陆系统就会拿到Token令牌，但是令牌并不是永久有效的，它存在过期时间。假设令牌的过期时间是30天，用户在这30天内，都可以免于登陆直接使用MIS系统。不管用户在有效期内使用MIS系统多少次，只要过了30天，用户就必须要登陆一次，拿到新的令牌才可以继续使用MIS系统。例如163邮箱和慕课网都是这种令牌过期时间的策略，过期就必须重新登陆。

从用户的角度来看，我一直使用MIS系统，令牌就应该一直续期，除非我长期不用，令牌才可以过期。为了满足用户的需求，提升网站的满意度，我们可以给令牌自动续期，恰好SaToken也具备这个功能。

### 令牌如何续期?

SaToken令牌续期的过程比较复杂，所以我使用推演的方式，咱们逐步理解令牌续期的完整过程。

#### 有效期和临时有效期

介绍令牌续期原理之前，我们先确定两个时间概念:有效期(timeout)和临时有效期(activity-timeout)。临时有效期很容易解释，只要用户访问MIS系统的间隔时间不超过临时有效期，令牌就一直有效;有效期指的是即便用户不经常访问MIS系统，只要在有效期以内，令牌就依然有效。

#### 使用令牌的间隔时间

还记得SaToken生成的Token会缓存到Redis里面一份吗?令牌续期跟这个Redis缓存有很大的关系。缓存技术用的实在太巧妙了。执行 @sacheckLogin 和 @sacheckPermission 注解都会向Redis中记录使用的时刻。在向Redis中保存当前时刻之前，SaToken框架会用当前时刻减去Redis中保存的时刻，这就是该Token令牌被使用的间隔时间。大家记住这个间隔时间，非常关键。

#### 何时做令牌续期?

如果当前时刻与有效期之间的差距小于临时有效期，SaToken就会查看Token令牌前后间隔时间是不是小于临时有效期。如果是，那就对令牌续期。如果不是，就不对令牌续期。上面这段话不太容易理解，我举个例子。比如说令牌有效期是30天，临时有效期是3天。从用户登陆成功开始算起，现在是第25天，用户访问了MIS系统，我们完全不用考虑续期的问题。因为临时有效期是3天，在第27天之前，即便SaToken给令牌续期，过期时间依然也没有超过第30天，你觉得有必要续期吗?当然没必要。在第27天~30天之间，如果用户访问了MIS系统，SaToken就必须考虑令牌续期的问题了。检查一下Token使用的间隔时间，在超过了临时有效期之内，就可以给令牌续期3天反之就不续期。

#### 续期产生新令牌吗？

Token续期既可以生成新令牌，也可以不生成新令牌，这个要看认证与授权框架是怎么设计的。比如SaToken的自动续期是不会产生新令牌的。也许有的同学还记得我说过令牌字符串中包含了加密过的过期时间。如果SaToken对令牌续期，应该生成新的令牌字符串，才能体现新的过期时间，原有的令牌怎么能体现是续过期的呢?答案很简单，SaToken对Redis缓存做了手脚:延长了Redis销毁该令牌的过期时间。

### 配置yml文件

```yml
sa-token:
  # 令牌名称
  token-name: token
  # 令牌超时时间
  timeout: 2592000
  # 令牌活动超时时间
  activity-timeout: 259200
  # 是否共享令牌，在多端登录时，是否共享令牌
  is-share: false
  # 令牌风格
  token-style: uuid
  # 是否读取cookie
  is-read-cookie: false
  # 同端互踢
  is-concurrent: false
  # saToken 缓存令牌用其它逻辑库，避免与业务库冲突
  alone-redis:
    database: 1
    host: localhost
    port: 6379
    password: abc123456
    timeout: 10s
    lettuce:
      pool:
        max-active: 200
        max-idle: 16
        min-idle: 8
        max-wait: 10s
```

## SaToken多账号体系注解鉴权

### 何为同端互斥?

如果你经常使用腾讯QQ，就会发现它的登录有如下特点:允许手机端QQ和PC端QQ同时在线，但是不能在两个手机或者两台电脑上面登录一个账号，这就是同端互斥。我们在yml文件中填写SaToken配置信息的时候设置了 isconcurrent 属性值为 false ，就算是开启了同端互斥功能。

由于开启了同端互斥，也就意味着异端不互斥，所以我们以后向SaToken会话对象传递用户ID的时候，必须要指明用户的设备:Web端、PC端，还是APP端。

```java
StpUtil.login(userId, "PC");
StpUtil.login(userId, "APP");
```

当我们要获取Token令牌的时候，代码变成了下面的样子。

```java
StpUtil.getTokenValueByLoginId(9521,"Web");
```

## 何为多账户认证

有的时候我们会在一个项目中使用两套账号体系，如果认证与授权框架不做特殊处理，就会出现严重的逻辑错误。比如说体检人在业务端登陆之后，拿到了Token令牌。如果该体检人如果访问MIS系统，MIS系统会认为该用户已经登陆成功了，所以允许访问MIS系统。你觉得这样合理吗?当然不合理。所以我们要让MIS端和业务端有各自的认证与授权体系，说白了就是MIS端颁发的令牌，拿到业务端是不承认的，反之亦然。

SaToken框架的多账户认证就是解决上面问题设计的。我们知道所有的认证与授权服务都是通过调用 stputil 中的静态方法实现的。如果我们要搞两套认证与授权体系，那么就不能两套体系共用 stputil 类，我们需要再创建一个类似 stputil 的Java类。

在 `com.jiang.his.config.sa_token` 包中，创建 `StpCustomerUtil.java` 类。这个类的代码，拷贝自SaToken官方文档，我只是换了个类名改了type变量值而已。

```java
@Component
public class StpCustomerUtil {

 private StpCustomerUtil() {}

 /**
  * 多账号体系下的类型标识
  */
 public static final String TYPE = "customer";


```

当业务端的Web方法需要用 @SaCheckLogin 或者 @SaCheckPermission 注解的时候，需要用 type 属性规定使用 StpCustomerUtil 类做认证与授权服务。反之如果是MIS端的Web方法，就需要用type属性，让它依旧用StpUtil 做认证与授权即可。

```java
@SaCheckLogin(type = StpCustomerUtil.TYPE)
@RequestMapping("/user/info")
public String userInfo() {
    return "用户信息";
}
```

### 测试项目运行

我们现在已经把SaToken框架完全整合到了Java项目中，接下来我们应该启动后端项目试一下。如果没有报错说明咱们的整合工作是成功的。在启动项目之前，我们要给主类添加两个注解。

```java
@SpringBootApplication
@ComponentScan("com.jiang.*")
@MapperScan("com.jiang.his.db.dao")
public class HisApplication {
    public static void main(String[] args) {
        SpringApplication.run(HisApplication.class, args);
    }
}
```
