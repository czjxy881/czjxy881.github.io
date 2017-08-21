---
layout: single
title: "基于OpenResty的API网关设计"
date: 2017-08-19 18:55:08 +0800
comments: true
categories: 架构设计
tags: lua nginx
---

### 背景
---
+ OpenResty 是一个支持lua的nginx，并且内置了一些常用的lua库。利用lua，我们就可以便捷得扩展nginx能力，甚至可以直接作为Web服务对外提供。<!--more--> [主页链接](https://openresty.org/cn/)
+ 由下图可以看出，我们可以在各个阶段进行干预。
![lua干预阶段](https://cloud.githubusercontent.com/assets/2137369/15272097/77d1c09e-1a37-11e6-97ef-d9767035fc3e.png)

### 设计
---
+ 本文介绍的API网关设计很简单，主要有路由，过滤器，拦截器三个部分组成。
+ 可以实现权限验证，日志记录，参数改写，限流限速等功能。


![IMAGE](https://gw.alipayobjects.com/zos/rmsportal/icBvPNHTAoWwWvGKBeGF.png)


### 实现关键
---
+ 这套方案实现并不复杂，主要是对nginx的干预要可控，对nginx主要的干预点有三个。
	1. init_worker_by_lua_file 注册路由信息
	2. access_by_lua_file 进行dispatch，过滤改写，权限校验和代理请求。如果代理请求的话，要记得`ngx.exit`将请求结束。
	3. balancer_by_lua_file 根据dispatch信息路由到不同真实负载服务器上

### 参考资料
---
+ [OpenResty最佳实践](https://www.gitbook.com/book/moonbingbing/openresty-best-practices/details)


{% include toc %}