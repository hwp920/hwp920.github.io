title: AVFoundation学习笔记十一 AVCaptureVideoDataOutput
author: Cyrus
tags: []
categories:
  - AVFoundation
date: 2018-11-12 22:25:00
---

### AVCaptureVideoDataOutput
AVCaptureVideoDataOutput是一个AVCaptureOutput子类，可以直接访问摄像头伟感器捕捉到的视频帧。这是一个强大的功能，因为这样我们就完全控制了视频数据的格式、时间和元数据，可以按照需求操作视频内容。
注：AVFoundation为处理音频数据提供了一个底层捕捉输出AVCaptureAudioDataOutput。使用方法类似。

与输出AVMetadataObject实例不同，AVCaptureVideoDataOutput输出的对象需要通过AVCaptureVideoDataOutputSampleBufferDelegate协议包含视频数据。

AVCaptureVideoDataOutputSampleBufferDelegate定义了下面两个方法：
```
//每当有一个新的视频帧写入时该方法就会被调用。数据会基于视频数据输出的videoSetting属性进行解码或重新编码。
- (void)captureOutput:(AVCaptureOutput *)output didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection

//每当一个迟到的视频帧被丢弃时就会调用该方法。通过是因为在didOutputSampleBuffer：调用中消耗了太多处理时间就会调用该方法。
- (void)captureOutput:(AVCaptureOutput *)output didDropSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection
```

### CMSampleBuffer
CMSampleBuffer是一个由Core Media框架提供的Core Foundation风格对象，用于在媒体管道传输数字样本。CMSampleBuffer的角色是将基础的样本数据进行封装并提供格式和时间信息，还会加上所有在转换和处理数据时用到的元数据。

### 样本数据
在使用AVCaptureVideoDataOutput时，sample buffer会包含一个CVPixelBuffer，它是一个带有带个视频帧原始像素数据的CoreVideo对象。

```
从sampleBuffer中获取图片
CVImageBufferRef imageBuffer =  CMSampleBufferGetImageBuffer(sampleBuffer);
    
    CVPixelBufferLockBaseAddress(imageBuffer, 0);
    void *baseAddress = CVPixelBufferGetBaseAddress(imageBuffer);
    size_t width = CVPixelBufferGetWidth(imageBuffer);
    size_t height = CVPixelBufferGetHeight(imageBuffer);
    size_t bufferSize = CVPixelBufferGetDataSize(imageBuffer);
    size_t bytesPerRow = CVPixelBufferGetBytesPerRowOfPlane(imageBuffer, 0);
    
    CGColorSpaceRef rgbColorSpace = CGColorSpaceCreateDeviceRGB();
    CGDataProviderRef provider = CGDataProviderCreateWithData(NULL, baseAddress, bufferSize, NULL);
     
    CGImageRef cgImage = CGImageCreate(width, height, 8, 32, bytesPerRow, rgbColorSpace, kCGImageAlphaNoneSkipFirst|kCGBitmapByteOrder32Little, provider, NULL, true, kCGRenderingIntentDefault);

    
    UIImage *image = [UIImage imageWithCGImage:cgImage];
     
    CGImageRelease(cgImage);
    CGDataProviderRelease(provider);
    CGColorSpaceRelease(rgbColorSpace);

    NSData* imageData = UIImageJPEGRepresentation(image, 1.0);
    image = [UIImage imageWithData:imageData];
    CVPixelBufferUnlockBaseAddress(imageBuffer, 0);
    return image;
```

### 格式描述
除了原始媒体样本本身之外，CMSampleBuffer还提供了以CMFormatDescription对象的形式来访问样本的信息。CMFormatDescription.h定义了大量函数用于访问媒体样本的更多细节。在头文件中带有CMFormatDescription的函数分别适用于获取视频和音频细节。
```
判断sampleBuffer格式
CMFormatDescriptionRef formatDescription =                              // 2
        CMSampleBufferGetFormatDescription(sampleBuffer);
    CMMediaType mediaType = CMFormatDescriptionGetMediaType(formatDescription);
    if (mediaType == kCMMediaType_Video) {
        CVPixelBufferRef pixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer);
        //process the frame of video
    } else if(mediaType == kCMMediaType_Audio) {
        CMBlockBufferRef blockBuffer = CMSampleBufferGetDataBuffer(sampleBuffer);
        //process audio samples
    }
```

### 时间信息
CMSampleBuffer还定义了关于媒体样本的时间信息。可以分别使用CMSampleBufferGetPresentationTimeStamp函数和CMSampleBufferGetDecodeTimeStamp函数提取时间信息来得到原始的表示时间戳和解码时间戳

### 附加的元数据
CoreMedia还在CMAttachment.h中定义了一个CMAttachment形式的元数据协议。API提供了读取和写入底层元数据的基础架构，比如可交换图片文件格式（Exif）标签。
从一个给定的CMSampleBuffer中获取Exif元数据。

```
CFDictionaryRef exifAttachments = CMGetAttachment(sampleBuffer, kCGImagePropertyExifDictionary, NULL);

//打印的信息
(lldb) po exifAttachments
{
    ApertureValue = "2.27500704749987";
    BrightnessValue = "3.366379752578831";
    ColorSpace = 1;
    DateTimeDigitized = "2018:11:12 15:22:21";
    DateTimeOriginal = "2018:11:12 15:22:21";
    ExposureBiasValue = 0;
    ExposureTime = "0.03333333333333333";
    FNumber = "2.2";
    Flash = 0;
    FocalLenIn35mmFilm = 155;
    FocalLength = "2.87";
    ISOSpeedRatings =     (
        64
    );
    LensMake = Apple;
    LensModel = "iPhone 8 front camera 2.87mm f/2.2";
    LensSpecification =     (
        "2.87",
        "2.87",
        "2.2",
        "2.2"
    );
    MeteringMode = 5;
    PixelXDimension = 640;
    PixelYDimension = 480;
    SceneType = 1;
    SensingMethod = 2;
    ShutterSpeedValue = "4.907121445283527";
    SubsecTimeDigitized = 923;
    SubsecTimeOriginal = 923;
    WhiteBalance = 0;
}
```