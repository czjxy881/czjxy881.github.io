---
layout: single
title: "【踩坑经验】阿里云 SLB 七层负载 造成 Apache Httpclient BrokenPipe"
date: 2019-03-12 23:47:25 +0800
comments: true
categories: Openresty,Nginx
---
### 现象
+ 最近在使用阿里云 SLB 的时候发现其七层负载（HTTP,HTTPS）如果后端 upstream 提前返回，比如 nginx 在 access 阶段返回401，或者是直接访问404这种直接 return 的 error_page，大文件情况下会出现异常
  + Apache HttpClient 小文件可以正常发送，大文件(>256KB)会直接卡死，然后报`Broken pipe (Write failed)`或`Protocol wrong type`
  + Curl 频繁访问会出现 `curl: (55) Send failure: Protocol wrong type for socket`

### 原因
+ 通过抓包可以看到，SLB 接受到了部分请求后就开始转发给后端 upstream，在 upstream 返回 401 后 SLB 没有响应已发送的数据包就直接发送 FIN 关闭请求，然后 Client 认为没有写入，继续尝试写，得到 RST，反复重发后最终 Broken pipe
+ ![](https://gw.alipayobjects.com/zos/antfincdn/9MX3oNZD3J/fb5ad724-058b-4dc1-aa3c-9442beb132e6.png)
+ 而正常的请求，一般为三种
  + 第一种为纯四层的，接收到请求逐包转发给后端，所以发送完头Client就接受到401了，就不必继续发 body 了，但有些客户端在发送完头后并不会判断，如 HttpClient，那么一般就如第三种情况
  ![](https://gw.alipayobjects.com/zos/antfincdn/POwc19J4oA/6f4b1018-b26f-4d34-baea-b33f8e96d312.png)
  + 第二种为接收完所有请求，再转发到后端，这种就是纯 nginx 的七层转发
  + 第三种为接收到请求，批量发往后端，但是接收到401后，暂不关闭请求，保持链接，由客户端发完后，自己响应 Response 然后由客户端发送FIN关闭请求。
+ 而阿里云的 SLB 交互图如下，在返回后端 response 后直接关闭请求，造成错误。
![](https://gw.alipayobjects.com/zos/antfincdn/79rbOYugJ3/4c7035d0-876e-46ba-b2e5-b7eddacba23f.png)
+ 查阅资料, 可知阿里云的 SLB 七层负载采用的是 Tengine, 而在 Tengine 2.2 有一特性是流式上传，初步怀疑是 SLB 未配置好造成此问题，已提交工单反馈
![](https://gw.alipayobjects.com/zos/antfincdn/v5S2x3fFoQ/ddc64445-5cca-4722-a016-033004427dac.png)

### 临时解法
+ 有两种临时解法，但最终还是得依赖阿里云升级修复方可解决
  + 一种是直接 TCP 代理，简单暴力
  + 一种就是收到请求头后不要直接返回状态码，即使是401这种，也接收完 body，如在 openresty 中可以使用`ngx.req.read_body();`读取 body


### 参考资料
+ [SLB 架构介绍](https://help.aliyun.com/document_detail/27544.html)
+ [Tengine 官网](http://tengine.taobao.org/)

### 版权声明
+ 自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
+ 本文首发于: http://czjxy881.coding.me/

{% include toc %}