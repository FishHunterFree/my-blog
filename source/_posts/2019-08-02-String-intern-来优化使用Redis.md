---
title: String.intern()来优化使用Redis
tags: [Java,String]
categories:
  - Java
reward: false
comment: false
top: 1
repo: FishHunterFree | my-blog
date: 2019-08-02 13:05:24
src:
---
本文记录使用String.intern()来优化使用Redis作为查询缓存的场景.
<!--more-->

---
### 使用场景

在一个接口中，该接口被多个线程并发访问，该接口主要做了以下工作：查询的时候是根据广告的类型查询符合该类型的广告，如果查询到广告，那么就返回该类型广告列表。同时，由于请求量比较的大，为了增加查询速度，减轻数据库的负担，我们在该层加入Redis。

我们会这样做，首先，我们去Redis查询该类型的广告，如果存在那么就直接返回，如果不存在，那么我们需要为该广告类型加锁，进行锁定，然后进行二次查询Redis（二次判空，保证查询排队的线程，只要有一个查询到了，那么其他的就不用查库了），然后查询数据库，再存入到Redis 中，返回即可。其他的线程，如果当时在排队，那么在二次判空的时候就可以拿到值，其他的则在第一次查询Redis中直接拿到了值。

### 案例分析

但是，要注意，在接口中我们是在线程内部的，我们锁定的只是一个字符串对象。首先，相同值的字符串，也可能是不同的对象；其次，该场景下字符串为局部变量（需注意是字面量赋值还是new对象），在线程内部，由于线程的栈封闭性，我们锁定的该字符串值，其他线程并不知道。

所以，我们需要一种策略，值相同的字符串就是一个对象，同时又是线程可见性的，那么字符串常量池就是一个很好的媒介，我们可以使用intern方法得到字符串常量池的引用，这样就保证了字符串值相同，那么就是一个对象，同时又是线程可见的。

但是，我们最好还是不要直接用字符串的intern方法，首先在Jdk1.6以及之前，字符串常量池是存储在永久代中的，也就是方法区中的，如果频繁使用该方法，那么就会造成该区域内存占有过大，造成垃圾收集器的GC，从而影响程序的运行。Java有一个很好的工具库， Guava ，其中封装了很多的工具类，其中很多平时都很常用，其中就有一个类，

```
Interner<String> pool = Interners.newWeakInterner();
```

该类对 intern 做了很多的优化，使用弱引用包装了你传入的字符串类型，所以，这样就不会对内存造成较大的影响，可以使用该类的 pool.intern(str)来进行对字符串intern。 这样就解决了内存的问题。

### 解决方案

伪码：


```
if (redis 存在) {
    // 直接return;
} else { 
    // 不存在
    Interner<String> pool = Interners.newWeakInterner();
    synchronized (pool.intern(str)) {
        if(redis 存在){
            return;
        }else{
            // 查库，入redis,返回
        }
    }
}
```

### 问题扩展

synchronized关键字针对共享变量，方法，类加锁；针对局部变量或者对象，无法起到锁的效果

[深入理解String#intern](https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html)

