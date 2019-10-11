title: AAC ADTS格式解析
author: Cyrus
tags:
  - adts
categories:
  - 音视频
date: 2019-10-11 22:33:00
---
参考：https://wiki.multimedia.cx/index.php?title=ADTS  
https://wiki.multimedia.cx/index.php?title=MPEG-4_Audio#Channel_Configurations

### 一、adts是啥，有什么用？
ADTS全称是(Audio Data Transport Stream)，是AAC的一种十分常见的传输格式。一般为7字节或9字节，包含用于aac音频解码的采样率、通道数、采样精度等信息。在音频传输过程中，只要在音频数据前加上adts数据，就可以在任意位置解码出音频数据（pcm）并播放。

### 二、adts长什么样？怎么看？
简单地说，adts就长下面这样子，每一个字母代表一个参数：
~~~
AAAAAAAA AAAABCCD EEFFFFGH HHIJKLMM MMMMMMMM MMMOOOOO OOOOOOPP (QQQQQQQQ QQQQQQQQ)

A:	 12位， 同步码，全为1
B:   MPEG版本: 0 for MPEG-4, 1 for MPEG-2
C:   Layer: 都是0     
D:   有否有CRC校验  1 没有 0 有  有的话adts后面跟两字节crc校验(即上面的Q)
E:   profile, the MPEG-4 Audio Object Type minus 1
F:   MPEG-4 Sampling Frequency Index，即采样率
G:   private bit, guaranteed never to be used by MPEG, set to 0 when encoding, ignore when decoding,也就是说，传0就对了
H:   MPEG-4 Channel Configuration, 通道数
I:   originality, set to 0 when encoding, ignore when decoding,无脑传0
J:   home, set to 0 when encoding, ignore when decoding,同样无脑传0
K：  copyrighted id bit, the next bit of a centrally registered copyright identifier, set to 0 when encoding, ignore when decoding， 同上
L：	copyright id start, signals that this frame's copyright id bit is the first bit of the copyright id, set to 0 when encoding, ignore when decoding， 同上
M: 	包含adts长度的帧长度， frame length, this value must include 7 or 9 bytes of header length: FrameLength = (ProtectionAbsent == 1 ? 7 : 9) + size(AACFrame)
O:  11位,Buffer fullness
P:  Number of AAC frames (RDBs) in ADTS frame minus 1, for maximum compatibility always use 1 AAC frame per ADTS frame
Q:	两字节crc校验
~~~

下面看看具体参数的值含义：
* 1、MPEG-4 Audio Object Types（E = type - 1）
~~~
* 0: Null 
* 1: AAC Main
* 2: AAC LC (Low Complexity)
* 3: AAC SSR (Scalable Sample Rate)
* 4: AAC LTP (Long Term Prediction)
* 5: SBR (Spectral Band Replication)
* 6: AAC Scalable
* 7: TwinVQ
* 8: CELP (Code Excited Linear Prediction)
* 9: HXVC (Harmonic Vector eXcitation Coding)
* 10: Reserved
* 11: Reserved
* 12: TTSI (Text-To-Speech Interface)
* 13: Main Synthesis
* 14: Wavetable Synthesis
* 15: General MIDI
* 16: Algorithmic Synthesis and Audio Effects
* 17: ER (Error Resilient) AAC LC
* 18: Reserved
* 19: ER AAC LTP
* 20: ER AAC Scalable
* 21: ER TwinVQ
* 22: ER BSAC (Bit-Sliced Arithmetic Coding)
* 23: ER AAC LD (Low Delay)
* 24: ER CELP
* 25: ER HVXC
* 26: ER HILN (Harmonic and Individual Lines plus Noise)
* 27: ER Parametric
* 28: SSC (SinuSoidal Coding)
* 29: PS (Parametric Stereo)
* 30: MPEG Surround
* 31: (Escape value)
* 32: Layer-1
* 33: Layer-2
* 34: Layer-3
* 35: DST (Direct Stream Transfer)
* 36: ALS (Audio Lossless)
* 37: SLS (Scalable LosslesS)
* 38: SLS non-core
* 39: ER AAC ELD (Enhanced Low Delay)
* 40: SMR (Symbolic Music Representation) Simple
* 41: SMR Main
* 42: USAC (Unified Speech and Audio Coding) (no SBR)
* 43: SAOC (Spatial Audio Object Coding)
* 44: LD MPEG Surround
* 45: USAC
~~~

* 2、MPEG-4 Sampling Frequency Index（即上面的F) 

~~~  
There are 13 supported frequencies:
* 0: 96000 Hz
* 1: 88200 Hz
* 2: 64000 Hz
* 3: 48000 Hz
* 4: 44100 Hz
* 5: 32000 Hz
* 6: 24000 Hz
* 7: 22050 Hz
* 8: 16000 Hz
* 9: 12000 Hz
* 10: 11025 Hz
* 11: 8000 Hz
* 12: 7350 Hz
* 13: Reserved
* 14: Reserved
* 15: frequency is written explictly
~~~

* 3、MPEG-4 Channel Configuration（即上面的H)  

~~~
These are the channel configurations:
* 0: Defined in AOT Specifc Config
* 1: 1 channel: front-center
* 2: 2 channels: front-left, front-right
* 3: 3 channels: front-center, front-left, front-right
* 4: 4 channels: front-center, front-left, front-right, back-center
* 5: 5 channels: front-center, front-left, front-right, back-left, back-right
* 6: 6 channels: front-center, front-left, front-right, back-left, back-right, LFE-channel
* 7: 8 channels: front-center, front-left, front-right, side-left, side-right, back-left, back-right, LFE-channel
* 8-15: Reserved

~~~

下面看一个具体adts头的解析：  
数据为：fff9 5040 017f fc  
首先，改写成二进制： 11111111 11111001 01010000 01000000 00000001 01111111 11111100
~~~
11111111 1111 	--> A 同步码，全为1
1				--> B 1 for MPEG-2
00				--> C 总是0
1				--> D 1代表没有CRC验证，即adts长度为7
01				--> E profile-1,即profile=2，查看上面的MPEG-4 Audio Object Types列表，可以知道是AAC LC (Low Complexity) 
0100 			--> F 采样率，查看上面的MPEG-4 Sampling Frequency Index，可以知道4是44100HZ
0				--> G 无脑0
001				--> H 通道数，1为单通道
0				--> I 无脑0
0				--> J 无脑0
0				--> K 无脑0
0				--> L 无脑0
00 00000001 011 --> M 包含adts长度的帧长度,这里是11，说明这一帧后面还有4字节的数据
11111 111111	--> O 11位,Buffer fullness
00				--> P 几帧aac数据带一个adts头减1，0代码每帧一个adts头
~~~


### 三、项目中adts怎么组
下面我们看一个OC的示例代码
~~~
- (NSData*) adtsDataForPacketLength:(NSUInteger)packetLength {
    int adtsLength = 7;
    char *packet = malloc(sizeof(char) * adtsLength);
    // Variables Recycled by addADTStoPacket
    int profile = 2;  //AAC LC
    //39=MediaCodecInfo.CodecProfileLevel.AACObjectELD;
    int freqIdx = 4;  //44.1KHz
    int chanCfg = 1;  //MPEG-4 Audio Channel Configuration. 1 Channel front-center
    NSUInteger fullLength = adtsLength + packetLength;
    // fill in ADTS data
    packet[0] = (char)0xFF; // 11111111     = syncword
    packet[1] = (char)0xF9; // 1111 1 00 1  = syncword MPEG-2 Layer CRC
    packet[2] = (char)(((profile-1)<<6) + (freqIdx<<2) +(chanCfg>>2));
    packet[3] = (char)(((chanCfg&3)<<6) + (fullLength>>11));
    packet[4] = (char)((fullLength&0x7FF) >> 3);
    packet[5] = (char)(((fullLength&7)<<5) + 0x1F);
    packet[6] = (char)0xFC;
    NSData *data = [NSData dataWithBytesNoCopy:packet length:adtsLength freeWhenDone:YES];
    return data;
}
~~~

