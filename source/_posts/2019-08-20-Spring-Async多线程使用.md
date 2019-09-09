---
title: Spring Async多线程使用
tags: [Java,Spring,Async]
categories:
  - Java
reward: false
comment: false
top: 1
repo: FishHunterFree | my-blog
date: 2019-08-20 14:43:25
src:
---
本文记录Spring Async对Java多线程的支持
<!--more-->

---
### 使用场景

Java在处理多线程时需要用到线程池及其相关的API,配置较为零散,学习成本较高.Spring提供了便捷的配置类来支持多线程的实现.

### 技术点

- Java多线程
- Spring Async

### Spring Async配置类

```
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;
import java.util.concurrent.ThreadPoolExecutor;

/**
 * 开启多线程操作，注解方式
 * 在需要进行异步处理的方法上加上@Async注解
 */
@Configuration
@EnableAsync
public class AsyncConfig {

    /**
     * 设置线程池基本大小值, 线程池维护线程的最少数量
     */
    private int corePoolSize = 10;
    /**
     * 设置线程池最大值
     */
    private int maxPoolSize = 200;
    /**
     * 线程池所使用的缓冲队列大小
     */
    private int queueCapacity = 1024;
    /**
     * 配置线程最大空闲时间
     */
    private int keepAliveSeconds = 50000;

    @Bean
    public Executor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(corePoolSize);
        executor.setMaxPoolSize(maxPoolSize);
        executor.setQueueCapacity(queueCapacity);
        executor.setKeepAliveSeconds(keepAliveSeconds);
        executor.setThreadNamePrefix("AsyncExecutor-");

        /*
         * rejection-policy：当pool已经达到max size的时候，如何处理新任务
         * CALLER_RUNS：不在新线程中执行任务，而是由调用者所在的线程来执行
         */
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}


```


### Spring Async服务类-控制层


```
import org.eureka.service.impl.AsyncService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;

/**
 * Description: 异步任务测试类
 */
@RestController
public class AsyncController {
    @Autowired
    private AsyncService asyncService;
 
    @RequestMapping(value = "/hello",method = RequestMethod.GET)
    public String testAsyncNoRetrun(){
        long start = System.currentTimeMillis();
         asyncService.doNoReturn();
         return String.format("任务执行成功,耗时{%s}", System.currentTimeMillis() - start);
    }

    @GetMapping("/hi")
    public Map<String, Object> testAsyncReturn() throws ExecutionException, InterruptedException {
        long start = System.currentTimeMillis();

        Map<String, Object> map = new HashMap<>();
        List<Future<String>> futures = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            Future<String> future = asyncService.doReturn(i);
            futures.add(future);
        }
        /*
         * 读取的时候,记得要批量读取不能单独读取,否则无法实现异步的效果
         * 且需要注意:
         * 异步方法调用一定需要通过Spring代理,@Async和@Transactional注解类似,只有在代理模式下生效
         */
        List<String> response = new ArrayList<>();
        for (Future future : futures) {
            String string = (String) future.get();
            response.add(string);
        }
        map.put("data", response);
        map.put("消耗时间", String.format("任务执行成功,耗时{%s}毫秒", System.currentTimeMillis() - start));
        return map;
    }
}


```

### Spring Async服务类-实现层


```
import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.AsyncResult;
import org.springframework.stereotype.Service;

import java.util.Date;
import java.util.concurrent.Future;

/**
 * Description: 异步任务测试类
 */
@Service
public class AsyncService {

    /**
     * TODO 处理无返回的异步任务
     * @param
     * @return void
     * @throws
     * @date 2019/8/20 16:34
     **/
    @Async
    public void doNoReturn(){
        try {
            // 这个方法执行需要三秒
            Thread.sleep(3000);
            System.out.println("方法执行结束" + new Date());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    /**
     * TODO 处理有返回的异步任务
     * @param i
     * @return java.util.concurrent.Future<java.lang.String>
     * @throws
     * @date 2019/8/20 16:35
     **/
    @Async
    public Future<String> doReturn(int i){
        try {
            // 这个方法需要调用500毫秒
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // 消息汇总
        return new AsyncResult<>(String.format("这个是第{%s}个异步调用的证书", i));
    }
}


```
