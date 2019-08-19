title: FLV文件格式解析（h264 + aac）
author: Cyrus
tags:
  - FLV
categories:
  - 音视频
date: 2019-08-08 22:19:00
---
在看本文之前， 可以先看一下上一篇关于rtmp的文章：
http://www.cyrus.fun/2019/08/06/RTMP协议解析/

### FLV文件结构
首先，看一下flv文件结构:
![](flv_1.png)
用软件看:
![](flv_2.png)

* FLV文件由FLV header 和FVL Body组成；
* FLV Body由一系列Tag组成，Tag分为Tag header 和Tag body两部分。
* 每个Tag后面有四个长节的长度，为之前Tag的Tag header + Tag body的大小。

### FLV header(9 byts)
![](flv_header.png)
* 前三个字节为‘FLV’
* 第四个字节为 Version 
* 第五个字节分为4部分，
    * 前5bit为保留字节，为0
    * 第6bit 是否包含音频
    * 第7bit 保留 0
    * 第8bit 是否包含视频
* 4字节 DataOffset，即header长度，Version为1长度为9

### FLV body
flv body部分是一系统的tag，主要分为三种script tag, video tag, audio tag
![](tag_1.png)
Tag 分为 tag header 和 tag data，
* tag header与rtmp header类似
* tag data 则与rtmp body一致

#### FLV Tag Header 结构
FLV Tag header 与 rtmp header对比，tag type与rtmp一致，主要是0x12(script), 0x08(audio), 0x09(video)
![](tag_2.png)

### Script tag
Script tag， 第一个tag，包含文件meta data，一般只出现一次
![](script.png)
Type id(0x12) 
Tag Data 参考上一篇rtmp amf解析
这里说一下 AMF type 8(array)的解析
~~~
08 type 8(array)
00 00 00 10 (metadata count 16)
… 与AMF type 3(object)解析一致[string: any]
00 00 09 结束标致
~~~

### video tag
#### Sps/pps
Sps/pps, 第一个video tag,一般只出现一次
![](sps.png)
* Type id = 0x09
* Body 前两字节为 17 00 与rtmp一致

#### I帧
![](I帧.png)
* Type id = 0x09
* Body 前两字节为 17 01 与rtmp一致

#### B/P帧
![](B帧.png)
* Type id = 0x09
* Body 前两字节为 27 01 与rtmp一致

### audio tag
#### AAC sequence参数
![](aac_1.png)
* Type id = 0x08
* Body 前两字节为 af 00 与rtmp一致

#### AAC raw data
![](aac_2.png)
* Type id = 0x08
* Body 前两字节为 af 01 与rtmp一致


### 总结
rtmp流保存为FLV文件步骤  
1、写入flv header, 第5字节暂存为0x00  
2、对接收到的rtmp流数据，修改rtmp header为flv tag header(简单调换下位置)写入flv 文件，rtmp body直接不作更改写入文件，写入4字节rtmp header+rtmp body 数据长度  
3、保存文件时，根据接收过程是否有视频流和音频流，修改flv header第5字节的值

