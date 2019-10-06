title: QT OpenGL渲染RGB格式数据
author: Cyrus
tags:
  - QT
categories:
  - 播放器
date: 2019-10-06 21:13:00
---
为了更好的根据具体情况显示YUV420的视频帧和RGB的视频帧，我们先创建一个基类Video_Render：
~~~
//video_render.h
#ifndef VIDEO_RENDER_H
#define VIDEO_RENDER_H

#include <QObject>
#include <QOpenGLWidget>
#include <QOpenGLFunctions>
#include <QOpenGLShaderProgram>

class Video_Render: protected QOpenGLFunctions
{
public:
    Video_Render();
    virtual ~Video_Render();

    virtual void initialize() = 0;
    virtual void render(uchar* ptr,int width,int height,int type) = 0;
};

#endif // VIDEO_RENDER_H


// video_render.cpp
#include "video_render.h"
Video_Render::Video_Render()
{

}

Video_Render::~Video_Render()
{

}
~~~

修改YUV420_Render,使之继承Video_Render
~~~
#ifndef YUV420_RENDER_H
#define YUV420_RENDER_H

#include "video_render.h"

class YUV420_Render: public Video_Render
{
public:
    YUV420_Render();
    ~YUV420_Render();

        //初始化gl
        void initialize();
        //刷新显示
        void render(uchar* py,uchar* pu,uchar* pv,int width,int height,int type);
        void render(uchar* ptr,int width,int height,int type);

    private:
        //shader程序
        QOpenGLShaderProgram m_program;
        //shader中yuv变量地址
        GLuint m_textureUniformY, m_textureUniformU , m_textureUniformV;
        //创建纹理
        GLuint m_idy , m_idu , m_idv;
};

#endif // YUV420_RENDER_H
~~~

创建RGB_Render，同样继承Video_Render
~~~
// rgb_render.h
#ifndef RGB_RENDER_H
#define RGB_RENDER_H

#include "video_render.h"

class RGB_Render: public Video_Render
{
public:
    RGB_Render();
    ~RGB_Render();

    //初始化gl
    void initialize();
    //刷新显示
    void render(uchar* ptr,int width,int height,int type);

private:
    //shader程序
    QOpenGLShaderProgram m_program;
    //shader中yuv变量地址
    GLuint m_textureUniform;
    //创建纹理
    GLuint m_id;
};

#endif // RGB_RENDER_H



// rgb_render.cpp
#include "rgb_render.h"

#define ATTRIB_VERTEX 0
#define ATTRIB_TEXTURE 1

RGB_Render::RGB_Render()
{

}

RGB_Render::~RGB_Render()
{

}

//初始化gl
void RGB_Render::initialize()
{
    qDebug() << "initializeGL";

    //初始化opengl （QOpenGLFunctions继承）函数
    initializeOpenGLFunctions();

    //顶点shader
    const char *vString =
            "attribute vec4 vertexPosition;\
            attribute vec2 textureCoordinate;\
    varying vec2 texture_Out;\
    void main(void)\
    {\
        gl_Position = vertexPosition;\
        texture_Out = textureCoordinate;\
    }";
    //片元shader
    const char *tString =
            "varying vec2 texture_Out;\
            uniform sampler2D tex;\
    void main(void)\
    {\
        gl_FragColor = texture2D(tex, texture_Out); \
    }";

    //m_program加载shader（顶点和片元）脚本
    //片元（像素）
    qDebug()<<m_program.addShaderFromSourceCode(QOpenGLShader::Fragment, tString);
    //顶点shader
    qDebug() << m_program.addShaderFromSourceCode(QOpenGLShader::Vertex, vString);

    //设置顶点位置
    m_program.bindAttributeLocation("vertexPosition",ATTRIB_VERTEX);
    //设置纹理位置
    m_program.bindAttributeLocation("textureCoordinate",ATTRIB_TEXTURE);

    //编译shader
    qDebug() << "m_program.link() = " << m_program.link();

    qDebug() << "m_program.bind() = " << m_program.bind();

    //传递顶点和纹理坐标
    //顶点
    static const GLfloat ver[] = {
        -1.0f,-1.0f,
        1.0f,-1.0f,
        -1.0f, 1.0f,
        1.0f,1.0f
        //        -1.0f,-1.0f,
        //        0.9f,-1.0f,
        //        -1.0f, 1.0f,
        //        0.9f,1.0f
    };
    //纹理
    static const GLfloat tex[] = {
        0.0f, 1.0f,
        1.0f, 1.0f,
        0.0f, 0.0f,
        1.0f, 0.0f
    };

    //设置顶点,纹理数组并启用
    glVertexAttribPointer(ATTRIB_VERTEX, 2, GL_FLOAT, 0, 0, ver);
    glEnableVertexAttribArray(ATTRIB_VERTEX);
    glVertexAttribPointer(ATTRIB_TEXTURE, 2, GL_FLOAT, 0, 0, tex);
    glEnableVertexAttribArray(ATTRIB_TEXTURE);

    //从shader获取地址
    m_textureUniform = m_program.uniformLocation("tex");

    //创建纹理
    glGenTextures(1, &m_id);
    //只需要一个texture
    glBindTexture(GL_TEXTURE_2D, m_id);
    //放大过滤，线性插值   GL_NEAREST(效率高，但马赛克严重)
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);

    glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
    glClear(GL_COLOR_BUFFER_BIT);

}


void RGB_Render::render(uchar* ptr,int width,int height,int type)
{
    glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
    glClear(GL_COLOR_BUFFER_BIT);

    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, m_id);
    //修改纹理内容(复制内存内容)
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE,ptr);
    //与shader 关联
    glUniform1i(m_textureUniform, 0);

    glDrawArrays(GL_TRIANGLE_STRIP,0,4);
    qDebug() << "paintGL";
}

~~~

修改MyOpenGLWidget
~~~
#ifndef MYOPENGLWIDGET_H
#define MYOPENGLWIDGET_H

#include <QOpenGLWidget>
#include "video_render.h"
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
    Video_Render *pRender;
    MPVideoFrame  *pCurrentFrame;
};

#endif // MYOPENGLWIDGET_H


#include "myopenglwidget.h"
#include "rgb_render.h"
#include "yuv420_render.h"

MyOpenGLWidget::MyOpenGLWidget(QWidget *parent, Qt::WindowFlags f) : QOpenGLWidget(parent, f)
{
    pRender = new RGB_Render();
    pCurrentFrame = NULL;
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