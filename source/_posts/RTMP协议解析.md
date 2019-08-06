title: RTMP协议解析
author: Cyrus
tags:
  - RTMP
categories:
  - 音视频
date: 2019-08-06 21:32:00
---
参考：https://www.jianshu.com/p/5ce11c20a9df  
	 http://www.imooc.com/article/68207  
     音频：https://my.oschina.net/u/2326611/blog/854488

### 一、协议概述
RTMP(Real Time Messaging Protocol)实时消息传送协议是AdobeSystems公司为Flash播放器和服务器之间音频、视频和数据传输 开发的开放协议。
它有多种变种：
* RTMP工作在TCP之上，默认使用端口1935；
* RTMPE在RTMP的基础上增加了加密功能；
* RTMPT封装在HTTP请求之上，可穿透防火墙；
* RTMPS类似RTMPT，增加了TLS/SSL的安全功能。

### 二、协议详细介绍
RTMP协议(Real Time Messaging Protocol)是被Flash用于对象，视频，音频的传输。这个协议建立在TCP协议或者轮询HTTP协议之上。
RTMP协议就像一个用来装数据包的容器,这些数据既可以是AMF格式的数据，也可以是FLV中的视/音频数据。
一个单一的连接可以通过不同的通道传输多路网络流，这些通道中的包都是按照固定大小的包传输的。

### 三、握手请求及应答
要建立一个有效的RTMP Connection链接，首先要“握手”:客户端要向服务器发送C0,C1,C2（按序）三个chunk，服务器向客户端发送S0,S1,S2（按序）三个chunk，然后才能进行有效的信息传输。RTMP协议本身并没有规定这6个Message的具体传输顺序，但RTMP协议的实现者需要保证这几点：
* 客户端要等收到S1之后才能发送C2
* 客户端要等收到S2之后才能发送其他信息（控制信息和真实音视频等数据）
* 服务端要等到收到C0之后发送S1
* 服务端必须等到收到C1之后才能发送S2
* 服务端必须等到收到C2之后才能发送其他信息（控制信息和真实音视频等数据）

如果每次发送一个握手chunk的话握手顺序会是这样： 
![](握手.png)

理论上来讲只要满足以上条件，如何安排6个Message的顺序都是可以的，但实际实现中为了在保证握手的身份验证功能的基础上尽量减少通信的次数，一般的发送顺序是这样的，这一点可以通过wireshark抓ffmpeg推流包进行验证：  
｜client｜Server｜   
｜－－－C0+C1—->|  
｜<－－S0+S1+S2–|  
｜－－－C2-－－－>｜ 

下面让我们看一下Wireshark的抓包数据:
![](握手_1.png)
![](握手_2.png)
![](握手_3.png)
从上面的过程可以看出： 
* C0 == S0 == 03 为协议版本
* C1/C2/S1/S2 分别为1536字节的data,且S2 == C1, C2 == S1

### 四、信息传输
rtmp的传输遵循RTMP Header + RTMP Body 的结构，如下图所示：
![](rtmp_chunk.png)
首先，让我们看一下RTMP Header的结构：

#### 1、RTMP Header
* RTMP Header 的第一个字节称为basic header, 包含 2bit的fmt + 6bit 的csid(chunk stream ID（流通道Id）。fmt的值决定着RTMP Header 的长度，fmt为0/2/3时，对应的header长度为12/8/4/1(包含basic header本身的一个字节）。
* Chunk Msg Header,根据fmt的值，长度为11/7/3/1， 以最长的11字节为例，分别为 时间戳（3字节，以ms为单位，可保存约4.66h的视频），message length(3字节，用来表示后面RTMP Body数据的长度，即图中Chunk data), 消息类型type id（1字节，来以区分该消息块的类型，如音频（0x08)、视频(0x09)、script(0x12)及其它操作,stream id消息流id,4字节。
* Extend Timestamp:0/4 bytes,该字段发送的时候必须是正常的时间戳设置成0xffffff时，当正常时间戳不为0xffffff时，该字段不发送。当时间戳比0xffffff小该字段不发送，当时间戳比0xffffff大时该字段必须发送，且正常时间戳设置成0xffffff。

#### 2、RTMP Body
对于RTMP Body，我们主要对照抓包数据，解析几种消息类型如音频（0x08)、视频(0x09)、script(0x12)的解析方法。
##### 首先，看一下script的解析
![](script.png)
RTMP Header: 
~~~
04 00 00 00 00 01 7c 12 01 00 00 00

首字节：0x04即 fmt=0, 确定了header总长度为12  csid=4
时间戳：00 00 00，还未正式传输音视频数据
body长度：00 01 7c（380，注意这里采用大端结构，部分平台需要转换字节序）
type id: 0x12(script类型，包含MetaData)
stream id：01 00 00 00（1）
~~~
RTMP Body,在解析body之后，我们需要先看一下
![](amf_type.png)
类型比较多，这边主要讲一下00(number,即double)，02(字符串)，03（object[string + value]组， 以00 00 09结束）

下面看一组抓包数据：
![](script.png)
~~~
02 00 0d 40 73 65 74 44 61 74 61 46 72 61 6d 65
02 00 0a 6f 6e 4d 65 74 61 44 61 74 61 03 00 06   
61 75 74 68 6f 72 02 00 00 00 09 63 6f 70 79 72
69 67 68 74 02 00 00 00 0b 64 65 73 63 72 69 70   
74 69 6f 6e 02 00 00 00 08 6b 65 79 77 6f 72 64   
73 02 00 00 00 06 72 61 74 69 6e 67 02 00 00 00 
05 74 69 74 6c 65 02 00 00 00 0a 70 72 65 73 65 
74 6e 61 6d 65 02 00 06 43 75 73 74 6f 6d 00 0c
63 72 65 61 74 69 6f 6e 64 61 74 65 02 00 19 53  
75 6e 20 4a 75 6e 20 30 34 20 30 30 3a 33 31 3a   
30 38 20 32 30 31 37 0a 00 0b 76 69 64 65 6f 64  
65 76 69 63 65 02 00 15 55 53 42 32 2e 30 20 56   
47 41 20 55 56 43 20 57 65 62 43 61 6d 00 09 66   
72 61 6d 65 72 61 74 65 00 40 2e 00 00 00 00 00  
00 00 05 77 69 64 74 68 00 40 74 00 00 00 00 00   
00 00 06 68 65 69 67 68 74 00 40 6e 00 00 00 00   
00 00 00 0c 76 69 64 65 6f 63 6f 64 65 63 69 64   
02 00 04 61 76 63 31 00 0d 76 69 64 65 6f 64 61  
74 61 72 61 74 65 00 40 7f 40 00 00 00 00 00 00   
08 61 76 63 6c 65 76 65 6c 00 40 3f 00 00 00 00  
00 00 00 0a 61 76 63 70 72 6f 66 69 6c 65 00 40  
50 80 00 00 00 00 00 00 17 76 69 64 65 6f 6b 65  
79 66 72 61 6d 65 5f 66 72 65 71 75 65 6e 63 79 
00 3f f0 00 00 00 00 00 00 00 00 09 
==============     解析       ==============
02(字符串) 00 0d(长度13) 40 73 65 74 44 61 74 61 46 72 61 6d 65(@setDataFrame)
02(字符串) 00 0a(长度10) 6f 6e 4d 65 74 61 44 61 74 61(onMetaData)
03(object)
每一组：00 06(长度为6的字符串) 61 75 74 68 6f 72(author) 02 00 00(长度为0的字符串，即"")
第二组:00 09(长度为9的字符串) 63 6f 70 79 72 69 67 68 74(copyright) 02 00 00(长度为0的字符串，即"")
。。。。。。
第十组：00 09(长度为9的字符串) 66 72 61 6d 65 72 61 74 65(framerate) 00(number, 后面跟8字节的值) 00 40 2e 00 00 00 00 00 00(大端值，转小端double后为15)                   
。。。。。。                    
00 00 09(object 结束标志）
~~~

##### 再看一下video的解析
视频数据主要分为3种，sps和pps参数集数据，I帧数据，B/P帧数据
* 先看看sps和pps数据解析
![](sps.png)

~~~
04 00 00 00 00 00 43 09 01 00 00 00 17 00 00 00
00 01 42 00 1f 03 01 00 2f 67 42 80 1f 96 52 02
83 f6 02 a1 00 00 03 00 01 00 00 03 00 3c e0 60
01 e8 40 00 11 8c 3f c6 38 c0 c0 03 d0 80 00 23
18 7f 8c 70 ed 0a 15 24 01 00 04 68 cb 8d 48
==============     解析       ==============
header:
	04(fmt+csid) 00 00 00(timestamp) 00 00 43(body size 67) 09(video) 01 00 00 00(stream id 1)

body:
	0x17 = 0b0001 0111
    	前4bit,1 = key frame (for AVC, a seekable frame)关键帧sps+pps/I帧
               2 = inter frame (for AVC, a non-seekable frame) B/P帧
               3 = disposable inter frame (H.263 only)
               4 = generated key frame (reserved for server use only)
               5 = video info/command frame
		后4bit,2 = Sorenson H.263
               3 = Screen video
               4 = On2 VP6
               5 = On2 VP6 with alpha channel
               6 = Screen video version 2
               7 = AVC
所以，0x17为AVC关键帧

	00：AVCPacketType，如果是avc，即首字节后四位为7，即
    	0 = AVC sequence header，序列参数集sps+pps
        1 = AVC NALU,视频裸数据
		2 = AVC end of sequence (lower level NALU sequence ender is not required or supported)
        
	00 00 00：CompositionTime
    	IF AVCPacketType == 1
        	Composition time offset
		ELSE
    		0
            
    01：configuration Version
 	42：AVCProfileIndication
 	00 1f 03：
    01：后5bit表示sps长度，即0x01&0x1F = 1,1个sps
    00 2f: sps长度
    SPS数据：67 42 80 1f 96 52 02 83 f6 02 a1 00 00 03 00 01 00 00 03 00 3c e0 60 01 e8 40 00 11 8c 3f c6 38 c0 c0 03 d0 80 00 23 18 7f 8c 70 ed 0a 15 24
    01： pps个数
    00 04： pps长度
    PPS数据：68 cb 8d 48
~~~

I帧：
![](I帧.png)

~~~
04 00 00 00 00 03 b8 09 01 00 00 00 17 01 00 02
4b 00 00 03 af 65 88 80 40 07 6c 29 89 00 01 0b
fc f6 ff f8 ec 73 78 59 62 f9 f1 47 53 e1 61 e7
bc 42 a0 f8 60 65 3d e2 b7 88 1e f2 f7 8a de 2b
78 a3 78 81 ef 77 88 7b dd e2 1c 7b 9f 10 f6 88
cb 8d 12 af 51 64 d7 fa 65 24 51 95 eb a5 a2 5c
ae e5 e5 dd e2 b2 e2 1e 5c a6 4b d7 .......
==============     解析       ==============
header:解析方式与sps/pps的header一致
	04 00 00 00 00 03 b8(长度952) 09（video） 01 00 00 00

body:
	17: avc 关键帧
    01: 视频裸数据，17 01即I帧
    00 02 4b: CompositionTime
    00 00 03 af: nalu size 裸数据长度943，body长度（952）- 9bytes(17 01 00 02 4b 00 00 03 af),刚好为943
    Raw Data: 65 88 80 40 07 6c 29 89 00 01 0b fc f6 ff f8 ec 73 78 59 62 f9 f1 47 53 e1 61 e7 bc 42 a0 f8 60 65 3d e2 b7 88 1e f2 f7 8a de 2b ..... 
    //裸数据首字节为0x65,0x65&ox1f=5,符合h264 nalu I帧定义 
~~~

B/P帧
![](b帧.png)

~~~
04 00 02 4b 00 01 66 09 01 00 00 00 27 01 00 00
70 00 00 01 5d 41 9a 02 05 8e df a8 fd f5 1f b1
9b bd 75 af ad 7d 6b eb 5f 5a c4 6f 11 8a fd 6b
11 8a fd 6b eb 5f 5a fa d7 d5 b1 98 ae a8 6e ef
89 f3 92 21 c8 a5 7f 5a c5 79 d5 fd 6b da d6 2b
fa d6 2b 78 ad e2 b7 f5 ac ff d6 be b5 62 71 5c
42 bc 5f af ad 67 7c 77 7c ea 97 ad 62 9d b8 9d
==============     解析       ==============
header:解析方式与其它视频帧一致
	04 00 02 4b 00 01 66 09 01 00 00 00
    
body:
	27: avc 非关键帧
    01: 视频裸数据，27 01即B帧/P帧
    00 00 70：CompositionTime
    00 00 01 5d：nalu size 裸数据长度349
    Raw Data: 41 9a 02 05 8e df a8 fd f5 1f b1
9b bd 75 af ad 7d 6b eb 5f 5a c4 6f 11 8a fd 6b
11 8a fd 6b eb 5f 5a fa d7 d5 b1 98 ae.........
	////裸数据首字节为0x41,0x41&ox1f=1,符合h264 nalu B/P帧定义

~~~

##### 最后再看看audio数据
首先看一下AAC sequence 参数块
![](audio参数.png)

~~~
header的type id为0x08，即为音频

body:
    1）第一个字节af，a就是10代表的意思是AAC，Format of SoundData. The following values are defined:
		0 = Linear PCM, platform endian
		1 = ADPCM
		2 = MP3
		3 = Linear PCM, little endian
		4 = Nellymoser 16 kHz mono
		5 = Nellymoser 8 kHz mono
		6 = Nellymoser
		7 = G.711 A-law logarithmic PCM
		8 = G.711 mu-law logarithmic PCM
		9 = reserved
		10 = AAC
		11 = Speex
		14 = MP3 8 kHz
		15 = Device-specific sound
		Formats 7, 8, 14, and 15 are reserved.
		AAC is supported in Flash Player 9,0,115,0 and higher.
		Speex is supported in Flash Player 10 and higher.
	2）第一个字节中的后四位f代表如下
		前2个bit的含义采样频率，这里是二进制11，代表44kHZ
			Sampling rate. The following values are defined:
			0 = 5.5 kHz
			1 = 11 kHz
			2 = 22 kHz
			3 = 44 kHz
		第3个bit，代表 音频用16位的
			Size of each audio sample. This parameter only pertains to uncompressed formats. Compressed formats always decode to 16 bits internally.
			0 = 8-bit samples
			1 = 16-bit samples
		第4个bit代表声道
			Mono or stereo sound
			0 = Mono sound
			1 = Stereo sound
    
    第二个字节00，0 = AAC sequence header 参数，1 = AAC raw 裸数据
    
    后面：12 10 56 e5 00 详见码流结构参见“ISO-14496-3 Audio”中描述
    或https://my.oschina.net/u/2326611/blog/854488
~~~

![](audio_raw.png)

~~~
header的type id为0x08，即为音频

body:
	首字节af，解析方式与AAC sequence 参数块相同
    第二个字节01 ，0 = AAC sequence header 参数，1 = AAC raw 裸数据
    
~~~
