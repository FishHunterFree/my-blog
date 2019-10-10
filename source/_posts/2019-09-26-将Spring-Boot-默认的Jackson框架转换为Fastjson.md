---
title: 将Spring Boot 默认的Jackson框架转换为Fastjson
tags:
  - [Spring Boot]
categories:
  - Spring Boot
reward: false
comment: false
top: 1
repo: FishHunterFree | my-blog
date: 2019-09-26 16:55:42
src:
---
本文记录将Spring Boot 默认的Jackson框架转换为Fastjson的实现方式及注意事项
<!--more-->

---
### 使用场景

在用Feign Client 接口调用，由于Jackson对null等特殊值处理存在异常，故改用Fastjson解析数据

### 操作步骤

1.增加文件FeignConfig，注入Bean，修改默认Feign默认的解析方式

2.由于Fastjson1.2.28后，增加了对Content-type验证，故添加多种MediaType

```
@Configuration
public class FeignConfig {

    @Bean
    public ResponseEntityDecoder feignDecoder() {
        HttpMessageConverter fastJsonConverter = createFastJsonConverter();
        ObjectFactory<HttpMessageConverters> objectFactory = () -> new HttpMessageConverters(fastJsonConverter);
        return new ResponseEntityDecoder(new SpringDecoder(objectFactory));
    }

    @Bean
    public SpringEncoder feignEncoder(){
        HttpMessageConverter fastJsonConverter = createFastJsonConverter();
        ObjectFactory<HttpMessageConverters> objectFactory = () -> new HttpMessageConverters(fastJsonConverter);
        return new SpringEncoder(objectFactory);
    }

    private HttpMessageConverter createFastJsonConverter() {

        //创建fastJson消息转换器
        FastJsonHttpMessageConverter fastConverter = new FastJsonHttpMessageConverter();

        //升级最新版本需加=============================================================
        List<MediaType> supportedMediaTypes = new ArrayList<>();
        supportedMediaTypes.add(MediaType.APPLICATION_JSON);
        supportedMediaTypes.add(MediaType.APPLICATION_JSON_UTF8);
        supportedMediaTypes.add(MediaType.APPLICATION_ATOM_XML);
        supportedMediaTypes.add(MediaType.APPLICATION_FORM_URLENCODED);
        supportedMediaTypes.add(MediaType.APPLICATION_OCTET_STREAM);
        supportedMediaTypes.add(MediaType.APPLICATION_PDF);
        supportedMediaTypes.add(MediaType.APPLICATION_RSS_XML);
        supportedMediaTypes.add(MediaType.APPLICATION_XHTML_XML);
        supportedMediaTypes.add(MediaType.APPLICATION_XML);
        supportedMediaTypes.add(MediaType.IMAGE_GIF);
        supportedMediaTypes.add(MediaType.IMAGE_JPEG);
        supportedMediaTypes.add(MediaType.IMAGE_PNG);
        supportedMediaTypes.add(MediaType.TEXT_EVENT_STREAM);
        supportedMediaTypes.add(MediaType.TEXT_HTML);
        supportedMediaTypes.add(MediaType.TEXT_MARKDOWN);
        supportedMediaTypes.add(MediaType.TEXT_PLAIN);
        supportedMediaTypes.add(MediaType.TEXT_XML);
        fastConverter.setSupportedMediaTypes(supportedMediaTypes);

        //创建配置类
        FastJsonConfig fastJsonConfig = new FastJsonConfig();
        //修改配置返回内容的过滤
        //WriteNullListAsEmpty  ：List字段如果为null,输出为[],而非null
        //WriteNullStringAsEmpty ： 字符类型字段如果为null,输出为"",而非null
        //DisableCircularReferenceDetect ：消除对同一对象循环引用的问题，默认为false（如果不配置有可能会进入死循环）
        //WriteNullBooleanAsFalse：Boolean字段如果为null,输出为false,而非null
        //WriteMapNullValue：是否输出值为null的字段,默认为false
        fastJsonConfig.setSerializerFeatures(
                SerializerFeature.DisableCircularReferenceDetect,
                SerializerFeature.WriteMapNullValue
        );
        fastConverter.setFastJsonConfig(fastJsonConfig);

        return fastConverter;
    }
}
```

### 实际调用链

1.通过Feign调用接口

2.默认进入org.springframework.http.converter.AbstractHttpMessageConverter 的 writeInternal 方法

3.FastJson实现该方法，进行数据转换处理com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter 中的 write 方法


- 参考资料

[FastJson 文档链接](https://github.com/alibaba/fastjson/wiki/FastJson-%E6%96%87%E6%A1%A3%E9%93%BE%E6%8E%A5)


