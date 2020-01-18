title: macox QT 集成ONVIF客户端
author: Cyrus
tags:
  - ONVIF
categories: []
date: 2020-01-18 22:37:00
---
### 一、编译openssl
上一篇编译的openssl已经包含了x86_64框架，可以直接拿过来用，如果觉得包含的框架太多，包太大，可以把脚本的第48行，*** ARCH_LIST=("armv7" "armv7s" "arm64" "i386" "x86_64") *** 改成 *** ARCH_LIST=("x86_64") ***，去掉多余的框架。

### 二、编译ffmpeg
这里选择用ffmpeg播放rtmp，具体的ffmpeg编译可以看前面的文章，要注意的是rtmp属于网络多媒体，编译的时候要把***”--disable-network“***去掉。具体的播放代码打算直接用前面kxMovie翻译版的播放器。

### 三、将onvif框架文件编译成静态库文件（可选）
如果想项目整洁一点，可以将前面生成的onvif框架文件生成静态库，导入到项目中。这里用cmake编译。
CMakeList.txt
~~~
cmake_minimum_required(VERSION 3.1.0)
PROJECT(onvifClient)

message(STATUS "${PROJECT_BINARY_DIR}")

include_directories("${PROJECT_BINARY_DIR}" ${PROJECT_BINARY_DIR}/../openssl/include)

add_library(libcrypto STATIC IMPORTED)
set_target_properties( libcrypto
        PROPERTIES
        IMPORTED_LOCATION ${PROJECT_BINARY_DIR}/../openssl/libcrypto.a )

add_library(libssl STATIC IMPORTED)
set_target_properties( libssl
        PROPERTIES
        IMPORTED_LOCATION ${PROJECT_BINARY_DIR}/../openssl/libssl.a )

file(GLOB SOURCES "*.c" )
add_library(onvifClient STATIC ${SOURCES} wsdd.nsmap)
target_link_libraries(onvifClient libcrypto libssl)
~~~

可以生成libonvifClient.a,将.h文件拷贝出来，放到include文件里，就ok了。
![](onvif_qt_1.png)

### 四、创建项目
1、.pro导入所需的qt库和第三方库
~~~
QT       += core gui multimedia concurrent

greaterThan(QT_MAJOR_VERSION, 4): QT += widgets

CONFIG += c++11

# The following define makes your compiler emit warnings if you use
# any Qt feature that has been marked deprecated (the exact warnings
# depend on your compiler). Please consult the documentation of the
# deprecated API in order to know how to port your code away from it.
DEFINES += QT_DEPRECATED_WARNINGS

# You can also make your code fail to compile if it uses deprecated APIs.
# In order to do so, uncomment the following line.
# You can also select to disable deprecated APIs only up to a certain version of Qt.
#DEFINES += QT_DISABLE_DEPRECATED_BEFORE=0x060000    # disables all the APIs deprecated before Qt 6.0.0

macx {

#INCLUDEPATH += $$PWD/libFFmpeg/include
INCLUDEPATH += $$PWD/onvifClient/include
INCLUDEPATH += $$PWD/openssl/include

LIBS += -L$$PWD/onvifClient
LIBS += -lonvifClient

LIBS += -L$$PWD/openssl
LIBS += -lcrypto  \
        -lssl

LIBS += -L /usr/lib -lbz2 -lm -lz -llzma

LIBS += -framework Foundation -framework AudioToolBox \
        -framework VideoToolBox -framework AVFoundation \
        -framework Cocoa -framework CoreMedia \
        -framework QuartzCore -framework VideoDecodeAcceleration \
        -framework Security

LIBS += -liconv
INCLUDEPATH += $$PWD/libFFmpeg/Macx/include

LIBS += -L$$PWD/libFFmpeg/Macx/lib

LIBS += -lavcodec   \
        -lavdevice  \
        -lavfilter  \
        -lavformat  \
        -lavutil    \
        -lswscale   \
        -lswresample
}

SOURCES += \
    main.cpp \
    mainwindow.cpp \
    mpaudiodevice.cpp \
    mpaudiooutput.cpp \
    mpmoviedecoder.cpp \
    mpplayer.cpp \
    mpvideoframe.cpp \
    myopenglwidget.cpp

HEADERS += \
    mainwindow.h \
    mpaudiodevice.h \
    mpaudiooutput.h \
    mpmoviedecoder.h \
    mpplayer.h \
    mpvideoframe.h \
    myopenglwidget.h

FORMS += \
    mainwindow.ui

# Default rules for deployment.
qnx: target.path = /tmp/$${TARGET}/bin
else: unix:!android: target.path = /opt/$${TARGET}/bin
!isEmpty(target.path): INSTALLS += target
~~~

mainwindow.h的代码：
~~~
#ifndef MAINWINDOW_H
#define MAINWINDOW_H

#include <QMainWindow>
#include <QVector>
#include <QQueue>

#include "OnvifControl.h"
#include "mpplayer.h"

QT_BEGIN_NAMESPACE
namespace Ui { class MainWindow; }
QT_END_NAMESPACE

class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    MainWindow(QWidget *parent = nullptr);
    ~MainWindow();

    void play();
    void pause();

    Ui::MainWindow 	*getUI();
    QStringList&    getXaddrs();

private:

public slots:

private:
    QStringList     xaddrs;	//数据源，存入搜索到的设备地址
    Ui::MainWindow  *ui;
    oc_device_t     *device;
    MPPlayer        *p_player;

};
#endif // MAINWINDOW_H
~~~

mainwindow.cpp的代码：
~~~
#include "mainwindow.h"
#include "ui_mainwindow.h"
#include <QDebug>

void cb_detect(void *userinfo, struct wsdd__ProbeMatchesType *wsdd__ProbeMatches)
{
    MainWindow *window = (MainWindow *)userinfo;

    if(wsdd__ProbeMatches != NULL) {
        for(int i = 0; i < wsdd__ProbeMatches->__sizeProbeMatch; i++) {
            struct wsdd__ProbeMatchType *probeMatch = wsdd__ProbeMatches->ProbeMatch + i;
            printf("%s\n", probeMatch->XAddrs);
            QString addr = QString(probeMatch->XAddrs);
            if(!window->getXaddrs().contains(addr)) {
                window->getXaddrs().append(addr);
            }
        }
    } else  {
        qDebug() << "no devices" << endl;
    }
    window->getUI()->listWidget->clear();
    window->getUI()->listWidget->addItems(window->getXaddrs());
}


MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
{
    ui->setupUi(this);

    setGeometry(100, 100, 1000, 600);
    setFixedSize(1000, 600);
    ui->label->setScaledContents(true);

    connect(ui->searchBtn, &QPushButton::released, [=](){
        this->getXaddrs().clear();
        this->getUI()->listWidget->clear();
        ONVIF_DetectDevice(cb_detect, (void *)this);
    });

    connect(ui->listWidget, &QListWidget::currentRowChanged, [=](int row){
        qDebug() << "current row is " << row << endl;
        if(row >= 0) ui->deviceNameLb->setText(xaddrs.at(row));
    });

    connect(ui->connectBtn, &QPushButton::released, [=](){
        QString url = ui->deviceNameLb->text();
        if(!url.startsWith("http")) return;
        if(ui->userName->text().isEmpty() || ui->password->text().isEmpty()) return ;

        device = createDevice(url.toUtf8(), ui->userName->text().toUtf8(), ui->password->text().toUtf8());
        ONVIF_GetCapabilities(device);
        ONVIF_GetProfile(device);

        char uri[128] = {0};
        ONVIF_GetStreamUri(device, 0, uri, 128);
        p_player = new MPPlayer(QString(uri), ui->label);
    });
}

MainWindow::~MainWindow()
{
    delete ui;
}

Ui::MainWindow *MainWindow::getUI()
{
    return this->ui;
}

QStringList& MainWindow::getXaddrs()
{
    return this->xaddrs;
}
~~~
![](onvif_qt_2.png)