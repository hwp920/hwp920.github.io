title: FFMpeg QT跨平台播放器六 Linux和windows兼容配置
author: Cyrus
tags:
  - 播放器
categories:
  - 音视频
date: 2019-09-21 17:41:00
---

### 一、Ubuntu下编译
Ubuntu下面的编译跟Mac下面一样，详见第一篇的FFMpeg编译。下面看看Ubuntu下面QT .pro文件的配置。
* 要注意的一点是，Ubuntu下的对库和依赖的顺序有要求，提示找不到链接符号什么的，需要根据pkgconfig修改下位置。

~~~
unix: !macx {
INCLUDEPATH += $$PWD/libFFmpeg/Linux/include
LIBS += -L$$PWD/libFFmpeg/Linux/lib
LIBS +=  -lavfilter  \
        -lavformat  \
        -lavcodec   \
        -lavutil    \
        -lswresample \
        -lswscale
LIBS += -lm -lxcb -lxcb-shm -lxcb-shape -lxcb -lxcb-xfixes \
        -lxcb-render -lxcb-shape -lxcb -lasound -lSDL2 -lsndio -lX11 -lva -lva-drm -lvdpau
}

~~~

### 二、windows下集成
windows下面我们采用动态库链接的方式，到官网下载地址https://ffmpeg.zeranoe.com/builds/，下载share(动态库)和dev(开发时提供符号和重定向信息)。
![](win_1.png)

开发时，我们把dev文件夹下面的include和lib放到项目下面。.pro文件配置如下：
~~~
win32 {
INCLUDEPATH += $$PWD/libFFmpeg/win32/include
LIBS += $$PWD/libFFmpeg/win32/lib/avfilter.lib  \
        $$PWD/libFFmpeg/win32/lib/avformat.lib  \
        $$PWD/libFFmpeg/win32/lib/avcodec.lib   \
        $$PWD/libFFmpeg/win32/lib/swscale.lib   \
        $$PWD/libFFmpeg/win32/lib/swresample.lib
}

~~~

编译通过后，再找到编译生成的文件夹，把shared/bin目录下的.dll动态库文件拷到程序同级目录。
![](win_2.png)

### 三、运行效果
mac
![](result_mac.png)

windows
![](result_win.png)

ubuntu
![](result_unix.png)

