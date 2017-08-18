---
layout: post
title: "RequestMap2Retrofit"
date: 2017-03-10 09:47:21 +0800
comments: true
categories: 
---
+ 最近写项目，各种Rest的接口调来调去，RequestMap总要反复写，服务发布方写一遍，调用方再写一遍，非常麻烦。
+ 业界目前有JAX-RS，resteasy通过发布facade包来避免这个问题。但是无力推进所有服务方修改，所以写了个小脚本用来将RequestMap转换成Retrofit2的接口。
<!-- more -->
+ 目前只实现了基本的功能，用node编写的，注释相对挺多，欢迎大家一起完善。git地址: [https://github.com/czjxy881/RequestMap2Retrofit/blob/master/RequestMap2Retrofit.js](https://github.com/czjxy881/RequestMap2Retrofit/blob/master/RequestMap2Retrofit.js)
+ 已实现功能:
  1. RequestMap的path和method的转换
  2. 参数的RequestBody和PathVariable两个注解的转换
  3. 注释保留
+ TODOList:
  1. 完善注解
  2. 自动生成整个类，而不是只有方法

+ 具体效果如下
  + 转换前的RequestMap

  ![转换前.png](https://zos.alipayobjects.com/rmsportal/lCrLbBWsznsIUcumKprT.png)
  + 转换后的Retrofit
  
  ![转换后.png](https://zos.alipayobjects.com/rmsportal/GnlEipyvAWBYtRlsOvOY.png)