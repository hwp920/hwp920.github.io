title: QT OpenGL渲染YUV420格式数据
author: Cyrus
tags:
  - QT
categories:
  - OpenGL
  - 播放器
date: 2019-10-06 20:09:00
---
### 一、代码
YUV420Render的代码参考https://www.cnblogs.com/nanqiang/p/10224891.html，在此表示感谢。

### 二、应用
应用方面，我们以之前的QT播放器为例：

#### 1、获取YUV420数据。注意，这里主要用于h.264解码后为YUV420的视频帧，如果是其他格式的，建议转成rgb后显示（QT OpenGL渲染rgb的下一篇讲）。
代码：
~~~
// 注意，这里默认_videoCodecCtx->pix_fmt == AV_PIX_FMT_YUV420P || _videoCodecCtx->pix_fmt == AV_PIX_FMT_YUVJ420P，使用YUV的显示的话，就不用设置SwsContext转格式了


MPVideoFrame *MPMovieDecoder::handleVideoFrame()
{
    qDebug() << "MPMovieDecoder::handleVideoFrame()" << endl;
    if (_videoFrame->data[0] == NULL) {
        qDebug() << "_videoFrame->data[0] no data" << endl;
        return NULL;
    }

    MPVideoFrame *frame = new MPVideoFrame();
    frame->type = MPMovieFrameTypeVideo;

    int lineSize = _videoFrame->linesize[0] > _videoCodecCtx->width ? _videoCodecCtx->width : _videoFrame->linesize[0];
    qDebug() << "linesize:" << lineSize << endl;

	// 这里把yuv的数据放在一起的方式，主要是为了兼容rgb的渲染，详见下一篇
    int size = lineSize * _videoCodecCtx->height;
    frame->databuffer = (unsigned char *)malloc(size * 3 / 2);
    frame->bufferSize = size * 3 / 2;
    copyFrameData(_videoFrame->data[0], frame->databuffer, _videoFrame->linesize[0], _videoCodecCtx->width, _videoCodecCtx->height);
    copyFrameData(_videoFrame->data[1], frame->databuffer + size, _videoFrame->linesize[1], _videoCodecCtx->width/2, _videoCodecCtx->height/2);
    copyFrameData(_videoFrame->data[2], frame->databuffer + size * 5 / 4, _videoFrame->linesize[2], _videoCodecCtx->width/2, _videoCodecCtx->height/2);

    frame->width = _videoFrame->lineSize;
    frame->height = _videoCodecCtx->height;
    qDebug() << "frame->width:" << frame->width <<  " frame->height:" << frame->height << endl;
    frame->position = av_frame_get_best_effort_timestamp(_videoFrame) * _videoTimeBase;

    const int64_t frameDuration = av_frame_get_pkt_duration(_videoFrame);
    if (frameDuration) {
        frame->duration = frameDuration * _videoTimeBase;
        frame->duration += _videoFrame->repeat_pict * _videoTimeBase * 0.5;
    } else {

        // sometimes, ffmpeg unable to determine a frame duration
        // as example yuvj420p stream from web camera
        frame->duration = 1.0 / fps;
    }
    return frame;
}

// 拷贝数据
void MPMovieDecoder::copyFrameData(unsigned char *src, unsigned char *ioData, int linesize, int width, int height)
{
    width = width > linesize ? linesize : width;
    int dst = 0;
    for (int i = 0; i < height; i++) {
        memcpy(ioData, src, width);
        src += linesize;
        ioData += width;
    }
}
~~~

#### 2、播放
代码：
~~~
// MyOpenGLWidget.h

#ifndef MYOPENGLWIDGET_H
#define MYOPENGLWIDGET_H

#include <QObject>
#include <QOpenGLWidget>
#include <QOpenGLFunctions>
#include <QOpenGLShaderProgram>
#include "yuv420_render.h"
#include "mpvideoframe.h"

class MyOpenGLWidget : public QOpenGLWidget
{
    Q_OBJECT
public:
    explicit MyOpenGLWidget(QWidget *parent = nullptr, Qt::WindowFlags f = Qt::WindowFlags());
    void setFrame(MPVideoFrame *frame);

protected:

    void initializeGL();
    void paintGL();
    void resizeGL(int w, int h);

private:
    YUV420_Render *pRender;
    MPVideoFrame  *pCurrentFrame;
};

#endif // MYOPENGLWIDGET_H



// MyOpenGLWidget.cpp
#include "myopenglwidget.h"
#include "rgb_render.h"

MyOpenGLWidget::MyOpenGLWidget(QWidget *parent, Qt::WindowFlags f) : QOpenGLWidget(parent, f)
{
    pRender = new YUV420_Render();
}

void MyOpenGLWidget::initializeGL()
{
    pCurrentFrame = NULL;
    pRender->initialize();
}

void MyOpenGLWidget::setFrame(MPVideoFrame *frame)
{
    if(this->pCurrentFrame != NULL) {
        delete this->pCurrentFrame;
    }
    this->pCurrentFrame = frame;
    update();
}

void MyOpenGLWidget::paintGL()
{
    if(pCurrentFrame == NULL) {
        return;
    }
//    pRender->render(pCurrentFrame->yData, pCurrentFrame->uData, pCurrentFrame->vData, pCurrentFrame->width, pCurrentFrame->height, 0);
    pRender->render(pCurrentFrame->databuffer, pCurrentFrame->width, pCurrentFrame->height, 0);
}

void MyOpenGLWidget::resizeGL(int w, int h)
{
    glViewport(0, 0, w, h);
}
~~~

MainWidget里面，将之前的QLabel改为MyOpenGLWidght
~~~
double MainWindow::presentVideoFrame(MPVideoFrame *frame)
{
//    displayRBGFrame((char *)frame->databuffer, frame->width, frame->height);

	// 直接调用setFrame就可以显示
    ui->openGLWidget->setFrame(frame);
    _moviePosition = frame->position;
    return frame->duration;
}
~~~

