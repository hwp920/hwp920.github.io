title: QT基础7——图片加载及保存
author: Cyrus
tags: []
categories:
  - QT
date: 2018-12-31 14:03:00
---
QT图片主要有QPixmap、QImage和QPicture三个类，他们的父类均为QPaintDevice。
* QPixmap/QBitmap: 针对屏幕进行了优化， 和平台相关, 不能对图片进行修改
* QImage: 和平台无关，可以对图片进行修改，可以在线程中绘图
* QPicture: 保存绘图的状态（二进制文件)

#### QPixmap/QBitmap
QBitmap 是QPixmap的子类，图片类型是黑白图片，可以减少图片的大小。

```
 	QPainter p(this);
	
    //把图片画到当前窗口上
    //方法一
    p.drawPixmap(0, 0, QPixmap("../Image/bg.png"));
    p.drawPixmap(300, 0, QBitmap("../Image/bg.png"));
	
    //方法二，先load加载，再drawPixmap
    QPixmap pixmap;
    pixmap.load("../Image/bg.png");
    p.drawPixmap(0, 200, pixmap);

    QBitmap bitmap;
    bitmap.load("../Image/bg.png");
    p.drawPixmap(300, 200, bitmap);
    
    
    //内存加载图片并保存（不显示到窗口上）
        QPixmap pixmap(400, 300);
    QPainter p(&pixmap);
    //通过画刷填充颜色
    p.fillRect(0, 0, 400, 300, QBrush(Qt::white));
    //直接用绘图设置填充颜色
    pixmap.fill(Qt::white);
    p.drawPixmap(0, 0, 80, 80, QPixmap("../Image/bg.png"));
    pixmap.save("../pixmap.jpg");
```


#### QImage
QImage与QPixmap使用方法基本相同，不同的地方在于可以获取和修改指定位置相素点的值
```
	//指定图片大小和像素格式
	QImage image(400, 300, QImage::Format_ARGB32);
    //加载图片
    image.load("../Image/bg.png");
    QPainter p;
    p.begin(&image);
    p.drawImage(0, 0, image);

    for(int i = 0; i <  50; i++)
    {
        for(int j = 0; j < 50; j++) {
        //将前50*50的像素点改为绿色
            image.setPixel(QPoint(i, j), qRgb(0, 255, 0));
        }
    }
	//保存图片
    image.save("../image.jpg");
    p.end();
```

#### QPicture
QPicture可以将图片保存为二进制文件（系统无法识别），可以通过QT加载打开。
```
  QPicture picture;
    QPainter p;
    p.begin(&picture);

    p.drawPixmap(0, 0, QPixmap("../Image/bg.png"));
    p.drawLine(50, 50, 150, 50);
    p.end();
    
    //必须p.end()之后才能调用save()函数
    picture.save("../pictuer.png");
    
    
    //加载保存在本地的picture文件
    QPicture pic;
    pic.load("../pictuer.png");
    QPainter p(this);
    p.drawPicture(0, 0, pic);
```


#### QImage 与 QPixmap相互转换
```
QPainter p(this);
    QPixmap pixmap;
    pixmap.load("../Image/bg.png");
	
    //pixmap转image
    QImage tempImage = pixmap.toImage();
	//image 转 pixmap
    QPixmap tempPixmap = QPixmap::fromImage(tempImage);

    p.drawPixmap(0, 0, tempPixmap);
    p.drawImage(300, 0, tempImage);
```