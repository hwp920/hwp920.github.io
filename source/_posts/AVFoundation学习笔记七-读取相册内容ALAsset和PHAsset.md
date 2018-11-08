title: AVFoundation学习笔记六 读取相册内容ALAsset和PHAsset
author: Cyrus
tags: []
categories:
  - AVFoundation
date: 2018-11-05 17:04:00
---

### ALAsset (iOS9.0已废弃)
1、导入头文件
```
#import <AssetsLibrary/AssetsLibrary.h>
```

2、具体流程
![](ALAsset.png)

3、相关代码
```
//1、创建ALAssetsLibrary
    ALAssetsLibrary *library = [[ALAssetsLibrary alloc] init];
    //2、根据传入的type，在block中获得ALAssetsGroup，即相应的相册
    [library enumerateGroupsWithTypes:ALAssetsGroupSavedPhotos usingBlock:^(ALAssetsGroup *group, BOOL *stop) {
        //3、相册设置筛选类型
        [group setAssetsFilter:[ALAssetsFilter allVideos]];
        //4、根据o筛选类型，遍历相册，在block回调中得到相应ALAsset
        [group enumerateAssetsAtIndexes:[NSIndexSet indexSetWithIndex:0] options:0 usingBlock:^(ALAsset *result, NSUInteger index, BOOL *stop) {
            //5、对ALAsset进行相关操作
            if (result) {
                ALAssetRepresentation *representation = [result defaultRepresentation];
                NSURL *url = representation.url;
                NSLog(@"%@", url);
            }
        }];

    } failureBlock:^(NSError *error) {
        NSLog(@"error: %@", error.localizedDescription);
    }];
```

### PHAsset (ALAsset的代替）
1、导入头文件
```
#import <Photos/Photos.h>
```

2、具体流程
![](PHAsset.png)

3、相关代码
```
/**
     * 1、PHAssetCollection（类似上面ALAssetsGroup相册的概念,有多个类，根据具体情况选择），调用fetch 开头的函数，得到一个PHFetchResult<PHAssetCollection *> *结果对象（相当于符合筛选的PHAssetCollection相册数组）
     *   PHFetchResult:类似于C++的模板类，提供统一的collection类、PHAsset类筛选结果的遍历、获取方法
     */
    PHFetchResult *results = [PHAssetCollection fetchAssetCollectionsWithType:PHAssetCollectionTypeAlbum subtype:PHAssetCollectionSubtypeAlbumRegular options:0];
    //2、遍历results,对象为上面调用fetch函数的collection类
    [results enumerateObjectsUsingBlock:^(PHAssetCollection *obj, NSUInteger idx, BOOL * _Nonnull stop) {
        //3、PHAsset调用fetch方法，获取相应PHAssetCollection相册内符合要求的PHAsset集合PHFetchResult
        PHFetchResult *assets = [PHAsset fetchAssetsInAssetCollection:obj options:nil];
        //4、遍历assets
        [assets enumerateObjectsUsingBlock:^(PHAsset *obj, NSUInteger idx, BOOL * _Nonnull stop) {
            //根据asset的mediaType类型，调用相应的PHImageManager request方法（注意不同类型传入的options参数不同）
            if (obj.mediaType == PHAssetMediaTypeImage) {
                [[PHImageManager defaultManager] requestImageForAsset:obj targetSize:CGSizeMake(obj.pixelWidth, obj.pixelHeight) contentMode:PHImageContentModeDefault options:[[PHImageRequestOptions alloc] init] resultHandler:^(UIImage * _Nullable result, NSDictionary * _Nullable info) {
                    NSLog(@"%@", [NSThread currentThread]);
                    UIImageView *imageView = [[UIImageView alloc] initWithFrame:self.view.bounds];
                    imageView.image = result;
                    [self.view addSubview:imageView];
                }];
            } else if (obj.mediaType == PHAssetMediaTypeVideo) {
                
            }
        }];
    }];
```

