title: Mac平台移植FFMpeg到Android
author: Cyrus
tags:
  - Android
categories:
  - FFMpeg
date: 2019-10-07 22:56:00
---
### 一、环境
1、macOS 10.13.6
2、Android Studio 3.4.1，Android SDK配置：
![](jni_1.png)

### 二、编译FFMpeg
编译方面，墙裂推荐这个脚本[ffmpeg-android-maker](https://github.com/Javernaut/ffmpeg-android-maker),使用方法：
* 1、打开ffmpeg-android-maker.sh  第111行找到**TOOLCHAIN_PATH=${ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/${HOST_TAG}**，其中ANDROID_NDK_HOME的路径，在你的~/.bash_profile中你的ndk路径配置一下就好了。我这边之前把ndk的路径配置成ANDROID_NDK_ROOT，所以这里我就把ANDROID_NDK_HOME改为ANDROID_NDK_ROOT：
~~~
cyurssiMac:~ cyrus$ cat .bash_profile 
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_221.jdk/Contents/Home
export ANDROID_NDK_ROOT=/Users/cyrus/Library/Android/sdk/ndk-bundle
export JAVA_HOME
export PATH=/Users/cyrus/development/flutter/bin:$PATH:$JAVA_HOME:$ANDROID_NDK_ROOT
~~~

ffmpeg-android-maker.sh第111行
~~~
TOOLCHAIN_PATH=${ANDROID_NDK_ROOT}/toolchains/llvm/prebuilt/${HOST_TAG}
~~~

2、进入ffmpeg-android-maker-master文件夹，运行ffmpeg-android-maker.sh
~~~
cd /{你的路径}/ffmpeg-android-maker-master
./ffmpeg-android-maker.sh
~~~
自动下载4.2.1版本的FFMpeg并编译armeabi-v7a、arm64-v8a、x86、x86_64四个平台的动态库。（如果需要其他版本可以改 FFMPEG_FALLBACK_VERSION=4.2.1 的值,编译需求可修改脚本163行开始的configure）

编译结果：
![](jni_2.png)


### 二、移植FFMpeg到Android Studio
* 1、新建一个Native C++项目
![](jni_3.png)

* 2、将编译完的.so库放到项目中，如下图：
![](jni_4.png)

* 3、修改CMakeLists.txt
~~~
cmake_minimum_required(VERSION 3.4.1)

#1.设置第三方库头文件所在位置，跟CMakeLists.txt同级，所以直接写include
include_directories(include)

#2.导入第三方动态库
add_library(avcodec SHARED IMPORTED)

set_target_properties( avcodec
        PROPERTIES
        IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/../jniLibs/${ANDROID_ABI}/libavcodec.so)
add_library(avformat SHARED IMPORTED)

set_target_properties( avformat
        PROPERTIES
        IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/../jniLibs/${ANDROID_ABI}/libavformat.so)

add_library(avutil SHARED IMPORTED)
set_target_properties( avutil
        PROPERTIES
        IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/../jniLibs/${ANDROID_ABI}/libavutil.so )

add_library(swscale SHARED IMPORTED)
set_target_properties( swscale
        PROPERTIES
        IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/../jniLibs/${ANDROID_ABI}/libswscale.so )

#3.创建动态库（自己将要生成的动态库,库名跟代码文件都可以根据需要改，这里演示就随便了）
add_library( TestJNI  SHARED native-lib.cpp)


find_library( # Sets the name of the path variable.
        log-lib

        # Specifies the name of the NDK library that
        # you want CMake to locate.
        log )


* 4.链接需要的第三方动态库
target_link_libraries( # Specifies the target library.
        TestJNI

        #the third library
        avcodec
        avformat
        avutil
        swscale

        # Links the target library to the log library
        # included in the NDK.
        ${log-lib} )
~~~

* 5、修改代码，看一下效果
~~~
// native-lib.cpp

#include <jni.h>
#include <string>

extern "C" {
#include "libavcodec/avcodec.h"
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_cyrus_ffmpegjni_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    std::string hello = avcodec_configuration();
    return env->NewStringUTF(hello.c_str());
}



// MainActivity
// CMakeLists.txt中把生成库改成TestJNI了，所以这里要改一下
    static {
        System.loadLibrary("TestJNI");
    }
~~~

效果：
![](jni_5.png)

* 6、自定义类导入FFMpeg
这里先说一个我觉得挺好的javah tool配置：
Android Studio -> Preferences ->External Tools -> "+"
~~~
Program填 		$JDKPath$/bin/javah
Arguments填		$FileClass$
Working direction填		$ModuleFileDir$/src/main/java
~~~
具体如下图：
![](jni_6.png)

创建一下类，这里就叫TestJNI
~~~
package com.cyrus.ffmpegjni;

public class TestJNI {
    public native String GetConfigString();

    static  {
        System.loadLibrary("TestJNI");
    }
}
~~~

选中TestJNI，右键External Tools -> javah,生成头文件com_cyrus_ffmpegjni_TestJNI.h，将头文件放到cpp文件夹下,新建一个TestJNI.cpp，代码：
~~~
#include "com_cyrus_ffmpegjni_TestJNI.h"
#include <string>

extern "C" {
#include "libavcodec/avcodec.h"
}

JNIEXPORT jstring JNICALL Java_com_cyrus_ffmpegjni_TestJNI_GetConfigString
        (JNIEnv *env, jobject obj)
{
    std::string hello = avcodec_configuration();
    return env->NewStringUTF(hello.c_str());
}
~~~

修改MainActivity
~~~
package com.cyrus.ffmpegjni;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
		
        //这个就是我们新建的JNI类
        TestJNI test = new TestJNI();

        // Example of a call to a native method
        TextView tv = findViewById(R.id.sample_text);
        tv.setText(test.GetConfigString());
    }

}
~~~

删掉native-lib.cpp,修改一下CMakeLists.txt
~~~
#3.创建动态库（自己将要生成的动态库,库名跟代码文件都可以根据需要改，这里演示就随便了）
// 这里的cpp改成我们自己建的TestJNI.cpp
add_library( TestJNI  SHARED TestJNI.cpp)
~~~

Build -> Clean Project, 再次运行，正常显示。