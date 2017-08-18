---
layout: post
title: "Node顺序执行Shell命令"
date: 2017-02-19 21:59:49 +0800
comments: true
categories: Node
---
+ 今天写个小工具，其中有一步是用node执行shell，第一次写，以为也是同步执行于是就直接一个 for-exec

```js
commands.map(command => {
  cp.exec(command, {}, (err, stdout, stderr) => {
    console.log(`stdout: ${stdout}\nstderr: ${stderr}`);
  });
});
```
<!--more-->
+ 结果死的很惨，各种并发。
+ 网上查了一圈，没找到同步的exec，基本都是要用shellJS.
+ 不想依赖太多，于是手写了一个执行队列，因为JS中没有wait，所以用递归实现持续执行

```js
const cp = require('child_process');
let doing = false;
const commandQueue = [];
function exec(command) {
  doing = true;
  console.log(command);
  cp.exec(command, {}, (err, stdout, stderr) => {
    console.log(`stdout: ${stdout}\nstderr: ${stderr}`);
    commandQueue.length > 0 ? exec(commandQueue.shift()) : doing = false;
  });
}
function execCommand(command) {
  commandQueue.push(command);
  if (!doing) {
    exec(commandQueue.shift());
  }
}
```