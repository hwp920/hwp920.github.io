title: AVFoundation学习笔记二  Speech Synthesis
author: Cyrus
tags:
  - 文本语音播报
categories:
  - AVFoundation
date: 2018-10-24 23:07:00
---
AVFoundation的文本朗诵主要由以下三个类组成：AVSpeechSynthesizer（语音合成器）、AVSpeechUtterance（语音内容）和AVSpeechSynthesisVoice（语音口音）。这三个类的关系如下图：
![](SpeechSynthesizer.png)
简单来说就是一个语音合成器（AVSpeechSynthesizer）可以播放一个或多个可以设置口音（AVSpeechSynthesisVoice）、语调、语速的语音内容（AVSpeechUtterance）。

具体代码步骤：
1、初始化一个AVSpeechSynthesizer对象
```
_synthesizer = [[AVSpeechSynthesizer alloc] init];
```

2、创建要朗诵的内容
```
_speechStrings = @[@"Hello AV Foundation. How are you?",
                           @"I'm well! Thanks for asking.",
                           @"Are you excited about the book?",
                           @"Very! I have always felt so misunderstood.",
                           @"What's your favorite feature?",
                           @"Oh, they're all my babies.  I couldn't possibly choose.",
                           @"It was great to speak with you!",
                           @"The pleasure was all mine!  Have fun!"
                           ];
```

3、为每个字符串创建一个AVSpeechUtterance对象,并传递给self.synthesizer进行播放
```
for (NSUInteger i = 0; i < self.speechStrings.count; i++) {
        AVSpeechUtterance *utterance = [[AVSpeechUtterance alloc] initWithString:self.speechStrings[i]];
        utterance.voice = [AVSpeechSynthesisVoice voiceWithLanguage:@"en-US"];
        utterance.rate = 0.4;
        utterance.pitchMultiplier = 0.8f;       //声调，0.5（低音调）-2.0（高音调）
        utterance.postUtteranceDelay = 0.1f;    //播放下一句之前有短时间的暂停
        [self.synthesizer speakUtterance:utterance];
    }
```

相关属性：

AVSpeechSynthesisVoice相关属性

（1）language:voice的主要属性，可以根据language创建voice。可以使用
```
+ (NSArray<AVSpeechSynthesisVoice *> *)speechVoices;
```
获取所有支持的language,具体如下：
* Arabic (ar-SA)
* Chinese (zh-CN, zh-HK, zh-TW)
* Czech (cs-CZ)
* Danish (da-DK)
* Dutch (nl-BE, nl-NL)
* English (en-AU, en-GB, en-IE, en-US, en-ZA)
* Finnish (fi-FI)
* French (fr-CA, fr-FR)
* German (de-DE)
* Greek (el-GR)
* Hebrew (he-IL)
* Hindi (hi-IN)
* Hungarian (hu-HU)
* Indonesian (id-ID)
* Italian (it-IT)
* Japanese (ja-JP)
* Korean (ko-KR)
* Norwegian (no-NO)
* Polish (pl-PL)
* Portuguese (pt-BR, pt-PT)
* Romanian (ro-RO)
* Russian (ru-RU)
* Slovak (sk-SK)
* Spanish (es-ES, es-MX)
* Swedish (sv-SE)
* Thai (th-TH)
* Turkish (tr-TR)

（2）***quality***：声音质量，枚举值
```
typedef NS_ENUM(NSInteger, AVSpeechSynthesisVoiceQuality) {
    AVSpeechSynthesisVoiceQualityDefault = 1,	//默认
    AVSpeechSynthesisVoiceQualityEnhanced		//增强
} NS_ENUM_AVAILABLE(10_14, 9_0);
```

AVSpeechUtterance相关属性

（1）***rate***:指定播放语音时的速率(0.0-1.0),即AVSpeechUtteranceMinimumSpeechRate和AVSpeechUtteranceMaximumSpeechRate之间

（2）***pitchMultiplier***：设置声调,属性值介于0.5(低音调)~2.0(高音调)之间

（3）***volume***:设置音量(0.0-1.0)

（4）***preUtteranceDelay***：postUtteranceDelay告诉synthesizer本句朗读结束后要延迟多少秒再接着朗读下一秒,对应的属性还有***preUtteranceDelay***

此外,<font color=0xff000000>***AVSpeechSynthesizerDelegate***</font>中还提供了一些监听朗读状态的方法.