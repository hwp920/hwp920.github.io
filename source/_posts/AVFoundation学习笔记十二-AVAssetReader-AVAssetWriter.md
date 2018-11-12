title: AVFoundation学习笔记十二 AVAssetReader AVAssetWriter
author: Cyrus
tags: []
categories:
  - AVFoundation
date: 2018-11-12 22:33:00
---

![](read_write.png)

### AVAssetReader
AVAsserReader用于从AVAsset实例中读取媒体样本。通常会配置一个或多个AVAssetReaderOutput实例，并通过copyNextSampleBuffer方法可以访问音频样本和视频帧。AVAssetReaderOutput是一个抽象类，不过框架定义了3个具本实例来从指定的AVAssetTrack中读取解码的媒体样本，从多音频轨道中读取混合输出，或者从多视频轨道中读取组合输出。一个资源读取器的内部通道都是以多线程的方式不断提取下一个可用样本的，这样可以在系统请求资源时最小化时延。

### AVAssetWriter
AVAssetWriter是AVAssetReader对应的兄弟类，它用于对资源进行编码并将其写入到容器文件中。它由一个或多个AVAssetWriterInput对象配置，用于附加将包含要写入容器的媒体样本的CMSampleBuffer对象。AVAssetWriterInput被配置为可以处理指定的媒体类型，比如音频或视频，并且附加在其后的样本会在最终输出时生成一个独立的AVAssetTrack.当使用一个配置了处理视频样本的AVAssetWriterInput时，开发者会经常用到一个专门的适配器对象AVAssetWriterInputPixelBufferAdaptor.这个类在附加被包装为CVPixelBuffer对象的视频样本时提供最优性能。
AVAssetWriter可用于实时操作和离线操作两种情况，不过对于每个场景都有不同的方法将样本buffer添加到写入对象的输入中：

***实时***：当处理实时资源时，比如从AVCaptureVideoDataOutput写入捕捉的样本时，AVAssetWriterInput应该令 expectsMediaDataInRealTime属性为YES来确保 readyForMoreMediaData值被正确计算。从实时资源写入数据优化了写入器，这样一来，与维持交错效果相比，快速写入样本具有更高的优先级。这一优化效果不错，视频和音频样本以大致相同的速率捕捉，传入数据自然交错。

***离线***： 当从离线资源读取媒体资源时，比如从AVAssetReader读取样本buffer，在附加样本前仍然需要观察写入的readyForMoreMediaData属性的状态，不过可以使用requestMediaDataWhenReadyOnQueue：usingBlock:方法控制数据的提供。传到这个方法中的代码会随写入器输入准备附加更多的样本而不断被调用，添加样本时开发者需要检索数据并从资源中找到下一个样本进行添加。

读资源
```
AVAsset *asset = ...;
    AVAssetTrack *track = [[asset tracksWithMediaType:AVMediaTypeVideo]firstObject];
    AVAssetReader *assetReader = [[AVAssetReader alloc] initWithAsset:asset error:nil];
    NSDictionary *readerOutputSettings = @{
                                           (id)kCVPixelBufferPixelFormatTypeKey : @(kCVPixelFormatType_32BGRA)
                                           };
    AVAssetReaderOutput *trackOutput = [[AVAssetReaderTrackOutput alloc] initWithTrack:track outputSettings:readerOutputSettings];
    [assetReader addOutput:trackOutput];
    [assetReader startReading];
```

写资源
```
NSURL *outputURL = ...;
    AVAssetWriter *assetWriter = [[AVAssetWriter alloc] initWithURL:outputURL fileType:AVFileTypeQuickTimeMovie error:nil];
    NSDictionary *writerOutputSettings = @{
                                           AVVideoCodecKey : AVVideoCodecH264,
                                           AVVideoWidthKey : @1280,
                                           AVVideoHeightKey : @720,
                                           AVVideoCompressionPropertiesKey : @{
                                                   AVVideoMaxKeyFrameIntervalKey : @1,
                                                   AVVideoAverageBitRateKey : @10500000,
                                                   AVVideoProfileLevelKey : AVVideoProfileLevelH264Main31
                                                   }
                                           };
    AVAssetWriterInput *writerInput = [[AVAssetWriterInput alloc] initWithMediaType:AVMediaTypeVideo outputSettings:writerOutputSettings];
    [assetWriter addInput:writerInput];
    [assetWriter startWriting];
```

从AVAssetReader拉取数据写入到文件
```
dispatch_queue_t dispatchQueue = dispatch_queue_create("com.tapharmonic.writerQueue", NULL);
    [assetWriter startSessionAtSourceTime:kCMTimeZero];
    [writerInput requestMediaDataWhenReadyOnQueue:dispatchQueue usingBlock:^{
        BOOL complete = NO;
        while ([writerInput isReadyForMoreMediaData] && !complete) {
            CMSampleBufferRef sampleBuffer = [trackOutput copyNextSampleBuffer];
            if (sampleBuffer) {
                BOOL result = [writerInput appendSampleBuffer:sampleBuffer];
                CFRelease(sampleBuffer);
                complete = !result;
            } else {
                [writerInput markAsFinished];
                complete = YES;
            }
        }
        if (complete) {
            [assetWriter finishWritingWithCompletionHandler:^{
                AVAssetWriterStatus status = assetWriter.status;
                if (status == AVAssetWriterStatusCompleted) {
                    //Handle success case
                } else {
                    //Handle failure case
                }
            }];
        }
    }];
```