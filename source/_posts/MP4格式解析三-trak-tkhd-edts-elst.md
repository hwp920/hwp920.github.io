title: MP4格式解析三 trak->tkhd & edts/elst
author: Cyrus
tags:
  - MP4
categories:
  - 音视频
date: 2019-08-14 22:10:00
---
参考：https://blog.jianchihu.net/mp4-elst-box.html

### 一、tkhd(Track Header Box)

首先看一下软件解析出来的情况：
![音频track header box](tkhd_1.png)
![视频track header box](tkhd_2.png)

下面看一下box的结构体表示：
~~~
tkhd(Track Header Box)
// tkhd box  
typedef struct {  
    u_char    size[4];  
    u_char    name[4];  
    u_char    version[1];  
    /* 
        flags标志位 
        按位或操作结果值，预定义如下： 
        0x000001 track_enabled，否则该track不被播放； 
        0x000002 track_in_movie，表示该track在播放中被引用； 
        0x000004 track_in_preview，表示该track在预览时被引用。 
        一般该值为7，如果一个媒体所有track均未设置track_in_movie和track_in_preview， 
        将被理解为所有track均设置了这两项；对于hint track，该值为0 
    */  
    u_char    flags[3];  
    u_char    creation_time[4];  
    u_char    modification_time[4];  
    u_char    track_id[4]; // id号，不能重复且不能为0  
    u_char    reserved1[4];  
    u_char    duration[4]; // track的时间长度  
    u_char    reserved2[8];  
    u_char    layer[2]; // 视频层，默认为0，值小的在上层  
    u_char    group[2]; // track分组信息，默认为0表示该track未与其他track有群组关系  
    u_char    volume[2]; // 音量  
    u_char    reverved3[2];  
    u_char    matrix[36]; // 视频变换矩阵  
    u_char    width[4]; // 宽  
    u_char    heigth[4]; // 高  
} mp4_tkhd_atom;

~~~

看一下具体的数据：
~~~
音频tkhd

00 00 00 5C 74 6B 68 64 00 00 00 01 D9 70 A8 7B ; ...\tkhd.....p.{
D9 70 A8 7B 00 00 00 01 00 00 00 00 00 00 DA BF ; .p.{............
00 00 00 00 00 00 00 00 00 00 00 00 01 00 00 00 ; ................
00 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ; ................
00 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ; ................
40 00 00 00 00 00 00 00 00 00 00 00             ; @...........

=====================  解析  =====================
00 00 00 5C: 长度92
74 6B 68 64: tkhd
00: version0
00 00 01: flag 1
D9 70 A8 7B:  creation_time (2019/08/07 16:10:35)
D9 70 A8 7B:  modifyTime
00 00 00 01: track id 1
00 00 00 00: reverse
00 00 DA BF: duration (55999) / time base(6000) = 9.33s
00 00 00 00 00 00 00 00: reverse2
00 00: layer, 视频层，默认为0，值小的在上层  
00 00: group, track分组信息，默认为0表示该track未与其他track有群组关系
00 00: 音量  
00 00:  reverse
00 01 00 00  00 00 00 00   00 00 00 00  
00 00 00 00  00 01 00 00   00 00 00 00 
00 00 00 00  00 00 00 00   40 00 00 00: matrix[36]; // 视频变换矩阵
00 00 00 00: width 0
00 00 00 00: height 0
~~~

~~~
视频tkhd
00 00 00 5C 74 6B 68 64 00 00 00 01 D9 70 A8 7B ; ...\tkhd.....p.{
D9 70 A8 7B 00 00 00 02 00 00 00 00 00 00 DA C0 ; .p.{............
00 00 00 00 00 00 00 00 00 00 00 00 01 00 00 00 ; ................
00 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ; ................
00 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ; ................
40 00 00 00 05 12 00 00 03 C6 00 00             ; @...........

=====================  解析  =====================
前面跟音频一样的，只看最后8字节的宽高解析，应该也是[16.16]的结构，即前16个字节为整数部分大小，后16字节为小数大小

05 12 00 00: 1298.0 (0X05120000>>16)
03 C6 00 00: 966.0 (0X03C60000>>16)
~~~

### 二、elst(Edit List Box)
首先看一下elst(Edit List Box)的解析伪代码：
~~~
aligned(8) class EditListBox extends FullBox(‘elst’, version, 0) {
   unsigned int(32)  entry_count;
   for (i=1; i <= entry_count; i++) {
        if (version==1) {
           unsigned int(64) segment_duration;
           int(64) media_time;
        } else { // version==0
           unsigned int(32) segment_duration;
           int(32)  media_time;
        }
            int(16) media_rate_integer;
            int(16) media_rate_fraction = 0;
    } 
}
~~~

再看两个实际数据：
![](edts_1.png)
看一下具体的数据：
~~~
00 00 00 24 65 64 74 73 00 00 00 1C 65 6C 73 74 ; ...$edts....elst
00 00 00 00 00 00 00 01 00 00 DA C0 00 00 01 90 ; ................
00 01 00 00    

=====================  解析  =====================
00 00 00 24: 长度36
65 64 74 73: edts
00 00 00 1C: 长度28
65 6C 73 74: elst
00: version 0
00 00 00 00: flag 0
00 00 00 01: Entry count 1
   00 00 DA C0: Segment duration 56000
   00 00 01 90: Media time 400
   00 01: media_rate_integer 1
   00 00: Media rate fraction 0
   
elst实例参数解析：
segment_duration: 表示该edit段的时长，以Movie Header Box（mvhd）中的timescale为单位,即 segment_duration/timescale = 实际时长（单位s）

media_time: 表示该edit段的起始时间，以track中Media Header Box（mdhd）中的timescale为单位。如果值为-1(FFFFFF)，表示是空edit，一个track中最后一个edit不能为空。

media_rate: edit段的速率为0的话，edit段相当于一个”dwell”，即画面停止。画面会在media_time点上停止segment_duration时间。否则这个值始终为1。

* 即上面的elst实例表示的含义为：从400/timescale 开始，以正常速率播放至结束
~~~

![](edts_2.png)
看一下具体的数据：
~~~
00 00 00 30 65 64 74 73 00 00 00 28 65 6C 73 74 ; ...0edts...(elst
00 00 00 00 00 00 00 02 00 00 08 2A FF FF FF FF ; ...........*....
00 01 00 00 00 00 D2 95 00 00 00 00 00 01 00 00 ; ................

=====================  解析  =====================
00 00 00 30: 长度48
00 00 00 28: 长度40
65 6C 73 74: elst
00: version 0
00 00 00 00: flag 0
00 00 00 02: Entry count 2
   00 00 08 2A: Segment duration 2090
   FF FF FF FF: Media time -1
   00 01: media_rate_integer 1
   00 00: Media rate fraction 0
   		* 该Entry表示，从开始到2090/timebase不播放该trak,即该trak整体延迟2090/timebase播放
        
   00 00 D2 95： Segment duration 53909
   00 00 00 00： Media time 0
   00 01: media_rate_integer 1
   00 00: Media rate fraction 0
   		* 该Entry表示，从2090/timebase开始到结束，正常播放
  
~~~