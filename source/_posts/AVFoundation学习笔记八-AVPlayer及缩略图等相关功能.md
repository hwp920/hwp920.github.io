title: AVFoundation学习笔记八 AVPlayer及缩略图等相关功能
author: Cyrus
tags: []
categories:
  - AVFoundation
date: 2018-11-08 23:25:00
---

![](player_classes.png)

### AVPlayer
一个用来播放基本时间的视听媒体的控制器对象。支持播放从本地、分步下载或通过HTTP Live Streaming协议得到的流媒体，并在多种播放场景中播放这些视频资源。AVPlayer是一个不可见组件，要将视频资源导出到用户界面的目标位置，需要使用AVPlyerLayer类。

### AVPlayerLayer
构建于Core animation上，是AVFoundation中能找到的为数不多的可见组件。AVPlayerLayer扩展了CoreAnimation的CALayer类，并通过框架在屏幕上显示视频内容。
```
AVPlayerLayer的videoGravity属性：
AVLayerVideoGravityResizeAspect：默认值，保持视频原始宽高比，可能会有黑边；
AVLayerVideoGravityResizeAspectFill： 保持原始宽高比，短边填充屏幕，长边可能会超出屏幕
AVLayerVideoGravityResize：填充屏幕，不保持宽高比，图像会变形，最不常用。
```

### AVPlayerItem
AVAsset包含一些媒体数据的静态信息，如创建日期、元数据和时长等，但是没有当前播放时间、音量和查找特定位置的方法这些动态数据，需要AVPlayerItem和AVPlayerItemTrack类构建相应的动态内容。

```
代码如下（简单，没啥说的）：
NSURL *assetURL = [[NSBundle mainBundle] URLForResource:@"1" withExtension:@"mp4"];
    AVAsset *asset = [AVAsset assetWithURL:assetURL];
    
    AVPlayerItem *item = [[AVPlayerItem alloc] initWithAsset:asset];
    AVPlayer *player = [[AVPlayer alloc] initWithPlayerItem:item];
    AVPlayerLayer *layer = [AVPlayerLayer playerLayerWithPlayer:player];
    layer.frame = self.view.bounds;
    [self.view.layer addSublayer:layer];
    [player play];
```

### CMTime
CMTime是Core Media的一种数据结构。CMTime为时间的正确表示给出了一种结构，即分数值的方式（分子、分母）,定义如下：
```
//value/timescale = seconds.
    typedef struct
    {
        CMTimeValue    value;       //分子 int64_t
        CMTimeScale    timescale;   //分母 int32_t
        CMTimeFlags    flags;
        CMTimeEpoch    epoch;
    } CMTime;
    
    创建
    //0.5s
    CMTime halfSecond = CMTimeMake(1, 2);
    //5s
    CMTime fiveSeconds = CMTimeMake(5, 1);
    //44.1kHz
    CMTime oneSample = CMTimeMake(1, 44100);
    //0
    CMTime zeroTime = kCMTimeZero;
```

### 播放过程中的进度监听
```
- (void)addPlayerItemTimeObserver {
    
    // Create 0.5 second refresh interval - REFRESH_INTERVAL == 0.5
    CMTime interval =
        CMTimeMakeWithSeconds(0.5, NSEC_PER_SEC);              // 1
    
    // Main dispatch queue
    dispatch_queue_t queue = dispatch_get_main_queue();                     // 2
    
    // Create callback block for time observer
 
    void (^callback)(CMTime time) = ^(CMTime time) {
        NSTimeInterval currentTime = CMTimeGetSeconds(time);
        NSTimeInterval duration = CMTimeGetSeconds(weakSelf.playerItem.duration);
        //do something with currentTime and duration
    };
    
    // Add observer and store pointer for future use
    self.timeObserver = [self.player addPeriodicTimeObserverForInterval:interval queue:queue usingBlock:callback];
}


可通过： [self.player removeTimeObserver:self.timeObserver]; 移除监听
```

### <font color=ff0000>视频进度缩略图</font>
AVAssetImageGenerator工具类。这个类可以从一个Asset视频曲目中提取图片。生成一个或多个缩略图。主要包含两个方法：
```
AVAssetImageGenerator工具类。这个类可以从一个Asset视频曲目中提取图片。生成一个或多个缩略图。主要包含两个方法：
/**
requestedTime: 指定获取缩略图的时间 传入数据
actualTime:    生成图片的准确时间   传出数据
*/

- (nullable CGImageRef)copyCGImageAtTime:(CMTime)requestedTime actualTime:(nullable CMTime *)actualTime error:(NSError * _Nullable * _Nullable)outError //生成指定时间的单张缩略图
```

代码如下：
```
//指定时间生成缩略图
//创建URL
    NSURL *url=[NSURL URLWithString:[_videoURLStr stringByAddingPercentEscapesUsingEncoding:NSUTF8StringEncoding]];
    AVURLAsset *urlAsset=[AVURLAsset assetWithURL:url];
    AVAssetImageGenerator *imageGenerator=[AVAssetImageGenerator assetImageGeneratorWithAsset:urlAsset];
    NSError *error=nil;
    CMTime time=CMTimeMakeWithSeconds(timeBySecond, 10);
    CMTime actualTime;
    CGImageRef cgImage= [imageGenerator copyCGImageAtTime:time actualTime:&actualTime error:&error];
    if(error){
        return;
    }
    CMTimeShow(actualTime);
    UIImage *image=[UIImage imageWithCGImage:cgImage];//转化为UIImage
    //保存到相册
    UIImageWriteToSavedPhotosAlbum(image,nil, nil, nil);
    CGImageRelease(cgImage);
```

```
//生成指定时间数组的一组缩略图
typedef void (^AVAssetImageGeneratorCompletionHandler)(CMTime requestedTime, CGImageRef _Nullable image, CMTime actualTime, AVAssetImageGeneratorResult result, NSError * _Nullable error);
- (void)generateCGImagesAsynchronouslyForTimes:(NSArray<NSValue *> *)requestedTimes completionHandler:(AVAssetImageGeneratorCompletionHandler)handler;
- (void)generateThumbnails {
    
    self.imageGenerator =                                                   // 1
        [AVAssetImageGenerator assetImageGeneratorWithAsset:self.asset];
    
    // Generate the @2x equivalent
    self.imageGenerator.maximumSize = CGSizeMake(200.0f, 0.0f);             // 2
    CMTime duration = self.asset.duration;
    NSMutableArray *times = [NSMutableArray array];                         // 3
    CMTimeValue increment = duration.value / 20;
    CMTimeValue currentValue = 2.0 * duration.timescale;
    while (currentValue <= duration.value) {
        CMTime time = CMTimeMake(currentValue, duration.timescale);
        [times addObject:[NSValue valueWithCMTime:time]];
        currentValue += increment;
    }
    __block NSUInteger imageCount = times.count;                            // 4
    __block NSMutableArray *images = [NSMutableArray array];
    AVAssetImageGeneratorCompletionHandler handler;                         // 5 handler会多次回调，每次生成一张缩略图/失败
    
    handler = ^(CMTime requestedTime,
                CGImageRef imageRef,
                CMTime actualTime,
                AVAssetImageGeneratorResult result,
                NSError *error) {
        if (result == AVAssetImageGeneratorSucceeded) {                     // 6
            UIImage *image = [UIImage imageWithCGImage:imageRef];
            id thumbnail =
                [THThumbnail thumbnailWithImage:image time:actualTime];
            [images addObject:thumbnail];
        } else {
            NSLog(@"Error: %@", [error localizedDescription]);
        }
        // If the decremented image count is at 0, we're all done.
        if (--imageCount == 0) {                                            // 7
            dispatch_async(dispatch_get_main_queue(), ^{
                NSString *name = THThumbnailsGeneratedNotification;
                NSNotificationCenter *nc = [NSNotificationCenter defaultCenter];
                [nc postNotificationName:name object:images];
            });
        }
    };
    [self.imageGenerator generateCGImagesAsynchronouslyForTimes:times completionHandler:handler];
}
```

### 字幕切换
AVFoundation在展示字幕和隐藏字幕方面提供了可靠方法。AVPlayerLayer会自动渲染这些元素，并且可以让开发者告诉应用程序哪些元素里面要渲染。完成这些操作需要用到AVMediaSelectionGroup和AVMediaSelectionOption两个类。
```
NSArray *mediaCharacteristics = asset.availableMediaCharacteristicsWithMediaSelectionOptions;
    for (NSString *characteristic in mediaCharacteristics) {
        //可选择的轨道组
        AVMediaSelectionGroup *group = [asset mediaSelectionGroupForMediaCharacteristic:characteristic];
        NSLog(@"%@", characteristic);
        for (AVMediaSelectionOption *option in group.options) {
            //option:轨道组中的轨道
            NSLog(@"Option:%@", option.displayName);
            
            //- (void)selectMediaOption:(nullable AVMediaSelectionOption *)mediaSelectionOption inMediaSelectionGroup:(AVMediaSelectionGroup *)mediaSelectionGroup
            //对所择轨道的切换，如切换字幕、音频等
            [item selectMediaOption:option inMediaSelectionGroup:group];
        }
    }
```
结果显示：
![](option_result.png)

### airplay
```
MPVolumeView *volumeView = [[MPVolumeView alloc] initWithFrame:CGRectZero];
volumeView.showsVolumeSlider = NO;
[volumeView sizeToFit];
[self.view addSubview:volumeView];
```
可以弹出airplay播放设备，如mac上的air server，但air Server闪退，原因未知












