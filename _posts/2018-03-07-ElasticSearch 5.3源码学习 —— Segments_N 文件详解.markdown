---
layout: single
title: "ElasticSearch 5.3源码学习 —— Segments_N 文件详解"
date: 2018-03-07 19:29:12 +0800
comments: true
categories: ElasticSearch
---
### 概览
+ Lucene当前活跃的Segment都会存在一个Segment Info文件里，也就是`segments_N`。如果有多个`segments_N`,那么序号最大的就是最新的。
+ segments_N用SegmentInfos进行操作
+ segments_N由`Header, LuceneVersion, Version, NameCounter, SegCount, MinSegmentLuceneVersion, <SegName, HasSegID, SegID, SegCodec, DelGen, DeletionCount, FieldInfosGen, DocValuesGen, UpdatesFiles>SegCount, CommitUserData, Footer` 这19个变量组成。

|变量|类型|
|:--|:--|
|Header|一个魔数(int),"segments"字符串和一个大版本(int)组成|
|LuceneVersion，MinSegmentLuceneVersion|3个vint组成|
|NameCounter, SegCount, DeletionCount |int32|
|Generation, Version, DelGen, Checksum, FieldInfosGen, DocValuesGen 为|long(int64)|
|HasSegID|int8|
|SegID|16位byte|
|SegName, SegCodec|String|
|CommitUserData|Map<String,String>|
|UpdatesFiles|Map<Int32,Set<String>>|
|Footer|魔数(int),algorithmID(int),CRC(long)|
  

### 文件实例(SegmentInfos#readCommit读取)
![IMAGE](https://gw.alipayobjects.com/zos/rmsportal/DgVDqEmNWiTcOkNIdOTS.png)

|序号|含义|
|:--|:--|
|1|Magic Number 硬编码在lucene中，一直为`0x3fd76c17`|
|2|"segments"字符串,用于校验文件|
|3|version为6|
|4|Commit ID, 16位byte|
|5|表示generation的string，这边是1e|
|6|lucene具体版本，6.4.1|
|7|index version: 622,表示index修改了622次了|
|8|counter为`0xf4`,表示下一个segment序号为244|
|9|numSegments为8，表示一共有8个active segment|
|10|segment里最小的lucene版本，为6.4.1|
|11|第一个segment name:_3d|
|12|第一个byte表示有没有segID,如果为1，那么后面16位就是segID|
|13|表示Codec，这里是Lucene62,用来找到对应segment的编码器，用于打开segment|
|14|DelGen,删除文件序号，为-1代表还没有删除，对应文件`{segname}_{delgen}.liv`，这里就是`_3d_1.liv`|
|15|删除的doc数目|
|16|fieldInfosGen，为-1代表没有，对应文件`{segname}_{delgen}.fnm`|
|17|docValuesGen |
|18|读取一个String Set,第一个vint为长度，此处为0。然后读取一个int代表DocValuesUpdatesFiles的长度，此处为0，如果不为0，则是一个Map<Int32,Set<String>>|
|19|第二个segment的开头，因为一共有8个segment所以后面就重复上面的7遍|
|20|CommitUserData的长度，此处为3，表示后面有6个string，依次读取作为kv|
|21|结尾魔数，是开头魔数的反码|
|22|algorithmID 此处为0|
|23|CRC校验码|

### 附录
1. vint: 用1-5bit表示int,符号位表示是否结束(为0代表结束)，后7位表示数值。低位在前高位在后
2. generation在文件名中都是转成36进制
3. SegmentInfos在readCommit时除了读取Segment_N，还会读取各segment的元文件获得maxID,在lucene62中为`.si`文件，下图标红处即为docNum,MaxDoc为63023 (Lucene62SegmentInfoFormat#read读取)
![IMAGE](https://gw.alipayobjects.com/zos/rmsportal/osOrfbVPzhCwYIKShzpa.png)
4. 其实Segment的这些数据在Rest API中均有展示，不过在5.3中不在一个api
```
// 获得当前Segments信息
GET /{index}/_segments
//获得CommitUserData
GET /{index}/_stats?filter_path=**.commit&level=shards
```


### 版权声明
+ 自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
+ 本文首发于: http://czjxy881.coding.me/

{% include toc %}