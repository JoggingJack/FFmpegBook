# 仿VLC接收RTSP流并播放

[这里下载直接可运行的源码，下载后有任何问题可加作者微信](https://mbd.pub/o/bread/mbd-ZZiXmZ9s)

## 效果

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/%E6%92%AD%E6%94%BERTSP.gif)

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/%E9%AA%8C%E8%AF%81%E6%92%AD%E6%94%BE.gif)

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/%E9%AA%8C%E8%AF%81%E9%94%99%E8%AF%AF.gif)

## 产生RTSP流

比播放文件复杂一点是，为了接收`RTSP`流，我们需要产生`RTSP`流。简单搭建一个`RTSP`推流环境：

用`EasyDarwin`开启`RTSP`服务作为`RTSP`服务器。

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/easydarwin.jpg)

用`ffmpeg`命令行作为客户端，向`EasyDarwin`循环推送一个视频文件。

~~~bash
./ffmpeg.exe -re -stream_loop -1 -i test.mp4 -c copy -f rtsp rtsp://127.0.0.1/stream
~~~

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/ffmpeg%E7%AE%80%E5%8D%95%E6%8E%A8%E6%B5%81.png)

这样就可以从EasyDarwin接收RTSP流了。

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/%E6%8E%A8%E6%B5%81%E5%85%B3%E7%B3%BB.png)

我们用`vlc`接收`RTSP`流看看。

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/vlc%E6%92%AD%E6%94%BErtsp%E6%B5%81.gif)

成功接收。

## FFmepg接收RTSP流代码

用`FFmpeg`接收`RTSP`流并播放的流程和播放`mp4`文件的流程差不多，只不过播放`mp4`文件时，文件作为播放源，而接收`RTSP`流时，`RTSP`流作为了播放源：

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E7%A8%8B.drawio.png)

我们依旧看下流程中的关键代码：

~~~C++
if (avformat_open_input(&fileFmtCtx, url.toStdString().c_str(), nullptr, nullptr) != 0) {
    qDebug() << "avformat_open_input() failed";
    return;
}
~~~

用于打开一个RTSP地址，跟打开一个文件相比，不仅要查找流信息，还需要和`RTSP`服务器建立连接，让`RTSP`服务器开始推流。

接收上述`RTSP`流后，我们打印`AVFormatContext`的相关属性：

~~~C++
qDebug() << "stream name: " << streamFmtCtx->url;
    qDebug() << "stream iformat: " << streamFmtCtx->iformat->name;
    qDebug() << "stream duration: " << streamFmtCtx->duration << " microseconds";
    qDebug() << "stream bit_rate: " << streamFmtCtx->bit_rate;
/* 
stream name:  rtsp://127.0.0.1/stream
stream iformat:  rtsp
stream duration:  -9223372036854775808  microseconds
stream bit_rate:  0
*/
~~~

这次由于是`RTSP`流，并不能获取准确的`duration`。继续打印流相关的信息：

~~~C++
qDebug() << "nb_streams:";
    for (unsigned int i = 0; i < streamFmtCtx->nb_streams; i++) {
        AVStream *stream = streamFmtCtx->streams[i];
        qDebug() << "Stream " << i + 1 << ":";
        qDebug() << "  Codec: " << avcodec_get_name(stream->codecpar->codec_id);
        qDebug() << "  Duration: " << stream->duration << " microseconds";
    }
/*
nb_streams:
Stream  1 :
  Codec:  h264
  Duration:  -9223372036854775808  microseconds
Stream  2 :
  Codec:  aac
  Duration:  -9223372036854775808  microseconds
*/
~~~

可以看到和上次直接读取文件的结果一样，包括1个`H264`视频流和1个`AAC`音频流。

~~~C++
swsCtx = sws_getContext(decoderCtx->width, decoderCtx->height, decoderCtx->pix_fmt,
                                     decoderCtx->width, decoderCtx->height, FMT_PIC_SHOW,
                                     SWS_BICUBIC, NULL, NULL, NULL);
qDebug() << "decoderCtx->pix_fmt:" << av_get_pix_fmt_name(decoderCtx->pix_fmt);
//decoderCtx->pix_fmt: yuv420p
~~~

`sws_getContext()`用于将`RTSP`流格式转换为将要显示的格式，这里是`yuv420p`=>`AV_PIX_FMT_RGB24`。

~~~C++
int numBytes = av_image_get_buffer_size(FMT_PIC_SHOW, decoderCtx->width, decoderCtx->height, 1);
showBuffer = (unsigned char*)av_malloc(static_cast<unsigned long long>(numBytes) * sizeof(unsigned char));
if(av_image_fill_arrays(showFrame->data, showFrame->linesize,
                        showBuffer, FMT_PIC_SHOW, decoderCtx->width, decoderCtx->height, 1) < 0)
{
    qDebug() << "av_image_fill_arrays() failed";
    return;
}
~~~

`av_image_get_buffer_size`计算了计算图像数据的缓冲区大小。`av_malloc`分配了1个内存块给`showBuffer`。`av_image_fill_arrays`用图像参数和`showBuffer`初始化`AVFrame`的`data`和`linesize`成员，并且让`AVFrame`和`showBuffer`关联。

~~~C++
while(av_read_frame(streamFmtCtx, packet) >= 0){
    if(packet->stream_index == nVideoIndex){
        if(avcodec_send_packet(decoderCtx, packet)>=0){
            while((ret = avcodec_receive_frame(decoderCtx, decodedFrame)) >= 0){
                //...
            }
        }
    }
}
~~~

和播放`mp4`文件类似的解码步骤，从`RTSP`流中读取一个数据包`AVPacket`，将`AVPacket`送入解码器进行解码，尝试从解码器中接收已解码的视频帧，并将接收到的帧数据存储在`decodedFrame`中。

经过上述基本步骤，我们的代码已经可以和`VLC`一样，从RTSP服务器接收`RTSP`流并播放了。

## RTSP协议简述及验证

`FFmpeg`内部将RTSP连接建立处理得很好，但我们有必要进一步学习一下`RTSP`协议。`RTSP`全称`Real Time Sreaming Protocol`，是`TCP/IP`协议体系中的一个应用层协议。数据传输由`RTP/RTCP`完成，底层通过`TCP/UDP`实现。

一个标准的RTSP的收流协议层的交互流程如下：

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/%E5%8D%8F%E8%AE%AE%E6%B5%81%E7%A8%8B%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

话不多说，我们直接在上面的推流环境下（由于`EasyDarwin`似乎加密了某些信息，我们选择了一个其他的`RTSP`服务器，效果是一样的），用`VLC`收流，并用`wireshark`抓包看看协议流程是不是这样的：

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/vlc%E6%8E%A5%E6%94%B6%E6%8A%93%E5%8C%85.jpg)

直接看看每条信息都是什么：

client => server

~~~http
Real Time Streaming Protocol
    Request: OPTIONS rtsp://127.0.0.1:554/stream RTSP/1.0\r\n
        Method: OPTIONS
        URL: rtsp://127.0.0.1:554/stream
    CSeq: 2\r\n
    User-Agent: LibVLC/3.0.18 (LIVE555 Streaming Media v2016.11.28)\r\n
    \r\n
~~~

client发送`OPTIONS`向`rtsp://127.0.0.1:554/stream`询问server支持哪些`RTSP`方法。

server=> client

~~~http
Real Time Streaming Protocol
    Response: RTSP/1.0 200 OK\r\n
        Status: 200
    CSeq: 2\r\n
    Session: 4J_bOCNSg
    Public: DESCRIBE, SETUP, TEARDOWN, PLAY, PAUSE, OPTIONS, ANNOUNCE, RECORD\r\n
    \r\n
~~~

server回复支持`DESCRIBE, SETUP, TEARDOWN, PLAY, PAUSE, OPTIONS, ANNOUNCE, RECORD`

client => server

~~~http
Real Time Streaming Protocol
    Request: DESCRIBE rtsp://127.0.0.1:554/stream RTSP/1.0\r\n
        Method: DESCRIBE
        URL: rtsp://127.0.0.1:554/stream
    CSeq: 3\r\n
    User-Agent: LibVLC/3.0.18 (LIVE555 Streaming Media v2016.11.28)\r\n
    Accept: application/sdp\r\n
    \r\n
~~~

client请求媒体描述文件，格式为`application/sdp`。

一般server会进行用户认证，如果未携带Authorization鉴权信息，或者认证失败，server会返回错误号为401的响应，client接收到401响应时，需要根据已知的用户鉴权信息，生成Authorization，再次发送`DESCRIBE`，如果认证成功，服务器返回携带有`SDP`的响应信息。

是否进行认证和`RTSP`服务器有关，这里我们没有为`EasyDarwin`设置认证。

server=> client

~~~http
Real Time Streaming Protocol
    Response: RTSP/1.0 200 OK\r\n
    CSeq: 3\r\n
    Session: _ZLZ7_NSR
    Content-type: application/sdp
    Content-length: 511
    \r\n
    Session Description Protocol
        Session Description Protocol Version (v): 0
        Owner/Creator, Session Id (o): - 0 0 IN IP4 127.0.0.1
        Session Name (s): No Name
        Connection Information (c): IN IP4 127.0.0.1
        Time Description, active time (t): 0 0
        Session Attribute (a): tool:libavformat 58.76.100
        Media Description, name and address (m): video 0 RTP/AVP 96
        Bandwidth Information (b): AS:1894
        Media Attribute (a): rtpmap:96 H264/90000
        Media Attribute (a): fmtp:96 packetization-mode=1; sprop-parameter-sets=Z2QAKqwspQFAFumoCAgKAAADAAIAAAMAYcTAAc/YABW+f4xwEA==,aOkJNSU=; profile-level-id=64002A
        Media Attribute (a): control:streamid=0
        Media Description, name and address (m): audio 0 RTP/AVP 97
        Bandwidth Information (b): AS:317
        Media Attribute (a): rtpmap:97 MPEG4-GENERIC/48000/2
        Media Attribute (a): fmtp:97 profile-level-id=1;mode=AAC-hbr;sizelength=13;indexlength=3;indexdeltalength=3; config=1190
        Media Attribute (a): control:streamid=1
~~~

server返回`SDP`信息，告诉client当前有哪些音视频流和属性，sdp协议不做展开。这里我们要关注的比较重要的信息是：server可以发送`streamid=0`的`H264`视频流和`streamid=1`的`AAC`音频流。

client => server

~~~http
Real Time Streaming Protocol
    Request: SETUP rtsp://127.0.0.1:554/stream/streamid=0 RTSP/1.0\r\n
        Method: SETUP
        URL: rtsp://127.0.0.1:554/stream/streamid=0
    CSeq: 4\r\n
    User-Agent: LibVLC/3.0.18 (LIVE555 Streaming Media v2016.11.28)\r\n
    Transport: RTP/AVP;unicast;client_port=52024-52025
    \r\n
~~~

client发送`SETUP`告诉server需要建立`streamid=0`即视频流的连接，这里`RTP/AVP`表示通过`UDP`传输，`unicast`表示单播，`client_port=52024-52025`需要单独解释一下，前面说到`RTSP`协议数据传输通过`RTP+RTCP`完成。`RTP`和`RTCP`都是建立在`UDP`之上的，`RTP`默认使用1个偶数端口号，而`RTCP`则默认使用`RTP`端口的下1个奇数端口号，就是这里的52024和52025。

server => client

~~~http
Real Time Streaming Protocol
    Response: RTSP/1.0 200 OK\r\n
        Status: 200
    CSeq: 4\r\n
    Session: 4J_bOCNSg
    Transport: RTP/AVP;unicast;client_port=52024-52025
    \r\n

~~~

server向client返回确认。

client => server

~~~http
Real Time Streaming Protocol
    Request: SETUP rtsp://127.0.0.1:554/stream/streamid=1 RTSP/1.0\r\n
        Method: SETUP
        URL: rtsp://127.0.0.1:554/stream/streamid=1
    CSeq: 5\r\n
    User-Agent: LibVLC/3.0.18 (LIVE555 Streaming Media v2016.11.28)\r\n
    Transport: RTP/AVP;unicast;client_port=52028-52029
    Session: 4J_bOCNSg
    \r\n
~~~

client告诉server需要建立`streamid=1`的音频流的连接，`RTP`和`RTCP`的端口分别在52028和52029。

server => client

~~~http
Real Time Streaming Protocol
    Response: RTSP/1.0 200 OK\r\n
        Status: 200
    Transport: RTP/AVP;unicast;client_port=52028-52029
    CSeq: 5\r\n
    Session: 4J_bOCNSg
    \r\n
~~~

server向client返回确认。

client=>server

~~~http
Real Time Streaming Protocol
    Request: PLAY rtsp://127.0.0.1:554/stream RTSP/1.0\r\n
        Method: PLAY
        URL: rtsp://127.0.0.1:554/stream
    CSeq: 6\r\n
    User-Agent: LibVLC/3.0.18 (LIVE555 Streaming Media v2016.11.28)\r\n
    Session: 4J_bOCNSg
    Range: npt=0.000-\r\n
    \r\n
~~~

client发送`PLAY`告诉server开始传输，`Range`代表媒体播放时间，server会根据`Range`的值播放指定段的数据流，对于实时流，一般只会指定起点，即`Range: npt=0.000-`

server=>client

~~~http
Real Time Streaming Protocol
    Response: RTSP/1.0 200 OK\r\n
        Status: 200
    CSeq: 6\r\n
    Session: 4J_bOCNSg
    Range: npt=0.000-\r\n
    \r\n
~~~

server返回确认，使用同一`Session`。

client=>server

~~~http
Real Time Streaming Protocol
    Request: TEARDOWN rtsp://127.0.0.1:554/stream RTSP/1.0\r\n
        Method: TEARDOWN
        URL: rtsp://127.0.0.1:554/stream
    CSeq: 7\r\n
    User-Agent: LibVLC/3.0.18 (LIVE555 Streaming Media v2016.11.28)\r\n
    Session: 4J_bOCNSg
    \r\n
~~~

client发送`TEARDOWN`发起停止流传输请求。

server=>client

~~~http
Real Time Streaming Protocol
    Response: RTSP/1.0 200 OK\r\n
        Status: 200
    CSeq: 7\r\n
    Session: 4J_bOCNSg
    \r\n
~~~

server返回确认，使用同一`Session`，停止流传输。

## 搭建摘要认证环境

上面说到了server可能会进行用户认证，那我们现在得创造一个需要认证的环境，直接看看`EasyDarwin`能不能直接选择认证，打开`easydarwin.ini`：

~~~ini
[http]
port=10008
default_username=admin
default_password=admin
#...
;是否使能向服务器推流或者从服务器播放时验证用户名密码. [注意] 因为服务器端并不保存明文密码，所以推送或者播放时，客户端应该输入密码的md5后的值。
;password should be the hex of md5(original password)
authorization_enable=0
#...
~~~

可以看到`authorization_enable`变量是控制认证的，把它的值改为1，重新启动服务。这时候发现原来的`ffmpeg`命令推流不成功了。

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/%E6%8E%A8%E6%B5%81%E4%B8%8D%E6%88%90%E5%8A%9F.jpg)

那就是说，向EasyDarwin推流的时候，也需要进行认证。从注释上来看，需要加入用户名和密码的`md5`值，我们用正确的参数再推流（下面mad5ofpassword换成你密码的`md5`）：

~~~bash
./ffmpeg.exe -re -stream_loop -1 -i test.mp4 -c copy -f rtsp rtsp://admin:mad5ofpassword@127.0.0.1/stream
~~~

成功了：

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/ffmpeg%E5%90%91%E6%9C%89%E8%AE%A4%E8%AF%81%E7%9A%84%E6%8E%A8%E6%B5%81%E6%88%90%E5%8A%9F.jpg)

这时候用`vlc`接收试试，果然要进行认证，要求输入用户名和密码：

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/vlc%E9%9C%80%E8%A6%81%E8%AE%A4%E8%AF%81.jpg)

注意这里密码也要输入`md5`后的值。输入正确的密码后，`vlc`可以接收`RTSP`流了:

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/vlc%E8%AE%A4%E8%AF%81%E5%90%8E%E6%92%AD%E6%94%BE.gif)

同样地，用`wireshark`抓包看看带有认证的流程是什么样的：

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/%E9%89%B4%E6%9D%83.jpg)

client=>server

~~~http
Real Time Streaming Protocol
    Request: DESCRIBE rtsp://127.0.0.1:554/stream RTSP/1.0\r\n
        Method: DESCRIBE
        URL: rtsp://127.0.0.1:554/stream
    CSeq: 6\r\n
    User-Agent: LibVLC/3.0.18 (LIVE555 Streaming Media v2016.11.28)\r\n
    Accept: application/sdp\r\n
    \r\n
~~~

首先client同样发起`DESCRIBE`

server=>client

~~~HTTP
Real Time Streaming Protocol
    Response: RTSP/1.0 401 Unauthorized\r\n
        Status: 401
    CSeq: 6\r\n
    Session: ayQBojNIg
    WWW-Authenticate: Digest realm="EasyDarwin", nonce="539c6afee35b8edd354e983a6af947bf", algorithm="MD5"\r\n
    \r\n
~~~

server返回401，`WWW-Authenticate: Digest`表示需要摘要认证，`realm`和`nonce`用于生成`response`，`algorithm="MD5"`表示需要`md5`算法生成`response`。

client=>server

~~~http
Real Time Streaming Protocol
    Request: DESCRIBE rtsp://127.0.0.1:554/stream RTSP/1.0\r\n
        Method: DESCRIBE
        URL: rtsp://127.0.0.1:554/stream
    CSeq: 7\r\n
    Authorization: Digest username="admin", realm="EasyDarwin", nonce="539c6afee35b8edd354e983a6af947bf", uri="rtsp://127.0.0.1:554/stream", response="d6a48b37f2010b3ddfad1eef18692648"\r\n
    User-Agent: LibVLC/3.0.18 (LIVE555 Streaming Media v2016.11.28)\r\n
    Accept: application/sdp\r\n
    \r\n
~~~

client用对应算法生成`response`并返回给server，`response`的计算方法单独再讲。

server=>client

~~~http
Real Time Streaming Protocol
    Response: RTSP/1.0 200 OK\r\n
        Status: 200
    Content-length: 511
    CSeq: 7\r\n
    Session: ayQBojNIg
    \r\n
    Data (511 bytes)
~~~

server验证`response`通过，则返回200。

这里其实和上面一样返回了`SDP`信息（Data 511 bytes中的信息），但`EasyDarwin`是做了加密处理还是什么，是无法解析出来的。

之后的流程就和没有摘要认证的过程是一样的了。

## 完善代码，处理摘要认证

既然可能会存在认证，那我们代码中得处理server有认证的情况，否则肯定收不到`RTSP`流。首先我们定位server的返回在哪里被捕捉了，经过一番尝试，发现在方法`avformat_open_input`中：

~~~C++
if ((ret = avformat_open_input(&streamFmtCtx, url.toStdString().c_str(), nullptr, nullptr)) != 0) {
    qDebug() << "ret:" << ret;
}
//打印输出
//ret: -825242872
//ffmpeg日志输出
//[rtsp @ 000001d2d3940ec0] method DESCRIBE failed: 401 Unauthorized
~~~

在需要认证的情况下，`avformat_open_input`直接返回了一个负数。再结合`ffmpeg`的日志，大致可以断定这是server返回Unauthorized时的情况。但我们需要更具体的确认，所以查看`avformat_open_input`的声明：

~~~C++
//avformat.h
/*
* @return 0 on success, a negative AVERROR on failure.
*/
int avformat_open_input(AVFormatContext **ps, const char *url, ff_const59 AVInputFormat *fmt, AVDictionary **options);
~~~

返回值是一个int，注释中写到如果是失败，则返回`AVERROR`，那么接下来，我们可以去`ffmpeg`的源码中，找关于`AVERROR`的内容了。

如果编译了`ffmpeg`源码，直接debug就可以看到最终是如何返回的，但现在我们不想花额外的时间去编译源码，所以我们用宇宙第一IDE——Visual Studio，打开`ffmpeg`的源码文件夹，直接搜索`AVERROR`，很方便找到了`AVERROR`的定义：

~~~C++
//error.h
#define AVERROR(e) (-(e))   ///< Returns a negative error code from a POSIX error code, to return from library functions.
~~~

可以看到`AVERROR`是用来取`POSIX`中标准错误相反数的宏，继续追踪没有发现相关返回的地方。但我们在头文件却看见了Unauthorized的相关定义：

~~~c++
//error.h
#define AVERROR_HTTP_UNAUTHORIZED  FFERRTAG(0xF8,'4','0','1')
#define FFERRTAG(a, b, c, d) (-(int)MKTAG(a, b, c, d))
//common.h
#define MKTAG(a,b,c,d) ((a) | ((b) << 8) | ((c) << 16) | ((unsigned)(d) << 24))
~~~

按照定义，`AVERROR_HTTP_UNAUTHORIZED`实际上是(0xF8,'4','0','1')组合的移位，按照定义计算后`AVERROR_HTTP_UNAUTHORIZED`确实等于-825242872。为了验证，我们把宏定义从ffmpeg源码中复制出来，直接在我们项目中打印：

~~~c++
//mainwindow.h
#define AVERROR_HTTP_UNAUTHORIZED  FFERRTAG(0xF8,'4','0','1')
#define MKTAG(a, b, c, d) ((a) | ((b) << 8) | ((c) << 16) | ((unsigned)(d) << 24))
#define FFERRTAG(a, b, c, d) (-(int)MKTAG(a, b, c, d))
//mainwindow.cpp
qDebug() << "AVERROR_HTTP_UNAUTHORIZED:" <<FFERRTAG(0xF8,'4','0','1');

//输出
//AVERROR_HTTP_UNAUTHORIZED: -825242872
~~~

输出和前面的日志输出还有我们计算出来的结果都是一样的，到这里我们确定报出了`AVERROR_HTTP_UNAUTHORIZED`错误。顺手把error.h中其他宏定义打印出来，`ffmpeg`常用错误码错误码表如下：

| **错误码宏定义**           | **错误码**  | **错误说明**                                                 |
| -------------------------- | ----------- | ------------------------------------------------------------ |
| AVERROR_BSF_NOT_FOUND      | -1179861752 | Bitstream filter   not found                                 |
| AVERROR_BUG                | -558323010  | Internal bug, also   see AVERROR_BUG2                        |
| AVERROR_BUFFER_TOO_SMALL   | -1397118274 | Buffer too small                                             |
| AVERROR_DECODER_NOT_FOUND  | -1128613112 | Decoder not found                                            |
| AVERROR_DEMUXER_NOT_FOUND  | -1296385272 | Demuxer not found                                            |
| AVERROR_ENCODER_NOT_FOUND  | -1129203192 | Encoder not found                                            |
| AVERROR_EOF                | -541478725  | End of file                                                  |
| AVERROR_EXIT               | -1414092869 | Immediate exit was   requested; the called function should not be restarted |
| AVERROR_EXTERNAL           | -542398533  | Generic error in   an external library                       |
| AVERROR_FILTER_NOT_FOUND   | -1279870712 | Filter not found                                             |
| AVERROR_INVALIDDATA        | -1094995529 | Invalid data found   when processing input                   |
| AVERROR_MUXER_NOT_FOUND    | -1481985528 | Muxer not found                                              |
| AVERROR_OPTION_NOT_FOUND   | -1414549496 | Option not found                                             |
| AVERROR_PATCHWELCOME       | -1163346256 | Not yet   implemented in FFmpeg, patches welcome             |
| AVERROR_PROTOCOL_NOT_FOUND | -1330794744 | Protocol not found                                           |
| AVERROR_STREAM_NOT_FOUND   | -1381258232 | Stream not found                                             |
| AVERROR_BUG2               | -541545794  |                                                              |
| AVERROR_UNKNOWN            | -1313558101 |                                                              |
| AVERROR_EXPERIMENTAL       | -733130664  |                                                              |
| AVERROR_INPUT_CHANGED      | -1668179713 |                                                              |
| AVERROR_OUTPUT_CHANGED     | -1668179714 |                                                              |
| AVERROR_HTTP_BAD_REQUEST   | -808465656  |                                                              |
| AVERROR_HTTP_UNAUTHORIZED  | -825242872  |                                                              |
| AVERROR_HTTP_FORBIDDEN     | -858797304  |                                                              |
| AVERROR_HTTP_NOT_FOUND     | -875574520  |                                                              |
| AVERROR_HTTP_OTHER_4XX     | -1482175736 |                                                              |
| AVERROR_HTTP_SERVER_ERROR  | -1482175992 |                                                              |

于是可以在代码中增加Unauthorized情况的处理，如果Unauthorized则让用户输入用户名和密码。

~~~C++
//ffmpegmanager.cpp
if ((ret = avformat_open_input(&streamFmtCtx, url.toStdString().c_str(), nullptr, nullptr)) != 0) {
    if (ret == AVERROR_HTTP_UNAUTHORIZED)
    {
        //...
        return;
    }else{
        //...
        return;
    }
}
~~~

`vlc`中，如果输入的用户名和密码无法通过验证，则会重新弹出验证框（且用户名不用重新输入），直至输入正确或取消输入（效果看开头）。所以我们也加入`RTSP`地址合法性的检查等操作：

~~~C++
//ffmpegmanager.cpp
int rtspIndex = url.indexOf("rtsp://");
int atIndex = url.lastIndexOf("@");
if(rtspIndex != -1 && atIndex != -1){
    QString couple = url.mid(rtspIndex + 7, atIndex - rtspIndex - 7);
    username = couple;
    if(couple.contains(':')){
        username = couple.mid(0, couple.lastIndexOf(':'));
    }
}
~~~

到这里，我们的代码可以适配需要摘要认证的情况了。

## 增加错误窗口

`vlc`在无法打开`RTSP`地址的时候会弹出错误窗口。

![](http://s5rlp5ps9.hn-bkt.clouddn.com/ffmpegbook/2%E6%8E%A5%E6%94%B6rtsp%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/vlc%E6%97%A0%E6%B3%95%E6%89%93%E5%BC%80.png)

我们也增加一个错误窗口，把所有错误都归为无法打开地址，并打印出来。

![](http://s5rlp5ps9.hn-bkt.clouddn.com/ffmpegbook/2%E6%8E%A5%E6%94%B6rtsp%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/%E6%97%A0%E6%B3%95%E6%92%AD%E6%94%BE%E5%9C%B0%E5%9D%80.gif)

## 解决内存泄漏

虽然程序可以正常接收`RTSP`流了，但出现了之前没出现的情况：内存持续增加。这种情况下一般是发生了内存泄露，之前读取MP4文件没有发现，可能是因为文件大小固定，现在持续收流，现象比较明显，我们得检查我们的代码。经过一番定位之后，我们发现是下面的代码块发生泄露：

~~~C++
while(av_read_frame(streamFmtCtx, packet) >= 0){
    if(packet->stream_index == nVideoIndex){
        if(avcodec_send_packet(decoderCtx, packet)>=0){
            while((ret = avcodec_receive_frame(decoderCtx, decodedFrame)) >= 0){
                //...
            }
        }
    }
}
~~~

接下来我们逐句排查，首先是`av_read_frame`，查看它的声明：

~~~C++
//avformat.h
/**
 *.....
 * On success, the returned packet is reference-counted (pkt->buf is set) and
 * valid indefinitely. The packet must be freed with av_packet_unref() when
 * it is no longer needed. 
 *.....
 */
int av_read_frame(AVFormatContext *s, AVPacket *pkt);
~~~

这里面有些有用的信息：`pkt`是`reference-counted`的，如果不`av_packet_unref() `，则它将永久有效。继续看它的定义，我们的目标是找出和`pkt`相关的进行`reference-counted`的语句：

~~~C++
//avformat.cpp
int av_read_frame(AVFormatContext *s, AVPacket *pkt){
    //...
    ret = read_frame_internal(s, pkt);
    ret = avpriv_packet_list_put(&s->internal->packet_buffer,
                                 &s->internal->packet_buffer_end,
                                 pkt, NULL, 0);
    //...
}
~~~

最终`pkt`都要执行这两个函数，`avpriv_packet_list_put`就是我们要找的地方，继续看它的声明和定义：

~~~C++
//packet_internal.h
/**
 * Append an AVPacket to the list.
 *
 * @param head  List head element
 * @param tail  List tail element
 * @param pkt   The packet being appended. The data described in it will
 *              be made reference counted if it isn't already.
 */
int avpriv_packet_list_put(PacketList **head, PacketList **tail,
                           AVPacket *pkt,
                           int (*copy)(AVPacket *dst, const AVPacket *src),
                           int flags);
//avpacket.c
int avpriv_packet_list_put(PacketList **packet_buffer,
                           PacketList **plast_pktl,
                           AVPacket      *pkt,
                           int (*copy)(AVPacket *dst, const AVPacket *src),
                           int flags)
{
    //...
    if (*packet_buffer)
        (*plast_pktl)->next = pktl;
    else
        *packet_buffer = pktl;
    *plast_pktl = pktl;
    return 0;
}
~~~

最后pkt添加到了buffered packet中。其他细节我们可以不用深究，只需要知道pkt被添加到了一个list中，那么这里的确会产生内存泄漏。根据前面声明中的提示，我们需要使用`av_packet_unref()`来释放pkt的引用，那么直接在读取和使用完1个`AVPacket`和结束时调用`av_packet_unref()`。

~~~C++
while(av_read_frame(streamFmtCtx, packet) >= 0){
    //...
    av_packet_unref(packet);
}
av_packet_unref(packet);
~~~

加上后发现，内存泄漏的问题被解决了，那就不再继续向下排查了。

## 遗留问题

至此，一个简单好用的`RTSP`收流功能就算是完成了，但别高兴的太早，事情往往没有我们想象的那么简单——经过测试，接收高分辨率视频一段时间后（甚至一开始），就会产生花屏现象：

![](https://raw.gitmirror.com/JoggingJack/PicRepo/main/FFmpegBook/2%E6%8E%A5%E6%94%B6RTSP%E6%B5%81%E5%B9%B6%E6%92%AD%E6%94%BE/%E8%8A%B1%E5%B1%8F.gif)

考虑到篇幅原因，后面单独篇章再去讨论解决这个问题，依旧是需要从源码切入：)

## TO-DO

- 适配`BASE`认证