title: QT基础4——可视化编程
author: Cyrus
tags: []
categories:
  - QT
date: 2018-12-16 22:37:00
---
前面的视图都是使用代码创建，QT也支持可视化编程，且性能强大。只需要在创建项目时勾选“创建界面”（默认勾选），生成的项目中就会自动生成支持可视化编程的.ui文件。

点击左侧“设计”按钮，.ui的样式如下：
![](design_all.png)

### 控件区
* Layouts 布局区，少用

* Spacers 填补布局时的间隙，缩放界面时自适应（详看上图）

1、Horizontal Spacer 可以填补水平间隙

2、Vertical Spacer	可以填补垂直间隙

* Buttons

1、Push Button	常用按钮

2、Tool Button 

3、Radio Button 单选按钮，必须处于同一父视图同一层级

4、Check Box	多选按钮

* Item Views 和 Item Widgets

* Containers 各种可以作为父视图的控件

* Input Widgets 各种输入控件

* Display Widgets 各种展示类控件

示例
![](design_add.png) ![](design_result.png)

### 视图层级区
可以查看当前窗口的所有视图及相应的层级分布，layout状态。可以修改相应控件的名称及进行其他操作。

### 控件属性区
属性区由控件的继承关系从上到下展示控件的可编辑属性，如控件名称，控件大小，是否可拉伸等等。

### 信号和槽区
信号和槽区可以对窗口控件直接添可信号和相当的槽响应，无需编写Connect函数。

### 代码调用.ui文件创建的控件
```
MainWindow::MainWindow(QWidget *parent) :
    QMainWindow(parent),
    ui(new Ui::MainWindow)
{
	//ui文件初始化，勾选“创建界面”后系统自动生成
    ui->setupUi(this);

	//ui->控件名，可以选取相应控件的指针，控件名可以在“视图层级区”查看及修改
    ui->myButton->setText("我了个去");
}
```

### 创建按钮槽事件
选择相应的按钮（如myButton)，右键，转到槽，选择相应的按钮信号（如released()），就会自动生成槽函数 void on_按钮名_信号名（）。
```
.h文件
private slots:
    void on_myButton_released();
    
.cpp文件
void MainWindow::on_myButton_released()
{
    
}
```
也可以创建好相应的槽函数后，在信号与槽编辑区直接添加绑定，不用调用Connect函数
![](design_slot.png)

### 在ui界面中使用自定义类
* 1、创建自定义类子类，如QPushButton子类 MyButton
* 2、选中一个QPushButton，右键->提升为，填写“提升的类名称”->MyButton， 勾选“全局包含”， 点击“添加”, 再点击“提升”即可。

将自定义降为父类：选中提升后的控件，右键选择“选择xxx的提升”。


