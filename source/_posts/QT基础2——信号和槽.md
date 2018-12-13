title: QT基础2——信号和槽
author: Cyrus
tags: []
categories:
  - QT
date: 2018-12-13 22:17:00
---
信号和槽是QT中用于对象间通信的一种机制，也是QT的核心机制。在GUI编程中，我们经常需要在改变一个组件的同时，通知另一个组件做出响应。

### 一、信号连接函数
```
//信号连接函数
QMetaObject::Connection QObject::connect(const QObject *sender, const char *signal, const QObject *receiver, const char *method, Qt::ConnectionType type = Qt::AutoConnection)

示例：
//btn为成员变量
	btn.setParent(this);
    btn.setText("hehe");
    btn.move(50, 50);
    connect(&btn, &QPushButton::pressed, this, &MyWidget::close);
    
    /*
    * QObject *sender, 即&btn，信号发出者,指针类型
    * const char *signal，即&QPushButton::pressed，处理的信号， &发送者类名::信号名
    * QObject *receiver，即this(这里是父视图，也可以是其他对象），信号接收者
    * const char *method，即&MyWidget::close，槽函数，&接收的类名::槽函数名字
    */
    
对比iOS的UIControl(UIButton的父类）添加事件的方法：
- (void)addTarget:(nullable id)target action:(SEL)action forControlEvents:(UIControlEvents)controlEvents;

示例：
UIButton *button = [UIButton buttonWithType:UIButtonTypeCustom];
    [button addTarget:self action:@selector(close) forControlEvents:UIControlEventTouchUpInside];
    
对比一下：
&btn 对应 button  -->事件产生者
&QPushButton::pressed 对应 UIControlEventTouchUpInside  -->事件的类型
this 对应  self   -->事件的接收者
&MyWidget::close 对应  @selector(close)  -->事件的响应函数
而且，事件类型和响应函数只需要字符数组类型，这说明槽函数有着类似iOS运行时的实现机制，可以通过函数名找到相应的函数实现。
```

### 二、自定义信号函数和槽函数
在我们的.h文件中:
```
#ifndef MYWIDGET_H
#define MYWIDGET_H

#include <QWidget>
#include <QPushButton>

class MyWidget : public QWidget
{
    Q_OBJECT
public:
    explicit MyWidget(QWidget *parent = nullptr);

//这里写信号函数
   /* 信号必须有signals关键字声明
    * 信号没有返回值，但可以有参数
    * 信号就是函数的声明，只需要声明，无需实现
    * 使用: emit  函数() 发送信号   
    */
signals:
	void mySignal();	//对应信息发送 emit mySignal();
    void mySignal1(int, QString);	//对应信息发送 emit mySignal1(123， “123”);
    
//这里写槽函数
	/* 槽函数特点：
	 * 可以为任意的成员函数，普通全局函数，静态函数
	 * 槽函数需要和信号一致（参数，返回值 ）
	 * 信号都没有返回值，所以，槽函数一定没有返回值 
    */
public slots:
    void mySlot();
    void mySlot1(int, QString);

private:
    QPushButton btn;
};

#endif // MYWIDGET_H
```

在.m文件中
```
#include "mywidget.h"
#include <QLabel>
#include <QDebug>

MyWidget::MyWidget(QWidget *parent) : QWidget(parent)
{
    btn.setParent(this);
    btn.setText("hehe");
    btn.move(50, 50);
    
    //自发自收，^-^
    connect(this, &MyWidget::mySignal, this, &MyWidget::mySlot);
   	connect(this, &MyWidget::mySignal1, this, &MyWidget::mySlot1);
    
    emit mySignal();
    emit mySignal1(123, "123");
}


void MyWidget::mySlot() {
    btn.setText("点击了");
}

void MyWidget::mySlot1(int a, QString str) {
    qDebug() << a << str;
}
```

### 三、信号连接函数的几种写法
```
1、QT5的写法
connect(&btn, &QPushButton::pressed, this, &MyWidget::close);

2、QT4的写法
//QT4槽函数必须有slots关键字来修饰 .h public slots:
connect(&btn, SIGNAL(released()), this, SLOT(mySlot()));

3、Lambda 表达式的写法
connect(&btn, &QPushButton::released,
            [=]() {
            btn.setText("456");
            qDebug() << "123";
                  }
    );
    
```

重点讲一下关于Lambda表达式的写法：
* C++11增加的新特性 .pro文件 CONFIG += C++11
* 配合信号一起使用，类似iOS的block
* [btn，a, b]传参，将btn,a, b传入表达式
* 可以直接用【=】，将外部所有局部变量、类中的所有成员变量传入表达式
*【=】() mutable {} 可以修改变量值，相当于iOS的__block
* [this] 类中的所有成员以值传递方式, [&] 把外部所有局部变量，引用符号
* 信号带参数 【】(参数列表){},参数列表与信号相对应，如上面的mySlot1信号
```
connect(this, &MyWidget::mySignal1,
          [=](int a, QString str) {
            qDebug() << a << str;
         }
        );
//发送信号
emit mySignal1(123, "123");
```
