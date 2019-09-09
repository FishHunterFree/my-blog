---
title: SpringCloud中Feign配置类使用
tags: [Spring Cloud,Feign]
categories:
  - Spring Cloud
reward: false
comment: false
top: 1
repo: FishHunterFree | my-blog
date: 2019-08-21 09:48:17
src:
---
本文记录SpringCloud中Feign配置的两种方式
<!--more-->

---
### 使用场景

目前环境下各系统间接口基本按照Restfull规范制定,Feign作为一个接口客户端,在SpringCloud体系下有定义明晰,开发便捷的优势.

同时,针对不同外部服务,我们可以通过自定义Feign的配置,来实现个性化的三方接口管理.本次介绍通过配置文件进行的全局配置及通过配置类的个性化配置两种实现方式

### 技术点

- SpringCloud Feign客户端
- SpringCloud Hystrix及Ribbon配置

### SpringCloud Feign的引入

对于如何搭建Feign,网络上有大量教程,此处引用其中一篇博客:

详细内容可以参考博文[SpringCloud实战五：Spring Cloud Feign 服务调用](https://blog.csdn.net/zhuyu19911016520/article/details/84933268)

需要注意的点为:

- 启动主类上必须添加@EnableFeignClients
- 接口类上使用@FeignClient注解,注解可选参数介绍如下

```
@FeignClient(value = "", fallback = , decode404 = true, url = "")

value:
必填,若在SpringCloud的Eureka体系下,指注册在Eureka中的服务名称(不区分大小写);若针对三方服务,仅仅唯一标识三方即可

fallback:
当前FeignClient的降级类,该降级类需要实现该接口

decode404:
默认false,设置为true后会对404类的异常进行捕捉而不是异常抛出,并返回处理后的Json结果

url:
指定调用的实际路径,若配置该参数,则在value无法匹配服务的情况下,会按照指定的url进行请求
```
- 接口中Get类型的请求,必须使用@RequestParam()注解入参

### SpringCloud Feign的配置

对于Feign的部分高级应用,网络上有大量教程,此处引用其中一篇博客:

详细内容可以参考博文[SpringCloud实战六：Spring Cloud Feign 高级应用](https://blog.csdn.net/zhuyu19911016520/article/details/84963568)

需要注意的点为:

- 对于Feign的熔断超时配置,需要同时针对Hystrix及Ribbon进行超时配置,且超时判断顺序为Feign -> Ribbon -> Hystrix,配置示例如下:

```
# 开启Feign熔断及Hystrix配置
feign:
  hystrix:
    enabled: true
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            # 连接超时时间
            timeoutInMilliseconds: 80000
 
# Ribbon 超时配置
ribbon:
  ReadTimeout: 60000
  ConnectTimeout: 60000
  # 同一实例最大重试次数，不包括首次调用。默认值为0
  MaxAutoRetries: 0
  # 同一个微服务其他实例的最大重试次数，不包括第一次调用的实例。默认值为1。
  # 此处配置需要结合业务场景,置为0后不再进行接口重试
  MaxAutoRetriesNextServer: 0
  # 是否所有操作（GET、POST等）都允许重试。默认值为false
  OkToRetryOnAllOperations: false
  
# 控制Feign客户端(proxy-external)熔断参数
# 此处配置会对全部@FeignClient(value="proxy-external")注解的类生效
feign.client.config:
  # 连接超时
  proxy-external.connectTimeout: 10000
  # 读超时
  proxy-external.readTimeout: 10000
```

- 可以通过配置类对每个接口类进行单独配置Feign规则,可参考

[SpringCloud系列十一：自定义Feign](https://www.cnblogs.com/jinjiyese153/p/8663763.html)

[SpringCloud系列十二：手动创建Feign](https://www.cnblogs.com/jinjiyese153/p/8664370.html)

[SpringCloud系列十三：Feign对继承、压缩、日志的支持以及构造多参数请求](https://www.cnblogs.com/jinjiyese153/p/8664968.html)

[Spring Cloud OpenFeign详解](https://blog.csdn.net/taiyangdao/article/details/81359394)

