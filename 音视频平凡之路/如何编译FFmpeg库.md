# 如何编译FFmpeg库
FFmpeg是一个开源的多媒体框架，音视频开发中最常见的库就是FFmpeg，学会裁剪和编译FFmpeg库是音视频工程师的入门课程，本文主要介绍怎么编译和裁剪FFmpeg，希望大家看完本文之后，能够掌握FFmpeg的编译原理。<br>
本文会从几个角度聊聊FFmpeg的相关知识:
- FFmpeg代码架构
- FFmpeg中关键数据结构
- FFmpeg编译选项
- PC上编译以及运行FFmpeg
- 交叉编译FFmpeg
- FFmpeg链接openssl/libx264/fdk-aac库

## 1.FFmpeg代码架构
第一步大家要下载FFmpeg源码，下载地址是：https://github.com/FFmpeg/FFmpeg <br>
下载完成后，发现FFmpeg源码目录下有一系列的libavXXX的模块，通过这些模块的命名，可以很好区分各个模块的分工以及主要的功能:
- libavformat，format，用于格式封装和解封装，场景的parser功能就在这个模块
- libavcodec，codec，音视频的编码和解码模块，也可以支持外接其他的编码和解码接口
- libavutil，util，通用音视频工具，像素计算、IO、时间等工具
- libavfilter，filter，过滤器，可以用作音视频特效处理
- libavdevice，device，设备（摄像头、拾音器）
- libswscale，scale，视频图像缩放，像素格式互换
- libavresample，resample，重采样，这个模块现在已经废弃了
- libswresample，也是重采样，类似图像缩放
- libpostproc，后期处理模块

其中核心的四个模块是libavformat、libavcodec、libavfilter、libavutil
### 1.1 libavformat模块
libavformat中主要功能有三个:
- 协议解析：解析file/http/https/tcp等协议
- 解封装：对mp4/mkv/avi/mp3/aac等音视频封装协议进行解析
- 封装：将数据封装成mp4/mkv/avi/mp3/aac等文件，是解封装的逆过程

libavformat中协议协议的索引文件是protocols.c，其中可以看到很多具体协议的的tag-name：
```
extern const URLProtocol ff_file_protocol;
extern const URLProtocol ff_ftp_protocol;
extern const URLProtocol ff_gopher_protocol;
extern const URLProtocol ff_hls_protocol;
extern const URLProtocol ff_http_protocol;
extern const URLProtocol ff_httpproxy_protocol;
extern const URLProtocol ff_https_protocol;
extern const URLProtocol ff_icecast_protocol;
extern const URLProtocol ff_mmsh_protocol;
extern const URLProtocol ff_mmst_protocol;
extern const URLProtocol ff_md5_protocol;
extern const URLProtocol ff_pipe_protocol;
extern const URLProtocol ff_prompeg_protocol;
extern const URLProtocol ff_rtmp_protocol;
extern const URLProtocol ff_tee_protocol;
extern const URLProtocol ff_tcp_protocol;
......

```
我们要分析某一个协议的解析代码时，可以根据这个协议的tag-name找到对应的文件，例如ff_http_protocol对应的具体文件是http.c，可以在其中找到ff_http_protocol的定义：
```
const URLProtocol ff_http_protocol = {
    .name                = "http",
    .url_open2           = http_open,
    .url_accept          = http_accept,
    .url_handshake       = http_handshake,
    .url_read            = http_read,
    .url_write           = http_write,
    .url_seek            = http_seek,
    .url_close           = http_close,
    .url_get_file_handle = http_get_file_handle,
    .url_get_short_seek  = http_get_short_seek,
    .url_shutdown        = http_shutdown,
    .priv_data_size      = sizeof(HTTPContext),
    .priv_data_class     = &http_context_class,
    .flags               = URL_PROTOCOL_FLAG_NETWORK,
    .default_whitelist   = "http,https,tls,rtp,tcp,udp,crypto,httpproxy"
};

```
每一个链接的指针函数都指向一个新的目标执行方法，这是FFmpeg中的特色用法。如果我们做优化需要了解并掌握底层的网络请求代码，这样可以更好地提升我们程序的执行效率。大家在平时工作中也能明白一个道理：就是把一件事情做出来不是很难，但是做好或者做全面就比较难了，那是因为我们对原理的掌握还不够深入，导致一些点没有吃透，没有吃透的前提下，更好地优化就无从谈起。<br><br>

libavformat中的解封装和封装模块是注册在同一个索引文件allformats.c：
```
/* (de)muxers */
extern AVInputFormat  ff_matroska_demuxer;
extern AVOutputFormat ff_matroska_muxer;
extern AVOutputFormat ff_matroska_audio_muxer;
extern AVInputFormat  ff_mgsts_demuxer;
extern AVInputFormat  ff_microdvd_demuxer;
extern AVOutputFormat ff_microdvd_muxer;
extern AVInputFormat  ff_mjpeg_demuxer;
extern AVOutputFormat ff_mjpeg_muxer;
extern AVInputFormat  ff_mjpeg_2000_demuxer;
extern AVInputFormat  ff_mlp_demuxer;
extern AVOutputFormat ff_mlp_muxer;
extern AVInputFormat  ff_mlv_demuxer;
extern AVInputFormat  ff_mm_demuxer;
extern AVInputFormat  ff_mmf_demuxer;
extern AVOutputFormat ff_mmf_muxer;
extern AVInputFormat  ff_mov_demuxer;
extern AVOutputFormat ff_mov_muxer;
......

```
这里面可以看出FFmpeg支持的解封装和封装协议真的非常多，几乎你能想到的所有协议这儿都能支持，我们平时场景的mp4协议封装和解封转可以简单谈谈，mp4也成为mov，所有mp4的解封装是ff_mov_demuxer，在mov.c文件中；mp4的封装是ff_mov_muxer，在movenc.c文件中。
```
mov.c

AVInputFormat ff_mov_demuxer = {
    .name           = "mov,mp4,m4a,3gp,3g2,mj2",
    .long_name      = NULL_IF_CONFIG_SMALL("QuickTime / MOV"),
    .priv_class     = &mov_class,
    .priv_data_size = sizeof(MOVContext),
    .extensions     = "mov,mp4,m4a,3gp,3g2,mj2",
    .read_probe     = mov_probe,
    .read_header    = mov_read_header,
    .read_packet    = mov_read_packet,
    .read_close     = mov_read_close,
    .read_seek      = mov_read_seek,
    .flags          = AVFMT_NO_BYTE_SEEK,
};

```
解封装的操作主要是:
- 探测：探测当前是否符合封装协议的要求
- 读取头部：安装封装格式的要求读取头部数据，mp4的头部就是moov信息
- 读取body：安装moov信息读取对应的mdat数据
- seek：拖动进度条的情况下如何快速seek数据

```
movenc.c

AVOutputFormat ff_mov_muxer = {
    .name              = "mov",
    .long_name         = NULL_IF_CONFIG_SMALL("QuickTime / MOV"),
    .extensions        = "mov",
    .priv_data_size    = sizeof(MOVMuxContext),
    .audio_codec       = AV_CODEC_ID_AAC,
    .video_codec       = CONFIG_LIBX264_ENCODER ?
                         AV_CODEC_ID_H264 : AV_CODEC_ID_MPEG4,
    .init              = mov_init,
    .write_header      = mov_write_header,
    .write_packet      = mov_write_packet,
    .write_trailer     = mov_write_trailer,
    .deinit            = mov_free,
    .flags             = AVFMT_GLOBALHEADER | AVFMT_ALLOW_FLUSH | AVFMT_TS_NEGATIVE,
    .codec_tag         = (const AVCodecTag* const []){
        ff_codec_movvideo_tags, ff_codec_movaudio_tags, 0
    },
    .check_bitstream   = mov_check_bitstream,
    .priv_class        = &mov_muxer_class,
};

```
封装是解封装的逆过程，我们在开发音视频SDK的过程中经常会用到音视频的封装，这儿大家了解即可，后续还要展开分析。

### 1.2 libavcodec模块
libavcodec的主要功能也有3个：
- 比特流过滤器 : 解封装之后的原始数据过滤器 
- 解码 : 音视频解码流程，将h264/hevc/aac等音视频解码成原始数据的过程
- 编码 : 解码的逆过程，将原始数据编码为h264/hevc/aac等过程

比特率过滤器是解码之前的重要步骤，我们解封装之后得到的数据离解码还有一小步，例如H264解析处最终的解码单元之前，还要读取SPS和PPS数据，这个比特率过滤器就是这样的一个过程，我们可以在bitstream_filters.c中看到每一个比特率过滤器的tag-name，然后再根据这个tag-name追踪到具体的执行文件：
```
extern const AVBitStreamFilter ff_aac_adtstoasc_bsf;
extern const AVBitStreamFilter ff_chomp_bsf;
extern const AVBitStreamFilter ff_dump_extradata_bsf;
extern const AVBitStreamFilter ff_dca_core_bsf;
extern const AVBitStreamFilter ff_eac3_core_bsf;
extern const AVBitStreamFilter ff_extract_extradata_bsf;
extern const AVBitStreamFilter ff_filter_units_bsf;
extern const AVBitStreamFilter ff_h264_metadata_bsf;
extern const AVBitStreamFilter ff_h264_mp4toannexb_bsf;
extern const AVBitStreamFilter ff_h264_redundant_pps_bsf;
extern const AVBitStreamFilter ff_hapqa_extract_bsf;
extern const AVBitStreamFilter ff_hevc_metadata_bsf;
extern const AVBitStreamFilter ff_hevc_mp4toannexb_bsf;
extern const AVBitStreamFilter ff_imx_dump_header_bsf;
extern const AVBitStreamFilter ff_mjpeg2jpeg_bsf;
extern const AVBitStreamFilter ff_mjpega_dump_header_bsf;
extern const AVBitStreamFilter ff_mp3_header_decompress_bsf;
extern const AVBitStreamFilter ff_mpeg2_metadata_bsf;
extern const AVBitStreamFilter ff_mpeg4_unpack_bframes_bsf;
extern const AVBitStreamFilter ff_mov2textsub_bsf;
extern const AVBitStreamFilter ff_noise_bsf;
extern const AVBitStreamFilter ff_null_bsf;
extern const AVBitStreamFilter ff_remove_extradata_bsf;
extern const AVBitStreamFilter ff_text2movsub_bsf;
extern const AVBitStreamFilter ff_trace_headers_bsf;
extern const AVBitStreamFilter ff_vp9_raw_reorder_bsf;
extern const AVBitStreamFilter ff_vp9_superframe_bsf;
extern const AVBitStreamFilter ff_vp9_superframe_split_bsf;

```
以我们熟知的ff_h264_metadata_bsf和ff_h264_mp4toannexb_bsf为例，ff_h264_mp4toannexb_bsf在h264_mp4toannexb_bsf.c中，ff_h264_metadata_bsf在h264_metadata_bsf.c中。
```
const AVBitStreamFilter ff_h264_mp4toannexb_bsf = {
    .name           = "h264_mp4toannexb",
    .priv_data_size = sizeof(H264BSFContext),
    .init           = h264_mp4toannexb_init,
    .filter         = h264_mp4toannexb_filter,
    .codec_ids      = codec_ids,
};


const AVBitStreamFilter ff_h264_metadata_bsf = {
    .name           = "h264_metadata",
    .priv_data_size = sizeof(H264MetadataContext),
    .priv_class     = &h264_metadata_class,
    .init           = &h264_metadata_init,
    .close          = &h264_metadata_close,
    .filter         = &h264_metadata_filter,
    .codec_ids      = h264_metadata_codec_ids,
};
```
h264_mp4toannexb_filter中是解析SPS和PPS的过程，同样在音视频编辑SDK中会用到这部分的知识点。大家有想法的可以深入分析一下，在音视频SDK中会用到这部分的知识。<br><br>

libavcodec中的解码和编码模块是非常核心的模块，其他的基本上围绕这两个模块开展，想了解编解码协议的同学，这一块可以下功夫去学习一下。目前编解码实在太多了，有上百种之多，可以在allcodecs.c中查找目前支持的所有codec：
```
extern AVCodec ff_bink_decoder;
extern AVCodec ff_h261_encoder;
extern AVCodec ff_h261_decoder;
extern AVCodec ff_h263_encoder;
extern AVCodec ff_h263_decoder;
extern AVCodec ff_h263i_decoder;
extern AVCodec ff_h263p_encoder;
extern AVCodec ff_h263p_decoder;
extern AVCodec ff_h263_v4l2m2m_decoder;
extern AVCodec ff_h264_decoder;
extern AVCodec ff_h264_crystalhd_decoder;
extern AVCodec ff_h264_v4l2m2m_decoder;
extern AVCodec ff_h264_mediacodec_decoder;
extern AVCodec ff_h264_mmal_decoder;
extern AVCodec ff_h264_qsv_decoder;
extern AVCodec ff_h264_rkmpp_decoder;
......

```
对一个音视频工程师而言，要了解常见的音视频的编解码知识，从了解音视频编码的基础知识开始，例如H264的编码过程，H264是如何提升压缩效率的？当然这一张如何展开讲太多了，不过我会在其他章节重点分析一下场景的编码的H264/HEVC等等。

### 1.3 libavfilter模块
libavfilter是处理数据的过滤器，不仅仅是我们理解的滤镜操作，甚至声音也有filter操作，混音、倍速都需要特定的filter处理，所有的filter都挂在allfilters.c文件中:
```
extern AVFilter ff_af_abench;
extern AVFilter ff_af_acompressor;
extern AVFilter ff_af_acontrast;
extern AVFilter ff_af_acopy;
extern AVFilter ff_af_acrossfade;
extern AVFilter ff_af_acrusher;
extern AVFilter ff_af_adelay;
extern AVFilter ff_af_aecho;
extern AVFilter ff_af_aemphasis;
extern AVFilter ff_af_aeval;
extern AVFilter ff_af_afade;
extern AVFilter ff_af_afftfilt;
extern AVFilter ff_af_afir;
extern AVFilter ff_af_aformat;
extern AVFilter ff_af_agate;
extern AVFilter ff_af_aiir;
extern AVFilter ff_af_ainterleave;
extern AVFilter ff_af_alimiter;
extern AVFilter ff_af_allpass;
......

```
FFmpeg能够实现的非常丰富的filter操作，从加水印、去水印、加滤镜、加字幕、改变声音、混音等操作，从改变声音、图片、字幕等各个数据源的角度改变原始数据。
举个例子，ff_vf_delogo就是去水印的filter，可以看看具体的一些操作，在vf_delogo.c文件中：
```
AVFilter ff_vf_delogo = {
    .name          = "delogo",
    .description   = NULL_IF_CONFIG_SMALL("Remove logo from input video."),
    .priv_size     = sizeof(DelogoContext),
    .priv_class    = &delogo_class,
    .init          = init,
    .query_formats = query_formats,
    .inputs        = avfilter_vf_delogo_inputs,
    .outputs       = avfilter_vf_delogo_outputs,
    .flags         = AVFILTER_FLAG_SUPPORT_TIMELINE_GENERIC,
};

```
需要开发者指定祛除水印的坐标和大小，计算出当前位置的水印数据，而祛除数据。不过祛除完成后还是能看到一些马赛克。<br><br>

ff_vf_amix就是混音操作，在af_amix.c文件中：
```
AVFilter ff_af_amix = {
    .name           = "amix",
    .description    = NULL_IF_CONFIG_SMALL("Audio mixing."),
    .priv_size      = sizeof(MixContext),
    .priv_class     = &amix_class,
    .init           = init,
    .uninit         = uninit,
    .activate       = activate,
    .query_formats  = query_formats,
    .inputs         = NULL,
    .outputs        = avfilter_af_amix_outputs,
    .flags          = AVFILTER_FLAG_DYNAMIC_INPUTS,
};

```
将多路音频流混合成一路流，这在音视频应用中还是非常常见的。混音操作也会是音视频SDK的重点。

### 1.4 libavutil模块
libavutil是FFmpeg中一个通用方法管理模块，例如日志模块、时间处理模块、狂平台兼容模块等等。大家在FFmpeg开发中会用到很多相关的API。就拿我们常见的日志模块来讲，FFmpeg提供了独有的日志模块，av_log模块，在log.h文件中。我们在调用FFmpeg库的时候想打开FFmpeg中日志怎么做到了？
- 设置log level : 什么级别的日志才能输出
- 设置log回调处理 : log以什么形式输出

具体的执行代码如下：
```
    av_log_set_level(AV_LOG_VERBOSE);
    av_log_set_callback(log_callback);

```

```
void log_callback(void *ptr, int level, const char *fmt, va_list vl) {
    //ffmpeg中level越大打样的log优先级越低
    if (level > av_log_get_level())
        return;
    TLogLevel android_level = kLevelAll;
    if (level > AV_LOG_DEBUG) {
        android_level = kLevelDebug;
    } else if (level > AV_LOG_VERBOSE) {
        android_level = kLevelVerbose;
    } else if (level > AV_LOG_INFO) {
        android_level = kLevelInfo;
    } else if (level > AV_LOG_WARNING) {
        android_level = kLevelWarn;
    } else if (level > AV_LOG_ERROR) {
        android_level = kLevelError;
    } else if (level > AV_LOG_FATAL) {
        android_level = kLevelFatal;
    }
    __LOGV__(android_level, FFMPEG_LOG_TAG, fmt, vl);
}

```
我们在Android平台上，可以将其转嫁到__android_log_print输出，在其他平台上也可以用其他的形式输出，最终的目的是方便我们查找问题。

## 2.FFmpeg中关键数据结构
开始学习FFmpeg源码，需要搞清楚有其内部有几个关键的数据结构，掌握这些数据结构对进一步学习有非常大的帮助:
- AVFormatContext
- AVIOContext
- AVInputFormat
- AVOutputFormat
- AVStream
- AVPacket和AVFrame

### 2.1 AVFormatContext
AVFormatContext是打开文件必需的一个结构体，Format是音视频的一个核心概念，操作一个音视频文件，第一步就是将其读取打开转化为AVFormatContext结构体，其中音视频的关键信息已经被读到了AVFormatContext结构体中。AVFormatContext将音视频的信息解析出来，它不会直接处理，交给下面的子结构去处理，它负责全局调度。<br>
它都有哪些子结构呢？
```
/// 解封装专用
struct AVInputFormat *iformat;

/// 封装专用
struct AVOutputFormat *oformat;

/// 解封装中需要先打开文件，封装过程中需要写数据到文件，主要控制IO操作
AVIOContext *pb;

/// 文件流个数
unsigned int nb_streams;

/// 文件流具体的指针
AVStream **streams;

/// 文件时长
int64_t duration;

/// 视频编解码的id
enum AVCodecID video_codec_id;

/// 视频编解码的实例
AVCodec *video_codec;

/// 音频编解码的id
enum AVCodecID audio_codec_id;

/// 音频编解码的实例
AVCodec *audio_codec;

/// 字幕编解码的id
enum AVCodecID subtitle_codec_id;

/// 字幕编解码的实例
AVCodec *subtitle_codec;

/// 音视频的具体信息都存在metadata结构中，是一个key-value的结构，可以通过av_dump输出
AVDictionary *metadata;

```
### 2.2 AVIOContext
AVIOContext结构体是音视频IO操作的时候用到的主要数据结构，例如我们需要输出一个mp4文件，首先肯定要打开这个文件，然后按照特定排列写入数据，这里面的主要是一些IO的执行方法：
```
/// 读数据包方法
int (*read_packet)(void *opaque, uint8_t *buf, int buf_size);

/// 写数据包方法
int (*write_packet)(void *opaque, uint8_t *buf, int buf_size);

/// seek情况下写操作
int64_t (*seek)(void *opaque, int64_t offset, int whence);

/// seek情况下读操作
int64_t (*read_seek)(void *opaque, int stream_index, int64_t timestamp, int flags);
```

### 2.3 AVInputFormat
AVInputFormat用于解封装操作，其核心执行方法都是读操作，可以看下其结构体中指向方法：
```
/// 探测封装信息，决定操作哪种解封装格式
int (*read_probe)(AVProbeData *);

/// 读取数据的头部信息
int (*read_header)(struct AVFormatContext *);

/// 读取数据的body信息
int (*read_packet)(struct AVFormatContext *, AVPacket *pkt);

/// 关闭读取操作
int (*read_close)(struct AVFormatContext *);

/// seek情况下读取操作
int (*read_seek)(struct AVFormatContext *,
                     int stream_index, int64_t timestamp, int flags);

/// 读取具体时间点的操作
int64_t (*read_timestamp)(struct AVFormatContext *s, int stream_index,
                              int64_t *pos, int64_t pos_limit);

/// 读取RTSP协议信息
int (*read_play)(struct AVFormatContext *);

/// 暂停读取RTSP信息
int (*read_pause)(struct AVFormatContext *);

/// 读取TS信息
int (*read_seek2)(struct AVFormatContext *s, int stream_index, int64_t min_ts, int64_t ts, int64_t max_ts, int flags);

```
### 2.4 AVOutputFormat
AVOutputFormat是封装的过程，是AVInputFormat逆结构，都是写操作：
```
/// 写头部信息
int (*write_header)(struct AVFormatContext *);

/// 写数据body
int (*write_packet)(struct AVFormatContext *, AVPacket *pkt);

/// 写尾部信息
int (*write_trailer)(struct AVFormatContext *);

```
### 2.5 AVStream
解封装出来的具体流信息，有音频流、视频流、字幕流等等。AVFormatContext是整个文件的信息，那么AVStream就是具体流的信息。
```
/// 编解码的上下文
AVCodecContext *codec;

/// 时间基，单位时间内有多少的传输单元存在
AVRational time_base;

/// 时长
int64_t duration;

/// 帧数
int64_t nb_frames;

/// 编解码参数，里面包含编解码的基本信息，这是非常重要编解码信息key-value
AVCodecParameters *codecpar;

```

其中AVCodecParameters的结构如下：
```
/// 类型，可以视频音频、视频、字幕等等
enum AVMediaType codec_type;

/// 编解码id
enum AVCodecID   codec_id;

/// format id，如果是图片，参考AVPixelFormat枚举；如果是音频，参考AVSampleFormat枚举
int format;

/// 比特率
int64_t bit_rate;

/// 宽和高
int width;
int height;

/// channel 布局
uint64_t channel_layout;

/// 声道个数
int      channels;

/// 采样率
int      sample_rate;

/// 声音帧大小，aac是1024，mp3是1152
int      frame_size;
```
特别要强调的一点是frame_size中aac的frame_size是1024，mp3的frame_size是1152，为什么特别强调一下，因为之前在做音视频SDK的时候遇到一个坑就是这个引起的，之前没有注意，后面谈到音视频SDK的时候会详细分析一下。
### 2.6 AVPacket和AVFrame
AVPacket是未解码之前的音视频原始数据，将文件解封装之后，可以通过av_read_frame来得到对应的AVPacket，但是AVPacket还没有经过解码，经过解码之后的数据就是AVFrame。
```
typedef struct AVPacket {
    AVBufferRef *buf;
    int64_t pts;    
    int64_t dts;
    uint8_t *data;
    int   size;
    int   stream_index;
    int   flags;
    AVPacketSideData *side_data;
    int side_data_elems;
    int64_t duration;
    int64_t pos;                            ///< byte position in stream, -1 if unknown

} AVPacket;

```
主要是av_read_frame函数让大家误解，还以为AVPacket是帧数据，AVPacket是包数据，AVFrame才是真正的帧数据。<br><br>


操作文件的完整流程：
- avformat_alloc_context() 创建输入媒体文件的AVFormatContext
- avformat_alloc_output_context2() 创建输出媒体文件的AVFormatContext
- av_dump_format() 打印format详情
- avformat_open_input() 打开媒体文件，探知媒体文件的封装格式
- avformat_close_input() 关闭媒体文件
- avformat_find_stream_info() 探知媒体文件中的流信息，几条流，每条流的基本信息
- av_read_frame() 读取媒体文件中每一个数据包，这是未解码之前的包
- avformat_write_header() 写入输出文件的媒体头部信息
- av_interleaved_write_frame() 写入输出文件的帧信息，此帧信息已经调整了帧与帧之间的关联了
- av_write_uncoded_frame() 写入输出文件的未编码的帧信息
- av_write_frame() 写入输出文件的已编码的帧信息
- av_write_trailer() 写入输出文件的媒体尾部信息


## 3.FFmpeg编译选项
FFmpeg的编译裁剪是非常重要的，它要求我们必须知道应该支持什么特性，不支持什么特性，那首先需要搞清楚，FFmpeg可以支持什么特性，查询FFmpeg是否支持这个特性是很重要的。下载了FFmpeg源码之后，在源码目录下有一个configure文件，这个文件非常庞大，里面包含了FFmpeg所有特性的配置选项。<br>
- ./configure --help
```
  --help                   print this message
  --quiet                  Suppress showing informative output
  --list-decoders          show all available decoders
  --list-encoders          show all available encoders
  --list-hwaccels          show all available hardware accelerators
  --list-demuxers          show all available demuxers
  --list-muxers            show all available muxers
  --list-parsers           show all available parsers
  --list-protocols         show all available protocols
  --list-bsfs              show all available bitstream filters
  --list-indevs            show all available input devices
  --list-outdevs           show all available output devices
  --list-filters           show all available filters
```

- 支持哪些解码选项        ./configure --list-decoders
想支持这个解码，需要使用--enable-decoder=XXX；<br>
不想支持这个解码，需要使用--disable-decoder=XXX<br>
- 支持哪些编码选项        ./configure --list-encoders
想支持这个编码，使用--enable-encoder=XXX；<br>
不想支持这个编码，使用--disable-encoder=XXX<br>
- 支持哪些硬件编解码选项   ./configure --list-hwaccels
```
h263_vaapi       hevc_dxva2       mpeg2_d3d11va        mpeg4_videotoolbox       vp9_dxva2
h263_videotoolbox    hevc_nvdec       mpeg2_d3d11va2       vc1_d3d11va          vp9_nvdec
h264_d3d11va         hevc_vaapi       mpeg2_dxva2          vc1_d3d11va2         vp9_vaapi
h264_d3d11va2        hevc_vdpau       mpeg2_nvdec          vc1_dxva2            wmv3_d3d11va
h264_dxva2       hevc_videotoolbox    mpeg2_vaapi          vc1_nvdec            wmv3_d3d11va2
h264_nvdec       mjpeg_nvdec          mpeg2_vdpau          vc1_vaapi            wmv3_dxva2
h264_vaapi       mjpeg_vaapi          mpeg2_videotoolbox       vc1_vdpau            wmv3_nvdec
h264_vdpau       mpeg1_nvdec          mpeg2_xvmc           vp8_nvdec            wmv3_vaapi
h264_videotoolbox    mpeg1_vdpau          mpeg4_nvdec          vp8_vaapi            wmv3_vdpau
hevc_d3d11va         mpeg1_videotoolbox   mpeg4_vaapi          vp9_d3d11va
hevc_d3d11va2        mpeg1_xvmc       mpeg4_vdpau          vp9_d3d11va2

```
上面列出的都是可以外接的硬件编解码模块。

- 支持哪些解封装选项      ./configure --list-demuxers
想支持这个解封装，使用--enable-demuxer=XXX；<br>
不想支持这个解封装，使用--disable-demuxer=XXX<br>
- 支持哪些封装选项        ./configure --list-muxers
想支持这个封装，使用--enable-muxer=XXX；<br>
不想支持这个封装，使用--disable-muxer=XXX<br>
- 支持哪些parser选项     ./configure --list-parsers
```
aac          dnxhd            h261             opus             vorbis
aac_latm         dpx              h263             png              vp3
ac3          dvaudio          h264             pnm              vp8
adx          dvbsub           hevc             rv30             vp9
bmp          dvd_nav          mjpeg            rv40             xma
cavsvideo        dvdsub           mlp              sbc
cook             flac             mpeg4video           sipr
dca          g729             mpegaudio        tak
dirac            gsm              mpegvideo        vc1

```
想支持这个parser选项，使用--enable-parser=XXX；<br>
不想支持这个parser选项，使用--disable-parser=XXX<br>
- 支持哪些协议           ./configure --list-protocols
```
async            ftp              librtmps         pipe             sctp
bluray           gopher           librtmpt         prompeg          srtp
cache            hls              librtmpte        rtmp             subfile
concat           http             libsmbclient         rtmpe            tcp
crypto           httpproxy        libsrt           rtmps            tee
data             https            libssh           rtmpt            tls
ffrtmpcrypt      icecast          md5              rtmpte           udp
ffrtmphttp       librtmp          mmsh             rtmpts           udplite
file             librtmpe         mmst             rtp              unix
```
想支持这个协议，使用--enable-protocol=XXX；<br>
不想支持这个协议，使用--disable-protocol=XXX<br>
- 支持哪些比特流过滤器     ./configure --list-bsfs
```
aac_adtstoasc        filter_units         hevc_mp4toannexb     mpeg2_metadata       trace_headers
chomp            h264_metadata        imx_dump_header      mpeg4_unpack_bframes     vp9_raw_reorder
dca_core         h264_mp4toannexb     mjpeg2jpeg           noise            vp9_superframe
dump_extradata       h264_redundant_pps   mjpega_dump_header       null             vp9_superframe_split
eac3_core        hapqa_extract        mov2textsub          remove_extradata
extract_extradata    hevc_metadata        mp3_header_decompress    text2movsub

```
想支持这个比特流过滤器，使用--enable-bsf=XXX；<br>
不想支持这个比特流过滤器，使用--disable-bsf=XXX<br>
- 支持哪些可用的输入设备   ./configure --list-indevs
```
alsa             dshow            kmsgrab          openal           vfwcap
android_camera       fbdev            lavfi            oss              xcbgrab
avfoundation         gdigrab          libcdio          pulse
bktr             iec61883         libdc1394        sndio
decklink         jack             libndi_newtek        v4l2

```
想支持这个输入设备，使用--enable-indev=XXX；<br>
不想支持这个输入设备，使用--disable-indev=XXX<br>
- 支持哪些可用的输出设备   ./configure --list-outdevs
```
alsa             fbdev            oss              sndio
caca             libndi_newtek        pulse            v4l2
decklink         opengl           sdl2             xv

```
想支持这个输出设备，使用--enable-outdev=XXX；<br>
不想支持这个输出设备，使用--disable-outdev=XXX<br>
- 支持哪些filter选项     ./configure --list-filters
想支持这个filter选项，使用--enable-filter=XXX；<br>
不想支持这个filter选项，使用--disable-filter=XXX<br>
FFmpeg中原生的filter非常多，也提供了扩张框架可以保证接入新的filter，如果你有好的filter，可以接入进来，这是FFmpeg强大的生命力保证。<br><br><br>

我们通过打开关闭这些编译选项，可以达到裁剪FFmpeg库的目的，要怎么裁剪，还要你根据实际情况来看，你的应用场景等等。
## 4.PC上编译以及运行FFmpeg
## 5.交叉编译FFmpeg
## 6.FFmpeg链接openssl/libx264/fdk-aac库