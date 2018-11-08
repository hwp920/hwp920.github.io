title: AVFoundation学习笔记三  AVAudioSession
author: Cyrus
tags: []
categories:
  - AVFoundation
date: 2018-10-25 10:49:00
---
注：
文章部分内容来自：[https://www.jianshu.com/p/3e0a399380df](https://www.jianshu.com/p/3e0a399380df)
及参考《AVFoundation 开发秘籍》

#### 一、AVAudioSession是用来干嘛的？

简单来说AVAudioSession是用来控制app的音频行为，比如插拔耳机后是否继续播放音频、接电话后返回是否继续播放、是否和其他音频数据混音等。当你遇到:

* 是进行录音还是播放？
* 当系统静音键按下时该如何表现？
* 是从扬声器还是从听筒里面播放声音？
* 插拔耳机后如何表现？
* 来电话/闹钟响了后如何表现？
* 其他音频App启动后如何表现？
* ...
这些场景的时候，就可以考虑一下“AVAudioSession”了。

#### 二、AVAudioSession是如何控制音频行为的？
AVFoundation定义了7种分类（category）来描述应用程序所使用的音频行为。
![](session_category.png)

7种类别各自的行为总结如下：
* ***AVAudioSessionCategoryAmbient***： 只用于播放音乐时，并且可以和QQ音乐同时播放，比如玩游戏的时候还想听QQ音乐的歌，那么把游戏播放背景音就设置成这种类别。同时，当用户锁屏或者静音时也会随着静音，这种类别基本使用所有App的背景场景。
* ***AVAudioSessionCategorySoloAmbient(默认)***：也是只用于播放,但是和***"AVAudioSessionCategoryAmbient"***不同的是，用了它就别想听QQ音乐了，比如不希望QQ音乐干扰的App，类似节奏大师。同样当用户锁屏或者静音时也会随着静音，锁屏了就玩不了节奏大师了。
* ***AVAudioSessionCategoryPlayback***：如果锁屏了还想听声音怎么办？用这个类别，比如App本身就是播放器，同时当App播放时，其他类似QQ音乐就不能播放了。所以这种类别一般用于播放器类App
* ***AVAudioSessionCategoryRecord***：有了播放器，肯定要录音机，比如微信语音的录制，就要用到这个类别，既然要安静的录音，肯定不希望有QQ音乐了，所以其他播放声音会中断。想想微信语音的场景，就知道什么时候用他了。
* ***AVAudioSessionCategoryPlayAndRecord***：如果既想播放又想录制该用什么模式呢？比如VoIP，打电话这种场景，PlayAndRecord就是专门为这样的场景设计的 。
* ***AVAudioSessionCategoryMultiRoute***：想象一个DJ用的App，手机连着HDMI到扬声器播放当前的音乐，然后耳机里面播放下一曲，这种常人不理解的场景，这个类别可以支持多个设备输入输出。
* ***AVAudioSessionCategoryAudioProcessing***: 主要用于音频格式处理，一般可以配合AudioUnit进行使用

#### 三、如何设置AVAudioSession

获取AVAudioSession单例,设置类别并激活。音频会话通常会在应用程序启动时进行一次配置，所以可以将代码写在- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions：中。
```
AVAudioSession *session = [AVAudioSession sharedInstance];
NSError *error;
    if (![session setCategory:AVAudioSessionCategoryPlayback error:&error]) {
        NSLog(@"Category Error:%@", [error localizedDescription]);
    }
    if (![session setActive:YES error:&error]) {
        NSLog(@"Activation Error:%@", [error localizedDescription]);
    }
```

```
因为AVAudioSession会影响其他App的表现，当自己App的Session被激活，其他App的就会被解除激活，如何要让自己的Session解除激活后恢复其他App Session的激活状态呢？

此时可以使用：

(BOOL)setActive:(BOOL)active
withOptions:(AVAudioSessionSetActiveOptions)options
error:(NSError * _Nullable *)outError;
这里的options传入AVAudioSessionSetActiveOptionNotifyOthersOnDeactivation 即可。

当然，也可以通过otherAudioPlaying变量来提前判断当前是否有其他App在播放音频。
```

#### 四、如何根据自己需求调整AVAudioSession
AVAudioSession的设置可以分三个层级
```
Category确定基调---> options微调 + mode微调
```

##### Category的选项options
上面介绍的这个七大类别，可以认为是设定了七种主场景，而这七类肯定是不能满足开发者所有的需求的。CoreAudio提供的方法是，首先定下七种的一种基调，然后在进行微调。CoreAudio为每种Category都提供了些许选项来进行微调。
在设置完类别后，可以通过
```
@property(readonly) AVAudioSessionCategoryOptions categoryOptions;
```
属性，查看当前类别设置了哪些选项，注意这里的返回值是AVAudioSessionCategoryOptions，实际是多个options的“|”运算。默认情况下是0。

| 选项 | 适用类别 | 作用 |
| ----- | ----- | ----- |
| AVAudioSessionCategoryOptionMixWithOthers | AVAudioSessionCategoryPlayAndRecord, AVAudioSessionCategoryPlayback, and AVAudioSessionCategoryMultiRoute | 是否可以和其他后台App进行混音 |
| AVAudioSessionCategoryOptionDuckOthers | AVAudioSessionCategoryAmbient, AVAudioSessionCategoryPlayAndRecord, AVAudioSessionCategoryPlayback, and AVAudioSessionCategoryMultiRoute | 是否压低其他App声音 |
| AVAudioSessionCategoryOptionAllowBluetooth | AVAudioSessionCategoryRecord and AVAudioSessionCategoryPlayAndRecord | 是否支持蓝牙耳机 |
| AVAudioSessionCategoryOptionDefaultToSpeaker | AVAudioSessionCategoryPlayAndRecord | 是否默认用免提声音 |

来看每个选项的基本作用：
* AVAudioSessionCategoryOptionMixWithOthers ： 如果确实用的AVAudioSessionCategoryPlayback实现的一个背景音，但是呢，又想和QQ音乐并存，那么可以在AVAudioSessionCategoryPlayback类别下在设置这个选项，就可以实现共存了。
* AVAudioSessionCategoryOptionDuckOthers：在实时通话的场景，比如QQ音乐，当进行视频通话的时候，会发现QQ音乐自动声音降低了，此时就是通过设置这个选项来对其他音乐App进行了压制。
* AVAudioSessionCategoryOptionAllowBluetooth：如果要支持蓝牙耳机电话，则需要设置这个选项
* AVAudioSessionCategoryOptionDefaultToSpeaker： 如果在VoIP模式下，希望默认打开免提功能，需要设置这个选项

通过接口：
```
- (BOOL)setCategory:(NSString *)category withOptions:(AVAudioSessionCategoryOptions)options error:(NSError **)outError
```
来对当前的类别进行选项的设置。

#### 七大模式（Mode）

刚讲完七大类别，现在再来七大模式。通过上面的七大类别，我们基本覆盖了常用的主场景，在每个主场景中可以通过Option进行微调。为此CoreAudio提供了七大比较常见微调后的子场景。叫做各个类别的模式。

| mode | 适用的类别 | 场景 |
| -- | -- | -- |
| AVAudioSessionModeDefault | 所有类别 | 默认的模式 |
| AVAudioSessionModeVoiceChat | AVAudioSessionCategoryPlayAndRecord | VoIP |
| AVAudioSessionModeGameChat | AVAudioSessionCategoryPlayAndRecord | 游戏录制，由GKVoiceChat自动设置，无需手动调用 |
| AVAudioSessionModeVideoRecording | AVAudioSessionCategoryPlayAndRecord AVAudioSessionCategoryRecord | 录制视频时 |
| AVAudioSessionModeMoviePlayback | AVAudioSessionCategoryPlayback | 视频播放 |
| AVAudioSessionModeMeasurement | AVAudioSessionCategoryPlayAndRecord AVAudioSessionCategoryRecord AVAudioSessionCategoryPlayback | 最小系统 |
| AVAudioSessionModeVideoChat | AVAudioSessionCategoryPlayAndRecord | 视频通话 |

每个模式有其适用的类别，所以，并不是有“七七 四十九”种组合。如果当前处于的类别下没有这个模式，那么是设置不成功的。设置完Category后可以通过：
```
@property(readonly) NSArray<NSString *> *availableModes;
```
属性，查看其支持哪些属性，做合法性校验。

来看具体应用：

* AVAudioSessionModeDefault： 每种类别默认的就是这个模式，所有要想还原的话，就设置成这个模式。
* AVAudioSessionModeVoiceChat：主要用于VoIP场景，此时系统会选择最佳的输入设备，比如插上耳机就使用耳机上的麦克风进行采集。此时有个副作用，他会设置类别的选项为"AVAudioSessionCategoryOptionAllowBluetooth"从而支持蓝牙耳机。
* AVAudioSessionModeVideoChat ： 主要用于视频通话，比如QQ视频、FaceTime。时系统也会选择最佳的输入设备，比如插上耳机就使用耳机上的麦克风进行采集并且会设置类别的选项为"AVAudioSessionCategoryOptionAllowBluetooth" 和 "AVAudioSessionCategoryOptionDefaultToSpeaker"。
* AVAudioSessionModeGameChat ： 适用于游戏App的采集和播放，比如“GKVoiceChat”对象，一般不需要手动设置
另外几种和音频APP关系不大，一般我们只需要关注VoIP或者视频通话即可。

通过调用：
```
- (BOOL)setMode:(NSString *)mode error:(NSError **)outError
```
可以在设置Category之后再设置模式。

#### 系统中断响应
<font color=color=#ff0000>AVAudioSessionInterruptionNotification</font>:电话、闹铃响等中断的通知,其回调回来的userInfo主要包含两个键：

* AVAudioSessionInterruptionTypeKey:取值为 <font color=#ff0000> AVAudioSessionInterruptionTypeBegan </font>:表示中断开始，我们应该暂停播放和采集，取值为<font color=color=#ff0000>AVAudioSessionInterruptionTypeEnded</font>表示中断结束，我们可以继续播放和采集。
* AVAudioSessionInterruptionOptionKey:当前只有一种值<font color=color=#ff0000>AVAudioSessionInterruptionOptionShouldResume</font>:表示此时也应该恢复继续播放和采集。

示例如下：

```
//注册通知
NSNotificationCenter *nsnc = [NSNotificationCenter defaultCenter];
        [nsnc addObserver:self selector:@selector(handleInterruption:) name:AVAudioSessionInterruptionNotification object:[AVAudioSession sharedInstance]];
        
//处理回调
- (void)handleInterruption: (NSNotification *)notification
{
    NSDictionary *info = notification.userInfo;
    AVAudioSessionInterruptionType type = [info[AVAudioSessionInterruptionTypeKey] unsignedIntegerValue];
    
    if (type == AVAudioSessionInterruptionTypeBegan) {
        //Handle AVAudioSessionInterruptionTypeBegan
    } else {
       // Handle AVAudioSessionInterruptionTypeEnd
        AVAudioSessionInterruptionOptions options = [info[AVAudioSessionInterruptionOptionKey] unsignedIntegerValue];
        if (options == AVAudioSessionInterruptionOptionShouldResume) {
        
        }
    }
}
```

<font color=#ff0000>AVAudioSessionSilenceSecondaryAudioHintNotification</font>:其他App占据AudioSession的通知，其回调回来的userInfo键为：

```
AVAudioSessionSilenceSecondaryAudioHintTypeKey
```

可能包含的值
* <font color=color=#ff0000>AVAudioSessionSilenceSecondaryAudioHintTypeBegin</font>:表示其他App开始占据Session
* <font color=color=#ff0000>AVAudioSessionInterruptionTypeEnded</font>:表示其他App开始释放Session


#### 外设改变

默认情况下，AudioSession会在App启动时选择一个最优的输出方案，比如插入耳机的时候，就用耳机。但是这个过程中，用户可能拔出耳机，我们App要如何感知这样的情况呢？

<font color=color=#ff0000>**AVAudioSessionRouteChangeNotification**</font> : 外设改变时通知，在NSNotificationCenter中对其进行注册，userInfo中有键：
* AVAudioSessionRouteChangeReasonKey ： 表示改变的原因

| 枚举值 | 意义 |
| ----- | ----- |
| AVAudioSessionRouteChangeReasonUnknown | 未知原因 |
| AVAudioSessionRouteChangeReasonNewDeviceAvailable | 有新设备可用 |
| AVAudioSessionRouteChangeReasonOldDeviceUnavailable | 老设备不可用 |
| AVAudioSessionRouteChangeReasonCategoryChange | 类别改变了 |
| AVAudioSessionRouteChangeReasonOverride | App重置了输出设置 |
| AVAudioSessionRouteChangeReasonWakeFromSleep | 从睡眠状态呼醒 |
| AVAudioSessionRouteChangeReasonNoSuitableRouteForCategory | 当前Category下没有合适的设备 |
| AVAudioSessionRouteChangeReasonRouteConfigurationChange | Rotuer的配置改变了 |

示例代码：
```
NSNotificationCenter *nsnc = [NSNotificationCenter defaultCenter];
[nsnc addObserver:self selector:@selector(handleRouteChange:) name:AVAudioSessionRouteChangeNotification object:[AVAudioSession sharedInstance]];

- (void)handleRouteChange:(NSNotification *)notification
{
    NSDictionary *info = notification.userInfo;
    AVAudioSessionRouteChangeReason reason = [info[AVAudioSessionRouteChangeReasonKey] unsignedIntegerValue];
    if (reason == AVAudioSessionRouteChangeReasonOldDeviceUnavailable) {
        //如果拔出耳机，停止播放
        AVAudioSessionRouteDescription *previousRoute = info[AVAudioSessionRouteChangePreviousRouteKey];
        AVAudioSessionPortDescription *previousOutput = previousRoute.outputs[0];
        NSString *portType = previousOutput.portType;
        if ([portType isEqualToString:AVAudioSessionPortHeadphones]) {
            [self stop];
        }
    }
}
```