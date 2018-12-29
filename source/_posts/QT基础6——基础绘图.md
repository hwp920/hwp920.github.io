title: QT基础6——基础绘图
author: Cyrus
tags: []
categories:
  - QT
date: 2018-12-29 15:40:00
---
#### 1、重写绘图事件
```
.h文件
protected:
    //1、重写绘图事件，虚函数
    //2、如果在窗口绘图，必须放在绘图事件里实现
    //3、绘图事件内部自动调用，窗口需要重绘的时候（状态改变，如缩放、鼠标点击之类）
    void paintEvent(QPaintEvent *);
```

#### 2、绘图事件中调用相关函数
绘图功能需要用到QPainter对象，QPainter有两种创建方式
* QPainter(QPaintDevice *device)，QWidget继承于QObject和QPaintDevice 所以所有视图均为QPaintDevice子类，可以写为QPainter p(this);

* QPainter p1;绘图代码写于p1.begin(this);和p1.end();之间

```
void Widget::paintEvent(QPaintEvent *)
{
    //QPaintDevice,widget多继承于QObject和QPaintDevice
    QPainter p(this);

    //创建画图对象
    //指定绘图对象的绘图设备
//    QPainter p1;
//    p1.begin(this);

	//画图片
//    p.drawPixmap(0, 0, width(), height(), QPixmap("../Image/bg.png"));

    //绘笔
    QPen pen;
    //设置常用颜色
//    pen.setColor(Qt::red);
	//	设置线宽
    pen.setWidth(5);
    //以rgba设置颜色
    pen.setColor(QColor(12, 34, 255, 255));
    //设置绘笔风格
    pen.setStyle(Qt::DashDotLine);
	
    //将绘笔赋予painter
    p.setPen(pen);
    //画线
    p.drawLine(50, 50, 50, 150);
    //画矩形
    p.drawRect(20, 30, 110, 200);
    //画椭圆
	p.drawEllipse(QPoint(150, 150), 60, 100);
    
    //笔刷
    QBrush  brush;
    //设置常用颜色
    brush.setColor(Qt::red);
    //设置笔刷风格
    brush.setStyle(Qt::Dense4Pattern);
    //将笔刷赋予painter
    p.setBrush(brush);
    p.drawEllipse(QPoint(150, 150), 60, 100);

//    p1.end();
}
```