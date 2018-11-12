title: AVFoundation学习笔记十 AVCaptureSession2
author: Cyrus
tags: []
categories:
  - AVFoundation
date: 2018-11-12 22:22:00
---

视频缩放
```
- (BOOL)cameraSupportsZoom {
 	return self.activeCamera.activeFormat.videoMaxZoomFactor > 1.0f;        // 1
}
- (CGFloat)maxZoomFactor {
 	return MIN(self.activeCamera.activeFormat.videoMaxZoomFactor, 4.0f);    // 2
}
- (void)setZoomValue:(CGFloat)zoomValue {                                   // 3
 	if (!self.activeCamera.isRampingVideoZoom) {
        NSError *error;
        if ([self.activeCamera lockForConfiguration:&error]) {              // 4
            // Provide linear feel to zoom slider
 			CGFloat zoomFactor = pow([self maxZoomFactor], zoomValue);      // 5
            self.activeCamera.videoZoomFactor = zoomFactor;
            [self.activeCamera unlockForConfiguration];                     // 6
 		} else {
            [self.delegate deviceConfigurationFailedWithError:error];
        }
 	}
}
- (void)rampZoomToValue:(CGFloat)zoomValue {                                // 1
    CGFloat zoomFactor = pow([self maxZoomFactor], zoomValue);
 	NSError *error;
 	if ([self.activeCamera lockForConfiguration:&error]) {
 		[self.activeCamera rampToVideoZoomFactor:zoomFactor                 // 2
                                        withRate:THZoomRate];
 		[self.activeCamera unlockForConfiguration];
 	} else {
 		[self.delegate deviceConfigurationFailedWithError:error];
 	}
}
```

人脸检测
```
- (BOOL)setupSessionOutputs:(NSError **)error {
//设置输出数据类型
    self.metadataOutput = [[AVCaptureMetadataOutput alloc] init];           // 2
    if ([self.captureSession canAddOutput:self.metadataOutput]) {
        [self.captureSession addOutput:self.metadataOutput];
//设置人脸检测
        NSArray *metadataObjectTypes = @[AVMetadataObjectTypeFace];         // 3
        self.metadataOutput.metadataObjectTypes = metadataObjectTypes;
        dispatch_queue_t mainQueue = dispatch_get_main_queue();
        [self.metadataOutput setMetadataObjectsDelegate:self                // 4
                                                  queue:mainQueue];
        return YES;
    } else {                                                                // 5
        if (error) {
            NSDictionary *userInfo = @{NSLocalizedDescriptionKey:
                                           @"Failed to still image output."};
            *error = [NSError errorWithDomain:THCameraErrorDomain
                                         code:THCameraErrorFailedToAddOutput
                                     userInfo:userInfo];
        }
        return NO;
    }
}
//人脸检测回调
- (void)captureOutput:(AVCaptureOutput *)captureOutput
didOutputMetadataObjects:(NSArray *)metadataObjects
       fromConnection:(AVCaptureConnection *)connection {
    for (AVMetadataFaceObject *face in metadataObjects) {                   // 2
        NSLog(@"Face detected with ID: %li", (long)face.faceID);
        NSLog(@"Face bounds: %@", NSStringFromCGRect(face.bounds));
    }
    [self.faceDetectionDelegate didDetectFaces:metadataObjects];            // 3
    
}
```

条形码检测
```
- (BOOL)setupSessionOutputs:(NSError **)error {
    self.metadataOutput = [[AVCaptureMetadataOutput alloc] init];
    if ([self.captureSession canAddOutput:self.metadataOutput]) {
        [self.captureSession addOutput:self.metadataOutput];
        dispatch_queue_t mainQueue = dispatch_get_main_queue();
        [self.metadataOutput setMetadataObjectsDelegate:self
                                                  queue:mainQueue];
        NSArray *types = @[AVMetadataObjectTypeQRCode,                      // 1
                           AVMetadataObjectTypeAztecCode,
                           AVMetadataObjectTypeUPCECode];
        self.metadataOutput.metadataObjectTypes = types;
    } else {
        NSDictionary *userInfo = @{NSLocalizedDescriptionKey:
                                       @"Failed to still image output."};
        *error = [NSError errorWithDomain:THCameraErrorDomain
                                     code:THCameraErrorFailedToAddOutput
                                 userInfo:userInfo];
        return NO;
    }
    return YES;
}
- (void)captureOutput:(AVCaptureOutput *)captureOutput
didOutputMetadataObjects:(NSArray *)metadataObjects
       fromConnection:(AVCaptureConnection *)connection {
    [self.codeDetectionDelegate didDetectCodes:metadataObjects];            // 2
}
```