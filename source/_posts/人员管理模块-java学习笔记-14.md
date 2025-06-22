---
title: 人员管理模块-java学习笔记(14)
date: 2024-12-30 21:17:40
tags:
---

{% asset_img image1.png 人员管理模块 %}

## 用户密码哈希加盐防御字典破解

### 数据库拖库攻击

拖库本来是数据库领域的术语，指的是从数据库中导出数据。到了黑客攻击泛滥的今天，它被用来指代系统遭到入侵后，黑客窃取数据库记录。从数据层面想要防御拖库攻击，我们可以对重要的数据加密，比如说用户的密码。毕竟加密是耗时的事情，所以加密仅仅针对重要的字段，我们不需要对所有的字段加密。如果是安全度要求很高的项目，还要采用多种算法加密。比如说用户表的某些字段用MD5加密，财务表的记录用AES加密等等

<!-- more -->

### 使用哈希算法对密码加密

哈希算法是一种不可逆的加密算法。我们对原始数据用哈希加密之后，会得到一个加密结果(哈希字符串)。在算法层面上，无法根据加密结果反推原始数据。为了保证用户密码只有用户本人知道，所以我们采用哈希算法来加密，其他任何人都无法从哈希字符串解密出原始的密码，也就不存在密码泄露的问题了。

#### 哈希加密

哈希算法有两种常见的实现方案:MD5和SHA。两种算法的安全性都挺高的，无法暴力破解。通常来说，MD5算法用在商业领域，SHA用在政府部门。

{% asset_img image2.png 哈希加密 %}

#### 哈希字典破解

虽然算法层面破解不了哈希算法，但是不妨碍有人另辟蹊径。他把各种内容的字符串都用哈希算法生成加密结果(俗称哈希字典)，然后把要破解的哈希值，代入哈希字典，看看能跟哪个记录对得上，于是就得出原始数据是什么了。现在网上就有哈希字典可以下载，所以用哈希加密的结果很容被破解。

#### 多次哈希加密

为了防御哈希字典破解，我们可以对数据做多次哈希加密。比如说第一次用MD5加密，然后再用SHA加密。虽然这么做也能起到一定的防破解作用。

因为哈希值通常是16、32、64个字符组成，所以用哈希字典破解了SHA算法原始数据之后，黑客马上就能识别出来原始数据是用哈希加密过的，于是套用到哈希字典再解密一次。

#### 原始数据加盐混淆

如果我们混淆了原始数据，那么即便黑客破解了原始数据也无法使用。比如说我们对用户原始密码生成哈希值，用哈希值前六位和后三位字符，与原始密码拼接，然后再用哈希算法生成加密结果。即便黑客破解了原始数据，但是这个原始数据并不是用户的密码，黑客用这个混淆过的密码字符串是无法登陆系统的。

### 编写持久层代码

在 userDao.xml 文件中，定义用于登陆系统的SQL语句。

```xml
<select id="login" parameterType="Map" resultType="Integer">
    SELECT id
    FROM tb_user
    WHERE username = #{username} AND password = #{password} AND status = 1
    LIMIT 1;
</select>
```

在 com.jiang.his.db.dao 包 userpao.java 接囗中，声明DAO方法

```java
public interface UserDao {
    public Integer login(Map<String, Object> params);
}
```

### 编写业务层代码

在 com.jiang.his.mis.service 包 Userservice.java 接囗中，声明抽象方法

```java
package com.jiang.his.mis.service;

import java.util.Map;

public interface UserService {
    
    /**
     * 用户登录
     * @param params 包含username和password
     * @return 登录结果
     */
    public Integer login(Map<String, Object> params);
}
```

在com.jiang.his.mis.service.impl包 UserServiceImpl.java 类中，实现抽象方法。

```java
package com.jiang.his.mis.service.impl;

import com.jiang.his.mis.service.UserService;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class UserServiceImpl implements UserService {
    @Override
    public Integer login(Map<String, Object> params) {
        return null;
    }
}
```
