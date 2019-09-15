title: FFMpeg QT跨平台播放器一 FFMpeg编译
author: Cyrus
tags:
  - 播放器
categories:
  - 音视频
date: 2019-09-15 21:50:00
---
### 一、下载
到ffmpeg官网 http://ffmpeg.org/download.html 根据你要的平台下载相应的库。如Mac/linux平台：下载最新的ffmpeg-4.2.1的代码。Windows的可以根据需要，下载32位/64位的静态或动态库，这里选择64位动态库。下载shared(动态库)和Dev(用于开发时链接符号)。

### 二、编译
对于下载源码的，需要进行编译，编译的优势是可以根据需求，选取相应的功能模块，减小应用体积。像上面的Windows动态库的，则无需编译直接使用。
#### 1、进入代码所在文件夹，查看编译选项
~~~
cd ffmpeg路径
./configure --help

// 配置
./configure  --prefix=/Users/cyrus/Downloads/ffmpeg-4.2/build  --disable-doc  --disable-htmlpages  --disable-manpages  --disable-network
~~~

* 这里讲解一下，--prefix=/xxx/xxx 是编译出来的包存放的地址，这里是在ffmpeg源码里新建一个build文件夹存放，可以根据需求设置。

![](make_1.png)
![](make_2.png)

报错，根据报错修改
~~~
./configure  --prefix=/Users/cyrus/Downloads/ffmpeg-4.2/build  --disable-doc  --disable-htmlpages  --disable-manpages  --disable-network  --disable-x86asm
~~~

![](make_3.png)
configure配置通过，开始编译
~~~
make -j8 && make install
// 这里也可以分开写 make 编译完再make install, -j8是开8个线程编译，可以根据电脑配置选择写或不写。多线程编译速度较快。
~~~

![](make_4.png)
![](make_5.png)
编译完成，build文件夹里面的include和lib就是我们需要的头文件和库文件。

![](make_6.png)
这里再说一个lib文件夹下面的pkgconfig，里面存放的是生成库的依赖关系。我们在qt的.pro文件里面需要用到。