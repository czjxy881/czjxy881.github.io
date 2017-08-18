---
layout: post
title: "Docker for Mac配置Sock5代理"
date: 2017-03-30 17:39:08 +0800
comments: true
categories: docker 科学上网
---

+ 今天需要用ES的镜像，发现Docker Hub的镜像已经废弃了，换成了`docker.elastic.co`镜像。
+ 于是喜闻乐见得被墙了。。。 `error pulling image configuration ... EOF`
![被墙](https://zos.alipayobjects.com/rmsportal/hrXjXfsUJLijmuyxEmrc.png)
<!--more-->
+ 尝试了下SS的全局代理，发现没有用

### 方案
+ 利用Privoxy开个Http代理，将http请求转发到Sock5，Docker for mac里配置Http代理。

### 具体步骤
1. `brew install privoxy`
2. 配置privoxy配置 `vi /usr/local/etc/privoxy/config`
3. 增加一行,代表把所有匹配/的请求(也就是所有请求)，以sock5协议转发到`127.0.0.1:1080`,最后一个.代表不转发到http代理
```
forward-socks5 / 127.0.0.1:1080 .
```
4. 将listen-address默认的`127.0.0.1`改为本机ip或者`0.0.0.0`
```
listen-address  10.15.232.101:8118
```
4. `brew services start privoxy`
5. Docker for Mac的Proxy代理均配为上面的listen-address
![Proxy配置](https://zos.alipayobjects.com/rmsportal/gDSMTFKtNjeTLmBBRIKP.png)
6. 至此，大功告成，可以愉快得拉镜像了

### 坑
+ 如果Docker for Mac的代理配成了`127.0.0.1:8118`那么就会出现`getsockopt: connection refused.`的错误。原因好像是docker命令实际上是运行在docker machine中的，也就是一个跑在Mac上的虚拟机中，所以配成127.0.0.1就会尝试走那台机器的代理，所以会出错。因此，需要配置成本机的真实ip，而不能是lookback address。今天被这个坑了2个小时，ಥ_ಥ
![错误](https://zos.alipayobjects.com/rmsportal/OaQRgfFqTNkHiOGUOnWT.png)

### 参考资料
+ [Beginner having trouble with docker behind company proxy](https://forums.docker.com/t/beginner-having-trouble-with-docker-behind-company-proxy/3968/3)
+ [用Privoxy转发socks代理，建http代理](http://www.cnblogs.com/another-wheel/archive/2011/11/16/setup-http-proxy-via-socks-and-privoxy.html)