title: QT程序windows环境打包发布
author: Cyrus
tags:
  - 打包发布
categories:
  - QT
date: 2019-10-10 00:02:00
---
参考链接：
* windeployqt生成依赖库：https://blog.csdn.net/windsnow1/article/details/78004265
* NSIS生成安装包：https://blog.csdn.net/m1223853767/article/details/79702064

### 一、依赖库生成
* 1、选择release环境，编辑，会生成release文件夹和.exe文件
* 2、如果有第三方动态库，将动态库拷贝到编译生成的release，运行程序(这一步可以省略)
![](winqt_1.png)
* 3、将.exe文件拷贝出来，找到QT用的编辑器（这里是QT 5.12.5(MinGw7.3.0 64bit)),执行下面的代码：
~~~
cd xxxxx\xxx\	//进入.exe所在的文件夹
windeployqt xxx.exe
~~~
![](winqt_2.png)
![](winqt_3.png)
* 4、将第三方运态库文件拷到.exe同级目录。完成！

这里说一下容易遇到的问题：
* 1、在release环境编译后，到release文件夹运行程序，提示缺少.dll文件  
百度及部分视频讲到qt安装目录下拷贝对应的.dll文件。这种方法在安装qt的电脑上可以运行，但是在没安装qt的电脑上还是会提示缺少其他底层的.dll文件。一个一个添加效率太低。
![编译后提示缺少库](error_1.png)
![在未安装qt的电脑上提示缺少底层库](error_2.png)

* 2、出现0xc000007b——应用程序无法正常启动  
这个问题主要是编译器选的不对，例如我们要生成MinGW64位的程序，在QT中却使用了clang x86的编译器。查看下面两个位置设置是否对应：
![](winqt_4.png)
![](winqt_2.png)


### 二、NSIS生成安装包
使用NSIS生成安装包，我们主要会用到两个软件：
* 1、NSIS，NSIS是一个基于脚本的打包软件，提供了安装、卸载、系统设置、文件解压缩等功能。但是脚本编写麻烦，所以又有了第二个软件NSIS Edit.
* 2、NSIS Edit，顾名思义，就是NSIS编辑器，可以通过可视化界面的方式配置脚本并编译生成安装包。

具体的使用过程 [NSIS制作软件安装包](https://blog.csdn.net/m1223853767/article/details/79702064) 这篇文篇已经讲得非常详细,这里就不再重复，主要讲几个需要注意的地方。

* 1、NSIS Edit 新建时，选择“文件”菜单中的 “新建脚本：向导“；
* 2、步骤4，授权文件，如果没有就删掉，不然编译会出错；
* 3、步骤5，先删除默认的两个示例文件。如果包含子文件夹，使用AddTreeDir添加。使用添加文件的话，文件夹不会添加，生成的安装包会缺少库。
![](winqt_5.png)