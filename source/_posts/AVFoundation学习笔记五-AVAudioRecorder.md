title: AVFoundation学习笔记五 AVAudioRecorder
author: Cyrus
tags: []
categories:
  - AVFoundation
date: 2018-11-02 15:28:00
---
![](recorder.png)
比较简单的类，主要说一下几个注意的地方：
### 一、配置AVAudioSession
```
AVAudioSession *session = [AVAudioSession sharedInstance];
    NSError *error;
    if (![session setCategory:AVAudioSessionCategoryPlayAndRecord error:&error]) {
        NSLog(@"Category Error: %@", [error localizedDescription]);
    }
    if (![session setActive:YES error:&error]) {
        NSLog(@"Activation Error: %@", [error localizedDescription]);
    }
```

### 二、settings参数
创建AVAudioRecorder的方法中，settings是最重要的参数，有着众多的配置参数：
```
- (nullable instancetype)initWithURL:(NSURL *)url settings:(NSDictionary<NSString *, id> *)settings error:(NSError **)outError;
```
settings中的key可以在***AVFoundation->framework文件夹->AVAudio->AVAudioSettings.h***中找到。

settings常见的key值：
* **AVFormatIDKey**：定义了写入内容的音频格式，值类型存在于 AudioFormatID枚举中，由相应四字节字符组成的32位整形，如：
![](audio_format.png)
<font color=ff0000>注意</font>：指定的类型必须与URL定义的文件名对应，比如录制一个test.wav，隐含的意思是录制的音频必须满足Waveform Audio File Format(WAVE)的格式要求，即低字节序（AVLinearPCMIsBigEndianKey 值为NO）、LinerPCM。如果AudioFormatID的值不是 kAudioFormatLinearPCM。NSError的错误信息为：
The  operation couldn’t be completed.(OSStatus error 118449215).
118449215 = ‘fmt?’,即不兼容格式
* **AVSampleRateKey**:采样率，对输入的模拟音频信号每一秒内的采样数，如8kHz,AM广播的录制效果，不件较小。44.1kHz，CD质量的采样率，文件比较大。尽量使用标准采样率，如8000、16000、22050和44100。
* **AVNumberOfChannelsKey**:通道数，1：单声道  2：立体声
* **AVLinearPCMBitDepthKey**:采样精度/位深， 8位或16位，用于lpcm
* **AVLinearPCMIsBigEndianKey**:是否大端保存数据，用于lpcm
* **AVLinearPCMIsFloatKey**:采样数据是否为浮点型
* **AVEncoderBitDepthHintKey**:编码位深，8-32，非lpcm使用
* **AVEncoderAudioQualityKey**:编码质量，非lpcm使用
```
typedef NS_ENUM(NSInteger, AVAudioQuality) {
 	AVAudioQualityMin    = 0,
 	AVAudioQualityLow    = 0x20,
 	AVAudioQualityMedium = 0x40,
 	AVAudioQualityHigh   = 0x60,
 	AVAudioQualityMax    = 0x7F
};
```