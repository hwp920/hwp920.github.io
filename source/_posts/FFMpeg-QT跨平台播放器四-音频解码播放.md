title: FFMpeg QT跨平台播放器四 音频解码播放
author: Cyrus
tags:
  - 播放器
categories:
  - 音视频
date: 2019-09-18 21:54:00
---
### 一、解码
音频解码的步骤跟视频类型，都是AVPaket->AVFrame再根据需要进行调整
~~~
QVector<MPVideoFrame *> *MPMovieDecoder::decodeFrames(float minDuration)
{
    if (_videoStream == -1 && _audioStream == -1) return NULL;
    QVector<MPVideoFrame *> *result = new QVector<MPVideoFrame *>();

    AVPacket packet;
    float decodedDuration = 0;

    bool finished = false;
    while (!finished) {
        if (av_read_frame(_formatCtx, &packet) < 0) {
            isEOF = true;
            break;
        }

        if (packet.stream_index == _videoStream) {
			// 视频解码，看上一篇
        }

        if(packet.stream_index == _audioStream) {
            int pktSize = packet.size;
            while (pktSize > 0) {

                int gotFrame = 0;

                int len = avcodec_decode_audio4(_audioCodecCtx, _audioFrame, &gotFrame, &packet);
                if (len < 0) {
                    qDebug() << "decode audio error, skip packet" << endl;
                    break;
                }

                if (gotFrame) {

                    qDebug() << "decode audio gotFrame" << endl;
                    MPVideoFrame *frame = handleAudioFrame();
                    if (frame != NULL) {
                        result->append(frame);
                    }

                    if (_videoStream == -1) {
                        _position = frame->position;
                        decodedDuration += frame->duration;
                        if (decodedDuration > minDuration) finished = true;
                    }
                }
                if (0 == len) break;
                pktSize -= len;
            }
        }

        av_free_packet(&packet);
    }

    qDebug() << "decode end" << endl;
    return result;
}
~~~

~~~
MPVideoFrame *MPMovieDecoder::handleAudioFrame()
{
    if (_audioFrame->data[0] == NULL) {
        qDebug() << "_audioFrame->data[0] no data" << endl;
        return NULL;
    }
	
    // 重采样上下文配置可以看第二篇分流
    if (_swrContext == NULL) {
        qDebug() << "_swrContext == NULL" << endl;
        return NULL;
    } else {
        qDebug() << "before swr_convert" << endl;

        int out_channels = av_get_channel_layout_nb_channels(AV_CH_LAYOUT_STEREO);
        int out_buffer_size = av_samples_get_buffer_size(NULL,out_channels ,_audioFrame->nb_samples,AV_SAMPLE_FMT_S16, 0);//计算转换后数据大小
        uint8_t *audio_out_buffer = (uint8_t *)av_malloc(out_buffer_size);//申请输出缓冲区

        int len = swr_convert(_swrContext,&audio_out_buffer, out_buffer_size,(const uint8_t **)_audioFrame->data , _audioFrame->nb_samples);
        if (len <= 0)
        {
            av_free(audio_out_buffer);
            return NULL;
        }
        int resampled_data_size = len * 2  * av_get_bytes_per_sample(AV_SAMPLE_FMT_S16);//每声道采样数 x 声道数 x 每个采样字节数
        MPVideoFrame *frame = new MPVideoFrame();
        frame->type = MPMovieFrameTypeAudio;
        frame->bufferSize = resampled_data_size;
        frame->databuffer = audio_out_buffer;
        frame->position = av_frame_get_best_effort_timestamp(_audioFrame) * _audioTimeBase;
        frame->duration = av_frame_get_pkt_duration(_audioFrame) * _audioTimeBase;


        if (frame->duration == 0) {
            frame->duration = frame->bufferSize / (sizeof(float) * 2 * 44100);
        }

       return frame;
    }

    return NULL;
}

// 主要是重采样的参数要传对，最终的数据大小要算准
~~~

### 二、播放
先说一下QT对于pcm数据的播放方法：
* 1、QIODevice负责提供用于播放的数据，所以我们的pcm数据都会写到QIODevice中；
* 2、QAudioOutput根据设定的QAudioFormat,播放QIODevice提供的数据。

由于人对音频比较敏感，控制不好的话，杂音频比较严重。借鉴https://blog.csdn.net/jklinux/article/details/72620102 我们自己继承QIODevice创建一下提供pcm数据的IODevice,最大可能的保证音频数据的连续性。

~~~
/// MPAudioDevice.h

#ifndef MPAUDIODEVICE_H
#define MPAUDIODEVICE_H

#include <QIODevice>
#include <QMutex>

class MPAudioDevice : public QIODevice
{
    Q_OBJECT
public:
    MPAudioDevice(); //创建对象传递pcm数据
   ~MPAudioDevice();

   void appendData(char *data, qint64 len);
   qint64 readData(char *data, qint64 maxlen); //重新实现的虚函数
   qint64 writeData(const char *data, qint64 len); //它是个纯虚函数， 不得不实现
   qint64 bytesAvailable() const;
private:
    QMutex m_audioMutex;
    QByteArray data_pcm; //存放pcm数据
    int        len_written; //记录已写入多少字节
};

#endif // MPAUDIODEVICE_H
~~~

~~~
/// MPAudioDevice.cpp

#include "mpaudiodevice.h"
#include <QDebug>

MPAudioDevice::MPAudioDevice()
{
    this->open(QIODevice::ReadOnly); // 为了解决QIODevice::read (QIODevice): device not open
    len_written = 0;
}

MPAudioDevice::~MPAudioDevice()
{
    this->close();
}

qint64 MPAudioDevice::readData(char *data, qint64 maxlen) // data为声卡的数据缓冲区地址， maxlen为声卡缓冲区最大能存放的字节数
{
//    qDebug() << "MPAudioDevice::readData" << endl;

    if(data_pcm.size() == 0) return 0;

    int len;

    //计算未播放的数据的长度
    len = maxlen > data_pcm.size() ? data_pcm.size() : maxlen;

    memcpy(data, data_pcm.data(), len); //把要播放的pcm数据存入声卡缓冲区里
    {
        QMutexLocker locker(&m_audioMutex);
        data_pcm.remove(0, len);
    }

    return len;
}

qint64 MPAudioDevice::writeData(const char *data, qint64 len)
{

}

void MPAudioDevice::appendData(char *data, qint64 len)
{
    QMutexLocker locker(&m_audioMutex);
    data_pcm.append(data, len);
}

qint64 MPAudioDevice::bytesAvailable() const
{
    return data_pcm.size() + QIODevice::bytesAvailable();
}
~~~

具体的播放
~~~
	// 1、配置audioFormat
    QAudioFormat fmt;//设置音频输出格式
    fmt.setSampleRate(44100);//1秒的音频采样率
    fmt.setSampleSize(16);//声音样本的大小
    fmt.setChannelCount(2);//声道
    fmt.setCodec("audio/pcm");//解码格式
    fmt.setByteOrder(QAudioFormat::LittleEndian);
    fmt.setSampleType(QAudioFormat::SignedInt);//设置音频类型
    QAudioDeviceInfo info = QAudioDeviceInfo::defaultOutputDevice();
    if (!info.isFormatSupported(fmt)) {
        qDebug() << "Raw audio format not supported by backend, cannot play audio.";
        fmt = info.nearestFormat(fmt);
    }
    
    // 2、 创建AudioOutput
    QAudioOutput *audioOut = new QAudioOutput(info, fmt, this);
    audioOut->setVolume(ui->volumSlider->value() / 100.0);
	
    // 3、 关联IODevice和AudioOutput
    audioDevice = new MPAudioDevice();
    audioOut->start(audioDevice);
    
    
    // 4、每次解码完音频数据后，就加到AudioDevice的播放缓冲里
    bool MainWindow::addFrames(QVector<MPVideoFrame *> *frames)
{
    if (decoder->validVideo()) {
		// 视频看上一篇
    }

    if (decoder->validAudio()) {
        QMutexLocker locker(&m_audioMutex);
        for (QVector<MPVideoFrame *>::iterator it = frames->begin(); it != frames->end(); ++it)
        {
            if ((*it)->type == MPMovieFrameTypeAudio) {
                audioDevice->appendData((char *)((*it)->databuffer), (*it)->bufferSize);
            }

        }
    }
    delete frames;
    return playing && _bufferedDuration < _maxBufferedDuration;
}

// 缺点：音视频同步不好。
~~~