### 一、实现工具
1. FFmpeg负责解码
2. GLES+GLSurfaceView负责渲染
### 二、播放器实现流程图
> #### 1. FFmpeg解码流程:

![](http://file.jinxianyun.com/ffmpeg_decoder.png)
[by 雷霄骅](https://blog.csdn.net/leixiaohua1020/article/details/50534150?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522161950611716780261974067%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=161950611716780261974067&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v2~rank_v29-3-50534150.nonecase&utm_term=%E9%9F%B3%E8%A7%86%E9%A2%91%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B)

> #### 2. FFmpeg解码具体流程:
1. 创建封装格式上下文
2. 打开输入文件，解封装
3. 获取音视频流信息
4. 获取音视频流索引
5. 获取解码器参数
6. 根据codec_id获取解码器
7. 创建解码器上下文
8. 打开解码器
9. 创建存储编码数据和解码数据的结构体
10. 解码循环

> #### 3. 渲染流程
1. 将解码后的帧数据加载到内存
2. 取帧数据用GLES实现GLSurface.Renderer的onSurfaceCreated/onSurfaceChanged/onDrawFrame
3. Android JNI调准备好的方法


### 三、编码流程
1. 初始化解码

> DecoderBase.cpp
```c++
void DecoderBase::InitFFDecoder() {
     //1.创建封装格式上下文
        m_AVFormatContext = avformat_alloc_context();

        //2.打开文件
        if(avformat_open_input(&m_AVFormatContext, m_Url, NULL, NULL) != 0)
        {
            LOGCATE("DecoderBase::InitFFDecoder avformat_open_input fail.");
            break;
        }

        //3.获取音视频流信息
        if(avformat_find_stream_info(m_AVFormatContext, NULL) < 0) {
            LOGCATE("DecoderBase::InitFFDecoder avformat_find_stream_info fail.");
            break;
        }

        //4.获取音视频流索引
        for(int i=0; i < m_AVFormatContext->nb_streams; i++) {
            if(m_AVFormatContext->streams[i]->codecpar->codec_type == m_MediaType) {
                m_StreamIndex = i;
                break;
            }
        }

        if(m_StreamIndex == -1) {
            LOGCATE("DecoderBase::InitFFDecoder Fail to find stream index.");
            break;
        }
        //5.获取解码器参数
        AVCodecParameters *codecParameters = m_AVFormatContext->streams[m_StreamIndex]->codecpar;

        //6.获取解码器
        m_AVCodec = avcodec_find_decoder(codecParameters->codec_id);
        if(m_AVCodec == nullptr) {
            LOGCATE("DecoderBase::InitFFDecoder avcodec_find_decoder fail.");
            break;
        }

        //7.创建解码器上下文
        m_AVCodecContext = avcodec_alloc_context3(m_AVCodec);
        if(avcodec_parameters_to_context(m_AVCodecContext, codecParameters) != 0) {
            LOGCATE("DecoderBase::InitFFDecoder avcodec_parameters_to_context fail.");
            break;
        }

        //8.打开解码器
        result = avcodec_open2(m_AVCodecContext, m_AVCodec, NULL);
        if(result < 0) {
            LOGCATE("DecoderBase::InitFFDecoder avcodec_open2 fail. result=%d", result);
            break;
        }
        result = 0;

        m_Duration = m_AVFormatContext->duration / AV_TIME_BASE * 1000;//us to ms
        //创建 AVPacket 存放编码数据
        m_Packet = av_packet_alloc();
        //创建 AVFrame 存放解码后的数据
        m_Frame = av_frame_alloc();
}
```

2. 循环解码
> DecoderBase.cpp
```c++
void DecoderBase::DecodingLoop() {
    ...
    for(;;) {
        ...

        if(DecodeOnePacket() != 0) {
            //解码结束，暂停解码器
            std::unique_lock<std::mutex> lock(m_Mutex);
            m_DecoderState = STATE_PAUSE;
        }
    }
    LOGCATE("DecoderBase::DecodingLoop end");
}

```
3. 读数据包

> DecoderBase.cpp

```c++

int DecoderBase::DecodeOnePacket() {
    int result = av_read_frame(m_AVFormatContext, m_Packet);
    while(result == 0) {
        if(m_Packet->stream_index == m_StreamIndex) {

            if(avcodec_send_packet(m_AVCodecContext, m_Packet) == AVERROR_EOF) {
                //解码结束
                result = -1;
                goto __EXIT;
            }

            //一个 packet 包含多少 frame?
            int frameCount = 0;
            while (avcodec_receive_frame(m_AVCodecContext, m_Frame) == 0) {
                //更新时间戳
                UpdateTimeStamp();
                //同步
                AVSync();
                //渲染
                LOGCATE("DecoderBase::DecodeOnePacket 000 m_MediaType=%d", m_MediaType);
                OnFrameAvailable(m_Frame);
                LOGCATE("DecoderBase::DecodeOnePacket 0001 m_MediaType=%d", m_MediaType);
                frameCount ++;
            }
            LOGCATE("BaseDecoder::DecodeOneFrame frameCount=%d", frameCount);
            //判断一个 packet 是否解码完成
            if(frameCount > 0) {
                result = 0;
                goto __EXIT;
            }
        }
        av_packet_unref(m_Packet);
        result = av_read_frame(m_AVFormatContext, m_Packet);
    }

    __EXIT:
    av_packet_unref(m_Packet);
    return result;
}
```

4. 将解码后的数据交给GLRender
> VideoDecoder.cpp
```c++
void VideoDecoder::OnFrameAvailable(AVFrame *frame) {
    if(m_VideoRender != nullptr && frame != nullptr) {
        NativeImage image;
        if(GetCodecContext()->pix_fmt == AV_PIX_FMT_YUV420P || GetCodecContext()->pix_fmt == AV_PIX_FMT_YUVJ420P) {
            image.format = IMAGE_FORMAT_I420;
            image.width = frame->width;
            image.height = frame->height;
            image.pLineSize[0] = frame->linesize[0];
            image.pLineSize[1] = frame->linesize[1];
            image.pLineSize[2] = frame->linesize[2];
            image.ppPlane[0] = frame->data[0];
            image.ppPlane[1] = frame->data[1];
            image.ppPlane[2] = frame->data[2];
            if(frame->data[0] && frame->data[1] && !frame->data[2] && frame->linesize[0] == frame->linesize[1] && frame->linesize[2] == 0) {
                // on some android device, output of h264 mediacodec decoder is NV12 兼容某些设备可能出现的格式不匹配问题
                image.format = IMAGE_FORMAT_NV12;
            }
        } else {
            ...
        }

        m_VideoRender->RenderVideoFrame(&image);
    }

    if(m_MsgContext && m_MsgCallback)
        m_MsgCallback(m_MsgContext, MSG_REQUEST_RENDER, 0);
}
```

5. GLRender将NativeImage数据拷贝存在m_RenderImage

> VideoGLRender.cpp
```c++
void VideoGLRender::RenderVideoFrame(NativeImage *pImage) {
    LOGCATE("VideoGLRender::RenderVideoFrame pImage=%p", pImage);
    if(pImage == nullptr || pImage->ppPlane[0] == nullptr)
        return;
    std::unique_lock<std::mutex> lock(m_Mutex);
    if (pImage->width != m_RenderImage.width || pImage->height != m_RenderImage.height) {
        if (m_RenderImage.ppPlane[0] != nullptr) {
            NativeImageUtil::FreeNativeImage(&m_RenderImage);
        }
        memset(&m_RenderImage, 0, sizeof(NativeImage));
        m_RenderImage.format = pImage->format;
        m_RenderImage.width = pImage->width;
        m_RenderImage.height = pImage->height;
        NativeImageUtil::AllocNativeImage(&m_RenderImage);
    }

    NativeImageUtil::CopyNativeImage(pImage, &m_RenderImage);
}
```
6. 实现GLSurface.Renderer的onDrawFrame(将m_RenderImage数据进行GLES绘制)

> VideoGLRender.cpp
```c++
void VideoGLRender::OnDrawFrame() {
    glClear(GL_COLOR_BUFFER_BIT);
    if(m_ProgramObj == GL_NONE|| m_RenderImage.ppPlane[0] == nullptr) return;

    m_FrameIndex++;

    // upload image data
    std::unique_lock<std::mutex> lock(m_Mutex);
    switch (m_RenderImage.format)
    {
        case IMAGE_FORMAT_RGBA:
            glActiveTexture(GL_TEXTURE0);
            glBindTexture(GL_TEXTURE_2D, m_TextureIds[0]);
            glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, m_RenderImage.width, m_RenderImage.height, 0, GL_RGBA, GL_UNSIGNED_BYTE, m_RenderImage.ppPlane[0]);
            glBindTexture(GL_TEXTURE_2D, GL_NONE);
            break;
        ...
        default:
            break;
    }
    lock.unlock();


    // Use the program object
    glUseProgram (m_ProgramObj);

    glBindVertexArray(m_VaoId);

    for (int i = 0; i < TEXTURE_NUM; ++i) {
        glActiveTexture(GL_TEXTURE0 + i);
        glBindTexture(GL_TEXTURE_2D, m_TextureIds[i]);
        char samplerName[64] = {0};
        sprintf(samplerName, "s_texture%d", i);
        GLUtils::setInt(m_ProgramObj, samplerName, i);
    }

    float offset = (sin(m_FrameIndex * MATH_PI / 40) + 1.0f) / 2.0f;
    GLUtils::setFloat(m_ProgramObj, "u_Offset", offset);
    GLUtils::setVec2(m_ProgramObj, "u_TexSize", vec2(m_RenderImage.width, m_RenderImage.height));
    GLUtils::setInt(m_ProgramObj, "u_nImgType", m_RenderImage.format);

    glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_SHORT, (const void *)0);

}
```

7. Android端GLSurfaceView实现GLSurfaceView.Renderer的onSurfaceCreated/onSurfaceChanged/onDrawFrame

> LoginActivity.kt
```kotlin
class LoginActivity : GLSurfaceView.Renderer, FFMediaPlayer.EventCallback {
    override fun onSurfaceCreated(gl10: GL10?, eglConfig: EGLConfig?) {
        FFMediaPlayer.native_OnSurfaceCreated(VIDEO_GL_RENDER)

    }

    override fun onSurfaceChanged(gl10: GL10?, w: Int, h: Int) {
        Log.d(TAG, "onSurfaceChanged() called with: gl10 = [$gl10], w = [$w], h = [$h]")
        FFMediaPlayer.native_OnSurfaceChanged(VIDEO_GL_RENDER, w, h)
    }

    override fun onDrawFrame(gl10: GL10?) {
        FFMediaPlayer.native_OnDrawFrame(VIDEO_GL_RENDER)
    }
        
}
```

8. 写java调c的桥接文件
> learn-ffmpeg.cpp
```c++
extern "C" JNIEXPORT void JNICALL
Java_com_jinxian_wenshi_media_FFMediaPlayer_00024Companion_native_1OnSurfaceCreated(JNIEnv *env,
                                                                                    jobject thiz,
                                                                                    jint render_type) {
    switch (render_type) {
        case VIDEO_GL_RENDER:
            VideoGLRender::GetInstance()->OnSurfaceCreated();
            break;
        case AUDIO_GL_RENDER:
            AudioGLRender::GetInstance()->OnSurfaceCreated();
            break;
        default:
            break;
    }
}

extern "C" JNIEXPORT void JNICALL
Java_com_jinxian_wenshi_media_FFMediaPlayer_00024Companion_native_1OnSurfaceChanged(JNIEnv *env,
                                                                                    jobject thiz,
                                                                                    jint render_type,
                                                                                    jint width,
                                                                                    jint height) {
    switch (render_type) {
        case VIDEO_GL_RENDER:
            VideoGLRender::GetInstance()->OnSurfaceChanged(width, height);
            break;
        case AUDIO_GL_RENDER:
            AudioGLRender::GetInstance()->OnSurfaceChanged(width, height);
            break;
        default:
            break;
    }
}
extern "C" JNIEXPORT void JNICALL
Java_com_jinxian_wenshi_media_FFMediaPlayer_00024Companion_native_1OnDrawFrame(JNIEnv *env,
                                                                               jobject thiz,
                                                                               jint render_type) {
    switch (render_type) {
        case VIDEO_GL_RENDER:
            VideoGLRender::GetInstance()->OnDrawFrame();
            break;
        case AUDIO_GL_RENDER:
            AudioGLRender::GetInstance()->OnDrawFrame();
            break;
        default:
            break;
    }
}
```

### 四、成果预览
![image](http://file.jinxianyun.com/ffmpeg_player.gif)
[image](http://file.jinxianyun.com/ffmpeg_player.gif)

### 五、[github](https://github.com/qq326646683/wenshi_android)

### 六、参考
https://juejin.cn/post/6863316927121621000

---
完结，撒花🎉


