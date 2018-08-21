---
layout: single
title: "【踩坑经验】记HttpClient的NoHttpResponse问题"
date: 2018-08-21 19:55:02 +0800
comments: true
categories: Java,Nginx
---
### 背景
+ 服务A上线以来，一直有同学反应用jest的客户端访问时会出现偶发性NoHttpResponse的问题
+ 因为之前HttpClient4.4存在链接管理不当的问题，一直以为是版本问题，就没有在意。
+ 但最近却有业务方反应明明是4.4.1以后的版本却依旧存在问题，才发现原来事情并不简单。

### 现象描述
![IMAGE](https://gw.alipayobjects.com/zos/rmsportal/nMfQbcvxQTTCYThEanVX.png)
+ 服务A 架构如上所示，采用Nginx作为反向代理，为了负载均衡，服务端设置了
```
keepalive_requests 10000;
keepalive_timeout 30s;
```
+ 在生产环境中，偶有同学反应会有HttpClient报NoHttpResponse的问题 `org.apache.http.NoHttpResponseException: The target server failed to respond`

### 问题解析
+ 遇到问题，首先打开Debug日志，我们可以看到这是因为尝试读数据是没读到，所以链接失败了
![](https://gw.alipayobjects.com/zos/rmsportal/GORaqJgwuQpEebocTgXw.png)
![](https://gw.alipayobjects.com/zos/rmsportal/ILPLYCjUyDNWXOXTfaNl.png)

+ 查阅TCP资料，我们可以知道一般读到end of stream都是因为服务端主动关闭连接造成的，此时客户端应该主动关闭连接，而不是继续复用。
+ 通过WireShark抓包，我们可以看到的确是服务端已经发送了`FIN`,而客户端依旧使用了这个链接进行访问，所以就收到了`RST`，造成整个连接失败
![](https://gw.alipayobjects.com/zos/rmsportal/XTaaQiUwgyNHtPKuHouT.png)

#### 那么为什么会继续复用已经关闭的链接呢？
+ 我们继续从代码出发，通过日志，我们可以看出每次链接前都有次向连接池请求连接的过程
![](https://gw.alipayobjects.com/zos/rmsportal/NAGMxBxMOltSDtGyaMGX.png)
+ 进入连接池管理代码，我们可以看到这边有两次检测是否要关闭:
![](https://gw.alipayobjects.com/zos/rmsportal/mHTimGsrhEnnVIbOzCnm.png)
  1. 首先是检测连接有没有过期，如果过期了就close
  2. 其次检测有没有配置validateAfterInactivity(默认为2000ms),如果超过了指定的时间间隔就进行validate
      + validate就是尝试读一次(超时1ms)来判断有没有失效
      + 如果读到-1(end of stream)或者发生异常了就认为失效了
      + 如果超时了就认为还是正常(因为httpclient阻塞读)
![](https://gw.alipayobjects.com/zos/rmsportal/cFhqOeRGhnlHoVCPxaQd.png)


+ 所以出现链接异常会有2种情况:
  1. keepalive时间太短,2s内就释放了，因此httpClient以为还能用就没有检测直接使用了,从日志可以看出，根本没有validate过程，在TCP的连接上表现为`FIN`后直接发起请求
 ![](https://gw.alipayobjects.com/zos/rmsportal/nYcQrPmUeYXtoMorLZFY.png)
![](https://gw.alipayobjects.com/zos/rmsportal/HEkrWmMhTAAwMYhwXnsR.png)
  + 对比有validate的日志
    + 连接可用 
    ![](https://gw.alipayobjects.com/zos/rmsportal/TqvLlBqAZgLzxhummuea.png)
    + 连接不可用，关闭重建
    ![](https://gw.alipayobjects.com/zos/rmsportal/PhSWPjUzFFjXdhfXWtpe.png)
    ![](https://gw.alipayobjects.com/zos/rmsportal/wGCutQxzbcKinCsaPXVl.png)
  2. 刚好服务端释放请求时发起访问。如之前例子所示，在服务端还没发送FIN是进行了validate，但是刚好在正式发送请求前服务端发送了FIN，因此造成了RST。或者是如下正好在服务端关闭时发送了请求
 ![](https://gw.alipayobjects.com/zos/rmsportal/IwscSdHlrCAELRaXISko.png)

### 解决方案
#### 服务端方案
+ 由上可见，要想根本解决问题，最好的方法就是客户端可以检测到服务已经过期了，主动关闭。
+ 那么如何设置呢？追踪代码我们可以看到是client是从HttpHeader中读取`Keep-Alive: timeout=xx`，如果没有读到就认为永不超时
![](https://gw.alipayobjects.com/zos/rmsportal/qslisMeZEDqpipZVqqTq.png)
+ 而Nginx配置的第二个参数就是这个，因此只要服务端将`keepalive_timeout 30s;`改为`keepalive_timeout 30s 29s;`即可，这里少1s为了防止网络延时造成的并发问题。
![](https://gw.alipayobjects.com/zos/rmsportal/KZQxgKHnlgdhknusqysx.png)

#### 客户端方案
+ 如果服务端不方便修改，那么从客户端这边修改则相对比较麻烦。
+ 首先保证用最新的httpClient
+ 其次有三种方法:
  1. Retry。HttpClient默认是打开自动重试的，不要调用`disableAutomaticRetries`即可
  2. 对于jest这类有定时closeIdleConnection的，可以将调度时间调整为keep alive时间的一半以下以保证超时前回收
  3. 根据keep Alive时间，调整validateAfterInactivity小于keepAlive Time,但这种方法依旧不能避免同时关闭

### 相关知识
1. Nginx的`keepalive_requests`为一次长连接最多响应的次数，在最后一次时会发送`Connection: Close`的`Response Header`
2. HttpClient在接受到服务端的`FIN`后链接会变成`CLOSE_WAIT`，直到下一次请求才会从连接池种拿出，判断要不要响应`FIN`,因此如果一直没有下一个连接，将会有大量`CLOSE_WAIT`
3. 如果读数据时，操作系统已经将链接关闭，则返回`I/O error: Connection reset`,如果还未关闭，则返回`end of stream`也就是`NoHttpResponse`

### 参考资料
+ [HttpClient NoHttpResponseException问题排查](http://zhanjindong.com/2017/10/11/http-client-no-http-response-exception)
+ [TCP三次握手与四次挥手](https://www.cnblogs.com/flystar32/p/6297887.html)
+ [几种TCP连接中出现RST的情况](https://webcache.googleusercontent.com/search?q=cache:nx5S7_fSCSYJ:https://my.oschina.net/costaxu/blog/127394+&cd=1&hl=zh-CN&ct=clnk)
+ HTTPClient 4.5.2



### 版权声明
+ 自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
+ 本文首发于: http://czjxy881.coding.me/

{% include toc %}