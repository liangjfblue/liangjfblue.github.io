---
layout:     post                  
title:      redis
subtitle:   redis存用户信息用key-value还是hash
date:       2019-08-11          
author:     Liangjf                  
header-img: img/post_bg_86.jpg
catalog: true                      
tags:                       
    - redis
---

# redis存储用户信息用key-value还是hash

在平时开发中，用户信息的存储是经常碰到的，比如经常把用户数据缓存到redis中，此时，有两种存储方式：

- 1、把整个用户信息json序列化后通过key-value的形式存储
- 2、把用户信息拆分成一个个字段通过hash结构来存储

这两种方式，在平时开发中，应该如何选择呢？

## 方案解释

### 方案1

> 将整个对象作为json编码的字符串存储在一个键中，并使用set(或者list，如果更合适的话)跟踪所有对象

    INCR id:users
    SET user:{id} '{"name":"Fred","age":25}'
    SADD users {id}

### 方案2

    INCR id:users
    HMSET user:{id} name "Fred" age 25
    SADD users {id}

## 两者的优缺点：

**方案1**

- 优点: 每个对象都是一个完整的Redis键。JSON解析非常快，尤其是当需要同时访问该对象的许多字段时。
- 缺点: 当只需要访问一个字段时，速度会变慢。

**方案2**

- 优点: 每个对象都是一个完整的Redis键。不需要解析JSON字符串。
- 缺点:当需要访问对象中的所有/大部分字段时，可能会比较慢。此外，嵌套对象(对象中的对象)也不容易存储。

最后来总结下，如何选择正确的方案。应该从效率，占有内存两个方向来考虑。

**选择方案1：**

- 在大多数访问中使用了大部分字段
- 键值不怎么会变化的，前后差异小的

**选择方案2：**

- 在大多数访问中只使用单个字段。
- 已知哪些字段是可用的

## 总结
总的来说，我们应该选择对大多数请求耗时最少，最少查询次数的那种方案。







