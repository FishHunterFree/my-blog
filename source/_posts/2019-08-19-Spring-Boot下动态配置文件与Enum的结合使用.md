---
title: Spring Boot下动态配置文件与Enum的结合使用
tags: [Java,Spring Boot]
categories:
  - Java
reward: false
comment: false
top: 1
repo: FishHunterFree | my-blog
date: 2019-08-19 15:35:10
src:
---
本文记录在Spring Boot框架下,动态配置文件与Enum的结合使用的问题.
<!--more-->

---
### 使用场景

Enum类作为枚举常量,需要从SpringBoot配置文件(*.yml)中动态获取相关属性

### 技术点

- @Value注解使用
- 枚举类定义
- 动态注入的参数赋值给静态的枚举常量

### 实现代码

静态变量定义:

```
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class TypeConstants {

    public static String springbootKey = "springbootKey";

    public static String springbootValue;

    /**
     * TODO 此处需要额外注意:
     * 1.由于参数为静态变量,注解需要添加到属性对应的set方法上
     * 2.IDEA等工具自动生成的set方法,会添加static的描述,必须去掉,否则无法注入
     * 3.本静态变量需要被Spring注解扫描,需要在类上添加@Component
     * @param springbootValue
     * @return void
     **/
    @Value("${custom.springbootValue}")
    public void setSpringbootValue(String springbootValue) {
        TypeConstants.springbootValue = springbootValue;
    }

    /***
     * TODO 将静态变量和枚举结合使用
     *
     **/
    public enum BusinessType {

        /**
         * 定义在配置文件中的属性
         */

        CUSTOM(springbootKey, springbootValue);

        private String key;
        private String value;

        BusinessType(String key, String value) {
            this.value = value;
            this.key = key;
        }

        public String getValue() {
            return this.value;
        }

        public String getKey() {
            return this.key;
        }

        /**
         * 通过Key获取Value
         */
        public static String getType(String springbootKey) {
            TypeConstants.BusinessType[] types = TypeConstants.BusinessType.values();
            for (TypeConstants.BusinessType type : types) {
                if (type.getKey().equals(springbootKey)) {
                    return type.getValue();
                }
            }
            return null;
        }
    }
}

```

模拟调用:

```
@RestController
public class DemoController {

    @GetMapping("/type-constants-demo")
    public String typeConstantsDemo() {
        String value = TypeConstants.BusinessType.getType(TypeConstants.springbootKey);
        return "Value:"+value;
    }


}

```
