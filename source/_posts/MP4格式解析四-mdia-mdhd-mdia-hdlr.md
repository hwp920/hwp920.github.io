title: MP4格式解析四 mdia->mdhd && mdia->hdlr
author: Cyrus
tags:
  - MP4
categories:
  - 音视频
date: 2019-08-15 22:35:00
---
### 一、mdia( Media Box)
mdia是一个contain box,主要包含mdhd(Media Header Box), hdlr(Handler Reference Box)和minf(Media Information Box),包含着解码track的关键信息。
~~~
00 00 08 EC: size 2284 
6D 64 69 61: mdia
~~~

### 二、mdhd(Media Header Box)
![](mdhd_1.png)
解析伪代码
~~~
aligned(8) class MediaHeaderBox extends FullBox(‘mdhd’, version, 0) 
	{
		if (version==1)
		{
			unsigned int(64) creation_time;
			unsigned int(64) modification_time;
			unsigned int(32) timescale;
			unsigned int(64) duration;
		}
		else
		{ // version==0
			unsigned int(32) creation_time;
			unsigned int(32) modification_time;
			unsigned int(32) timescale;
			unsigned int(32) duration;
		}
		bit(1) pad = 0;
		unsigned int(5)[3] language; // ISO-639-2/T language code
		unsigned int(16) pre_defined = 0;
	}
    

creation_time: track中数据的创建时间,多同‘tkhd’中creation_time.
modification_time: track中数据的修改时间,多同‘tkhd’中modification_time.
timescale: 媒体中时间尺度. 通常和'mvhd'中的timescale不同,精度更高.
duration: 媒体的长度(时间尺度表示).
language：语言,符合ISO-639-2/T标准.
~~~

下面看一下具体数据解析：
~~~
00 00 00 20 6D 64 68 64 00 00 00 00 D9 70 A8 7B ; ... mdhd.....p.{
D9 70 A8 7B 00 00 BB 80 00 06 A0 00 55 C4 00 00 ; .p.{.......U...

======================== 解析 ========================
00 00 00 20： size 32
6D 64 68 64:  mdhd
00 00 00 00: version = 0, flag = 0
D9 70 A8 7B: creation_time
D9 70 A8 7B: modification_time
00 00 BB 80: timescale 48000
00 06 A0 00: duration 434176  duration/timescale = 434176/48000=9s
55 C4: language und(0X15+0x60, 0X0E+0x60, 0X04+0x60)
	0101 0101 1100 0100 ->10101(0x15)  01110(0x0E) 00100(0x04)
~~~

### hdlr(Handler Reference Box)
handler, declares the media (handler) type  
可获取track类型信息，主要是有字段handler_type(uint32_t)区分，具体含义如下：
   * 'vide' Video track
   * 'soun' Audio track
   * 'hint' Hint track

![](hdlr_1.png)
下面看一下具体数据解析：
~~~
00 00 00 31 68 64 6C 72 00 00 00 00 00 00 00 00 ; ...1hdlr........
73 6F 75 6E 00 00 00 00 00 00 00 00 00 00 00 00 ; soun............
43 6F 72 65 20 4D 65 64 69 61 20 41 75 64 69 6F ; Core Media Audio
00        ; .

======================== 解析 ========================
00 00 00 31: size 49
68 64 6C 72: hdlr 
00 00 00 00: version flag 都为0
00 00 00 00: component type: 全0
73 6F 75 6E: component subtype soun 代表该track为 audio track
00 00 00 00: Component manufacturer 。 Reserved. Set to 0.
00 00 00 00: Component flags。Reserved. Set to 0.
00 00 00 00  Component flags mask。Reserved. Set to 0。
43 6F 72 65 20 4D 65 64 69 61 20 41 75 64 69 6F 00:Name（Core Media Audio.）
~~~

再看一下视频的hdlr数据：
~~~
00 00 00 31 68 64 6C 72 00 00 00 00 00 00 00 00 ; ...1hdlr........
76 69 64 65 00 00 00 00 00 00 00 00 00 00 00 00 ; vide............
43 6F 72 65 20 4D 65 64 69 61 20 56 69 64 65 6F ; Core Media Video
00                                              ; .

======================== 解析 ========================
前面与音频相同
76 69 64 65： component subtype vide
43 6F 72 65 20 4D 65 64 69 61 20 56 69 64 65 6F 00: Name(Core Media Video.)
~~~