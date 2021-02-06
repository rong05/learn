# ffplay源码分析

熟悉FFmpeg项目从源码看起，以下是我阅读FFplay的源代码的总结；FFplay是FFmpeg项目提供的播放器示例，它的源代码的量也是不少的，其中很多知识点是我们可以学习和借鉴的。

## 总结构图

参照雷神（[雷霄骅](https://blog.csdn.net/leixiaohua1020)）的FFplay的总体函数调用结构图，自己总结了一个最新版本的结构，其中还有诸多不足，以后有机会慢慢完善；如下图所示。

<img src="./img/ffplay.png" alt="总结构图" style="zoom:50%;" />



这就不对主要函数分别解析，我来学习一下其中关键性的思想和ffplay的体系结构。

## 视频部分

### ffplay video的线程模式

<img src="./img/ffplay_video.png" style="zoom:50%;" />

ffplay选择了sdl作为显示SDK，以实现跨平台支持；因为使用了SDL，而video的显示也依赖SDL的窗口显示系统，所以先从main函数的SDL初始化看起：

```c
int main(int argc, char **argv)
{
  ...
       /* register all codecs, demux and protocols */
#if CONFIG_AVDEVICE
    avdevice_register_all(); //注册所有解码器
#endif
    avformat_network_init();

    init_opts();

    signal(SIGINT, sigterm_handler);  /* Interrupt (ANSI).    */
    signal(SIGTERM, sigterm_handler); /* Termination (ANSI).  */

    show_banner(argc, argv, options); //打印ffmpag库版本信息，编译时间，编译选项，类库信息等

    parse_options(NULL, argc, argv, options, opt_input_file); //解析输入的命令。
...
     if (SDL_Init(flags))
    { //初始化sdl
        av_log(NULL, AV_LOG_FATAL, "Could not initialize SDL - %s\n", SDL_GetError());
        av_log(NULL, AV_LOG_FATAL, "(Did you set the DISPLAY variable?)\n");
        exit(1);
    }

    SDL_EventState(SDL_SYSWMEVENT, SDL_IGNORE);
    SDL_EventState(SDL_USEREVENT, SDL_IGNORE);
		
  	//注册flush packet 只是一个标记作用，用于packet队列中，在对packet队列分析时有说明
    av_init_packet(&flush_pkt);
    flush_pkt.data = (uint8_t *)&flush_pkt;
  
 ...
       if (!display_disable)
    {
        int flags = SDL_WINDOW_HIDDEN;
        if (alwaysontop)
#if SDL_VERSION_ATLEAST(2, 0, 5)
            flags |= SDL_WINDOW_ALWAYS_ON_TOP;
#else
            av_log(NULL, AV_LOG_WARNING, "Your SDL version doesn't support SDL_WINDOW_ALWAYS_ON_TOP. Feature will be inactive.\n");
#endif
        if (borderless)
            flags |= SDL_WINDOW_BORDERLESS;
        else
            flags |= SDL_WINDOW_RESIZABLE;
         //创建sdl 窗口
        window = SDL_CreateWindow(program_name, SDL_WINDOWPOS_UNDEFINED, SDL_WINDOWPOS_UNDEFINED, default_width, default_height, flags);
        SDL_SetHint(SDL_HINT_RENDER_SCALE_QUALITY, "linear");
        if (window)
        {
          //创建sdl 窗口的渲染器
            renderer = SDL_CreateRenderer(window, -1, SDL_RENDERER_ACCELERATED | SDL_RENDERER_PRESENTVSYNC);
            if (!renderer)
            {
                av_log(NULL, AV_LOG_WARNING, "Failed to initialize a hardware accelerated renderer: %s\n", SDL_GetError());
                renderer = SDL_CreateRenderer(window, -1, 0);
            }
            if (renderer)
            {
                if (!SDL_GetRendererInfo(renderer, &renderer_info))
                    av_log(NULL, AV_LOG_VERBOSE, "Initialized %s renderer.\n", renderer_info.name);
            }
        }
        if (!window || !renderer || !renderer_info.num_texture_formats)
        {
            av_log(NULL, AV_LOG_FATAL, "Failed to create window or renderer: %s", SDL_GetError());
            do_exit(NULL);
        }
    }

  	is = stream_open(input_filename, file_iformat); //创建read_thread
    if (!is)
    {
        av_log(NULL, AV_LOG_FATAL, "Failed to initialize VideoState!\n");
        do_exit(NULL);
    }

    event_loop(is); //主线程 ，sdl——event事件监听和处理

}
```

而这里我们主要的两个函数是stream_open和event_loop；stream_open函数的作用是创建read_thread，read_thread会打开文件，解析封装，获取AVStream信息，启动解码器（创建解码线程），并开始读取文件；event_loop函数的作用是处理SDL事件队列中的事件和刷新显示数据，下面会针对这两个函数视频部分的内容进行详细说明。

#### stream_open

其实在stream_open函数里的关键内容都是初始化一些参数，主要的处理逻辑在read_thread中进行。

```c
static VideoState *stream_open(const char *filename, AVInputFormat *iformat)
{
 VideoState *is;

    is = av_mallocz(sizeof(VideoState)); //创建VideoState 这个很重要，它贯穿整个ffplay
  ...
    /* start video display */ //初始化解码后的帧队列
    if (frame_queue_init(&is->pictq, &is->videoq, VIDEO_PICTURE_QUEUE_SIZE, 1) < 0)
        goto fail;
  ...
     //初始化解码前的帧队列
    if (packet_queue_init(&is->videoq) < 0 ||
        ...
        //创建读线程的条件信号
    if (!(is->continue_read_thread = SDL_CreateCond()))
    {
        av_log(NULL, AV_LOG_FATAL, "SDL_CreateCond(): %s\n", SDL_GetError());
        goto fail;
    }
    //初始化时钟
    init_clock(&is->vidclk, &is->videoq.serial);
    ...
    is->read_tid = SDL_CreateThread(read_thread, "read_thread", is); //创建读线程
    ...
    fail:
        stream_close(is);//出错free一些初始化参数
        return NULL;
}
```

VideoState结构的参数详细说明：

```c
/*视频状态器，贯穿整个ffplay的结构*/
typedef struct VideoState
{
    SDL_Thread *read_tid;      //解复用（或读）线程
    AVInputFormat *iformat;    //输入格式
    int abort_request;         //中止请求
    int force_refresh;         //强制刷新
    int paused;                //暂停
    int last_paused;           //最后一次暂停状态
    int queue_attachments_req; //队列附件请求
    int seek_req;              //快进请求
    int seek_flags;            //快进标志
    int64_t seek_pos;          //快进位置
    int64_t seek_rel;
    int read_pause_return; //读暂停
    AVFormatContext *ic;   //解码格式上下文
    int realtime;          //是否实时码流

    Clock audclk; //音频时钟
    Clock vidclk; //视频时钟
    Clock extclk; //外部时钟

    FrameQueue pictq; //视频队列
    FrameQueue subpq; //字幕队列
    FrameQueue sampq; //pcm流队列

    Decoder auddec; //音频解码器
    Decoder viddec; //视频解码器
    Decoder subdec; //字幕解码器

    int audio_stream; //音频码流id

    int av_sync_type; //时钟同步类型

    double audio_clock;
    int audio_clock_serial; //音频时钟序号
    double audio_diff_cum;  // 用于音频差分计算 /* used for AV difference average computation */
    double audio_diff_avg_coef;
    double audio_diff_threshold;        //音频差分阈值
    int audio_diff_avg_count;           // 平均差分数量
    AVStream *audio_st;                 // 音频码流
    PacketQueue audioq;                 // 音频源包队列
    int audio_hw_buf_size;              // 硬件缓冲大小
    uint8_t *audio_buf;                 // 音频缓冲区
    uint8_t *audio_buf1;                // 音频缓冲区1
    unsigned int audio_buf_size;        // 音频缓冲大小 /* in bytes */
    unsigned int audio_buf1_size;       // 音频缓冲大小 1
    int audio_buf_index; /* in bytes */ // 音频缓冲索引
    int audio_write_buf_size;           // 音频写入缓冲大小
    int audio_volume;                   // 音量
    int muted;                          // 是否静音
    struct AudioParams audio_src;       // 音频参数
#if CONFIG_AVFILTER
    struct AudioParams audio_filter_src; // 音频过滤器
#endif
    struct AudioParams audio_tgt; // 音频参数
    struct SwrContext *swr_ctx;   // 音频转码上下文
    int frame_drops_early;
    int frame_drops_late;

    enum ShowMode
    { // 显示类型
        SHOW_MODE_NONE = -1,
        SHOW_MODE_VIDEO = 0,
        SHOW_MODE_WAVES,
        SHOW_MODE_RDFT,
        SHOW_MODE_NB
    } show_mode;
    int16_t sample_array[SAMPLE_ARRAY_SIZE]; // 采样数组
    int sample_array_index;                  // 采样索引
    int last_i_start;                        // 上一开始
    RDFTContext *rdft;                       // 自适应滤波器上下文
    int rdft_bits;                           // 自使用比特率
    FFTSample *rdft_data;                    // 快速傅里叶采样
    int xpos;
    double last_vis_time;
    SDL_Texture *vis_texture; // 音频Texture
    SDL_Texture *sub_texture; // 字幕Texture
    SDL_Texture *vid_texture; // 视频Texture

    int subtitle_stream;   // 字幕码流Id
    AVStream *subtitle_st; // 字幕码流
    PacketQueue subtitleq; // 字幕源包队列

    double frame_timer;                 // 帧计时器
    double frame_last_returned_time;    // 上一次返回时间
    double frame_last_filter_delay;     // 上一个过滤器延时
    int video_stream;                   // 视频码流Id
    AVStream *video_st;                 // 视频码流
    PacketQueue videoq;                 // 视频包队列
    double max_frame_duration;          // 最大帧间显示时间     // maximum duration of a frame - above this, we consider the jump a timestamp discontinuity
    struct SwsContext *img_convert_ctx; // 视频转码上下文
    struct SwsContext *sub_convert_ctx; // 字幕转码上下文
    int eof;                            // 结束标志

    char *filename;                 // 文件名
    int width, height, xleft, ytop; // 宽高，其实坐标
    int step;

#if CONFIG_AVFILTER
    int vfilter_idx;                   // 过滤器索引
    AVFilterContext *in_video_filter;  // the first filter in the video chain
    AVFilterContext *out_video_filter; // the last filter in the video chain
    AVFilterContext *in_audio_filter;  // the first filter in the audio chain
    AVFilterContext *out_audio_filter; // the last filter in the audio chain
    AVFilterGraph *agraph;             // audio filter graph
#endif

    int last_video_stream, last_audio_stream, last_subtitle_stream;

    SDL_cond *continue_read_thread;
} VideoState;
```

#### 读取线程`read_thread`

read_thread主要按以下步骤执行：

1. 准备阶段：打开文件，检测Stream信息，打开解码器
2. 主循环读数据，解封装：读取Packet，存入PacketQueue

read_thread的函数比较长，这里不贴完整代码，直接根据其功能分步分析。

