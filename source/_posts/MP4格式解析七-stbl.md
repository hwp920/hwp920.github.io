title: MP4格式解析七 stbl
author: Cyrus
tags:
  - MP4
categories:
  - 音视频
date: 2019-08-17 20:41:00
---
### 一、stbl (Sample Table Box)
 “stbl”包含了关于track中sample所有时间和位置的信息，以及sample的编解码等信息。利用这个表，可以解释sample的时序、类型、大小以及在各自存储容器中的位置。“stbl”是一个container box，其子box包括：sample description box（stsd）、time to sample box（stts）、sample size box（stsz或stz2）、sample to chunk box（stsc）、chunk offset box（stco或co64）、composition time to sample box（ctts）、sync sample box（stss）等。

![](stbl_1.png)
 
 ### 二、stsd(Sample Description Box)
 
 ![](stsd_1.png)
 
下面看一下数据:
 ![](stsd_2.png)
  
~~~
00 00 00 67 73 74 73 64 00 00 00 00 00 00 00 01 ; ...gstsd........
00 00 00 57 6D 70 34 61 00 00 00 00 00 00 00 01 ; ...Wmp4a........
00 00 00 00 00 00 00 00 00 02 00 10 00 00 00 00 ; ................
BB 80 00 00 00 00 00 33 65 73 64 73 00 00 00 00 ; ......3esds....
03 80 80 80 22 00 00 00 04 80 80 80 14 40 15 00 ; .".....@..
18 00 00 01 F4 00 00 01 F4 00 05 80 80 80 02 11 ; .............
88 06 80 80 80 01 02                            ; ....

==============  解析  ==============
00 00 00 67: size 103
73 74 73 64: stsd
00 00 00 00: version和flag均为1
00 00 00 01: Number of entries 1
00 00 00 57 .... : entries数据
~~~

#### mp4a && avc1
mp4a和avc1均为stsd的entries数据类型，分别对应着音频和视频(根据封装的音视频数据的不同，还可以有其他的类型)
#### 1、mp4a
下面看一下数据:
![](mp4a_1.png)

~~~
00 00 00 57 6D 70 34 61 00 00 00 00 00 00 00 01 ; ...Wmp4a........
00 00 00 00 00 00 00 00 00 02 00 10 00 00 00 00 ; ................
BB 80 00 00 00 00 00 33 65 73 64 73 00 00 00 00 ; ......3esds....
03 80 80 80 22 00 00 00 04 80 80 80 14 40 15 00 ; .".....@..
18 00 00 01 F4 00 00 01 F4 00 05 80 80 80 02 11 ; .............
88 06 80 80 80 01 02                            ; ....

==============  解析  ==============
00 00 00 57: size 87
6D 70 34 61: mp4a
00 00 00 00 00 00:  6字节保留，必须为0
00 01: index 1
00 00: unsigned int(16) pre_defined = 0;
00 00: const unsigned int(16) reserved = 0;
00 00 00 00 00 02 00 10 00 00 00 00: unsigned int(32)[3] pre_defined = 0;
BB 80 00 00: 暂时不知道什么作用
00 00 00 33 65 73 64 73 00 00 00 00....: esds box数据
~~~

#### esds box（ES Descriptor Box）解析
esds box中主要是存放Element Stream Descriptors（ESDs），该box的前四个字节为version&flag，一般为0x 00 00 00 00；
从偏移第四个字节开始，为ESDs。
ESDs中可以分为三层，每层为包含关系，分别为MP4ESDescr（0x03开始，一般7个字节），MP4DecConfigDescr（0x04开始，一般13个字节），MP4DecSpecificDescr，每层的结构都类似如下：
~~~
typedef esdsStruct{
uint8_t tag;
<不定长，最长4字节> size;
uint8_t[size] data;
}esdsStruct；
~~~
下面看具体数据：
~~~
00 00 00 33 65 73 64 73 00 00 00 00 03 80 80 80 ; ...3esds.....
22 00 00 00 04 80 80 80 14 40 15 00 18 00 00 01 ; ".....@......
F4 00 00 01 F4 00 05 80 80 80 02 11 88 06 80 80 ; ...........
80 01 02                                        ; ..

==============  解析  ==============
00 00 00 33: size 51
65 73 64 73: esds
00 00 00 00: version 和 flag 均为 0
03: ES_DescrTag
80 80 80 22: 4字节size（一般就一字节），这里有点奇怪，4字节长度，但只取最后一个0x22当长度
00 00：ES_ID
00: 
	:0000 0000(bits)
     0:		steamDependenceFlag，如果为1，则有16bits的dependsOn_ES_IS
      0:		URL_Flag,如果为1，后边则有8bits URLlength, 和相应的URLstring(URLlength)
       0:		OCRstreamFlag, 如果为1，有16bits OCR_ES_id;
        0 0000: streamPriority
        
04: DecoderConfigDescriptor TAG
80 80 80 14: 4字节size（一般就一字节），这里有点奇怪，4字节长度，但只取最后一个0x14(20)当长度
40: objectTypeIndication (14496-1 Table8), 0x40是Audio (ISO/IEC 14496-3)
15:
	:0001 0101
     0001 01        :streamType  5是Audio Stream, 14496-1 Table9
            0       :upStream
             1      :reserved
00 18 00: bufferSizeDB: 6144
01 F4 00: Max bitrate 128000  //可以获取最大码率
01 F4 00: Avg bitrate 128000  //可以获取平均码率

05: DecSpecificInfotag
80 80 80 02: 4字节size（一般就一字节），这里有点奇怪，4字节长度，但只取最后一个0x02当长度
11 88:
	0001 0001 1000 1000:
    0001 0                     :audioObjectType 2 GASpecificConfig
          001 1                :samplingFrequencyIndex
               000 1		   :channelConfiguration 1 
                    00		   :cpConfig
                      0		   :directMapping
                      
06: SLConfigDescrTag
80 80 80 01: 长度1
02: predefined 0x02 Reserved for use in MP4 files

~~~

#### 2、avc1
看数据：
~~~
00 00 00 C1 61 76 63 31 00 00 00 00 00 00 00 01 ; ....avc1........
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ; ................
05 12 03 C6 00 48 00 00 00 48 00 00 00 00 00 00 ; .....H...H......
00 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ; ................
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ; ................
00 00 00 18 FF FF 00 00 00 34 61 76 63 43 01 4D ; .........4avcC.M
00 20 FF E1 00 1D 27 4D 00 20 89 8B 60 29 03 DF ; . ....'M. ..`)..
11 36 02 D4 04 04 06 93 03 00 05 DC 00 01 77 05 ; .6............w.
EF 7C 14 01 00 04 28 EE 1F 20                                 

==============  解析  ==============
00 00 00 C1: size 193
61 76 63 31: avc1
00 00 00 00 00 00: reserve
00 01: index 1
00 00: pre_defined
00 00: reserved
00 00 00 00: pre_defined
00 00 00 00: pre_defined
00 00 00 00: pre_defined
05 12: width 1298 
03 C6: height 966
00 48 00 00: Horiz resolution 4718592 
00 48 00 00: Ver resolution   4718592 
00 00 00 00: reverse
00 01: frame count 0
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00: 32字节compressr_name
00 18：bit_depth
FF FF：pre_defined

00 00 00 34： size 52
61 76 63 43: avcC
01: version
4D: AVC profile indication(77, Main profile)
00: AVC profile compatibility
20: AVC level indication 32
FF: nulu size(4 暂时不知道怎么来的)
E1: SPS个数，低五位有效，1个sps
00 1D: sps长度，29
27 4D 00 20 89 8B 60 29 03 DF 11 36 02 D4 04 04 06 93 03 00 05 DC 00 01 77 05 EF 7C 14： sps
01: pps个数
00 04： pps长度
28 EE 1F 20： pps数据
~~~


### stts(Decoding Time to Sample Box)
“stts”存储了sample的duration，描述了sample时序的映射方法，我们通过它可以找到任何时间的sample。“stts”可以包含一个压缩的表来映射时间和sample序号，用其他的表来提供每个sample的长度和指针。表中每个条目提供了在同一个时间偏移量里面连续的sample序号，以及samples的偏移量。递增这些偏移量，就可以建立一个完整的time to sample表。

直接看数据:
~~~
00 00 00 18 73 74 74 73 00 00 00 00 00 00 00 01 ; ....stts........
00 00 00 8C 00 00 01 90                         ; ........

==============  解析  ==============
00 00 00 18: size 24
73 74 74 73: stts
00 00 00 00: version 和 flag
00 00 00 01: Entry count 1
00 00 00 8C: sample count 140
00 00 01 90: Sample delta 400

* 在mvhd的timescale为6000，这里sample delta为400，6000/400=15,即1秒15帧

* stts只有一个entry,sample count140,delta 400, 140*400=56000, 与mvhd的duration相对应
~~~

### ctts(Composition Time to Sample Box)
这个box是用来重新构建因为h264 B帧存在而导致的pts dts不同的问题。  
例如sample的解码顺序是这样的：
~~~
I P B  P B  ...  
~~~
但是它的显示顺序却是这样的：
~~~
I B P B P ... 
~~~
当带有B帧的Nalus流封装后，再次解码显示，此时PTS 和 DTS 不能一一对应，因为B帧的时间戳小于P帧，此时CTS 可以记录这个偏差，用以回复解码的时间戳。
 ctts = CTS-DTS  
 
 语法：
 ~~~
 class CompositionOffsetBox
    extends FullBox(‘ctts’, version = 0, 0) {
    unsigned int(32) entry_count;
    int i;
    if (version==0) {
        for (i=0; i < entry_count; i++) {
            unsigned int(32) sample_count;
            unsigned int(32) sample_offset;
        }
    }
    else if (version == 1) {
        for (i=0; i < entry_count; i++) {
            unsigned int(32) sample_count;
            signed int(32) sample_offset;
        }
    }
}
 ~~~
 
 看一下解析数据：
 ~~~
 00 00 04 70 63 74 74 73 00 00 00 00 00 00 00 8C ; ...pctts........
00 00 00 01 00 00 01 90 00 00 00 01 00 00 03 20 ; ............... 
00 00 00 01 00 00 00 00 00 00 00 01 00 00 03 20 ; ............... 
00 00 00 01 00 00 00 00 00 00 00 01 00 00 03 20 ; ............... 
00 00 00 01 00 00 00 00 00 00 00 01 00 00 03 20 ; ............... 
00 00 00 01 00 00 00 00 00 00 00 01 00 00 03 20 ; ............... 
00 00 00 01 00 00 00 00 00 00 00 01 00 00 03 20 ; ............... 
00 00 00 01 00 00 00 00 00 00 00 01 00 00 03 20 ; ............... 
00 00 00 01 00 00 00 00 00 00 00 01 00 00 01 90 ; ................
00 00 00 01 00 00 03 20 00 00 00 01 00 00 00 00 ; ....... ........
00 00 00 01 00 00 03 20 00 00 00 01 00 00 00 00 ; ....... ........

==============  解析  ==============
00 00 04 70: size 1136
63 74 74 73: ctts
00 00 00 00: version flag 0
00 00 00 8C: entry_count 140

entry datas:
00 00 00 01: Sample count 1
00 00 01 90: Sample offset 400

00 00 00 01: Sample count 1
00 00 03 20: Sample offset 800

00 00 00 01: Sample count 1
00 00 00 00: Sample offset 0

.... 循环
 ~~~
 由于edts->elst 的media time 为400，该track整体延迟400播放
 看一下示意图：
 ![](ctts.png)

### stss(Sync Sample Box)
 “stss”确定media中的关键帧。对于压缩媒体数据，关键帧是一系列压缩序列的开始帧，其解压缩时不依赖以前的帧，而后续帧的解压缩将依赖于这个关键帧。“stss”可以非常紧凑的标记媒体内的随机存取点，它包含一个sample序号表，表内的每一项严格按照sample的序号排列，说明了媒体中的哪一个sample是关键帧。如果此表不存在，说明每一个sample都是一个关键帧，是一个随机存取点。->关键帧序号
 
 
看一下解析数据：
~~~
00 00 00 38 73 74 73 73 00 00 00 00 00 00 00 0A ; ...8stss........
00 00 00 01 00 00 00 10 00 00 00 1F 00 00 00 2E ; ................
00 00 00 3D 00 00 00 4C 00 00 00 5B 00 00 00 6A ; ...=...L...[...j
00 00 00 79 00 00 00 88                         ; ...y....

==============  解析  ==============
00 00 00 38: size 56
73 74 73 73: stss
00 00 00 00: version flag
00 00 00 0A: entry count 10,有10个关键帧
00 00 00 01: 第一个关键帧位于第1帧
00 00 00 10: 第二个关键帧位于第16帧
00 00 00 1F：第三个关键帧位于第31帧
。。。。。
00 00 00 88：第十个关键帧位于第136帧
~~~
 
 
 ### sdtp(Independent and Disposable Samples Box)
语法：
~~~
aligned(8) class SampleDependencyTypeBox
   extends FullBox(‘sdtp’, version = 0, 0) {
   for (i=0; i < sample_count; i++){
        unsigned int(2) is_leading;
        unsigned int(2) sample_depends_on;
        unsigned int(2) sample_is_depended_on;
        unsigned int(2) sample_has_redundancy;
    } 
}

说明： 
* is_leading(可不管): takes one of the following four values: 
0: the leading nature of this sample is unknown;(
该样本的主要性质未知;)
1: this sample is a leading sample that has a dependency before the referenced I‐picture (and is therefore not decodable);(
此sample是一个主要样本，它在引用的I帧之前具有依赖性（因此不可解码）;)
2: this sample is not a leading sample;（此样本不是主要样本）
3: this sample is a leading sample that has no dependency before the referenced I‐picture (and is therefore decodable);（
此sample是一个主要示例，在引用的I帧之前没有依赖关系（因此可解码））

* sample_depends_on：takes one of the following four values:
0: the dependency of this sample is unknown;（未知依赖关系）
1: this sample does depend on others (not an I picture)（依赖其他帧，非I帧）; 
2: this sample does not depend on others (I picture)（I帧）;
3: reserved（保留）

* sample_is_depended_on takes one of the following four values: 
0: the dependency of other samples on this sample is unknown;（是否有其他帧依赖该帧未知） 
1: other samples may depend on this one (not disposable)（可能有其他帧依赖该帧）;
2: no other sample depends on this one (disposable)（没有其他帧依赖该帧）;
3: reserved

sample_has_redundancy takes one of the following four values:
0: it is unknown whether there is redundant coding in this sample（尚不清楚该sample中是否存在冗余编码）;
1: there is redundant coding in this sample（该sample有冗余编码）;
2: there is no redundant coding in this sample（没有冗余编码）;
3: reserved

~~~

看数据：
~~~
00 00 00 98 73 64 74 70 00 00 00 00 20 10 18 10 ; ....sdtp.... ...
18 10 18 10 18 10 18 10 18 10 18 20 10 18 10 18 ; ........... ....
10 18 10 18 10 18 10 18 10 18 20 10 18 10 18 10 ; .......... .....
18 10 18 10 18 10 18 10 18 20 10 18 10 18 10 18 ; ......... ......
10 18 10 18 10 18 10 18 20 10 18 10 18 10 18 10 ; ........ .......
18 10 18 10 18 10 18 20 10 18 10 18 10 18 10 18 ; ....... ........
10 18 10 18 10 18 20 10 18 10 18 10 18 10 18 10 ; ...... .........
18 10 18 10 18 20 10 18 10 18 10 18 10 18 10 18 ; ..... ..........
10 18 10 18 20 10 18 10 18 10 18 10 18 10 18 10 ; .... ...........
18 10 18 20 10 18 10 18                         ; ... ....

==============  解析  ==============
00 00 00 98: size 152
73 64 74 70: sdtp 
00 00 00 00: version flag

20:
	0010 0000:
   	00			:is_leading 0
      10		:sample_depends_on 2 I帧
         00		:sample_is_depended_on 0
           00	:sample_has_redundancy

10:
	0001 0000:
    00			:is_leading 0
   	  01		:sample_depends_on 1 非I帧
 		 00		:sample_is_depended_on 0
           00	:sample_has_redundancy
           
18:
	0001 1000:
    00			:is_leading 0
   	  01		:sample_depends_on 1 非I帧
         10		:sample_is_depended_on 2 没有其他帧依赖该帧 B帧
           00	:sample_has_redundancy

...

~~~



### stsc(Sample To Chunk Box)
    用chunk组织sample可以方便优化数据获取，一个thunk包含一个或多个sample。“stsc”中用一个表描述了sample与chunk的映射关系，查看这张表就可以找到包含指定sample的thunk，从而找到这个sample。  
可以参考：https://www.cnblogs.com/shakin/p/8543719.html  
语法：
~~~
aligned(8) class SampleToChunkBox
extends FullBox(‘stsc’, version = 0, 0) { 
    unsigned int(32) entry_count;
    for (i=1; i u entry_count; i++) {
    unsigned int(32) first_chunk;
    unsigned int(32) samples_per_chunk; 
    unsigned int(32) sample_description_index;
    } 
}

关键点在于他们里面的三个字段: first_chunk,samples_per_chunk,sample_description_index。

first_chunk: 每一个 entry 开始的 chunk 位置。
samples_per_chunk: 每一个 chunk 里面包含多少的 sample
sample_description_index: 每一个 sample 的描述。一般可以默认设置为 1。

这 3 个字段实际上决定了一个 MP4 中有多少个 chunks，每个 chunks 有多少个 samples。这里顺便普及一下 chunk 和 sample 的相关概念。在 MP4 文件中，最小的基本单位是 Chunk 而不是 Sample。

sample: 包含最小单元数据的 slice。里面有实际的 NAL 数据。
chunk: 里面包含的是一个一个的 sample。为了是优化数据的读取，让 I/O 更有效率。
~~~

现在看一下具体数据的解析:
~~~
00 00 00 DC 73 74 73 63 00 00 00 00 00 00 00 11 ; ....stsc........
00 00 00 01 00 00 00 08 00 00 00 01 00 00 00 02 ; ................
00 00 00 07 00 00 00 01 00 00 00 03 00 00 00 08 ; ................
00 00 00 01 00 00 00 04 00 00 00 07 00 00 00 01 ; ................
00 00 00 05 00 00 00 08 00 00 00 01 00 00 00 06 ; ................
00 00 00 07 00 00 00 01 00 00 00 07 00 00 00 08 ; ................
00 00 00 01 00 00 00 08 00 00 00 07 00 00 00 01 ; ................
00 00 00 09 00 00 00 08 00 00 00 01 00 00 00 0A ; ................
00 00 00 07 00 00 00 01 00 00 00 0B 00 00 00 08 ; ................
00 00 00 01 00 00 00 0C 00 00 00 07 00 00 00 01 ; ................
00 00 00 0D 00 00 00 08 00 00 00 01 00 00 00 0E ; ................
00 00 00 07 00 00 00 01 00 00 00 0F 00 00 00 08 ; ................
00 00 00 01 00 00 00 10 00 00 00 0F 00 00 00 01 ; ................
00 00 00 11 00 00 00 0C 00 00 00 01             ; ............

==============  解析  ==============
00 00 00 DC: size 220
73 74 73 63: stsc
00 00 00 00: version flag 0
00 00 00 11: entry count 17

00 00 00 01: first_chunk 1
00 00 00 08: samples_per_chunk 8
00 00 00 01: sample_description_index 1

00 00 00 02: first_chunk 2
00 00 00 07: samples_per_chunk 7
00 00 00 01: sample_description_index 1
......
~~~

学会了解析数据，下面通过几个例子来看看解析出来的数据如何使用：
![](stsc_1.png)
~~~
第一个entry数据： 从第一个chunk开始，每个chunk包含8个sample,下一个entry从第二个chunk开始 -> 1-8 sample
第二个entry数据： 从第二个chunk开始，每个chunk包含7个sample，下一个entry从第三个chunk开始 -> 9-15 sample
第三个entry数据： 从第三个chunk开始，每个chunk包含8个sample，下一个entry从第四个chunk开始 -> 16-24 sample
。。。。。。
~~~

![](stsc_2.png)
~~~
第一个entry数据： 从第一个chunk开始，每个chunk包含1个sample,下一个entry从第6960个chunk开始 -> 第1个chunk到第6959个chunk,每个chunk包含一个sample,1-6959 sample
第二个entry数据： 从第6960个chunk开始，每个chunk包含2个sample -> 6960-6961 sample
~~~

![](stsc_3.png)
~~~
第一个entry数据： 从第1个chunk开始，每个chunk包含5个sample,下一个entry从第2个chunk开始 -> 1-5 sample
第二个entry数据： 从第2个chunk开始，每个chunk包含4个sample,下一个entry从第3个chunk开始 -> 6-9 sample 
第三个entry数据： 从第3个chunk开始，每个chunk包含5个sample,下一个entry从第36个chunk开始 -> 第3个chunk到第35个chunk,每个chunk包含5个sample,10-169 sample
第四个entry数据： 从第36个chunk开始，每个chunk包含4个sample,下一个entry从第37个chunk开始 -> 170-173 sample
第五个entry数据： 从第37个chunk开始，每个chunk包含5个sample,下一个entry从第69个chunk开始 -> 第37个chunk到第68个chunk,每个chunk包含5个sample,174-333 sample
。。。。。。。
~~~
    
### stsz(Sample Size Boxes)
“stsz” 定义了每个sample的大小，包含了媒体中全部sample的数目和一张给出每个sample大小的表。这个box相对来说体积是比较大的。  
语法：
~~~
aligned(8) class SampleSizeBox extends FullBox(‘stsz’, version = 0, 0) {
   unsigned int(32)  sample_size;
   unsigned int(32)  sample_count;
   if (sample_size==0) {
      for (i=1; i <= sample_count; i++) {
      unsigned int(32)  entry_size;
      }
} }

~~~

数据解析：
~~~
00 00 02 44 73 74 73 7A 00 00 00 00 00 00 00 00 ; ...Dstsz........
00 00 00 8C 00 04 CF BA 00 00 C9 D9 00 00 3C 31 ; ..............<1
00 00 B0 96 00 00 26 5C 00 00 A7 FA 00 00 44 5C ; ......&\......D\
00 00 9C DD 00 00 41 26 00 00 B3 22 00 00 44 04 ; ......A&..."..D.
00 00 C1 69 00 00 53 1D 00 00 9E 1D 00 00 51 D1 ; ...i..S.......Q.
00 04 C3 05 00 00 C8 63 00 00 51 F5 00 00 AE AA ; .......c..Q.....
00 00 55 50 00 00 B6 3A 00 00 55 C8 00 00 A6 B8 ; ..UP...:..U.....
00 00 50 17 00 00 A6 AA 00 00 57 D8 00 00 BC 9F ; ..P.......W.....
00 00 63 1F 00 00 C4 CA 00 00 64 3F 00 04 C5 E5 ; ..c.......d?....
00 00 E8 6C 00 00 6B 9A 00 00 C0 F1 00 00 67 54 ; ...l..k.......gT
。。。。。。。。。。。。。。


==============  解析  ==============
00 00 02 44: size 580 
73 74 73 7A: stsz
00 00 00 00: version flag
00 00 00 00: Sample size 0
00 00 00 8C: Sample count 140 视频track可用于确定视频帧数

00 04 CF BA: 第一个sample(帧)大小 315322
00 00 C9 D9: 第二个sample(帧)大小 51673
00 00 3C 31: 第三个sample(帧)大小 15409
00 00 B0 96: 第四个sample(帧)大小 45206
。。。。。。。。。。。
~~~

### stco(Chunk Offset Box)
“stco”定义了每个thunk在媒体流中的位置。位置有两种可能，32位的和64位的，后者对非常大的电影很有用。在一个表中只会有一种可能，这个位置是在整个文件中的，而不是在任何box中的，这样做就可以直接在文件中找到媒体数据，而不用解释box。需要注意的是一旦前面的box有了任何改变，这张表都要重新建立，因为位置信息已经改变了。  
数据解析：
~~~
00 00 00 54 73 74 63 6F 00 00 00 00 00 00 00 11 ; ...Tstco........
00 00 7F 0C 00 08 D5 94 00 0C 31 91 00 14 E6 30 ; .........1....0
00 18 9C C3 00 21 93 51 00 25 B2 61 00 2E 7D 04 ; .....!.Q.%.a..}.
00 31 C0 3C 00 3A 7F B6 00 3D E2 FE 00 46 D2 76 ; .1.<.:..=...F.v
00 4A 46 3F 00 53 49 9C 00 56 A9 39 00 5F C4 5E ; .JF?.SI..V.9._.^
00 6C 3D FE                                     ; .l=.

==============  解析  ==============
00 00 00 54: size 84
73 74 63 6F: stco
00 00 00 00: version flag
00 00 00 11: entry count 17,有17个chunk

00 00 7F 0C: 第一个Chunk offset 32524，与stsc(chunk包含sample的个数)和stsz(每个sample的大小)一起，可以找到具体序号的sample的位置和大小，用于读取数据
00 08 D5 94: 第二个Chunk offset 578964
00 0C 31 91: 第三个Chunk offset 799121
。。。。。。。。。

* 检验
1、由stsc知道，第一个chunk包含8个sample
2、由stsz知道，前8个sample的大小分别为：315322，51673，15409，45206，9820，43002，17500，40157
3、由stco知道，第一个chunk的起始为置为32524，加上前8个sample大小后为570613，小于578964，查看音频track，可找到chuck offset起始地址为570613。说明音视频数据在文件中，以chunk为单位，交错分布。
~~~


