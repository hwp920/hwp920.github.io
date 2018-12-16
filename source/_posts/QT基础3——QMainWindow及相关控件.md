title: QT基础3——QMainWindow及相关控件
author: Cyrus
tags: []
categories:
  - QT
date: 2018-12-15 18:09:00
---
MainWindow 是QT创建应用程序三种界面中的一种，也是比较常用的一种。与QWidget相比，MainWindow自带了许多控件，如QMenuBar、QToolBar、QStatusBar等等。

### 菜单栏
```
//获取菜单栏
QMenuBar *mBar = menuBar();
//添加菜单
QMenu *pFile = mBar->addMenu(QString("File"));
//菜单下拉选项
QAction *pNew = pFile->addAction(QString("New"));
//选项绑定事件
connect(pNew, &QAction::triggered,
      	[=]() {
			qDebug() << "New";
      	}
);
    
//菜单添加分割线
pFile->addSeparator();

//添加另一个下拉选项和对应的事件
QAction *pOpen = pFile->addAction(QString("Open"));
connect(pOpen, &QAction::triggered,
        [=]() {
                qDebug() << "Open";
        }
);
```

### 工具栏
```
	//工具栏， 菜单栏对应的快捷方式
    QToolBar *toolBar = addToolBar("toolBar");
    //工具栏添加快捷键
    toolBar->addAction(pNew);
    toolBar->addAction(pOpen);

    QPushButton *b = new QPushButton(this);
    b->setText("*_*");
    //添加小控件
    toolBar->addWidget(b);
    connect(b, &QPushButton::clicked,
            [=](){
        b->setText("^-^");
    });
```

### 状态栏
```
    //状态栏
    QStatusBar *sbar = statusBar();
    QLabel *label = new QLabel(this);
    label->setText("long long time ago");
    sbar->addWidget(label);
    //从左往右加
    sbar->addWidget(new QLabel("hehe", this));
    //从右往左加
    sbar->addPermanentWidget(new QLabel("xixi", this));
```

### 设置核心控件和浮动窗口
```
	QTextEdit *textEdit = new QTextEdit(this);
    //将文本编辑设为核心控件
    setCentralWidget(textEdit);
    
    //浮动窗口
    QDockWidget *dock = new QDockWidget(this);
    addDockWidget(Qt::LeftDockWidgetArea, dock);
```

### 对话框
```
//一、模态对话框
//模态对话框，使用exex()调用，无法操作当前窗口，关闭窗口后才能继续运行MainWindow代码
QDialog dlg;
dlg.exec();


//二、非模态对话框
//非模态对话框，使用show()调用，不会阻塞MainWindow代码

错误写法如下：对话框一闪而过
QDialog dlg;
dlg.show();

修正写法：
1、使用new创建对象，缺点是不能主动释放，生成太多对话框的话内存高
QDialog *dlg = new QDialog(this);
dlg->show();

2、将QDialog设为成员变量，直接调用show()

3、将QDialog属性设置为关闭时释放,即方法1的修正写法
QDialog *dlg = new QDialog;
dlg->setAttribute(Qt::WA_DeleteOnClose);
dlg->show();
```

### 选择弹框
```
//多个按钮由|分隔，选择的按钮存在于返回值中
    int ret = QMessageBox::warning(this, tr("MyApplication"), tr("The document has been modified.\n" "Do you want to save your changes?"),
QMessageBox::Save | QMessageBox::Discard | QMessageBox::Cancel, QMessageBox::Save);
	//枚举返回值进行相应处理
    switch (ret) {
        case QMessageBox::Save:
            qDebug() << "save";
        break;
        case QMessageBox::Discard:
            qDebug() << "Discard";
         break;
        case QMessageBox::Cancel:
            qDebug() << "Cancel";
        break;
    default:
        break;
    }
    
    //文件名选择框，返回选择的文件名
    QString fileName = QFileDialog::getOpenFileName(this,tr("Open Image"), "../", tr("Image Files (*.png *.jpg *.bmp)"));
    qDebug() << fileName;
```
