---
title: Spring AOP日志记录实现
tags: [Java,Spring]
categories:
  - Java
reward: false
comment: false
top: 1
repo: FishHunterFree | my-blog
date: 2019-08-19 16:26:26
src:
---
本文记录Spring AOP日志记录实现.
<!--more-->

---
### 使用场景

通过切面编程,为Rest请求记录入参及回参的日志,同时对于整合链路跟踪的项目,记录请求Trace信息

### 技术点

- @Aspect及其派生的注解使用
- 获取实际客户端Ip工具类
- 链路跟踪相关工具类

### 实现代码

链路跟踪Pom依赖:

```
        <!-- 链路跟踪模块 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-sleuth</artifactId>
        </dependency>
```


日志记录切面类:

```
import brave.Span;
import brave.Tracer;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestAttributes;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;
import org.springframework.web.multipart.MultipartFile;

import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import java.util.Map;
import java.util.Objects;
import java.util.UUID;
/**
 * TODO 日志记录切面类
 **/
@Component
@Aspect 
@Slf4j
public class LoggerAspect {

    @Autowired
    Tracer tracer;

    /**
     * TODO 指定切点位置,主要针对RestController及Controller修饰的对外接口
     * @param
     * @return void
     * @throws
     * @date 2019/8/19 16:45
     **/
    @Pointcut("@within(org.springframework.web.bind.annotation.RestController) || @within(org.springframework.stereotype.Controller)")
    public void annotationController() {
    }

    /**
     * TODO 此处使用环绕注解,通过joinPoint.proceed()获取返回结果,需要注意点为异常处理
     * @param joinPoint
     * @return java.lang.Object
     * @throws
     * @date 2019/8/19 16:47
     **/
    @Around(value = "annotationController()")
    public Object showBeginLog(ProceedingJoinPoint joinPoint) throws Throwable {
        // 开始时间
        Long startTime = System.currentTimeMillis();
        String traceId = null;
        Object result = null;
        try {
            RequestAttributes requestAttributes = RequestContextHolder.getRequestAttributes();
            if (requestAttributes instanceof ServletRequestAttributes) {
                ServletRequestAttributes servletRa = (ServletRequestAttributes) requestAttributes;
                if (Objects.nonNull(servletRa)) {
                    traceId = extractTraceId(this.tracer);
                    HttpServletRequest request = servletRa.getRequest();
                    StringBuffer requestURL = request.getRequestURL();
                    String requestURI = request.getRequestURI();
                    String method = request.getMethod();
                    String ipAddress = getIpAddr(request);

                    //封装入参Request列表
                    Map<String, String[]> parameterMap = request.getParameterMap();
                    // 封装入参Body列表
                    String requestStr = getRequestParam(joinPoint);
                    requestStr = parameterHandle(requestStr, 10000);

                    // url
                    log.info("traceId={}，请求的url={}", traceId, requestURL);
                    // uri
                    log.info("traceId={}，请求的uri={}", traceId, requestURI);
                    // method
                    log.info("traceId={}，请求method={}", traceId, method);
                    // ip
                    log.info("traceId={}，请求源ip={}", traceId, ipAddress);
                    // class_method
                    log.info("traceId={}，请求方法class_method={}", traceId,
                            joinPoint.getSignature().getDeclaringTypeName() + "_" + joinPoint.getSignature().getName());
                    // args[]
                    log.info("traceId={}，请求参数args={}", traceId, JSON.toJSONString(parameterMap));
                    // body
                    log.info("traceId={}，请求体body={}", traceId, requestStr);


                    /*
                     * 执行完方法的返回值：调用proceed()方法，就会触发切入点方法执行
                     * result的值就是被拦截方法的返回值
                     * 需要注意,当此方法之前抛出异常,会导致接口无法被执行,对之前代码可能出现的异常,一定需要处理并不再向上抛出
                     */
                    result = joinPoint.proceed();

                    return result;
                }
            }
        } catch (Exception e) {
            log.info("traceId=" + traceId + "，异常信息exception=", e);
			throw e;
        } finally {
            // 执行时间
            long handleTime = System.currentTimeMillis() - startTime;
            String responseStr = result == null ? "无" : JSON.toJSONString(result);
            responseStr = parameterHandle(responseStr, 10000);
            log.info("traceId={}，耗时={}ms,响应结果response={}", traceId, handleTime, responseStr);
        }
        return result;
    }


    /**
     * @param paramStr
     * @param strlength
     * @return
     * @Description : 参数处理，超过指定长度字符的，只显示1000...
     */
    private String parameterHandle(String paramStr, int strlength) {
        if (paramStr.length() > strlength) {
            paramStr = paramStr.substring(0, 1000) + "...";
        }
        if (paramStr.length() > 10) {
            paramStr = "[" + paramStr + "]";
        }
        return paramStr;
    }

    /***
     * @Description : 获取请求参数
     * @param point
     * @return
     */
    private String getRequestParam(ProceedingJoinPoint point) {
        Object[] paramsArray = point.getArgs();
        String requestStr = argsArrayToString(paramsArray);
        return requestStr;
    }

    /**
     * 请求参数拼装,由于point.getArgs()获得的参数包含ServletRequest,ServletResponse等参数,需要在Json转换处理时特别注意
     *
     * @param args
     * @return
     */
    private String argsArrayToString(Object[] args) {
        Object[] paramsArray = new Object[args.length];
        for (int i = 0; i < args.length; i++) {
            if (args[i] instanceof ServletRequest || args[i] instanceof ServletResponse || args[i] instanceof MultipartFile) {
                // ServletRequest不能序列化，从入参里排除，否则报异常：java.lang.IllegalStateException: It is illegal to call this method if the current request is not in asynchronous mode (i.e. isAsyncStarted() returns false)
                // ServletResponse不能序列化 从入参里排除，否则报异常：java.lang.IllegalStateException: getOutputStream() has already been called for this response
                continue;
            }
            paramsArray[i] = args[i];
        }

        String params = "";
        if (paramsArray != null && paramsArray.length > 0) {
            try {
                params = JSONObject.toJSONString(paramsArray);
            } catch (Exception e) {
                params = paramsArray.toString();
            }
        }
        return params.trim();
    }

    /**
     * 记录浏览器访问路径,仅当上下游项目开启链路跟踪时可用
     *
     * @param tracer
     * @return
     */
    public static String extractTraceId(Tracer tracer) {
        Span span = tracer.currentSpan();
        Long parentId = span.context().parentIdAsLong();
        String traceId = (span == null ? null : String.valueOf(parentId));
        if (traceId == null || traceId.isEmpty()) {
            traceId = "[NoClientTraceId]" + UUID.randomUUID().toString();
        }
        return traceId;
    }

    /**
     * 获取客户端IP
     * @param request
     * @return
     */
    public static String getIpAddr(HttpServletRequest request) {
        String ip = request.getHeader("x-forwarded-for");

        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("WL-Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddr();
        }
        if (ip != null && ip.equalsIgnoreCase("0:0:0:0:0:0:0:1")) {
            ip = "127.0.0.1";
        }
        /*
         * 对于通过多个代理的情况，第一个IP为客户端真实IP,多个IP按照','分割
         * "***.***.***.***".length() = 15
         */
        if(ip!=null && ip.length()>15){
            if(ip.indexOf(",")>0){
                ip = ip.substring(0,ip.indexOf(","));
            }
        }
        return ip;
    }
}


```

