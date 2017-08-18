---
layout: post
title: "记一次UTF-8引发的宕机"
date: 2017-03-06 23:23:06 +0800
comments: true
categories: JAVA Jetty
---

+ 原本以为今天是个风平浪静的周一，没想到下午一个报警打破了全部的宁静

### 现象
+ 线上集群报警激增，客户端大量报错"connect reset"，服务端也报错"connect reset by peer"
+ 线上机器java进程cpu飙升至1500%+，rest服务响应非常缓慢，jstack甚至无法连接socket
+ 重启进程后很快又陷入缓慢状态。
<!-- more -->

### 排查
+ dump后发现大量线程BLOCKED，都在等待`sun.nio.cs.FastCharsetProvider.charsetForName`

![BLOCKED](https://zos.alipayobjects.com/rmsportal/oPdVBZYffSSFkHOHJtDt.jpg)
![blocked detail.png](https://zos.alipayobjects.com/rmsportal/gaCyaAFQVqtdVHfWNeFM.png)
+ 查看源码，可以看到这是个同步方法

![DingTalk20170306224956.png](https://zos.alipayobjects.com/rmsportal/YrkKfOrXunvRpTNdivhJ.png)
+ 而这个罪魁祸首实际上非常常见,就是下面这一行代码
```java
IOUtils.toString(input, "UTF-8");
```
+ 他会调用forName来找到具体的Charset，最后就会调用到上面的同步代码
![DingTalk20170306231318.png](https://zos.alipayobjects.com/rmsportal/GGPZllkIiAODhMgAKMqO.png)


### 原因
+ 至此原因基本明了，因为Charset的阻塞，造成了大量jetty线程block，jetty线程用完，被迫关闭连接，Connect Reset
+ 具体是不是这样等过两天重现下

### 解决方案
+ 而这个问题的解决方案也非常简单，只需要将所有"UTF-8"换成`Charsets.UTF_8`即可，`Charsets.UTF_8`是个final变量，所以不会造成阻塞

### 反思
+ 细节非常重要，要多学习学习开源代码

### 后记(2017.03.26) 
+ 今天想用spring boot 1.5.2重现一次bug，但发现没法重现，试了狂压，多个异步线程池同时IOUtils，但是效果都不好，都没有造成connect reset，可能UTF8的锁也只是个表象，更有深层原因，也有可能是底层升级已经修复了。等后面再碰到再仔细研究下吧



