title: QT基础10——UDP
author: Cyrus
tags: []
categories:
  - QT
date: 2019-01-01 21:17:00
---
```
1、创建socket
socket = new QUdpSocket(this);

2、绑定端口
socket->bind(10000);

3、处理收到数据信号
connect(socket, &QUdpSocket::readyRead,
            [=](){
        char buf[2048] = {0};
        QHostAddress cliAddr;
        quint16 port;
        
        //udp需要使用读取数据报的函数，否则无法成功读取数据
        qint64 length = socket->readDatagram(buf, sizeof(buf), &cliAddr, &port);
        if(length > 0) {
            QString str = QString("[%1:%2]:%3").arg(cliAddr.toString()).arg(port).arg(buf);
            ui->textEdit->setText(str);
        }

    });
    
    //发送数据
    QString str = "hello world";
    socket->writeDatagram(str.toUtf8(), QHostAddress("127.0.0.1"), 10000);
    
    //若写数据报时地址改为255.255.255.255则为广播
    QString str = "hello world";
    socket->writeDatagram(str.toUtf8(), QHostAddress("255.255.255.255"), 10000);
    
    //如果想要组播，则需要加入组播地址
    //socket->joinMulticastGroup(xxx)
// socket->leaveMulticastGroup(xxx)
```