title: AVFoundation学习笔记九 AVCaptureSession1
author: Cyrus
tags: []
categories:
  - AVFoundation
date: 2018-11-12 22:11:00
---

![](capture_session1.png)

### AVCaptureSession 
AVCaptureSession是AVFoundation捕捉栈的核心类。一个session相当于一个虚拟的”插线板”，用于连接输入和输出资源。捕捉会话被处理从物理设备得到的数据流，比如摄像头和麦克风设备，输出到一个或多个目的地。可以动态地配置输入和输出的线路，让开发者能够在会话进行中按需重新配置捕捉环境。

session还可以额外配置一个会话预设值（session preset）,用来控制捕捉数据的格式和质量。会话预设值默认为AVCaptureSessionPresetHigh,它适用于大多数情况。

### AVCaptureDevice
AVCaptureDevice为诸如摄像头或麦克风等物理设备定义了一个接口。AVCaptureDevice针对物理硬件设备定义了大量的控制方法，比如控制摄像头的对焦、曝光、白平衡和闪光灯等。
AVCaptureDevice定义了大量类方法用于访问系统的捕捉设备，最常用的一个方法是defaultDeviceWithMediaType:,它会根据给定的媒体类型返回一个系统指定的默认设备。
```
+ (NSArray<AVCaptureDevice *> *)devicesWithMediaType:(AVMediaType)mediaType
+ (nullable AVCaptureDevice *)defaultDeviceWithMediaType:(AVMediaType)mediaType;
```

### AVCaptureDeviceInput
在使用捕捉设备进行处理前，需要将它添加为session的输入。不过一个captureDevice不能直接添加到captureSession中，需要将它封装到一个AVCaptureDeviceInput实例来添加。
```
+ (nullable instancetype)deviceInputWithDevice:(AVCaptureDevice *)device error:(NSError * _Nullable * _Nullable)outError;
```

### AVCaptureOutput
AVFoundation定义了AVCaptureOutput的许多扩展类。AVCaptureOutput是一个抽象基类，用于为从捕捉会话得到的数据寻找输出目的地。框架定义了一些高级扩展类，如：
```
AVCaptureStillImageOutput:捕捉静态照片
AVCaptureMovieFileOutput: 捕捉视频
AVCaptureAudioDataOutput: 捕捉底层音频数据
AVCaptureVideoDataOutput: 捕捉底层视频数据
```

### AVCaptureConnection
session首先需要确定由给定captureInput渲染的媒体类型，并自动建立其到能够接收该媒体类型的captureOutput的连接.比如AVCaptureMovieFileOutput可以授受音频和视频数据，所以会话会确定哪些输入产生视频，哪些输入产生音频，正确地建立该连接。

### AVCaptureVideoPreviewLayer
AVCaptureVideoPreviewLayer展示正在捕捉的场景。previewLayer是一个Core Animation的CALayer子类，对捕捉视频数据进行实时预览。

代码：
```
//创建session和输入输出
self.captureSession = [[AVCaptureSession alloc] init];                  // 1
    self.captureSession.sessionPreset = AVCaptureSessionPresetHigh;
 	// Set up default camera device
 	AVCaptureDevice *videoDevice =                                          // 2
        [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
    AVCaptureDeviceInput *videoInput =                                      // 3
        [AVCaptureDeviceInput deviceInputWithDevice:videoDevice error:error];
    if (videoInput) {
        if ([self.captureSession canAddInput:videoInput]) {                 // 4
            [self.captureSession addInput:videoInput];
            self.activeVideoInput = videoInput;
        }
    } else {
        return NO;
    }
    // Setup default microphone
    AVCaptureDevice *audioDevice =                                          // 5
        [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeAudio];
    AVCaptureDeviceInput *audioInput =                                      // 6
        [AVCaptureDeviceInput deviceInputWithDevice:audioDevice error:error];
    if (audioInput) {
        if ([self.captureSession canAddInput:audioInput]) {                 // 7
            [self.captureSession addInput:audioInput];
        }
    } else {
        return NO;
    }
 	// Setup the still image output
    self.imageOutput = [[AVCaptureStillImageOutput alloc] init];            // 8
    self.imageOutput.outputSettings = @{AVVideoCodecKey : AVVideoCodecJPEG};
 	if ([self.captureSession canAddOutput:self.imageOutput]) {
 		[self.captureSession addOutput:self.imageOutput];
 	}
    return YES;
```

切换摄像头
```
- (AVCaptureDevice *)activeCamera {                                         // 3
    return self.activeVideoInput.device;
}
- (AVCaptureDevice *)inactiveCamera {                                       // 4
    AVCaptureDevice *device = nil;
    if (self.cameraCount > 1) {
        if ([self activeCamera].position == AVCaptureDevicePositionBack) {  // 5
            device = [self cameraWithPosition:AVCaptureDevicePositionFront];
        } else {
            device = [self cameraWithPosition:AVCaptureDevicePositionBack];
        }
    }
    return device;
}
- (BOOL)canSwitchCameras {                                                  // 6
    return self.cameraCount > 1;
}

- (BOOL)switchCameras {
    if (![self canSwitchCameras]) {                                         // 1
        return NO;
    }
    NSError *error;
    AVCaptureDevice *videoDevice = [self inactiveCamera];                   // 2
    AVCaptureDeviceInput *videoInput =
    [AVCaptureDeviceInput deviceInputWithDevice:videoDevice error:&error];
    if (videoInput) {
        [self.captureSession beginConfiguration];                           // 3
        [self.captureSession removeInput:self.activeVideoInput];            // 4
        if ([self.captureSession canAddInput:videoInput]) {                 // 5
            [self.captureSession addInput:videoInput];
            self.activeVideoInput = videoInput;
        } else {
            [self.captureSession addInput:self.activeVideoInput];
        }
        [self.captureSession commitConfiguration];                          // 6
    } else {
        [self.delegate deviceConfigurationFailedWithError:error];           // 7
        return NO;
    }
    return YES;
}
```

闪光灯
```
- (BOOL)cameraHasFlash {
    return [[self activeCamera] hasFlash];
}
- (AVCaptureFlashMode)flashMode {
    return [[self activeCamera] flashMode];
}
- (void)setFlashMode:(AVCaptureFlashMode)flashMode {
    AVCaptureDevice *device = [self activeCamera];
    if (device.flashMode != flashMode &&
        [device isFlashModeSupported:flashMode]) {
        NSError *error;
        if ([device lockForConfiguration:&error]) {
            device.flashMode = flashMode;
            [device unlockForConfiguration];
        } else {
            [self.delegate deviceConfigurationFailedWithError:error];
        }
    }
}
```

手电筒
```
- (BOOL)cameraHasTorch {
    return [[self activeCamera] hasTorch];
}
- (AVCaptureTorchMode)torchMode {
    return [[self activeCamera] torchMode];
}
- (void)setTorchMode:(AVCaptureTorchMode)torchMode {
    AVCaptureDevice *device = [self activeCamera];
    if (device.torchMode != torchMode &&
        [device isTorchModeSupported:torchMode]) {
        NSError *error;
        if ([device lockForConfiguration:&error]) {
            device.torchMode = torchMode;
            [device unlockForConfiguration];
        } else {
            [self.delegate deviceConfigurationFailedWithError:error];
        }
    }
}
```

对焦
```
- (BOOL)cameraSupportsTapToFocus {                                          // 1
    return [[self activeCamera] isFocusPointOfInterestSupported];
}
- (void)focusAtPoint:(CGPoint)point {                                       // 2
    AVCaptureDevice *device = [self activeCamera];
    if (device.isFocusPointOfInterestSupported &&                           // 3
        [device isFocusModeSupported:AVCaptureFocusModeAutoFocus]) {
        NSError *error;
        if ([device lockForConfiguration:&error]) {                         // 4
            device.focusPointOfInterest = point;
            device.focusMode = AVCaptureFocusModeAutoFocus;
            [device unlockForConfiguration];
        } else {
            [self.delegate deviceConfigurationFailedWithError:error];
        }
    }
}
```

曝光
```
- (BOOL)cameraSupportsTapToExpose {                                         // 1
    return [[self activeCamera] isExposurePointOfInterestSupported];
}
// Define KVO context pointer for observing 'adjustingExposure" device property.
static const NSString *THCameraAdjustingExposureContext;
- (void)exposeAtPoint:(CGPoint)point {
    AVCaptureDevice *device = [self activeCamera];
    AVCaptureExposureMode exposureMode =
    AVCaptureExposureModeContinuousAutoExposure;
    if (device.isExposurePointOfInterestSupported &&                        // 2
        [device isExposureModeSupported:exposureMode]) {
        NSError *error;
        if ([device lockForConfiguration:&error]) {                         // 3
            device.exposurePointOfInterest = point;
            device.exposureMode = exposureMode;
            if ([device isExposureModeSupported:AVCaptureExposureModeLocked]) {
                [device addObserver:self                                    // 4
                         forKeyPath:@"adjustingExposure"
                            options:NSKeyValueObservingOptionNew
                            context:&THCameraAdjustingExposureContext];
            }
            [device unlockForConfiguration];
        } else {
            [self.delegate deviceConfigurationFailedWithError:error];
        }
    }
}
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary *)change
                       context:(void *)context {
    if (context == &THCameraAdjustingExposureContext) {                     // 5
        AVCaptureDevice *device = (AVCaptureDevice *)object;
        if (!device.isAdjustingExposure &&                                  // 6
            [device isExposureModeSupported:AVCaptureExposureModeLocked]) {
            [object removeObserver:self                                    // 7
                        forKeyPath:@"adjustingExposure"
                           context:&THCameraAdjustingExposureContext];
            dispatch_async(dispatch_get_main_queue(), ^{                    // 8
                NSError *error;
                if ([device lockForConfiguration:&error]) {
                    device.exposureMode = AVCaptureExposureModeLocked;
                    [device unlockForConfiguration];
                } else {
                    [self.delegate deviceConfigurationFailedWithError:error];
                }
            });
        }
    } else {
        [super observeValueForKeyPath:keyPath
                             ofObject:object
                               change:change
                              context:context];
    }
}
```

AVCaptureStillImageOutput生成图片并保存
```
- (void)captureStillImage {
    AVCaptureConnection *connection =                                   
        [self.imageOutput connectionWithMediaType:AVMediaTypeVideo];
    if (connection.isVideoOrientationSupported) {                       
        connection.videoOrientation = [self currentVideoOrientation];
    }
    id handler = ^(CMSampleBufferRef sampleBuffer, NSError *error) {
        if (sampleBuffer != NULL) {
            NSData *imageData =
                [AVCaptureStillImageOutput
                    jpegStillImageNSDataRepresentation:sampleBuffer];
            UIImage *image = [[UIImage alloc] initWithData:imageData];
            [self writeImageToAssetsLibrary:image];                         // 1
        } else {
            NSLog(@"NULL sampleBuffer: %@", [error localizedDescription]);
        }
    };
    // Capture still image
    [self.imageOutput captureStillImageAsynchronouslyFromConnection:connection
                                                  completionHandler:handler];
}
- (void)writeImageToAssetsLibrary:(UIImage *)image {
    ALAssetsLibrary *library = [[ALAssetsLibrary alloc] init];              // 2
    [library writeImageToSavedPhotosAlbum:image.CGImage                    // 3
                              orientation:(NSInteger)image.imageOrientation // 4
                          completionBlock:^(NSURL *assetURL, NSError *error) {
                              if (!error) {
                                  [self postThumbnailNotifification:image]; // 5
                              } else {
                                  id message = [error localizedDescription];
                                  NSLog(@"Error: %@", message);
                              }
                          }];
}
```

AVCaptureMovieFileOutput生成文件并保存
```
/开始录制
[self.movieOutput startRecordingToOutputFileURL:self.outputURL recordingDelegate:self];

//停止录制
[self.movieOutput stopRecording];

#pragma mark - AVCaptureFileOutputRecordingDelegate - --
- (void)captureOutput:(AVCaptureFileOutput *)captureOutput
didFinishRecordingToOutputFileAtURL:(NSURL *)outputFileURL
      fromConnections:(NSArray *)connections
                error:(NSError *)error {
 	if (error) {                                                            // 1
        [self.delegate mediaCaptureFailedWithError:error];
 	} else {
        [self writeVideoToAssetsLibrary:[self.outputURL copy]];
 	}
    self.outputURL = nil;
}

- (void)writeVideoToAssetsLibrary:(NSURL *)videoURL {
    ALAssetsLibrary *library = [[ALAssetsLibrary alloc] init];              // 2
    if ([library videoAtPathIsCompatibleWithSavedPhotosAlbum:videoURL]) {   // 3
        ALAssetsLibraryWriteVideoCompletionBlock completionBlock;
        completionBlock = ^(NSURL *assetURL, NSError *error){               // 4
            if (error) {
                [self.delegate assetLibraryWriteFailedWithError:error];
            } else {
                [self generateThumbnailForVideoAtURL:videoURL];
            }
        };
        [library writeVideoAtPathToSavedPhotosAlbum:videoURL                // 8
                                    completionBlock:completionBlock];
    }
}
```