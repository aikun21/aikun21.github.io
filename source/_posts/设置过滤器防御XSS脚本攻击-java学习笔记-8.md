---
title: 设置过滤器防御XSS脚本攻击-java学习笔记(8)
date: 2024-12-29 10:30:37
tags:
---

首先我们要创建一个执行转义的封装类，这个类继承 HttpservletRequestwrapper 父类。在Web项目中，我们无法修改 HttpservletRequest 实现类的内容，因为请求的实现类是由各个Web容器厂商自己扩展的。但是有时候我们还想修改请求类中的内容，这该怎么办呢?Java语言给我们留出了缺口，我们只要继承Java Web内置的HttpservletRequestwrapper 父类，就能修改请求类的内容。如果我们能修改请求类的内容，我要修改获取请求数据的函数，返回的并不是客户端Form表单或者Ajax提交的数据，而是经过转义之后数据。

<!-- more -->

在 com.jiang.his.config.xss 包中创建 XssHttpServletRequestwrapper 类。把获取请求头和请求体数据的方法都要重写，返回的是经过XSS转义后的数据。

```java
package com.jiang.his.config.xss;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletRequestWrapper;
import cn.hutool.core.util.StrUtil;
import cn.hutool.http.HtmlUtil;
import java.util.HashMap;
import java.util.Map;
import javax.servlet.ServletInputStream;
import javax.servlet.ReadListener;
import java.io.*;
import org.apache.commons.io.IOUtils;
import cn.hutool.json.JSONObject;
import cn.hutool.json.JSONUtil;

public class XssHttpServletRequestWrapper extends HttpServletRequestWrapper {
    
    // 保存原始请求体内容
    private final byte[] body;
    
    public XssHttpServletRequestWrapper(HttpServletRequest request) throws IOException {
        super(request);
        // 读取并保存请求体
        try (InputStream in = request.getInputStream()) {
            body = IOUtils.toByteArray(in);
        }
    }

    @Override
    public String getHeader(String name) {
        String value = super.getHeader(name);
        return escapeXss(value);
    }

    @Override
    public String getParameter(String name) {
        String value = super.getParameter(name);
        return escapeXss(value);
    }

    @Override
    public String[] getParameterValues(String name) {
        String[] values = super.getParameterValues(name);
        if (values == null) {
            return null;
        }
        String[] escapeValues = new String[values.length];
        for (int i = 0; i < values.length; i++) {
            escapeValues[i] = escapeXss(values[i]);
        }
        return escapeValues;
    }

    @Override
    public Map<String, String[]> getParameterMap() {
        Map<String, String[]> parameterMap = super.getParameterMap();
        Map<String, String[]> escapedParameterMap = new HashMap<>();
        
        for (Map.Entry<String, String[]> entry : parameterMap.entrySet()) {
            String[] values = entry.getValue();
            String[] escapedValues = new String[values.length];
            for (int i = 0; i < values.length; i++) {
                escapedValues[i] = escapeXss(values[i]);
            }
            escapedParameterMap.put(entry.getKey(), escapedValues);
        }
        
        return escapedParameterMap;
    }
    
    @Override
    public ServletInputStream getInputStream() throws IOException {
        // 如果是空body，直接返回原始流
        if (body == null || body.length == 0) {
            return super.getInputStream();
        }

        // 转换请求体内容为字符串并进行XSS过滤
        String bodyString = new String(body);
        try {
            // 使用hutool解析JSON
            JSONObject jsonObject = JSONUtil.parseObj(bodyString);
            // 递归处理JSON中的所有字符串值
            cleanXssJson(jsonObject);
            bodyString = jsonObject.toString();
        } catch (Exception e) {
            // 如果不是JSON格式，直接进行XSS过滤
            bodyString = escapeXss(bodyString);
        }

        // 将过滤后的内容转换为InputStream
        final ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(bodyString.getBytes());

        // 返回自定义的ServletInputStream
        return new ServletInputStream() {
            @Override
            public boolean isFinished() {
                return false;
            }

            @Override
            public boolean isReady() {
                return true;
            }

            @Override
            public void setReadListener(ReadListener readListener) {
                // 不需要实现
            }

            @Override
            public int read() {
                return byteArrayInputStream.read();
            }
        };
    }

    // 递归清理JSON对象中的XSS
    private void cleanXssJson(JSONObject jsonObject) {
        for (String key : jsonObject.keySet()) {
            Object value = jsonObject.get(key);
            if (value instanceof String) {
                jsonObject.set(key, escapeXss((String) value));
            } else if (value instanceof JSONObject) {
                cleanXssJson((JSONObject) value);
            }
        }
    }

    /**
     * XSS转义处理
     */
    private String escapeXss(String value) {
        if (StrUtil.isBlank(value)) {
            return value;
        }
        // 使用hutool的HtmlUtil进行HTML转义
        return HtmlUtil.cleanHtmlTag(value);
    }
} 
```

在 com.jiang.his.config.xss 包中，创建 XssFilter 类。把 HttpServletRequest 包装成 XssHttpServletRequestWrapper，然后传给Web方法。

```java
package com.jiang.his.config.xss;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * XSS过滤器
 */
public class XssFilter implements Filter {
    
    // 排除链接
    private List<String> excludes = new ArrayList<>();
    
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        // 获取排除的URL
        String temp = filterConfig.getInitParameter("excludes");
        if (temp != null) {
            String[] url = temp.split(",");
            for (int i = 0; i < url.length; i++) {
                excludes.add(url[i]);
            }
        }
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        // 判断是否需要过滤
        if (handleExcludeURL(httpRequest)) {
            chain.doFilter(request, response);
            return;
        }
        
        // 判断是否是文件上传请求
        if (isMultipartContent(httpRequest)) {
            chain.doFilter(request, response);
            return;
        }
        
        // 包装请求对象，处理XSS
        XssHttpServletRequestWrapper xssRequest = new XssHttpServletRequestWrapper(httpRequest);
        chain.doFilter(xssRequest, response);
    }
    
    private boolean handleExcludeURL(HttpServletRequest request) {
        if (excludes == null || excludes.isEmpty()) {
            return false;
        }
        String url = request.getServletPath();
        for (String pattern : excludes) {
            Pattern p = Pattern.compile("^" + pattern);
            Matcher m = p.matcher(url);
            if (m.find()) {
                return true;
            }
        }
        return false;
    }
    
    private boolean isMultipartContent(HttpServletRequest request) {
        String contentType = request.getContentType();
        return contentType != null && contentType.toLowerCase().startsWith("multipart/");
    }

    @Override
    public void destroy() {
        // 销毁时不需要特殊处理
    }
}
```

在 com.jiang.his.config.xss 包中，创建 XssConfig 类。在类中，创建一个过滤器注册方法，把 XssFilter 注册到过滤器链中。

```java
package com.jiang.his.config.xss;

import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * XSS过滤器配置
 */
@Configuration
public class XssConfig {

    @Bean
    public FilterRegistrationBean<XssFilter> xssFilterRegistrationBean() {
        FilterRegistrationBean<XssFilter> registration = new FilterRegistrationBean<>();
        
        // 注册XSS过滤器
        registration.setFilter(new XssFilter());
        // 过滤所有URL
        registration.addUrlPatterns("/*");
        // 过滤器名称
        registration.setName("XssFilter");
        // 优先级，值越小优先级越高
        registration.setOrder(1);
        
        return registration;
    }
} 
```

## 允许跨域访问CORS

在 com,jiang.his.config包中创建 CorsConfig.java类，扩展 WebMvcConfigurer 接囗。

```java
package com.jiang.his.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

/**
 * 跨域配置
 */
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")  // 所有接口
                .allowedOriginPatterns("*")  // 允许所有来源
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")  // 允许的HTTP方法
                .allowedHeaders("*")  // 允许所有请求头
                .allowCredentials(true)  // 允许发送cookie
                .maxAge(3600L);  // 预检请求的有效期，单位为秒
    }
} 
```
