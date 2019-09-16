title: FFMpeg QT跨平台播放器二 qt配置及多媒体文件音视频分流
author: Cyrus
tags:
  - 播放器
categories:
  - 音视频
date: 2019-09-16 22:32:00
---
项目是基本iOS的KxMovie仿写的C++版本简易跨平台播器练练手，有兴趣的可以去看看KxMovie的源码。项目主要实现了： 
* FFMPEG视频处理原理以及实现；
* QT界面设计；
* FFMPEG音频处理原理以及实现；
* 视频播放进度控制及音量控制；
* 代码MacOS/Windos/Ubuntu跨平台通用。   
* 后面有时间可以考虑研究一下使用QT的OPenGL Widget显示。

### 一、QT .pro文件引入FFMpeg
~~~
//开发平台为Mac平台
macx {

LIBS += -L /usr/lib -lbz2 -lm -lz -llzma
LIBS += -framework Foundation -framework AudioToolBox -framework VideoToolBox -framework AVFoundation -framework Cocoa -framework CoreMedia -framework QuartzCore -framework VideoDecodeAcceleration -liconv

INCLUDEPATH += $$PWD/libFFmpeg/Macx/include
LIBS += -L$$PWD/libFFmpeg/Macx/lib
LIBS += -lavcodec   \
        -lavdevice  \
        -lavfilter  \
        -lavformat  \
        -lavutil    \
        -lswscale   \
        -lswresample
}


// Linux跟windows的等
unix:!macx {

}

win32 {

}
~~~
* 相关依赖可以看FFMpeg QT跨平台播放器一说的，编译完FFMpeg后，lib/pkgconfig 里面的.pc文件
![](pro_1.png)

### 二、QT找开多媒体文件
~~~
//mainwindow.cpp

//1、配置按钮的信号和槽
connect(ui->fileBtn, &QPushButton::clicked, this, &MainWindow::onOpenFile);
//创建解码对象
decoder = new MPMovieDecoder();

//2、QT打开文件
void MainWindow::onOpenFile()
{
    //弹窗选择文件
    QString path = QFileDialog::getOpenFileName(this, "open", "../", "all(*.*)");

    if(path.isEmpty() == true) {
        return;
    }

    // 解码对象打开文件
    int status = decoder->openFile(path);
    if (0 != status) {
        return;
    }
    
    // 创建两个队列用于存放解码后音视频帧
    _videoFrames = new QQueue<MPVideoFrame *>();
    _audioFrames = new QQueue<MPVideoFrame *>();

    if (decoder->isNetwork) {

        _minBufferedDuration = NETWORK_MIN_BUFFERED_DURATION;
        _maxBufferedDuration = NETWORK_MAX_BUFFERED_DURATION;

    } else {

        _minBufferedDuration = LOCAL_MIN_BUFFERED_DURATION;
        _maxBufferedDuration = LOCAL_MAX_BUFFERED_DURATION;
    }

    decoding = false;
    playing = false;
	
    // 获取文件时长并显示
    int duration = (int)decoder->getDuration();
   	ui->totalTime->setText(formatTime(duration));
	
    //播放
    play();
}

~~~

下面看看FFMpeg的如何实现音视频分流
~~~
//mpmoviedecoder.h

// 1、导入FFMpeg
extern "C"{

#include "libavcodec/avcodec.h"
#include "libavformat/avformat.h"
#include "libswscale/swscale.h"
#include "libswresample/swresample.h"
#include "libavutil/pixdesc.h"
}

~~~

~~~
// mpmoviedecoder.cpp

// 2、注册解码器
av_register_all();

// 3、通过路径打开文件
int MPMovieDecoder::openFile(QString path)
{
    if(path == NULL) {
        qDebug() << "null path" << endl;
        return -1;
    }

    _swsContext = NULL;
    _swrContext = NULL;
    _videoFrame = NULL;
    _videoCodecCtx = NULL;
    _formatCtx = NULL;

    isNetwork = isNetworkPath(path);

    static bool needNetworkInit = true;
    if (needNetworkInit && isNetwork) {
        needNetworkInit = false;
        avformat_network_init();
    }
    int errCode = 0;
    this->path = path;
    // 4、成功找到多媒体文件解码器的话
    if ((errCode = openInput(path)) >= 0) {
    	// 5、打开py
        int videoErr = openVideoStream();
        int audioErr = openAudioStream();
        if((0 != videoErr) && (0 != audioErr)) {
            errCode = -1;
        } else {
            _subtitleStreams = collectStreams(_formatCtx, AVMEDIA_TYPE_SUBTITLE);
        }
    } else {
        qDebug() << "open input failure" << endl;
    }

    if(errCode != 0) {
        closeFile();
        return errCode;
    }
    return 0;
}

int MPMovieDecoder::openInput(QString path)
{
    AVFormatContext *formatCtx = NULL;
    // 1、avformat_open_input
    int status = avformat_open_input(&formatCtx, (const char *)(path.toUtf8()), NULL, NULL);
    if (status < 0) {
        qDebug() << "avformat_open_input: failure"  << endl;
        if (formatCtx)
            avformat_free_context(formatCtx);
        return -1;
    }
	
    // 2、avformat_find_stream_info
    if (avformat_find_stream_info(formatCtx, NULL) < 0) {
        qDebug() << "avformat_find_stream_info failure" << endl;
        avformat_close_input(&formatCtx);
        return -1;
    }
	// 3、av_dump_format
    av_dump_format(formatCtx, 0, path.toUtf8(), false);

    _formatCtx = formatCtx;
    return 0;
}

~~~

先看看分流的方法
~~~
 QVector<int> *MPMovieDecoder::collectStreams(AVFormatContext *formatCtx, enum AVMediaType codecType)
 {
     QVector<int> *ma = new QVector<int>();
     for (int i = 0;i < formatCtx->nb_streams; i++) {
         if (codecType == formatCtx->streams[i]->codec->codec_type) {
             ma->append(i);
         }
     }
     return ma;
 }
 
 // FFMpeg里面，音视、视频、字幕都是用stream定义，而且可能不止一个stream，例如文件支持多语音频及字幕，那么就会有多个音频stream及多个字幕stream。AVMEDIA_TYPE_AUDIO（音频）、AVMEDIA_TYPE_SUBTITLE（字幕）、AVMEDIA_TYPE_VIDEO（视频）。这里把相关类型的所有stream所在的formatCtx->streams数组中的下标都存起来，如果stream不止一个，有需要的可以根据切换相应的下标中英文之类的。
~~~

~~~
// 找到视频流
int MPMovieDecoder::openVideoStream()
{
    qDebug() << "openVideoStream" << endl;
    int errCode = 0;
    _videoStream = -1;
    _artworkStream = -1;
	// 找到文件中的所有视频流
    _videoStreams = collectStreams(_formatCtx, AVMEDIA_TYPE_VIDEO);
    qDebug() << "_videoStreams count:" << _videoStreams->count() << endl;
    for (int i= 0; i<_videoStreams->count(); i++) {
    	// 通过&操作，判断视频是否以图片的形式存在，0不是，非0是
        if(0 == (_formatCtx->streams[i]->disposition & AV_DISPOSITION_ATTACHED_PIC)) {
        	// 查找对应的视频流的编解码信息
            errCode = openVideoStream(_videoStreams->at(i));
            if (0 == errCode) {
                break;
            }
        } else {
            _artworkStream = i;
        }
    }

    return errCode;
}


int MPMovieDecoder::openVideoStream(int videoStream)
{
//    // get a pointer to the codec context for the video stream
        AVCodecContext *codecCtx = _formatCtx->streams[videoStream]->codec;

//        // find the decoder for the video stream
        AVCodec *codec = avcodec_find_decoder(codecCtx->codec_id);
        if (codec == NULL)
            return -1;

//        // open codec
        if (avcodec_open2(codecCtx, codec, NULL) < 0)
            return -2;

        _videoFrame = av_frame_alloc();

        if (!_videoFrame) {
            avcodec_close(codecCtx);
            return -3;
        }

        _videoStream = videoStream;
        _videoCodecCtx = codecCtx;

        AVStream *st = _formatCtx->streams[_videoStream];
        avStreamFPSTimeBase(st, 0.04, &this->fps, &_videoTimeBase);

        QString info = QString("video codec size: %1:%2 fps: %3 tb: %4").arg(frameWidth).arg(frameHeight).arg(fps).arg(_videoTimeBase);
        qDebug() << info << endl;
        qDebug() << "video start time " << st->start_time * _videoTimeBase << endl;
        qDebug() << "video disposition " << st->disposition << endl;
        return 0;
}
~~~

同理，找音频流
~~~
int MPMovieDecoder::openAudioStream()
{
    int errCode = 0;
    _audioStream = -1;
    _audioStreams = collectStreams(_formatCtx, AVMEDIA_TYPE_AUDIO);
    for (int i= 0; i<_audioStreams->count(); i++) {
         errCode = openAudioStream(_audioStreams->at(i));
         if (0 == errCode) {
             break;
         }
    }
    return errCode;
}

int MPMovieDecoder::openAudioStream(int audioStream)
{
    uint64_t out_channel_layout=AV_CH_LAYOUT_STEREO; //定义目标音频参数
    AVCodecContext *codecCtx = _formatCtx->streams[audioStream]->codec;
    SwrContext *swrContext = NULL;

    AVCodec *codec = avcodec_find_decoder(codecCtx->codec_id);
    if(codec == NULL)
        return -1;

    if (avcodec_open2(codecCtx, codec, NULL) < 0) return -2;

    swrContext = swr_alloc_set_opts(NULL,
                                    AV_CH_LAYOUT_STEREO,
                                    AV_SAMPLE_FMT_S16,
                                    44100,
                                    codecCtx->channel_layout,
                                    codecCtx->sample_fmt,
                                    codecCtx->sample_rate,
                                    0,
                                    NULL);

    if (!swrContext || swr_init(swrContext)) {

        if (swrContext) swr_free(&swrContext);
        avcodec_close(codecCtx);
        return -3;
    }

    _audioFrame = av_frame_alloc();

    if (!_audioFrame) {
        if (swrContext)
            swr_free(&swrContext);
        avcodec_close(codecCtx);
        return -4;
    }

    _audioStream = audioStream;
    _audioCodecCtx = codecCtx;
    _swrContext = swrContext;

    AVStream *st = _formatCtx->streams[_audioStream];
    avStreamFPSTimeBase(st, 0.025, 0, &_audioTimeBase);

    QString info = QString("audio codec smr: %1 fmt:%2 chn: %3 tb: %4 %5").arg(_audioCodecCtx->sample_rate).arg(_audioCodecCtx->sample_fmt).arg(_audioCodecCtx->channels).arg(_audioTimeBase).arg(_swrContext ? "resample" : "");
    qDebug() << info << endl;
    return 0;
}

// 这里讲解一下FFMpeg的音频重采样SwrContext。这里的做法是，不管文件的音频是怎样的，我们都把它转化成双通道（AV_CH_LAYOUT_STEREO）， 16位精度（AV_SAMPLE_FMT_S16）， 44100采样率的pcm数据。
1、swr_alloc_set_opts 创建SwrContext转化参数
2、swr_init SwrContext初始化
~~~