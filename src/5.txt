#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <SDL2/SDL.h>
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libswscale/swscale.h>
#include <libavutil/imgutils.h>
#include <libavutil/time.h>

// 播放器状态
typedef enum {
    PLAYER_STATE_STOPPED,  // 停止
    PLAYER_STATE_PLAYING,  // 播放
    PLAYER_STATE_PAUSED,   // 暂停
    PLAYER_STATE_SEEKING   // 快进/快退
} PlayerState;

// 帧队列结构
typedef struct FrameQueue {
    AVFrame **frames;
    int size;
    int capacity;  //  队列容量
    int read_index;
    int write_index;
    SDL_mutex *mutex;
    SDL_cond *cond_full;
    SDL_cond *cond_empty;
} FrameQueue;

// 播放器上下文
typedef struct VideoPlayer {
    // FFmpeg相关
    AVFormatContext *format_ctx;
    AVCodecContext *video_codec_ctx;
    AVStream *video_stream;
    int video_stream_index;
    struct SwsContext *sws_ctx;
    
    // SDL相关
    SDL_Window *window;
    SDL_Renderer *renderer;
    SDL_Texture *texture;
    
    // 线程相关
    SDL_Thread *decode_thread;
    SDL_Thread *video_thread;
    
    // 帧队列
    FrameQueue *frame_queue;
    
    // 播放控制
    PlayerState state;
    int quit;
    double video_clock;
    double frame_timer;
    int speed;
    
    // 同步相关
    SDL_mutex *state_mutex;
    SDL_cond *state_cond;
    
    // 视频信息
    int video_width;
    int video_height;
    double fps;
    
} VideoPlayer;

// 创建帧队列
FrameQueue* frame_queue_create(int capacity) {
    FrameQueue *queue = (FrameQueue*)calloc(1, sizeof(FrameQueue));
    if (!queue) return NULL;
    
    queue->frames = (AVFrame**)calloc(capacity, sizeof(AVFrame*));
    if (!queue->frames) {
        free(queue);
        return NULL;
    }
    
    for (int i = 0; i < capacity; i++) {
        queue->frames[i] = av_frame_alloc();
        if (!queue->frames[i]) {
            // 清理已分配的帧
            for (int j = 0; j < i; j++) {
                av_frame_free(&queue->frames[j]);
            }
            free(queue->frames);
            free(queue);
            return NULL;
        }
    }
    
    queue->capacity = capacity;
    queue->mutex = SDL_CreateMutex();
    queue->cond_full = SDL_CreateCond();
    queue->cond_empty = SDL_CreateCond();
    
    return queue;
}

// 销毁帧队列
void frame_queue_destroy(FrameQueue *queue) {
    if (!queue) return;
    
    for (int i = 0; i < queue->capacity; i++) {
        av_frame_free(&queue->frames[i]);
    }
    free(queue->frames);
    
    SDL_DestroyMutex(queue->mutex);
    SDL_DestroyCond(queue->cond_full);
    SDL_DestroyCond(queue->cond_empty);
    free(queue);
}

// 向队列推入帧
int frame_queue_put(FrameQueue *queue, AVFrame *frame) {
    SDL_LockMutex(queue->mutex);
    
    // 等待队列有空间
    while (queue->size >= queue->capacity) {
        SDL_CondWait(queue->cond_full, queue->mutex);
    }
    
    // 复制帧数据
    av_frame_unref(queue->frames[queue->write_index]);
    av_frame_ref(queue->frames[queue->write_index], frame);
    
    queue->write_index = (queue->write_index + 1) % queue->capacity;
    queue->size++;
    
    SDL_CondSignal(queue->cond_empty);
    SDL_UnlockMutex(queue->mutex);
    
    return 0;
}

// 从队列获取帧
int frame_queue_get(FrameQueue *queue, AVFrame *frame) {
    SDL_LockMutex(queue->mutex);
    
    // 等待队列有数据
    while (queue->size == 0) {
        SDL_CondWait(queue->cond_empty, queue->mutex);
    }
    
    // 复制帧数据
    av_frame_unref(frame);
    av_frame_ref(frame, queue->frames[queue->read_index]);
    
    queue->read_index = (queue->read_index + 1) % queue->capacity;
    queue->size--;
    
    SDL_CondSignal(queue->cond_full);
    SDL_UnlockMutex(queue->mutex);
    
    return 0;
}

// 获取当前时间（秒）
double get_time() {
    return av_gettime() / 1000000.0;
}

// 解码线程
int decode_thread_func(void *arg) {
    VideoPlayer *player = (VideoPlayer*)arg;
    AVPacket *packet = av_packet_alloc();
    AVFrame *frame = av_frame_alloc();
    int ret;
    
    printf("解码线程启动\n");
    
    while (1) {
        // 检查退出标志
        SDL_LockMutex(player->state_mutex);
        if (player->quit) {
            SDL_UnlockMutex(player->state_mutex);
            break;
        }
        
        // 检查播放状态
        while (player->state == PLAYER_STATE_PAUSED && !player->quit) {
            SDL_CondWait(player->state_cond, player->state_mutex);
        }
        
        if (player->quit) {
            SDL_UnlockMutex(player->state_mutex);
            break;
        }
        SDL_UnlockMutex(player->state_mutex);
        
        // 读取数据包
        ret = av_read_frame(player->format_ctx, packet);
        if (ret < 0) {
            if (ret == AVERROR_EOF) {
                printf("文件读取完成\n");
                break;
            } else {
                printf("读取数据包失败: %s\n", av_err2str(ret));
                break;
            }
        }
        
        // 只处理视频流
        if (packet->stream_index == player->video_stream_index) {
            // 发送数据包到解码器
            ret = avcodec_send_packet(player->video_codec_ctx, packet);
            if (ret < 0) {
                printf("发送数据包到解码器失败: %s\n", av_err2str(ret));
                av_packet_unref(packet);
                continue;
            }
            
            // 接收解码后的帧
            while (ret >= 0) {
                ret = avcodec_receive_frame(player->video_codec_ctx, frame);
                if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF) {
                    break;
                } else if (ret < 0) {
                    printf("接收解码帧失败: %s\n", av_err2str(ret));
                    break;
                }
                
                // 计算帧的时间戳
                if (frame->pts != AV_NOPTS_VALUE) {
                    // 仅将原始 PTS 从“视频时间基”转换为“秒”，再转微秒（保持相对时间）
                    double pts = frame->pts * av_q2d(player->video_stream->time_base);
                    frame->pts = (int64_t)(pts * 1000000); 
                }
                
                // 将帧放入队列
                frame_queue_put(player->frame_queue, frame);
            }
        }
        
        av_packet_unref(packet);
    }
    
    av_frame_free(&frame);
    av_packet_free(&packet);
    
    printf("解码线程结束\n");
    return 0;
}

// 视频播放线程
int video_thread_func(void *arg) {
    VideoPlayer *player = (VideoPlayer*)arg;
    AVFrame *frame = av_frame_alloc();
    uint8_t *rgb_buffer = NULL;
    int rgb_buffer_size = 0;
    
    printf("视频播放线程启动\n");
    
    // 分配RGB缓冲区
    rgb_buffer_size = av_image_get_buffer_size(AV_PIX_FMT_RGB24, 
                                              player->video_width, 
                                              player->video_height, 1);
    rgb_buffer = (uint8_t*)av_malloc(rgb_buffer_size);

    while (1) {
        // 检查退出标志
        SDL_LockMutex(player->state_mutex);
        if (player->quit) {
            SDL_UnlockMutex(player->state_mutex);
            break;
        }
        
        // 检查播放状态
        while (player->state == PLAYER_STATE_PAUSED && !player->quit) {
            SDL_CondWait(player->state_cond, player->state_mutex);
        }
        
        if (player->quit) {
            SDL_UnlockMutex(player->state_mutex);
            break;
        }
        SDL_UnlockMutex(player->state_mutex);
        
        // 从队列获取帧
        frame_queue_get(player->frame_queue, frame);
        
        // 计算显示时间
        double pts = 0;
        if (frame->pts != AV_NOPTS_VALUE) {
            pts = frame->pts / 1000000.0; // 转换为秒
        } else {
            pts = player->video_clock + 1.0 / player->fps;
        }
        
        // 同步延迟
        
        double delay = pts - player->video_clock;
        double frame_duration = 1.0 / (player->fps* player->speed);
        if(delay > 0 && delay < frame_duration){
            SDL_Delay((uint64_t)(delay * 1000));
        }
       
        
        // 转换YUV到RGB
        uint8_t *rgb_data[1] = { rgb_buffer };
        int rgb_linesize[1] = { player->video_width * 3 };
        
        sws_scale(player->sws_ctx,
                 (const uint8_t *const *)frame->data, frame->linesize,
                 0, player->video_height,
                 rgb_data, rgb_linesize);
        
        // 更新纹理
        SDL_UpdateTexture(player->texture, NULL, rgb_buffer, 
                         player->video_width * 3);
        
        // 渲染
        SDL_SetRenderDrawColor(player->renderer, 0, 0, 0, 255);
        SDL_RenderClear(player->renderer);
        SDL_RenderCopy(player->renderer, player->texture, NULL, NULL);
        SDL_RenderPresent(player->renderer);
        
        player->video_clock = pts;
    }
    
    av_free(rgb_buffer);
    av_frame_free(&frame);
    
    printf("视频播放线程结束\n");
    return 0;
}


// 初始化播放器
int init_player(VideoPlayer *player, const char *filename) {
    int ret;
    
    // 初始化FFmpeg
    av_log_set_level(AV_LOG_INFO);
    
    // 打开输入文件
    ret = avformat_open_input(&player->format_ctx, filename, NULL, NULL);
    if (ret < 0) {
        printf("无法打开文件: %s\n", av_err2str(ret));
        return -1;
    }
    
    // 获取流信息
    ret = avformat_find_stream_info(player->format_ctx, NULL);
    if (ret < 0) {
        printf("无法获取流信息: %s\n", av_err2str(ret));
        return -1;
    }
    
    // 查找视频流
    player->video_stream_index = av_find_best_stream(player->format_ctx, 
                                                    AVMEDIA_TYPE_VIDEO, 
                                                    -1, -1, NULL, 0);
    if (player->video_stream_index < 0) {
        printf("未找到视频流\n");
        return -1;
    }
    
    player->video_stream = player->format_ctx->streams[player->video_stream_index];
    
    // 查找解码器
    const AVCodec *codec = avcodec_find_decoder(player->video_stream->codecpar->codec_id);
    if (!codec) {
        printf("未找到解码器\n");
        return -1;
    }
    
    // 创建解码器上下文
    player->video_codec_ctx = avcodec_alloc_context3(codec);
    if (!player->video_codec_ctx) {
        printf("无法分配解码器上下文\n");
        return -1;
    }
    
    // 复制解码器参数
    ret = avcodec_parameters_to_context(player->video_codec_ctx, 
                                       player->video_stream->codecpar);
    if (ret < 0) {
        printf("无法复制解码器参数: %s\n", av_err2str(ret));
        return -1;
    }
    
    // 打开解码器
    ret = avcodec_open2(player->video_codec_ctx, codec, NULL);
    if (ret < 0) {
        printf("无法打开解码器: %s\n", av_err2str(ret));
        return -1;
    }

    // 获取视频信息
    player->video_width = player->video_codec_ctx->width;
    player->video_height = player->video_codec_ctx->height;
    player->fps = av_q2d(player->video_stream->avg_frame_rate);
    if (player->fps <= 0) {
        player->fps = 25.0; // 默认帧率
    }
    
    printf("=== 视频信息 ===\n");
    printf("分辨率: %dx%d\n", player->video_width, player->video_height);
    printf("帧率: %.2f fps\n", player->fps);
    printf("编解码器: %s\n", codec->name);
    printf("时长: %.2f 秒\n", 
           player->format_ctx->duration / (double)AV_TIME_BASE);
    
    // 初始化SwsContext
    player->sws_ctx = sws_getContext(
        player->video_width, player->video_height, player->video_codec_ctx->pix_fmt,
        player->video_width, player->video_height, AV_PIX_FMT_RGB24,
        SWS_BILINEAR, NULL, NULL, NULL
    );
    
    // 初始化SDL
    if (SDL_Init(SDL_INIT_VIDEO | SDL_INIT_TIMER) < 0) {
        printf("SDL初始化失败: %s\n", SDL_GetError());
        return -1;
    }
    
    // 创建窗口
    player->window = SDL_CreateWindow(
        "多线程视频播放器",
        SDL_WINDOWPOS_CENTERED, SDL_WINDOWPOS_CENTERED,
        player->video_width, player->video_height,
        SDL_WINDOW_SHOWN | SDL_WINDOW_RESIZABLE
    );
    
    if (!player->window) {
        printf("创建窗口失败: %s\n", SDL_GetError());
        return -1;
    }
    
    // 创建渲染器
    player->renderer = SDL_CreateRenderer(player->window, -1, 
        SDL_RENDERER_ACCELERATED | SDL_RENDERER_PRESENTVSYNC);
    
    if (!player->renderer) {
        printf("创建渲染器失败: %s\n", SDL_GetError());
        return -1;
    }
    
    // 创建纹理
    player->texture = SDL_CreateTexture(player->renderer,
        SDL_PIXELFORMAT_RGB24,
        SDL_TEXTUREACCESS_STREAMING,
        player->video_width, player->video_height);
    
    if (!player->texture) {
        printf("创建纹理失败: %s\n", SDL_GetError());
        return -1;
    }
    
    // 创建帧队列
    player->frame_queue = frame_queue_create(20);
    if (!player->frame_queue) {
        printf("创建帧队列失败\n");
        return -1;
    }
    
    // 创建同步互斥锁
    player->state_mutex = SDL_CreateMutex();
    player->state_cond = SDL_CreateCond();
    
    // 设置初始状态
    player->state = PLAYER_STATE_PLAYING;
    player->quit = 0;
    player->video_clock = 0;
    player->frame_timer = get_time();
    player->speed = 1;
    return 0;
}

// 启动播放器
void start_player(VideoPlayer *player) {
    printf("\n=== 启动播放器 ===\n");
    printf("控制说明:\n");
    printf("  空格键: 暂停/播放\n");
    printf("  S键: 显示当前时间\n");
    printf("  ESC键: 退出\n");
    printf("\n");
    
    // 启动线程
    //player->event_thread = SDL_CreateThread(event_thread_func, "EventThread", player);
    player->decode_thread = SDL_CreateThread(decode_thread_func, "DecodeThread", player);
    player->video_thread = SDL_CreateThread(video_thread_func, "VideoThread", player);

    if (!player->decode_thread || !player->video_thread) {
        printf("创建线程失败\n");
        player->quit = 1;
    }
}

// 等待播放器结束
void wait_player(VideoPlayer *player) {
    // 等待线程结束
    if (player->decode_thread) SDL_WaitThread(player->decode_thread, NULL);
    if (player->video_thread) SDL_WaitThread(player->video_thread, NULL);
    //if (player->event_thread) SDL_WaitThread(player->event_thread, NULL);
}

// 清理播放器
void cleanup_player(VideoPlayer *player) {
    // 设置退出标志
    SDL_LockMutex(player->state_mutex);
    player->quit = 1;
    SDL_CondBroadcast(player->state_cond);
    SDL_UnlockMutex(player->state_mutex);
    
    // 等待线程结束
    wait_player(player);
    
    // 释放SDL资源
    if (player->texture) SDL_DestroyTexture(player->texture);
    if (player->renderer) SDL_DestroyRenderer(player->renderer);
    if (player->window) SDL_DestroyWindow(player->window);
    SDL_Quit();
    
    // 释放FFmpeg资源
    if (player->sws_ctx) sws_freeContext(player->sws_ctx);
    if (player->video_codec_ctx) avcodec_free_context(&player->video_codec_ctx);
    if (player->format_ctx) avformat_close_input(&player->format_ctx);
    
    // 释放队列和同步对象
    if (player->frame_queue) frame_queue_destroy(player->frame_queue);
    if (player->state_mutex) SDL_DestroyMutex(player->state_mutex);
    if (player->state_cond) SDL_DestroyCond(player->state_cond);
    
    printf("播放器资源清理完成\n");
}

int main(int argc, char *argv[]) {
    VideoPlayer player = {0};
    char* filename = "C:/Users/c/Desktop/C++/C++/study05/src/Forrest_Gump_IMAX.mp4";
    
    if (argc > 1) {
        filename = argv[1];
    }
    
    // 初始化播放器
    if (init_player(&player, filename) < 0) {
        printf("初始化播放器失败\n");
        return -1;
    }
    
    // 启动播放器
    start_player(&player);
    SDL_Event event;
    int running = 1;
    while (running) {
        while (SDL_PollEvent(&event)) {
            switch (event.type) {
            case SDL_QUIT:
                SDL_LockMutex(player.state_mutex);
                player.quit = 1;
                SDL_CondBroadcast(player.state_cond);
                SDL_UnlockMutex(player.state_mutex);
                running = 0;
                break;
            case SDL_KEYDOWN:
                SDL_LockMutex(player.state_mutex);
                if (event.key.keysym.sym == SDLK_ESCAPE) {
                    player.quit = 1;
                    SDL_CondBroadcast(player.state_cond);
                    running = 0;
                } else if (event.key.keysym.sym == SDLK_SPACE) {
                    if (player.state == PLAYER_STATE_PLAYING) {
                        player.state = PLAYER_STATE_PAUSED;
                        printf("暂停播放\n");
                    } else if (player.state == PLAYER_STATE_PAUSED) {
                        player.state = PLAYER_STATE_PLAYING;
                        SDL_CondBroadcast(player.state_cond);
                        printf("继续播放\n");
                    }
                } else if (event.key.keysym.sym == SDLK_s) {
                    printf("当前播放时间: %.2f秒\n", player.video_clock);
                }
                else if (event.key.keysym.sym == SDLK_UP) {
                    player.speed+=0.1;
                    printf("播放速度: %dx\n", player.speed);
                } else if (event.key.keysym.sym == SDLK_DOWN) {
                     player.speed-=0.1;
                    if(player.speed <0) player.speed = 0.1;
                    printf("播放速度: %dx\n", player.speed);
                }
                SDL_UnlockMutex(player.state_mutex);
                break;
            case SDL_WINDOWEVENT:
                printf("窗口事件: %d\n", event.window.event);
                if (event.window.event == SDL_WINDOWEVENT_RESIZED) {
                    printf("窗口大小改变: %dx%d\n", event.window.data1, event.window.data2);
                    // 这里可以重建纹理或调整渲染逻辑
                }
                break;
            default:
                break;
            }
        }
        SDL_Delay(10); // 防止主循环空转
        SDL_LockMutex(player.state_mutex);
        if (player.quit) running = 0;
        SDL_UnlockMutex(player.state_mutex);
    }
    // 等待播放结束
    wait_player(&player);
    
    // 清理资源
    cleanup_player(&player);
    
    return 0;
}
