title: FFMpeg QT跨平台播放器三 视频解码播放
author: Cyrus
tags:
  - 播放器
categories:
  - 音视频
date: 2019-09-17 22:07:00
---
### 一、MPVideoFrame 存放解码后的音视频数据
~~~
// 帧数据类型
typedef enum {
    MPMovieFrameTypeAudio,
    MPMovieFrameTypeVideo,
    MPMovieFrameTypeArtwork,
    MPMovieFrameTypeSubtitle,
} MPMovieFrameType;


class MPVideoFrame
{
public:
    MPVideoFrame();
    MPVideoFrame(const MPVideoFrame &other);
    MPVideoFrame& operator=(MPVideoFrame& t);
    ~MPVideoFrame();
    
    MPMovieFrameType    type;
    int                 width;
    int                 height;
    int                 bufferSize;
    unsigned char*      databuffer;
    double              position;
    double              duration;
};

~~~

### 二、视频解码
简单说一下解码的过程：
* 1、首先从文件中读出一块数据，可能是一帧或多帧音视频数据，存到AVPacket结构里面
* 2、判断packet是音频还是视频，分别调用相应的解码函数，如视频的avcodec_decode_video2和音频的avcodec_decode_audio4，视频解码出来的就是AVCodec里面的AVPixelFormat类型数据，如YUV420，YUV422,RGB等。音频解码出来的是pcm数据，并将数据存到AVFrame结构体中。这时的数据就是可以播放的数据了。
* 3、对于视频，如何不采用OpenGL显示，则可以用sws_scale函数，将数据转成RGB，再将RGB转成图片显示。对于音频，我们可以选择适合设备的参数，对音频数据进行重采样播放（下一篇讲）。

下面看代码：
~~~
QVector<MPVideoFrame *> *MPMovieDecoder::decodeFrames(float minDuration)
{
    if (_videoStream == -1 && _audioStream == -1) return NULL;
    QVector<MPVideoFrame *> *result = new QVector<MPVideoFrame *>();
    AVPacket packet;
    float decodedDuration = 0;
    bool finished = false;
    while (!finished) {
    	// 1、读取一块数据
        if (av_read_frame(_formatCtx, &packet) < 0) {
            isEOF = true;
            break;
        }
        // 2、判断是音频还是视频
        if (packet.stream_index == _videoStream) {
            int pktSize = packet.size;
            while (pktSize > 0) {
                int gotframe = 0;
				// 解码，
                int len = avcodec_decode_video2(_videoCodecCtx, _videoFrame, &gotframe, &packet);
                if (len < 0) {
                    qDebug() << "decode video error, skip packet" << endl;
                    break;
                }
                if (gotframe) {
                	// 将解码数据转成RGB并存到MPVideoFrame中
                    MPVideoFrame *frame = handleVideoFrame();
                    if (frame != NULL) {
//                        QMutexLocker locker(&m_decodeMutex);
                        result->append(frame);
                        _position = frame->position;
                        decodedDuration += frame->duration;
                        if (decodedDuration > minDuration) finished = true;
                    }
                }
                if (0 == len) break;
                pktSize -= len;
            }
        }
        if(packet.stream_index == _audioStream) {
    // 音频解码
		}
        av_free_packet(&packet);
    }
    return result;
}
~~~

~~~
/** 简单讲一下解码完的数据存在方式
* 1、数据类型为AVCodecContext->AVPixelFormat/AVCodec->AVPixelFormat
* 2、数据存在AVFrame结构体中的uint8_t *data[AV_NUM_DATA_POINTERS];
* 3、相应的数据长度存在AVFrame结构体中int linesize[AV_NUM_DATA_POINTERS];
* 如YUV420 即data[0]是Y分量数据，linesize[0]是Y分量数据长度。data[1]是U分量数据，linesize[1]是U分量数据长度.data[2]是V分量数据，linesize[2]是V分量数据长度.
*/
MPVideoFrame *MPMovieDecoder::handleVideoFrame()
{
    if (_videoFrame->data[0] == NULL) {
        qDebug() << "_videoFrame->data[0] no data" << endl;
        return NULL;
    }
    if (_swsContext == NULL) {
    	// 设置转换器
        bool result = setupScaler();
        if (!result) {
            qDebug() << "fail setup video scaler" << endl;
            return NULL;
        }
    }
	// sws_scale 可以实现1.图像色彩空间转换；2.分辨率缩放；3.前后图像滤波处理。解码完成的数据可以存在AVPicture中
    int height = sws_scale(_swsContext,
              (const uint8_t **)_videoFrame->data,
              _videoFrame->linesize,
              0,
              _videoCodecCtx->height,
              _picture.data,
              _picture.linesize);
  
    MPVideoFrame *frame = new MPVideoFrame();
    frame->type = MPMovieFrameTypeVideo;
    frame->bufferSize = _picture.linesize[0] * _videoCodecCtx->height;
    frame->databuffer = (unsigned char*)malloc(frame->bufferSize);
    memcpy(frame->databuffer, _picture.data[0], frame->bufferSize);
    frame->width = _videoCodecCtx->width;
    frame->height = _videoCodecCtx->height;
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
~~~

~~~
bool MPMovieDecoder::setupScaler()
{
    closeScaler();

    _pictureValid = avpicture_alloc(&_picture,
                                    AV_PIX_FMT_RGB24,
                                    _videoCodecCtx->width,
                                    _videoCodecCtx->height) == 0;

    if (!_pictureValid)
        return false;

    _swsContext = sws_getCachedContext(_swsContext,
                                       _videoCodecCtx->width,
                                       _videoCodecCtx->height,
                                       _videoCodecCtx->pix_fmt,
                                       _videoCodecCtx->width,
                                       _videoCodecCtx->height,
                                       AV_PIX_FMT_RGB24,
                                       SWS_FAST_BILINEAR,
                                       NULL, NULL, NULL);

    return _swsContext != NULL;
}

void MPMovieDecoder::closeScaler()
{
    qDebug() << "MPMovieDecoder::closeScaler start" << endl;
    if (_swsContext != NULL) {
        sws_freeContext(_swsContext);
        _swsContext = NULL;
    }
    qDebug() << "MPMovieDecoder::closeScaler end" << endl;
}

~~~

### 三、视频播放
这里简单说一下播放的逻辑，
* 1、设定一个一定时长的缓冲区间，如1-2秒，少于1s开始异步解码并缓存起来，缓存的视频长度大于2s则停止。
* 2、设定一个定时器，每显示一帧就把该帧的duration作为定时器的定时时长，在定时器的timeout信号槽里显示下一帧。

~~~
// 定时器，处理超时信号
 m_tickTimer = new QTimer(this);
    m_tickTimer->setSingleShot(true);
    connect(m_tickTimer, SIGNAL(timeout()), this, SLOT(handleTickTimeout()));


void MainWindow::handleTickTimeout()
{
    qDebug()<<"Enter timeout processing function\n";
    if(m_tickTimer->isActive()){
        m_tickTimer->stop();
    }

    qDebug() << "_videoFrames->length() = " << _videoFrames->length() << endl;
	// tick函数播放视频、判断是否继续解码缓存及根据播放帧的duration定时下一次超时
    tick();
}
~~~

// 具体讲播放前，先看一下异步解码的代码
~~~
void MainWindow::asyncDecodeFrames()
{
    qDebug() << "MainWindow::asyncDecodeFrames() start" << endl;
    if (decoding) return;

    const float duration = decoder->isNetwork ? .0f : 0.1f;

    decoding = true;
    qDebug() << "MainWindow::asyncDecodeFrames() start2" << endl;

	// 异步解码，并分别加到缓存的音频帧队列里
    std::function<void()> decode = [=]() {
        bool good = true;
        while (good) {
            if (decoder && (decoder->validVideo() || decoder->validAudio())) {
                QVector<MPVideoFrame *> *frames = decoder->decodeFrames(duration);
                if (!frames->isEmpty()) {
                    good = addFrames(frames);
                }
            }
        }
        decoding = false;
        qDebug() << "MainWindow::asyncDecodeFrames() end" << endl;
    };
    QtConcurrent::run(decode);
}

bool MainWindow::addFrames(QVector<MPVideoFrame *> *frames)
{
    if (decoder->validVideo()) {
        {
            QMutexLocker locker(&m_videoMutex);
            for (QVector<MPVideoFrame *>::iterator it = frames->begin(); it != frames->end(); ++it)
            {
                if ((*it)->type == MPMovieFrameTypeVideo) {
                    _videoFrames->enqueue(*it);
                    _bufferedDuration += (*it)->duration;
                }
            }
        }
    }

    if (decoder->validAudio()) {
        // 音频，下一篇讲
    }
    delete frames;
    return playing && _bufferedDuration < _maxBufferedDuration;
}
~~~

~~~
void MainWindow::tick()
{
    if (_buffered && ((_bufferedDuration > _minBufferedDuration) || decoder->isEOF)) {
        _tickCorrectionTime = 0;
        _buffered = false;
    }

    double interval = 0;


    if (playing) {

        if (!_buffered)  interval = presentFrame();

        if (decoder->isEOF) {
            qDebug() << "MainWindow::tick() eof" << endl;
            pause();
            return;
        }

        const int leftFrames =
        (decoder->validVideo() ? _videoFrames->length() : 0);

        if (0 == leftFrames) {

            if (_minBufferedDuration > 0 && !_buffered) {
                _buffered = true;
            }
        }

        if (!leftFrames ||
            !(_bufferedDuration > _minBufferedDuration)) {
            asyncDecodeFrames();
        }

        const double time = interval > 0.01 ? interval : 0.01;
        m_tickTimer->start(int(time * 1000));
    }
}
~~~

具体显示图像
~~~
double MainWindow::presentFrame()
{
    double interval = 0;

    if (decoder->validVideo()) {

        MPVideoFrame *frame = NULL;
        {
            QMutexLocker locker(&m_videoMutex);
            if (_videoFrames->length() > 0) {
                frame = _videoFrames->dequeue();
                _bufferedDuration -= frame->duration;
            }
        }
        if (frame != NULL) {
            interval = presentVideoFrame(frame);
            if(playing) {
                updateCurrentTime();
            }
            delete frame;
        }
    } 
    return interval;
}

double MainWindow::presentVideoFrame(MPVideoFrame *frame)
{
    displayRBGFrame((char *)frame->databuffer, frame->width, frame->height);
    _moviePosition = frame->position;
    return frame->duration;
}
~~~
