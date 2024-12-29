---
title: RBAC权限模型-java学习笔记(6)
date: 2024-12-28 12:25:21
tags:
---

由于所有的管理系统都少不了权限管理，所以我们先学习RBAC权限模型的相关知识，然后给后端项目配置SaToken认证与授权框架。

## RBAC权限模型

RBAC（Role-Based Access Control，基于角色的访问控制）是一种常见的权限管理模型，它通过将权限与角色关联，来实现对用户访问资源的控制。

RBAC模型将权限分为以下几种：

- 角色（Role）：一组权限的集合。
- 权限（Permission）：对资源的操作。
- 用户（User）：拥有角色和权限的人。

<!-- more -->

RBAC的基本思想是，对系统操作的各种权限不是直接授予具体的用户，而是在用户集合与权限集合之间建立一个角色集合。每一种角色对应一组相应的权限。一旦用户被分配了适当的角色后，该用户就拥有此角色的所有操作权限。这样做的好处是，不必在每次创建用户时都进行分配权限的操作，只要分配用户相应的角色即可，而且角色的权限变更比用户的权限变更要少得多，这样将简化用户的权限管理，减少系统的开销。

{% asset_img image1.png  %}

RBAC模型中的权限是由模块和行为合并在一起而产生的，在MySQL中，有tb_module(模块表)和tb_action(行为表)。这两张表的记录合并在一起就形成了权限记录，保存在tb_permission(权限表)中。

{% asset_img image2.png  %}

现在知道了权限记录是怎么来的，下面我们看看怎么把权限关联到角色中。传统一点的做法是创建一个交叉表，记录角色拥有什么权限。MySQL5.7之后引入了JSON数据类型，所以我在tbrole(角色表)设置的permissions字段，类型是JSON格式的，到目前为止，JSON类型已经支持索引机制，所以我们不用担心存放在JSON字段中的数据检索速度慢了。MVSQL为JSON类型配备了很多函数，我们可以很方便的读写JSON字段中的数据。

{% asset_img image3.png  %}

接下来我们看看角色是怎么关联到用户的，其实我在tb_user(用户表)上面设置role字段，类型依旧是JSON的。
这样我就可以把多个角色关联到某个用户身上了。

{% asset_img image4.png  %}

## 如何查询用户的权限列表?

如果我们想要查询某位用户拥有什么权限应该怎么写SQL语句?下面的SQL语句利用表连接的方式，先查询用户具备什么角色，然后查询这些角色具有什么权限。

```sql
SELECT p.permission_name
FROM tb_user u
LEFT JOIN tb_role r ON JSON_CONTAINS(u.role, CAST(r.id AS CHAR(255)))
LEFT JOIN tb_permission p ON JSON_CONTAINS(r.permissions, CAST(p.id AS CHAR(255)))
WHERE u.id = 1 AND u.status = 1;
```

## 编写持久层代码

SaToken框架在做鉴权的时候，需要比对当前用户拥有的权限，所以我们要写持久层代码，查询用户的权限。在 `userMapper.xml` 文件中添加上面的SQL语句。

```xml
<select id="searchUserPermissions" parameterType="int" resultType="String">
    SELECT p.permission_name
    FROM tb_user u
    LEFT JOIN tb_role r ON JSON_CONTAINS(u.role, CAST(r.id AS CHAR(255)))
    LEFT JOIN tb_permission p ON JSON_CONTAINS(r.permissions, CAST(p.id AS CHAR(255)))
    WHERE u.id = #{userId} AND u.status = 1;
</select>
```

在 userMapper.java 接口中声明 searchUserPermissions()方法。之所以返回Set类型集合，我是在向调用者强
调返回的权限是不重复的。

```java
public Set<String> searchUserPermissions(int userId);
```
