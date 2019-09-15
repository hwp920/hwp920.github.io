title: MP4 seek实现
author: Cyrus
tags:
  - MP4
categories:
  - 音视频
date: 2019-08-20 08:50:00
---
前面讲了几篇mp4文件的解析，这篇讲一下如何运用前面了解到的知识来完成一个seek操作。那么为了实现seek到指定时间，在mp4中需要做如下工作(这里简化了，没有考虑edts box的影响)：

* 在mvhd中找到timescale和duration，目标时间*timescale将标准化。（[MP4格式解析二](http://www.cyrus.fun/2019/08/13/MP4格式解析二-ftyp/)）
* 在stts中找到根据sample delta，可以换算出给定时间的sample number(帧序号)。([MP4格式解析七](http://www.cyrus.fun/2019/08/17/MP4格式解析七-stbl/))（注意，有时sample delta数值不准，建议用duration / 总帧数 对比一下）
* stss中保存了I帧的序号，在stss中查找距离sample number最近的sync sample。
* stsc记录了每个chunk里面sample的数目，根据这个表可以计算出每个sample对应的chunk。
* stsz记录了每个sample的大小，结合stsc的内容，可以得到每个sample在其对应的chunk中的相对位置。
* stco记录了每个chunk在文件中的绝对位置。（注意是“绝对”位置，offset以文件开头为准，不是以mdat box为准）
* 根据chunk在文件中的绝对位置及sample在chunk中的相对位置，即可计算出sample在文件中的绝对位置。

下面看一下例子：  
我们seek 5s画面:  

1、先看下timescale和duration,timescale = 6000, duration = 56000, 5s标准化的时间为30000;
![](seek_1.png)

2、再看一下stts:
![](seek_2.png)
只有一个entry,sample delta为400，56000/140=400,说明sample delta的值没问题。即5s的帧数为：30000/400=75帧。 

3、看stss中的I帧位置:
![](seek_3.png)
可以看到距离5s所处的75帧最近的两个同步帧(I帧)分别为61和76,这里选取较近的76帧。（如果精度要求高，可以选61帧，然后一帧帧解码到75帧。）  

4、到stsc中找76帧所处的chuck:
![](seek_4.png)
通过计算可以知道，76帧是第11个chuck的第一个sample。

5、在stco找到第11个chuck在文件中的绝对位置:
![](seek_5.png)
chuck offset = 4055806

6、在stsz中找到第76帧在chuck中的偏移和大小（这里是chuck中的第1帧，偏移为0，如果不是第1帧，则偏移为该chuck前面帧数size累加值）,sample size = 318782
![](seek_6.png)

所以，只需从文件开始，偏移4055806，读取318782大小的数据，解码出来就是我们要seek的画面。当然要解码，还需要找到视频的解码数据如sps、pps(详见‘stsd’)。