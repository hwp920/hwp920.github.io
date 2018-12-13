title: mac 编译FFmpeg iOS库
author: Cyrus
tags: []
categories:
  - 音视频
date: 2018-11-12 22:46:00
---

新装了主机，编译FFmpeg的iOS库，记录一下：
### 一、准备部分
1、去FFmpeg官网下载所需要的版本的源码：官网源码下载地址（若没下载，编译脚本也可以指定版本自动下载）

2、安装Homebrew：
```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

3、安装yasm：
```
brew install yasm
```

4、下载：[https://github.com/libav/gas-preprocessor](https://github.com/libav/gas-preprocessor) ，复制gas-preprocessor.pl到/usr/local/bin下，若需要修改文件权限 ：
```
chmod 777 /usr/local/bin/gas-preprocessor.pl
```

### 二、编译ffmpeg
1、下载脚本：[https://github.com/kewlbear/FFmpeg-iOS-build-script](https://github.com/kewlbear/FFmpeg-iOS-build-script)

2、解压后执行build-ffmpeg.sh即可。如果同级目录ffmpeg文件夹不存在，格式如“ffmpeg-3.4.5”，则自动从官网下载后再编译；
```
...
# directories
FF_VERSION="3.4.5" //与你下载的版本一致，或者没下载，等脚本自动下载
...
```

3、合并成一个.a文件
```
./build-ffmpeg.sh lipo
```

### 出现的问题：
提示：<font color=ff0000>xcrun -sdk iphoneos clang is unable to create an executable file.</font>
原因：新安装xcode,命令行工具没有联系起来。
解决：终端输入：
```
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer/
```

