title: iOS使用openfec
author: Cyrus
tags: []
categories:
  - 流媒体
date: 2019-11-29 23:03:00
---
#### 一、什么是FEC
我们都知道IP协议是不可靠的传输协议，而建立在IP协议之上的TCP和UDP，一个是可靠协议，一个是不可靠协议。TCP协议之所以可靠，是因为它有滑动窗口、重传等一系列措施保障，而UDP却没有。

当然，TCP由于增加了一系列措施，所以延时较高，UDP简单，所以延时相对较低。但是有的时候，我们又想延时低，又想保证数据的可靠性，没错，就是这么贪心，怎么办呢？于是就出现了常见的两种方式：FEC和ARQ。

* ARQ 自动重传请求（Automatic Repeat-reQuest，ARQ）,简单地说就是接收方发现丢包后，去发送方请求重传。具体实现可以看看KCP(https://github.com/skywind3000/kcp),整个协议只有 ikcp.h, ikcp.c两个源文件，可以方便的集成到项目中，这里就不细说了。

* FEC 前向纠错也叫前向纠错码(Forward Error Correction，简称FEC)。FEC是前向冗余，举个例子，发送数据A和B，增加发送一个数据C等于A和B的异或。接收方接到这3个包的任意2个包，异或一下就可以得到第3个包。

两者的说明，区别等在网上都可以查得到，如https://blog.csdn.net/dxpqxb/article/details/79567297  这些都不是本文的重点。fec常用的开源项目有好几个，这里只讲openfec。

#### 二、编译openfec
1、下载源码
~~~
http://openfec.org/accueil.html
~~~

2、正常编译   
在源码目录下的“README”就有教如何编译，当然是基于cmake的，具体怎么装自行百度。
~~~
// 进入源码目录
mkdir build  //创建build文件夹，避免cmake生成文件与源文件混在一起，常规操作
cd build

cmake .. -DDEBUG:STRING=OFF		// 生成Release版本
cmake .. -DDEBUG:STRING=ON		// 生成Debug版本

生成的文件在源码目录的 bin/{Debug|Release} 文件夹里
~~~

#### 三、编译iOS使用的openfec库
下面进入正题，说说如何编译iOS使用的版本：

1、 在源码的src文件夹里，打开CMakeLists.txt：
~~~
将第三行的：
add_library(openfec  SHARED  ${openfec_sources})

改为：
add_library(openfec  STATIC  ${openfec_sources})

// 这样修改是将编译成的库由动态库改为静态库，iOS用动态库有时会出问题，建议使用静态库。普通编译不想用动态库也可以这么改。
~~~

2、在源码目录，创建build文件夹，进入并执行 cmake -G Xcode ..
~~~
mkdir build
cd build
cmake -G Xcode ..
~~~
这样，就会在build目录下生成一个xcode项目，如下图：
![](fec_1.png)
相信对于从事iOS的人来说，相对于cmake,看到.xcodeproj肯定倍感亲切。

3、打开项目，现在是项目还是基于Mac平台的，操作如下图：
![](fec_2.png)
![](fec_3.png)
![](fec_4.png)
恭喜你，你已经得到了模拟器和真机运行需要的静态库了。

4、合成fat库
![](fec_5.png)

5、将需要的头文件拷贝出来
![](fec_6.png)
现在，我们已经得到iOS环境下的静态库和头文件夹了，赶快建个项目测试一下吧。

6、项目测试
![](fec_7.png)
![](fec_8.png)