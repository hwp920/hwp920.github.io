title: H264 profile类型、特点及解析
author: Cyrus
tags: []
categories:
  - 音视频
date: 2019-01-24 23:14:00
---
#### 一、profile的几种类型
H264 主要包括Baseline,  Ext，Main, High这几种常用profile和一些特殊用途的profies，如Constrain baseline， SVC，MVC和一系列high-Fidelity profiles 等等，各种profile是根据不同的应用场景设计的，具体余下：
* Baseline主要是用于可视电话，会议电视，无线通讯等实时通信。要实时，就要减少视频decode和display的时延，所以没有B frame；为了提高针对网络丢包的容错能力，特意添加了FMO，ASO和冗余slice；
* Main用于数字广播电视和数字视频存储，侧重点在于提高压缩率，所以有了CABAC，MBAFF，Interlace，B frame等。
* Extend用于改进误码性能和码流切换（SP和SI slice），侧重于码流切换（SI，SP slice）和error resilience（数据分割）。
*  High主要用于高压缩效率和质量， 引入8x8 DCT，选择量化矩阵等。

各个不同profile的具体feature对比如下：
![](profile_feather.png)

#### 二、解码过程中profile的解析
profile参数存放在SPS之中，在做视频播放器时，为了让后续的解码过程可以使用SPS中包含的参数，必须对其中的数据进行解析。其中H.264标准协议中规定的SPS格式位于文档的7.3.2.1.1部分，如下图所示：
![](profile_decode.png)

各种类型的profile在SPS对应的profile_idc为：
* <font color=#ff0000>baseline</font>:  profile_idc = 66 or constraint_set0_flag = 1
* <font color=#ff0000>constrain baseline</font>: profile_idc = 66 && constraint_set1_flag = 1
* <font color=#ff0000>main profile</font>: profile_idc = 77  or  constraint_set1_flag = 1
* <font color=#ff0000>extend profile</font>: profile_idc = 88 or constraint_set2_flag = 1
* <font color=#ff0000>high profile</font>: profile_idc = 100
* <font color=#ff0000>High 10</font>: profile_idc = 110
* <font color=#ff0000>High 422</font>: profile_idc = 122
* <font color=#ff0000>High 444</font>: profile_idc = 244
* <font color=#ff0000>Multiview</font>: profile_idc = 118
* <font color=#ff0000>Stereo Hight</font>: profile_idc = 128

Reference：
* https://blog.csdn.net/lixiaowei16/article/details/22370217
* https://www.cnblogs.com/wainiwann/p/7477794.html