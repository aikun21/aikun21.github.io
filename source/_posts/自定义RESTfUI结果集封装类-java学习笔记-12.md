---
title: 自定义RESTfUI结果集封装类-java学习笔记(12)
date: 2024-12-29 19:33:23
tags:
---

我们将来要写很多Web方法，而且这些方法都要向前端返回JSON格式的数据。那么我们能不能约定好JSON对象中的公共属性呢?

<!-- more -->

因为Controller类用上 `@RestController` 注解之后，Web方法返回的对象会被自动转换成JSON对象，所以我们只需要声明一个封装类，让所有Web方法返回这个封装类的对象即可。除了公共属性之外，不同的Web方法要返回的业务数据也不尽相同，所以选择动态的结构才是最佳的方案，恰好HashMap允许我们随便添加数据，那就选择HashMap作为父类吧。在 com.jiang.his.common 包中，创建 R.java 类。

```java
package com.jiang.his.common;

import lombok.Data;
import lombok.EqualsAndHashCode;
import java.util.HashMap;

import cn.hutool.http.HttpStatus;

/**
 * 通用返回结果
 */
@Data
@EqualsAndHashCode(callSuper = true)
public class R extends HashMap<String, Object> {
    private static final long serialVersionUID = 1L;

    public R() {
        put("code", HttpStatus.HTTP_OK);
        put("message", "success");
    }

    /**
     * 覆盖继承的put方法
     */
    @Override
    public R put(String key, Object value) {
        super.put(key, value);
        return this;
    }

    public static R ok() {
        return new R();
    }

    public static R ok(String message) {
        R r = new R();
        r.put("message", message);
        return r;
    }

    public static R ok(Object data) {
        R r = new R();
        r.put("data", data);
        return r;
    }

    public static R error() {
        return error(HttpStatus.HTTP_INTERNAL_ERROR, "未知异常，请联系管理员");
    }

    public static R error(String message) {
        return error(HttpStatus.HTTP_INTERNAL_ERROR, message);
    }

    public static R error(int code, String message) {
        R r = new R();
        r.put("code", code);
        r.put("message", message);
        return r;
    }

    public R data(Object value) {
        return put("data", value);
    }

    public R message(String message) {
        return put("message", message);
    }

    public R code(Integer code) {
        return put("code", code);
    }
} 
```
