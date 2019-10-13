title: MP4V2的MP4SetTrackESConfiguration和MP4SetVideoProfileLevel
author: Cyrus
tags:
  - MP4V2
categories:
  - 音视频
date: 2019-10-13 23:21:00
---
参考：
* https://blog.csdn.net/dxpqxb/article/details/42266873
* https://linux.die.net/man/3/mp4setvideoprofilelevel

MP4V2是一个C级别的MP4文件封装工具，网上的资料很多，这里就不讲代码封装了，简单地讲一下编译和MP4SetTrackESConfiguration和MP4SetVideoProfileLevel两个方法。

### 一、编译
编译的脚本网上也有很多，这里以iOS为例，讲一个最简单的：
* 1、到github [https://github.com/JQJoe/mp4v2-for-iOS](https://github.com/JQJoe/mp4v2-for-iOS)下载仓库；
* 2、解压仓库文件夹里的mp4v2-2.0.0.tar.bz2；
* 3、到仓库目录运行 ./build-libmp4v2-for-iOS.sh
没错，就是这么简单，其他平台的也可以在github试着找找。

### 二、MP4SetTrackESConfiguration函数
上面仓库下载的mp4v2的源码里面有一个mp4record的使用demo，使用到的api也不多，照着demo简单地改一改，封装一下就能用了。这里讲一下demo问题的地方：
~~~
if(_naluData[0]==0 && _naluData[1]==0 && _naluData[2]==0 && _naluData[3]==1 && _naluData[4]==0x67){
        index = _NALU_SPS_;
    }
    
    if(index!=_NALU_SPS_ && recordCtx->m_vTrackId == MP4_INVALID_TRACK_ID){
        return index;
    }
    if(_naluData[0]==0 && _naluData[1]==0 && _naluData[2]==0 && _naluData[3]==1 && _naluData[4]==0x68){
        index = _NALU_PPS_;
    }
    if(_naluData[0]==0 && _naluData[1]==0 && _naluData[2]==0 && _naluData[3]==1 && _naluData[4]==0x65){
        index = _NALU_I_;
    }
    if(_naluData[0]==0 && _naluData[1]==0 && _naluData[2]==0 && _naluData[3]==1 && _naluData[4]==0x41){
        index = _NALU_P_;
    }
    
 // 上面这一块nalu判断类型的代码有点问题，这里用的是00 00 00 01开头的nalu格式，但是nalu类型判断并不是0x67/0x68/0x65和0x41,真正的nalu类型判断只取后面5个字节作判断，所以应该是 (_naluData[4] & 0x1F)==0x07/0x08/0x05/0x01。
~~~

好，下面进入正题，讲一下MP4SetTrackESConfiguration函数，先看demo里面怎么写的：
~~~
    MP4SetAudioProfileLevel(recordCtx->m_mp4FHandle, 0x2);
    uint8_t aacConfig[2] = {18,16};
    MP4SetTrackESConfiguration(recordCtx->m_mp4FHandle,recordCtx->m_aTrackId,aacConfig,2);
~~~

这ProfileLevel的0x2代表什么？TrackESConfiguration的{18，16}又是啥？下面让我们来看看具体的定义：
~~~
bool MP4SetAudioProfileLevel(
	MP4FileHandle hFile,
	u_int8_t profileLevel
)

Description
MP4SetAudioProfileLevel sets the minumum profile/level of MPEG-4 audio support necessary to render the contents of the file.

ISO/IEC 14496-1:2001 MPEG-4 Systems defines the following values:
0x00 Reserved
0x01 Main Profile @ Level 1
0x02 Main Profile @ Level 2
0x03 Main Profile @ Level 3
0x04 Main Profile @ Level 4
0x05 Scalable Profile @ Level 1
0x06 Scalable Profile @ Level 2
0x07 Scalable Profile @ Level 3
0x08 Scalable Profile @ Level 4
0x09 Speech Profile @ Level 1
0x0A Speech Profile @ Level 2
0x0B Synthesis Profile @ Level 1
0x0C Synthesis Profile @ Level 2
0x0D Synthesis Profile @ Level 3
0x0E-0x7F Reserved
0x80-0xFD User private
0xFE No audio profile specified
0xFF No audio required

// 照描述，说的是设置所支持的最小的profile,这里照demo写0x2,影响不大。
~~~

~~~
bool MP4SetTrackESConfiguration(
MP4FileHandle hFile,
MP4TrackId trackId
const u_int8_t* pConfig,
u_int32_t configSize
)

上面的{18，16}就是pConfig了，2就是configSize。那么{18， 16}代表什么呢？

首先，config有2个字节组成，共16位，具体含义如下：
5 bits | 4 bits | 4 bits | 3 bits
第一欄    第二欄    第三欄    第四欄

第一欄：AAC Object Type
第二欄：Sample Rate Index
第三欄：Channel Number
第四欄：Don't care，設 0

/* AAC object types */
#define MAIN 1
#define LOW  2
#define SSR  3
#define LTP  4

/* Returns the sample rate index */
int GetSRIndex(unsigned int sampleRate)
{
   if (92017 <= sampleRate) return 0;
   if (75132 <= sampleRate) return 1;
   if (55426 <= sampleRate) return 2;
   if (46009 <= sampleRate) return 3;
   if (37566 <= sampleRate) return 4;
   if (27713 <= sampleRate) return 5;
   if (23004 <= sampleRate) return 6;
   if (18783 <= sampleRate) return 7;
   if (13856 <= sampleRate) return 8;
   if (11502 <= sampleRate) return 9;
   if (9391 <= sampleRate) return 10;

   return 11;
}

OK，现在再来看{18， 16}是啥：
18 16 --> 0x12 0x10
二进制： 0001 0020 0001 0000
AAC object types： 0001 0 --> 2 就是LOW
Returns the sample rate index: 020 0 --> 4, 就是46009~37566， 其实就是44100
Channel Number： 001 0	--> 2，就是双通道
~~~

### 三、MP4SetVideoProfileLevel函数
~~~
函数原型：
bool MP4SetVideoProfileLevel(
MP4FileHandle hFile,
u_int8_t profileLevel
)

首先，看一下profileLevel都有哪些值：
Description
MP4SetVideoProfileLevel sets the minumum profile/level of MPEG-4 video support necessary to render the contents of the file.

ISO/IEC 14496-1:2001 MPEG-4 Systems defines the following values:
0x00 Reserved
0x01 Simple Profile @ Level 3
0x02 Simple Profile @ Level 2
0x03 Simple Profile @ Level 1
0x04 Simple Scalable Profile @ Level 2
0x05 Simple Scalable Profile @ Level 1
0x06 Core Profile @ Level 2
0x07 Core Profile @ Level 1
0x08 Main Profile @ Level 4
0x09 Main Profile @ Level 3
0x0A Main Profile @ Level 2
0x0B N-Bit Profile @ Level 2
0x0C Hybrid Profile @ Level 2
0x0D Hybrid Profile @ Level 1
0x0E Basic Animated Texture @ Level 2
0x0F Basic Animated Texture @ Level 1
0x10 Scalable Texture @ Level 3
0x11 Scalable Texture @ Level 2
0x12 Scalable Texture @ Level 1
0x13 Simple Face Animation @ Level 2
0x14 Simple Face Animation @ Level 1
0x15-0x7F Reserved
0x80-0xFD User private
0xFE No audio profile specified
0xFF No audio required

demo里面写的是
MP4SetVideoProfileLevel(recordCtx->m_mp4FHandle, 0x7F); //  Simple Profile @ Level 3
参照上面的值，应该是错误的，Simple Profile @ Level 3 应该是0x01。

关于视频的profileLevel,需要根据实际编码的profile设置，不然会出现解码出来的视频不清晰的问题。
~~~