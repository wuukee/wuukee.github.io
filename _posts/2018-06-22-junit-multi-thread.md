---
layout: post
category: "java"
title:  "基于junit的多线程单元测试"
tags: [java,多线程,单元测试]
---

**背景**  

&#8194;用Spring的线程池调度器ThreadPoolTaskScheduler进行多线程的调度。

**问题**  

&#8194;在main方法中正常运行，然而用junit进行单元测试的时候，子线程并没有按照预设的情况执行，并且出现以下信息：
```
support.GenericWebApplicationContext: Closing org.springframework.web.context.support.GenericWebApplicationContext@63a5e46c: startup date [Fri Jun 22 11:42:16 CST 2018]; root of context hierarchy
```

**分析**  

&#8194;运行过程中没有报异常，在子线程并没有结束的情况下，主线程先终止了，这说明主线程并没有对子线程进行监控。找到junit.textui包下的TestRunner源码，我们看到这样的代码：
```
    public static void main(String args[]) {
        TestRunner aTestRunner = new TestRunner();
        try {
            TestResult r = aTestRunner.start(args);
            if (!r.wasSuccessful()) {
                System.exit(FAILURE_EXIT);
            }
            System.exit(SUCCESS_EXIT);
        } catch (Exception e) {
            System.err.println(e.getMessage());
            System.exit(EXCEPTION_EXIT);
        }
    }
```
&#8194;再点到TestResult中
```
    /**
     * Returns whether the entire test was successful or not.
     */
    public synchronized boolean wasSuccessful() {
        return failureCount() == 0 && errorCount() == 0;
    }
```
&#8194;总结这两段代码：当aTestRunner调用start方法后不会去等待子线程执行完毕在关闭主线程，而是直接调用TestResult.wasSuccessful()方法，不论返回什么，主线程都会执行相应的System.exit()，这个方法会结束当前正在运行中的java虚拟机，所以使用junit测试多线程方法的结果就不正常了

**解决**  

&#8194;解决这个问题的关键是想办法让主线程暂时不终止：  
1. 让主线程休眠一段时间，直到子线程能够运行结束。  
```
Thread.sleep(60 * 1000);
```  
2. 使用CountDownLatch工具类，让主线程阻塞。
```
CountDownLatch countDownLatch = new CountDownLatch(5);
countDownLatch.await(70, TimeUnit.SECONDS);
``` 
