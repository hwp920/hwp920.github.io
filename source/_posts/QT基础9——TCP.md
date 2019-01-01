title: QT基础9——TCP
author: Cyrus
tags: []
categories:
  - QT
date: 2019-01-01 20:56:00
---
#### 服务端
常用的tcp服务端会有两个套接字，一个监听套接字和三次握手之后正常通讯 的套接字。QT的tcp服务端把这两个套接字设计成两个类：QTcpServer（监听套接字）和QTcpSocket（通讯套接字，客户端只用通讯套接字）。
```
//1、创建监听套接字
server = new QTcpServer(this);

//2、监听地址和端口，相当于socket的 bind+listen
server->listen(QHostAddress::Any, 10000);

//3、当有客户端连接时，会发出newConnection信号
connect(server, &QTcpServer::newConnection,
            [=] () {
            
        //4、获取三次握手成功的客户端链接，相当于accept
        tcpSocket = server->nextPendingConnection();
		
        //获取客户端IP
        QString ip = tcpSocket->peerAddress().toString();
        //获取客户端商品
        quint16 port = tcpSocket->peerPort();
        QString temp = QString("[%1:%2]: 连接成功").arg(ip).arg(port);
        qDebug() << ip << port;
        ui->textRead->setText(temp);
		
        //5、当接收到客户端数据时，会发出readyRead信号
        connect(tcpSocket, &QTcpSocket::readyRead,
                [=]() {
             //读数据，获有接口文档的，可以读定长数据
            QByteArray array = tcpSocket->readAll();
            ui->textRead->append(array);
        });
		
        //6、客户端断开时，会发出disconnected信号，处理断开事件
        connect(tcpSocket, &QTcpSocket::disconnected,
                [=]() {
            tcpSocket->disconnectFromHost();
            tcpSocket->close();
        });

    });
    
    //写数据到客户端
    QString str = @"Hello world";
    tcpSocket->write(str.toUtf8().data());
    
    //主动断开连接
    tcpSocket->disconnectFromHost();
    tcpSocket->close();
    tcpSocket = NULL;
```

#### 客户端
```
//1、创建socket
client = new QTcpSocket(this);

//2、连接服务端
client->connectToHost("127.0.0.1", 8888, QIODevice::ReadWrite);

//3、处理连接成功信号
connect(client, &QTcpSocket::connected,
            [=](){
        QString ip = client->peerAddress().toString();
        quint16 port = client->peerPort();
        QString temp = QString("[%1 : %2]").arg(ip).arg(port);
        ui->editRead->setText(temp);
    });
    
//4、处理收到数据信号
connect(client, &QTcpSocket::readyRead,
            [=](){
        QByteArray array = client->readAll();
        ui->editRead->append(array);
 });
 
 5、处理断开连接信号
 connect(client, &QTcpSocket::disconnected, 
            [=](){
        //服务端断开连接
    });
    
    
//写数据
QString str = @"Hello world";
client->write(str.toUtf8().data());

//断开连接
client->disconnectFromHost();
client->close();
ui->editRead->setText("disconnected");
client = NULL;
```