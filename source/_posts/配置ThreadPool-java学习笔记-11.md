---
title: 配置ThreadPool-java学习笔记(11)
date: 2024-12-29 19:15:38
tags:
---

给后端Java项目添加线程池功能，将来有些复杂耗时的任务，我们可以交给线程池去做。比如说生成体检报告这种事情就非常耗时，如果用上了线程池，每天的体检报告用多线程去生成，速度肯定比单线程快很多。

在com.jiang.his.config包中，创建 ThreadPoolConfig.java 类

```java
package com.jiang.his.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;
import java.util.concurrent.ThreadPoolExecutor;

/**
 * 线程池配置
 */
@Configuration
@EnableAsync
public class ThreadPoolConfig {

    /**
     * 核心线程数
     */
    private static final int CORE_POOL_SIZE = 8;

    /**
     * 最大线程数
     */
    private static final int MAX_POOL_SIZE = 16;

    /**
     * 队列容量
     */
    private static final int QUEUE_CAPACITY = 1000;

    /**
     * 线程空闲时间
     */
    private static final int KEEP_ALIVE_SECONDS = 60;

    /**
     * 线程名前缀
     */
    private static final String THREAD_NAME_PREFIX = "his-async-";

    @Bean("asyncTaskExecutor")
    public Executor asyncTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        
        // 核心线程数：线程池创建时候初始化的线程数
        executor.setCorePoolSize(CORE_POOL_SIZE);
        
        // 最大线程数：线程池最大的线程数，只有在缓冲队列满了之后才会申请超过核心线程数的线程
        executor.setMaxPoolSize(MAX_POOL_SIZE);
        
        // 缓冲队列：用来缓冲执行任务的队列
        executor.setQueueCapacity(QUEUE_CAPACITY);
        
        // 允许线程的空闲时间：超过了核心线程数之外的线程，在空闲时间到达之后会被销毁
        executor.setKeepAliveSeconds(KEEP_ALIVE_SECONDS);
        
        // 线程池名的前缀：设置好了之后可以方便定位处理任务所在的线程池
        executor.setThreadNamePrefix(THREAD_NAME_PREFIX);
        
        // 缓冲队列满了之后的拒绝策略：由调用线程处理（一般是主线程）
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        
        // 初始化线程池
        executor.initialize();
        
        return executor;
    }
} 
```

## 拒绝策略

### AbortPolicy策略

直接抛出RejectedExecutionException异常，属于线程池默认的拒绝方案。

### CallerRunsPolicy策略

线程池拒绝执行，反而推给主线程去执行任务，可以保证线程任务一定能执行，而不是随便放弃。比如说生成体检报告，如果线程池放弃任务，就意味着某个体检报告无法生成，这肯定是不行的。

### DiscardOldestPolicy策略

丢弃队列中最早的未处理任务，然后尝试重新执行任务。

### DiscardPolicy策略

直接丢弃任务，不予任何处理也不抛出异常。

## 开启异步线程

通常我们使用线程池的时候需要我们自己往线程池里面添加任务，但是SpringBoot提供了异步线程技术，我们规定好任务类，SpringBoot会把任务对象交给线程池去执行。之所以叫做异步线程，是因为线程池中的线程相对于主线程来说就是异步的:主线程把任务推给线程池，然后就继续往下做别的事情了，并不会等待线程池的执行结果，所以是一种异步的效果。

在 HisApiApplication.java 主类的声明中，添加上 @EnableAsync 注解，就可以开启异步线程功能了

将来我们想要使用异步线程功能的时候，需要自己创建一个任务类，然后给里面的方法添加上 @Async ，那么这个方法将会推给线程池的空闲线程去执行，也就意味着生成体检报告这种耗时费力的事情已经脱离主线程了。

```java
@Service
public class AsyncService {

    @Async("asyncTaskExecutor")
    public void asyncTask() {
        System.out.println("异步线程执行任务");
    }
}
```
