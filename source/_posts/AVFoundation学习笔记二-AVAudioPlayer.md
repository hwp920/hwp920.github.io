title: AVFoundation学习笔记四  AVAudioPlayer
author: Cyrus
tags: []
categories:
  - AVFoundation
date: 2018-11-01 23:10:00
---
![](audioplayer.png)

AVAudioPlayer 比较简单，主要说几个有用的方法和属性。
#### 属性
* 1、pan:立体声设置，0:立体声  -1：左声道  1：右声道
* 2、volume:  音量    0.0 - 1.0
* 3、rate: 播放速率    0.5 - 2.0      0.5：半速  1.0：正常速度   2.0：倍速
* 4、currentTime: 音频文件的播放时间
* 5、deviceCurrentTime: 输出设备播放音频的时间，注意如果播放中被暂停此时间也会继续累加
* 6、numberOfLoop:  循环播放次数，默认为1， -1为无限循环
* 7、settting: 文件的基本信息
```
{
    AVAudioFileTypeKey = 1667327590;	\\大端序数据，转主机序后为lpcm
    AVChannelLayoutKey = <02006500 03000000 00000000>;
    AVEncoderBitRateKey = 0;
    AVFormatIDKey = 1819304813;		\\大端序数据，转主机序后为caff
    AVLinearPCMBitDepthKey = 16;	\\采样精度
    AVLinearPCMIsBigEndianKey = 1;	\\数据是否以大端序保存
    AVLinearPCMIsFloatKey = 0;		
    AVLinearPCMIsNonInterleaved = 0;
    AVNumberOfChannelsKey = 2;		\\通道数
    AVSampleRateKey = 44100;		\\采样率
}
```
* 8、duration: 文件总时长
* 9、numberOfChannels： 该音频的声道数
* 10、meteringEnabled： 是否允许测量声道平均值和峰值

#### 方法
* 1、- (BOOL)prepareToPlay;  取得需要的音频硬件并预加载AudioQueue缓冲区.
* 2、- (BOOL)play;
* 3、- (BOOL)playAtTime:(NSTimeInterval)time； 播放没有调用prepareToPlay的话会隐式调用
* 4、- (void)pause;
* 5、- (void)stop;  停止播放。<font color=ff0000>区别：stop会把prepareToPlay所做的准备释放掉，pause不会，即pause后调用play响应较快。</font>
* 6、-（void）updateMeters;  meteringEnabled为true时，刷新对应声道强度的峰值和平均值
* 7、- (float)peakPowerForChannel:(NSUInteger)channelNumber; 对应声道强度的峰值
* 8、- (float)averagePowerForChannel:(NSUInteger)channelNumber; 对应声道强度的平均值

<font color=ff0000>注：7、8有效的前提条件为 meteringEnabled = YES && 调用了updateMeters。此计数是以对数刻度计量的，-160表示完全安静，0表示最大输入值。</font>可用于动态展示音频强度状态。

