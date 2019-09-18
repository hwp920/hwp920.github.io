title: FFMpeg QT跨平台播放器五 音量调节和seek操作
author: Cyrus
tags:
  - 播放器
categories:
  - 音视频
date: 2019-09-18 23:11:00
---
### 一、音量调节
QT的音量调节是通过**void QAudioOutput::setVolume(qreal volume)** 方法实现的，所以结合QSlider，可以这样写：
~~~
// slider的范围为0-100， volume的范围为0-1
connect(ui->volumSlider, &QSlider::valueChanged,
            [=](int value) {
                audioOut->setVolume(value / 100.0);
    });
~~~

### 二、seek
FFMpeg的seek主要是能过 **int avformat_seek_file(AVFormatContext *s, int stream_index, int64_t min_ts, int64_t ts, int64_t max_ts, int flags); **实现。算出seek时间相对于音频/视频TimeBase的值并调用即可；
~~~
void MPMovieDecoder::setPosition(float seconds)
{
    _position = seconds;
    isEOF = false;

    if(_videoStream != -1) {
        int64_t ts = (int64_t)(seconds / _videoTimeBase);
        avformat_seek_file(_formatCtx, _videoStream, ts, ts, ts, AVSEEK_FLAG_FRAME);
        avcodec_flush_buffers(_videoCodecCtx);
    }

    if (_audioStream != -1) {
        int64_t ts = (int64_t)(seconds / _audioTimeBase);
        avformat_seek_file(_formatCtx, _audioStream, ts, ts, ts, AVSEEK_FLAG_FRAME);
        avcodec_flush_buffers(_audioCodecCtx);
    }
}
~~~

对于seek，我们通常会与slider配合使用，所以可以这样写：
~~~
// 准备拉slider的时候，停止播放
connect(ui->processSlider, &QSlider::sliderPressed, this, &MainWindow::onProcessSliderPress);

// 这里不用valueChange信号，主要是因为会与播放进度显示冲突
    connect(ui->processSlider, &QSlider::sliderReleased, this, &MainWindow::onProcessSliderRelease);
~~~

~~~
// 停止播放并记下当前是播放还是暂停，决定滑动后的状态
void MainWindow::onProcessSliderPress()
{
    playMode = playing;
    playing = false;
}
~~~

~~~
void MainWindow::onProcessSliderRelease()
{
	//获取slider的值，计算相对于文件总时长的时间，这里是之前停止播放有关，不停止播放，更新播放进度会回跳
    float total = decoder->getDuration();
    float position = total * ui->processSlider->value() / 100.0;
    setMoviePositon(position);
}

void MainWindow::setMoviePositon(float seconds)
{
    m_processTimer = new QTimer(this);
    connect(m_processTimer, &QTimer::timeout,
            [=](){
        updatePosition(seconds, playMode);
    });
    m_processTimer->start(100);
}

void MainWindow::updatePosition(float position, bool playMode)
{

    if(m_processTimer->isActive()) {
        m_processTimer->stop();
    }
    if (m_processTimer != NULL) {
        delete  m_processTimer;
        m_processTimer = NULL;
    }
	// 清除缓存的数据
    freeBufferedFrames();
    float tmp = position > 0 ? position : 0;
    position = (decoder->getDuration() - 1 ) > tmp ? tmp : decoder->getDuration() - 1;
    qDebug() << "updatePosition " << position << endl;

    if(playMode) {
        decoder->setPosition(position);
        _moviePosition = decoder->getPosition();
        updateCurrentTime();
        play();
    } else {
        decoder->setPosition(position);
        decodeFrames();
        _moviePosition = decoder->getPosition();
        updateCurrentTime();
        presentFrame();
    }
}

void MainWindow::updateCurrentTime()
{
    ui->currentTime->setText(formatTime((int)_moviePosition));
    int value = (_moviePosition / (decoder->getDuration()) * 100);
    ui->processSlider->setValue(value);
}

~~~