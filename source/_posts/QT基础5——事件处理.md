title: QT基础5——事件处理
author: Cyrus
tags: []
categories:
  - QT
date: 2018-12-20 22:15:00
---
QT的事件是由事件基类QEvent派生出来的，而控件的事件是从QObject和QWidget继续来的虚函数，可以重写实现。

### 控件事件
示例代码：
```
//.h文件
#ifndef EVENTLABEL_H
#define EVENTLABEL_H

#include <QObject>
#include <QLabel>
#include <QWidget>

class EventLabel : public QLabel
{
    Q_OBJECT
public:
    explicit EventLabel(QWidget *parent = nullptr);

protected:
	//鼠标按下事件
    void mousePressEvent(QMouseEvent *ev);
	//鼠标移动事件
    void mouseMoveEvent(QMouseEvent *event);
	//鼠标释放事件
    void mouseReleaseEvent(QMouseEvent *event);
	//鼠标离开控件事件 virtual关键字可写可不写
    virtual void leaveEvent(QEvent *event);
	//鼠标进入控件事件
    virtual void enterEvent(QEvent *event);

signals:

public slots:
};

#endif // EVENTLABEL_H



.cpp 文件
#include "eventlabel.h"
#include <QMouseEvent>
#include <QDebug>

EventLabel::EventLabel(QWidget *parent) : QLabel(parent)
{

}

void EventLabel::mousePressEvent(QMouseEvent *ev)
{
    int i = ev->x();
    int j = ev->y();

    if (ev->button() == Qt::LeftButton) {
        qDebug() << "left";
    } else if (ev->button() == Qt::RightButton) {
        qDebug() << "Right";
    } else if (ev->button() == Qt::MidButton) {
        qDebug() << "Mid";
    }

    QString text = QString("<center><h1>Mouse Press: (%1, %2)</h1></center>").arg(i).arg(j);
    this->setText(text);
}

void EventLabel::mouseReleaseEvent(QMouseEvent *ev) {
    QString text = QString("<center><h1>Mouse Release: (%1, %2)</h1></center>").arg(ev->x()).arg(ev->y());
    this->setText(text);

}

void EventLabel::mouseMoveEvent(QMouseEvent *ev)
{
    QString text = QString("<center><h1>Mouse Move: (%1, %2)</h1></center>").arg(ev->x()).arg(ev->y());
    this->setText(text);
}

void EventLabel::leaveEvent(QEvent *event)
{
    QString text = QString("<center><h1>leave</h1></center>");
    this->setText(text);
}

void EventLabel::enterEvent(QEvent *event)
{
    QString text = QString("<center><h1>enter</h1></center>");
    this->setText(text);
}
```

### 键盘事件
```
.h文件
protected:
    void keyPressEvent(QKeyEvent *);
    
.cpp 文件
void MainWindow::keyPressEvent(QKeyEvent *e)
{
	//QKeyEvent 键盘按键事件
    //e->key() 被下的按键
    //Qt::Key_x   键盘按键的枚举值
    qDebug() << (char)e->key();
    if (e->key() == Qt::Key_A)
    {
        qDebug() << "Qt::Key_A";
    }
}
```

### 定时器事件
```
.h 文件
protected:
	void timerEvent(QTimerEvent *e);
    
private:
	int timerID;
    
    
.cpp 文件
//启动定时器,单位为ms
timerID = this->startTimer(1000);

//定时器处理函数，所有定时器事件都通过些函数处理，不像OC的NSTimer单独指定
void MainWindow::timerEvent(QTimerEvent *e)
{
	//e->timerId()获取触发事件的定时器
    if (timerID == e->timerId()) {  
        static int sec = 0;
        sec++;

        ui->label->setText(QString("<center><h1>timer out: %1</h1></center>").arg(sec));

        if (sec == 5) {
        	//关闭定时器
            killTimer(timerID);
        }
    }
}
```

### 事件的接收和忽略
```
//e->accept() 接收事件，即处理事件
//e->ignore() 忽略事件，不处理
void MainWindow::closeEvent(QCloseEvent *e) {
    int ret = QMessageBox::question(this, "question", "是否关闭窗口");
    if(ret == QMessageBox::Yes) {
        //接收事件，执行事件， 事件不再传递，关闭事件->关闭窗口
        e->accept();
    } else {
        //忽略事件，事件传递给父组件直到找到事件接收者或者无父组件丢弃
        e->ignore();
    }
}
```

### 事件分发
```

  class MyClass : public QWidget
  {
      Q_OBJECT

  public:
      MyClass(QWidget *parent = 0);
      ~MyClass();

		//事件处理函数 返回true停止事件传递， false 继续事件传递
      bool event(QEvent* ev) override
      {	
      	//判断事件类型，分发给相应的处理函数
          if (ev->type() == QEvent::PolishRequest) {
              // overwrite handling of PolishRequest if any
              doThings();
              return true;
          } else  if (ev->type() == QEvent::Show) {
              // complement handling of Show if any
              doThings2();
              //调用父类处理函数，即进行默认处理
              QWidget::event(ev);
              return true;
          }
          // Make sure the rest of events are handled
          //调用父类处理函数，即进行默认处理
          return QWidget::event(ev);
      }
  };
```


### 事件过滤器
```
  class MainWindow : public QMainWindow
  {
  public:
      MainWindow();

  protected:
  //1、重写事件过滤函数
      bool eventFilter(QObject *obj, QEvent *ev) override;

  private:
      QTextEdit *textEdit;
  };

  MainWindow::MainWindow()
  {
      textEdit = new QTextEdit;
      setCentralWidget(textEdit);
      
	//2、控件初始化过滤
      textEdit->installEventFilter(this);
  }

//3、具体事件处理
  bool MainWindow::eventFilter(QObject *obj, QEvent *event)
  {
      if (obj == textEdit) {
          if (event->type() == QEvent::KeyPress) {
              QKeyEvent *keyEvent = static_cast<QKeyEvent*>(event);
              qDebug() << "Ate key press" << keyEvent->key();
              return true;
          } else {
              return false;
          }
      } else {
          // pass the event on to the parent class
          return QMainWindow::eventFilter(obj, event);
      }
  }
```