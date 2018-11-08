title: AVFoundation学习笔记一 iOS多媒体环境
author: Cyrus
tags: []
categories:
  - AVFoundation
date: 2018-10-24 20:14:00
---
![](ios_media.png)

### Core Audio
Core Audio是OS X和iOS系统上处理所有音频事件的框架。
* 1、为音频和HIDI内容的录制、播放和处理提供相应接口；
* 2、高层级AudioQueueServers框架提供的处理基本音频播放和录音相关的功能；
* 3、低层级Audio Units针对音频信号进行完全控制的功能。

### Core Video
Core Video是OS X和iOS系统上针对数字视频所提供的管道模式。Core Video为其相对的Core Media<font color=ff0000>提供图片缓存和缓存池支持，提供了一个能够对数字视频逐帧访问的接口</font>。

### Core Media
Core Media是AVFoundation所用到的低层级媒体管道的一部分。提供针对音频样本和视频帧处理所需的低层级数据类型和接口。Core Media还提供了CMTime数据类型的时基模型。

### Core Animation
Core Animation是OS X和iOS提供的合成及动画相关框架，封装了OpenGL和OpenGL ES功能的基本Objective-C的各种类。（似乎已经改为<font color=ff0000>Metal</font>作为Core Animation的低层渲染）

### AVFoundation (本系列主要内容)
处于高层级框架和低层级框架之间，以Objective-C接口方式提供了很多低层级框架才能实现的功能和性能,可以和高层级的框架无缝衔接。
* 1、文本朗诵 （AVSpeechSynthesizer）
* 2、音频播放和录制（AVAudioPlayer和AVAudioRecorder）
* 3、媒体文件元数据（AVMetadataItem）
* 4、视频播放 （AVPlayer 和 AVPlayerItem）
* 5、媒体捕捉 （AVCaptureSession）
* 6、媒体读写 （AVAssetReader和AVAssetWriter）
* 7、媒体编辑