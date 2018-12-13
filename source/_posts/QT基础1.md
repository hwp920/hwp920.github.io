title: QT基础1
author: Cyrus
tags: []
categories:
  - QT
date: 2018-12-12 22:48:00
---
QT是一个跨平台的C++应用程序开发框架。支持Linux、Windows、Mac、iOS、Android等几乎所有的平台。
详见维基百科[https://zh.wikipedia.org/wiki/Qt]

F1:帮助文档，不熟悉QT的情况下，F1是最有用的快捷键了。

QT创建应用程序时自带三种视图：

1、QMainWindow: 用于PC端，带菜单栏

2、QDialog: 对话框 

3、QWidget: QT视图基类，相当于iOS的UIView
一般可以选用QMainWindow或QWidget。

QT文件结构
![](qt_files.png)

首先，看一下main文件
```
#include  “mywidget.h"
//应用程序类
#include <QApplication>

int main(int argc, char *argv[])
{
	//有且只有一个应用程序类的对象，配置运行环境
	QApplication a(argc, argv);
	//创建窗口
	MyWidget w;
	//显示窗口
	w.show();

	//运行程序，死循环，等待事件发生，相当于iOS的 主线程runloop
	//思路与iOS的main函数相当，只是iOS是在AppDelegate中创建界面
	return a.exec();
	}
```

.pro配置文件
```
#模块 头文件 f1 查找文档可以找到所有QT类需要添加的模块
QT += core gui

#高于4的版本，添加 QT += widgets, 为了兼容QT4
greaterThan(QT_MAJOR_VERSION, 4): QT += widgets

#应用程序的名字
TARGET = QtTest
#指定makefile的类型， app/lib库
TEMPLATE = app

#使用C++11需要添加
CONFIG += c++11

#源文件，.cpp文件，创建类后自动添加
SOURCES += \
main.cpp \
mywidget.cpp

#头文件 .h文件，创建类后自动添加
HEADERS += \
mywidget.h
```

<font color=ff0000>这里，QT += 模块，可以在使用相关类时，在类名处按F1,进入帮助文档界面查找。</font>
![](help_add_frame.png)

最后，看一下代码创建控件
```
//mywidget.h

#ifndef MYWIDGET_H
#define MYWIDGET_H

#include <QWidget>
#include <QPushButton>

class MyWidget : public QWidget
{
    Q_OBJECT
public:
    explicit MyWidget(QWidget *parent = nullptr);

signals:

public slots:
    void mySlot();

private:
	//非指针对象需要添加为成员变量
    QPushButton btn;
};

#endif // MYWIDGET_H


//mywidget.cpp
#include "mywidget.h"
#include <QLabel>


MyWidget::MyWidget(QWidget *parent) : QWidget(parent)
{
	//指针变量，用new创建
    QLabel *lb = new QLabel(this);
    //设置内容
    lb->setText(QString("hello world"));

	//成员变量
    btn.setParent(this);
    btn.setText("hehe");
    btn.move(50, 50);

	//错误的创建方法,为局部变量，父视图没有强引用控件，不显示
    QLabel lb(this);
    //设置内容
    lb.setText(QString("hello world"));
}
```

