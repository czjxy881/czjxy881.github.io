---
layout: post
title: "聪明的shingling"
date: 2017-02-15 00:06:11 +0800
comments: true
categories: 搜索引擎 信息检索
---
+ 今天看书，看到了一个计算文档相似度的公式，一开始完全没看懂，后来看懂后发现这么机智，给大家分享下
$$
J(S(d_1),S(d_2))=P(x_1^\pi=x_2^\pi)
$$

## 传统 Shingling
+ 首先介绍下传统的 Shingling 算法，Shingling 是 shingle 的动词，也就是叠瓦片，顾名思义，就是把一篇文章分成很多瓦片，叠在一起。
<!--more-->

### shingle 定义
+ 常用的是4-shingle, 即4个单词一个瓦片。看一下 wiki 上的例子
>The document, "a rose is a rose is a rose" can be tokenized as follows:
(a,rose,is,a,rose,is,a,rose)
The set of all contiguous sequences of 4 tokens (4-grams) is
{ (a,rose,is,a), (rose,is,a,rose), (is,a,rose,is), (a,rose,is,a), (rose,is,a,rose) } = { (a,rose,is,a), (rose,is,a,rose), (is,a,rose,is) }
+ "a rose is a rose is a rose"用 `4-shingle` 拆开就是`{(a,rose,is,a), (rose,is,a,rose), (is,a,rose,is)}`

### 相似度 Resemblance
+ Shingling 算法认为两个文档的相似度就是两个文档共同的 shingle 的占比 
$$
  r(A,B)=J(S(A),S(B))=\frac{|S(A)\cap S(B)|}{|S(A)\cup S(B)|}
$$
+ 其中 J 是 Jaccard 系数(Jaccard coefficient),一个常用的相似度公式
+ 具体实现可以将 S(X) 映射到一个long的整形上，相当于做次 hash

## Shingling算法优化
+ 传统算法中需要对两个集合取交和取并，这个计算量在 IR 系统中是不可想象的，所以就需要进行优化。
+ 常用的优化方式有两种:

### 位运算优化
+ 如果放弃次数信息，只判断 shingle 出现与否。
+ 那么就可以利用位运算: $A\cap B=A\&B $ , $A \cup B = A \| B$   ,然后再求出1的个数就可以快速算出相似度了。

### 概率论优化
+ 但是如果带上出现次数，位运算就不行了，于是就需要文初的方法了。
+ 首先将 S(x) 映射的集合称为 H(x),$\pi$是一个随机置换，也就是相当于对 H(x)随机排序取得集合 $\Pi(x)$ , $x_1^\pi$就是$\Pi(x)$中的第一个
+ $x_1^\pi$相当于对 H(x) 的随机抽样，因此
$$
P(x_1^\pi=x_2^\pi)=\frac{|S(d_1)\cap S(d_2)|}{|S(d_1)\cup S(d_2)|}=J(S(d_1),S(d_2))
$$
+ 这种方法巧妙的用概率论方法绕过了大量计算，在实际中，我们常取200次随机置换$x_A^\pi$的结果集合作为梗概(sketch)$\psi_A$ , $r(A,B)=\|\psi_A \cap \psi_B\|/200$

## 更多优化
+ 虽然概率论方法节省了单次比较的大量运算，但是如果放眼全局仍是$N^2$的算法，对于大规模 IR 系统仍很有挑战，目前常用的优化方法如下：
  1. 去掉无用 shingle
  2. 将相似文档建立并查集，避免重复比较
  3. 将$\psi_A$排序，然后再做一次 shingle,如果这次 shingle 仍有一样的，那么则再计算真正相似度

## 参考资料
+ [W-shingling 的 Wiki](https://en.wikipedia.org/wiki/W-shingling)
+ [Jaccard 系数](https://en.wikipedia.org/wiki/Jaccard_index)
+ Chrisopher D.Manning,王斌译,信息检索导论[M],人民邮电出版社,2010,300-303

