---
layout: post
category: "java"
title:  "基于junit的多线程单元测试"
tags: [java,多线程,单元测试]
---

#####背景
&#8194;用Spring的线程池调度器ThreadPoolTaskScheduler进行多线程的调度。在main方法中正常运行，然而用junit进行单元测试的时候，子线程并没有按照预设的情况执行，并且出现以下信息：
```
support.GenericWebApplicationContext: Closing org.springframework.web.context.support.GenericWebApplicationContext@63a5e46c: startup date [Fri Jun 22 11:42:16 CST 2018]; root of context hierarchy
```

&#8194;

![preface1](https://github.com/wuukee/wuukee.github.io/raw/master/images/preface1.jpg)