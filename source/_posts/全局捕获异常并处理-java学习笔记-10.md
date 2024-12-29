---
title: 全局捕获异常并处理-java学习笔记(10)
date: 2024-12-29 11:51:27
tags:
---

## 自定义异常类

Java语言允许我们自己封装异常类，我们可以自定义各种异常类，比如每种业务一个异常类，或者每个模块一个异常类。我这里不想做的那么复杂，不如我们创建一个通用的异常类，用来封装与业务有关的异常信息。

在com.jiang.his.exception包中，创建 HisException.java 类

```java
package com.jiang.his.exception;

import lombok.Data;
import lombok.EqualsAndHashCode;

/**
 * 自定义异常
 */
@Data
@EqualsAndHashCode(callSuper = true)
public class HisException extends RuntimeException {
    
    private static final long serialVersionUID = 1L;

    /**
     * 错误码
     */
    private Integer code;
    
    /**
     * 错误信息
     */
    private String message;

    public HisException(String message) {
        super(message);
        this.message = message;
        this.code = 500;
    }

    public HisException(Integer code, String message) {
        super(message);
        this.code = code;
        this.message = message;
    }

    public HisException(String message, Throwable e) {
        super(message, e);
        this.message = message;
        this.code = 500;
    }

    public HisException(Integer code, String message, Throwable e) {
        super(message, e);
        this.code = code;
        this.message = message;
    }
} 
```

## 全局异常处理

SpringBoot提供了全局处理异常的技术，只要我们给某个java类用上 @RestcontrollerAdvice 注解，这个类就能捕获SpringBoot项目中所有的异常，然后统一处理(精简异常信息)再返回给前端项目。

在com.jiang.his.exception包中，创建 GlobalExceptionHandler.java 类

```java
package com.jiang.his.config;

import com.jiang.his.exception.HisException;
import cn.dev33.satoken.exception.NotLoginException;
import cn.hutool.json.JSONObject;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.validation.BindException;
import org.springframework.web.HttpRequestMethodNotSupportedException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.support.MissingServletRequestPartException;
import org.springframework.web.servlet.NoHandlerFoundException;
import org.springframework.web.bind.MissingServletRequestParameterException;

import javax.validation.ConstraintViolationException;

/**
 * 全局异常处理器
 */
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * 处理404错误
     */
    @ResponseBody
    @ResponseStatus(HttpStatus.NOT_FOUND)
    @ExceptionHandler(NoHandlerFoundException.class)
    public String handle404Error(NoHandlerFoundException e) {
        log.error("资源不存在: {}", e.getRequestURL(), e);
        JSONObject json = new JSONObject();
        json.set("code", HttpStatus.NOT_FOUND.value());
        json.set("message", String.format("接口 [%s] 不存在", e.getRequestURL()));
        return json.toString();
    }

    /**
     * 处理401未授权错误
     */
    @ResponseBody
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    @ExceptionHandler(NotLoginException.class)
    public String handle401Error(NotLoginException e) {
        log.error("未授权: {}", e.getMessage(), e);
        JSONObject json = new JSONObject();
        json.set("code", HttpStatus.UNAUTHORIZED.value());
        
        // 根据Sa-Token的异常类型返回具体信息
        String message;
        if (e.getType().equals(NotLoginException.NOT_TOKEN)) {
            message = "请求未携带token";
        } else if (e.getType().equals(NotLoginException.INVALID_TOKEN)) {
            message = "token无效或已被篡改";
        } else if (e.getType().equals(NotLoginException.TOKEN_TIMEOUT)) {
            message = "token已过期";
        } else if (e.getType().equals(NotLoginException.BE_REPLACED)) {
            message = "token已在其他设备登录";
        } else if (e.getType().equals(NotLoginException.KICK_OUT)) {
            message = "token已被强制下线";
        } else {
            message = "当前会话未登录";
        }
        
        json.set("message", message);
        return json.toString();
    }

    /**
     * 处理400参数错误
     */
    @ResponseBody
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler({MethodArgumentNotValidException.class, BindException.class, ConstraintViolationException.class})
    public String handle400Error(Exception e) {
        String message;
        if (e instanceof MethodArgumentNotValidException) {
            MethodArgumentNotValidException ex = (MethodArgumentNotValidException) e;
            message = String.format("参数校验失败: %s", 
                ex.getBindingResult().getAllErrors().get(0).getDefaultMessage());
            log.error("参数校验失败: {}", ex.getBindingResult().getAllErrors(), e);
        } else if (e instanceof BindException) {
            BindException ex = (BindException) e;
            message = String.format("参数绑定失败: %s", 
                ex.getBindingResult().getAllErrors().get(0).getDefaultMessage());
            log.error("参数绑定失败: {}", ex.getBindingResult().getAllErrors(), e);
        } else if (e instanceof ConstraintViolationException) {
            ConstraintViolationException ex = (ConstraintViolationException) e;
            message = String.format("参数约束失败: %s", ex.getMessage());
            log.error("参数约束失败: {}", ex.getConstraintViolations(), e);
        } else {
            message = String.format("参数错误: %s", e.getMessage());
            log.error("参数错误", e);
        }
        
        JSONObject json = new JSONObject();
        json.set("code", HttpStatus.BAD_REQUEST.value());
        json.set("message", message);
        return json.toString();
    }

    /**
     * 处理自定义业务异常
     */
    @ResponseBody
    @ExceptionHandler(HisException.class)
    public String handleBusinessError(HisException e) {
        log.error("业务异常: {}", e.getMessage(), e);
        JSONObject json = new JSONObject();
        json.set("code", e.getCode());
        json.set("message", e.getMessage());
        return json.toString();
    }

    /**
     * 处理500系统错误
     */
    @ResponseBody
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler(Exception.class)
    public String handle500Error(Exception e) {
        JSONObject json = new JSONObject();
        json.set("code", HttpStatus.INTERNAL_SERVER_ERROR.value());
        
        if (e instanceof HttpRequestMethodNotSupportedException) {
            log.error("请求方法不支持", e);
            json.set("message", "请求方法不支持");
        } else if (e instanceof HttpMessageNotReadableException) {
            log.error("请求参数解析失败", e);
            json.set("message", "请求参数解析失败");  
        } else if(e instanceof BindException) {
            BindException ex = (BindException) e;
            log.error("请求参数绑定失败", ex);
            json.set("message", ex.getBindingResult().getAllErrors().get(0).getDefaultMessage());
        } else if (e instanceof MissingServletRequestParameterException) {
            log.error("请求参数缺失", e);
            json.set("message", "请求参数缺失");  
        } else if (e instanceof MethodArgumentNotValidException) {
            MethodArgumentNotValidException ex = (MethodArgumentNotValidException) e;
            json.set("message", ex.getBindingResult().getAllErrors().get(0).getDefaultMessage());
        } else if (e instanceof MissingServletRequestPartException) {
            log.error("请求体缺失", e);
            json.set("message", "请求体缺失");  
        } else if (e instanceof NullPointerException) {
            log.error("空指针异常", e);
            json.set("message", "系统出现空指针异常");
        } else if (e instanceof IllegalArgumentException) {
            log.error("非法参数: {}", e.getMessage(), e);
            json.set("message", "非法的参数: " + e.getMessage());
        } else if (e instanceof IllegalStateException) {
            log.error("非法状态: {}", e.getMessage(), e);
            json.set("message", "非法的状态: " + e.getMessage());
        } else if (e instanceof HisException) {
            log.error("业务异常: {}", e.getMessage(), e);
            json.set("message", e.getMessage());
        } else {
            log.error("系统错误", e);
            json.set("message", "系统内部错误，请联系管理员");
        }
        
        return json.toString();
    }
}
```
