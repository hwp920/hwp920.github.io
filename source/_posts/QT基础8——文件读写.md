title: QT基础8——文件读写
author: Cyrus
tags: []
categories:
  - QT
date: 2019-01-01 20:42:00
---
#### 读文件
```
//弹窗选择文件
	QString path = QFileDialog::getOpenFileName(this, "open", "../", "TXT(*.txt)");

    if(path.isEmpty() == true) {
        return;
    }
    //打开文件
    bool success = file.open(QIODevice::ReadOnly);
        if(success == true) {
//        //读文件,默认只识别utf-8编码
	      //小文件的话可以一次读完		
//        QByteArray array =  file.readAll();
//        ui->textEdit->setText(array);
		
        //大文件的话一次读一行
        QByteArray array;
        while (file.atEnd() == false) {
            array += file.readLine();
        }
        ui->textEdit->setText(array);
    }
    file.close();
```

#### 写文件
```
//弹框选择保存文件的路径
		QString path = QFileDialog::getSaveFileName(this, "save", "../", "TXT(*.txt)");
    if (path.isEmpty() == true) {
        return;
    }

    QFile file;
    file.setFileName(path);
    bool success = file.open(QIODevice::WriteOnly);

    if (success) {
        QString str = ui->textEdit->toPlainText();

        // str.toUtf8() -->QString->QByteArray
//        file.write(str.toUtf8());

        //QString->C++ String -> char *
        file.write(str.toStdString().data());
    }
    file.close();
```

#### 文件信息
```
	QFileInfo info(path);
     qDebug() << "文件名字：" << info.fileName();
     qDebug() << "文件后缀" << info.suffix();
     qDebug() << "文件大小" << info.size();
     qDebug() << "创建时间" << info.birthTime().toString("yyyy-MM-dd hh:mm:ss");
```

#### 二进制读写文件
```
  	//写文件
	QFile file("../123.txt");
    if (file.open(QIODevice::WriteOnly) == false) {
        qDebug() << "Open failure";
        return;
    }

    QDataStream stream(&file);
    stream << QString("试一试") << 123;
    file.close();
    
    
    //读文件
    QFile file("../123.txt");
    if (file.open(QIODevice::ReadOnly) == false) {
        qDebug() << "Open failure";
        return;
    }

    QDataStream stream(&file);
    QString str;
    int a;
    stream >> str >> a;
    qDebug() << str.toUtf8().data() << a;
    file.close();
    
    //文本格式读写文件,用法与QDataStream类似，区别是可以设置编码类型
     QTextStream textStream;
     textStream.setCodec("UTF-8");
```