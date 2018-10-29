# [ffmpeg入门手册](http://dranger.com/ffmpeg/tutorial01.html)

在教程结束时，我们会有一个可用的不到1000行代码的视频播放器。在视频播放器中，我们使用[SDL](http://www.libsdl.org/)用于音视频的输出。SDL是一个跨平台的多媒体库，被MPEG播放软件、模拟器、视频游戏等使用。

## 截图

### 概述

视频文件有一些基础的组件。首先，文件本身被称为一个容器，容器的类型决定了文件里的信息的存放位置。AVI和QuickTime都是容器。接下来，我们有一些流，通常我们会有一个音频流和一个视频流。流里面的有一系列被称为帧的元素。每一个流被一种编解码器编码。编码器定义了真实的数据如何被编解码。DivX和MP3都是编码器。Packets被从流中读出，它们是一小段数据可以被解码成原始的帧，以便让应用程序进行处理。一个packet可能包含一个视频帧或者多个音频帧，也可能几个packet组成一个帧。

在最简单的层面上，处理音视频流非常简单：
```
10 打开video.avi的视频流
20 从视频流中读取packet解码到frame
30 如果当前frame不完整 GOTO 20
40 处理视频帧
50 GOTO 20
```

用ffmpeg处理视频基本上都是上述的流程，只是某些程序处理视频帧部分非常复杂。在本例子中，我们打开一个文件，读入其中的视频流，我们的处理视频帧为将帧写入到一个ppm文件。

### 打开文件

首先，我们打开一个文件。使用ffmpeg时，首先需要初始化库
```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <ffmpeg/swscale.h>
...
int main(int argc, charg *argv[]) {
av_register_all();
```
上述代码将所有可用的文件格式和编解码器注册，所有当我们打开文件时对应的格式和编解码器会自动的被使用。我们只需要调用av_register_all()一次，所以我们在main中调用。我们也可以只注册指定类型的文件格式和编解码器，但是一般情况下并不必要。

接下来我们打开文件

```c
AVFormatContext *pFormatCtx = NULL;

// Open video file
if(avformat_open_input(&pFormatCtx, argv[1], NULL, 0, NULL)!=0)
  return -1; // Couldn't open file
```

我们将文件名作为第一个参数，该函数读取文件的头信息并将文件格式相关的信息存储在我们传入的AVFormatContext结构体中。后面三个参数用户指定文件的格式、缓存的大小和格式的参数，通过设置为NULL或者0，libavformat将会自动检测它们。

这个函数只检测视频头，我们需要继续检查文件中的流信息：
```c
// Retrieve stream information
if(avformat_find_stream_info(pFormatCtx, NULL)<0)
  return -1; // Couldn't find stream information
```
这个函数将流信息存储到pFormatCtx->streams，我们使用一个debug函数来展示其内部的数据：
```c
// Retrieve stream information
// Dump information about file onto standard error
av_dump_format(pFormatCtx, 0, argv[1], 0);
```
pFormatCtx->streams是指针的数组，其大小为pFormatCtx->nb_streams，因此我们可以遍历直到找到视频流。
```c
int i;
AVCodecContext *pCodecCtxOrig = NULL;
AVCodecContext *pCodecCtx = NULL;

// Find the first video stream
videoStream=-1;
for(i=0; i<pFormatCtx->nb_streams; i++)
  if(pFormatCtx->streams[i]->codec->codec_type==AVMEDIA_TYPE_VIDEO) {
    videoStream=i;
    break;
  }
if(videoStream==-1)
  return -1; // Didn't find a video stream

// Get a pointer to the codec context for the video stream
pCodecCtxOrig=pFormatCtx->streams[videoStream]->codec;
```

流关于编解码器的信息被称为编解码器的上下文，它包含着这个流使用的编解码器的全部信息，现在我们有了一个指向它的指针。我们需要找到真正的编解码器并将其打开。
```c
AVCodec *pCodec = NULL;

// Find the decoder for the video stream
pCodec=avcodec_find_decoder(pCodecCtxOrig->codec_id);
if(pCodec==NULL) {
  fprintf(stderr, "Unsupported codec!\n");
  return -1; // Codec not found
}
// Copy context
pCodecCtx = avcodec_alloc_context3(pCodec);
if(avcodec_copy_context(pCodecCtx, pCodecCtxOrig) != 0) {
  fprintf(stderr, "Couldn't copy codec context");
  return -1; // Error copying codec context
}
// Open codec
if(avcodec_open2(pCodecCtx, pCodec)<0)
  return -1; // Could not open codec
```
需要注意的是，我们不能直接使用视频流的AVCodecContext，所以我们需要使用avcodec_copy_context()将该上下文拷贝到一个新的位置。

## 存储数据

我们需要一个地方存储视频帧
```c
AVFrame *pFrame = NULL;

// Allocate video frame
pFrame=av_frame_alloc();
```

因为我们准备输出ppm格式，它以24-bit的RGB存储，所以我们需要将帧从它原生的格式转为RGB。ffmpeg可以完成这种转换，我们需要分配一块区域来存储转换后的帧

```c
// Allocate an AVFrame structure
pFrameRGB=av_frame_alloc();
if(pFrameRGB==NULL)
  return -1;
```
我们还需要分配一块区域去存储转换生成的帧的原始数据，我们使用avpicture_get_size函数去获取我们需要的大小，并且手工分配内存区域
```c
uint8_t *buffer = NULL;
int numBytes;
// Determine required buffer size and allocate buffer
numBytes=avpicture_get_size(PIX_FMT_RGB24, pCodecCtx->width,
                            pCodecCtx->height);
buffer=(uint8_t *)av_malloc(numBytes*sizeof(uint8_t));
```
av_malloc是ffmpeg对malloc的简单包装，以便确保分配的内存地址是对齐的。它无法避免内存泄露、释放两次以及其它的malloc问题。

接下来我们使用avpicture_fill来将帧和新分配的buffer关联。关于AVPicture：AVPicture是AVFrame的子集，AVFrame开始的部分与AVPicture结构体相同。

```c
// Assign appropriate parts of buffer to image planes in pFrameRGB
// Note that pFrameRGB is an AVFrame, but AVFrame is a superset
// of AVPicture
avpicture_fill((AVPicture *)pFrameRGB, buffer, PIX_FMT_RGB24,
                pCodecCtx->width, pCodecCtx->height);
```

接着我们就可以读取数据了。

## 读取数据

我们接下来需要读整个视频流，读入一个packet，解码到frame，如果frame是完整的，就转换并保存它。

```c
struct SwsContext *sws_ctx = NULL;
int frameFinished;
AVPacket packet;
// initialize SWS context for software scaling
sws_ctx = sws_getContext(pCodecCtx->width,
    pCodecCtx->height,
    pCodecCtx->pix_fmt,
    pCodecCtx->width,
    pCodecCtx->height,
    PIX_FMT_RGB24,
    SWS_BILINEAR,
    NULL,
    NULL,
    NULL
    );

i=0;
while(av_read_frame(pFormatCtx, &packet)>=0) {
  // Is this a packet from the video stream?
  if(packet.stream_index==videoStream) {
	// Decode video frame
    avcodec_decode_video2(pCodecCtx, pFrame, &frameFinished, &packet);
    
    // Did we get a video frame?
    if(frameFinished) {
    // Convert the image from its native format to RGB
        sws_scale(sws_ctx, (uint8_t const * const *)pFrame->data,
		  pFrame->linesize, 0, pCodecCtx->height,
		  pFrameRGB->data, pFrameRGB->linesize);
	
        // Save the frame to disk
        if(++i<=5)
          SaveFrame(pFrameRGB, pCodecCtx->width, 
                    pCodecCtx->height, i);
    }
  }
    
  // Free the packet that was allocated by av_read_frame
  av_free_packet(&packet);
}
```

上述过程较为简单，首先av_read_frame()读取一个packet并且存储到一个AVPacket结构体中。需要注意的是，我们只分配了结构体的内存，ffmpeg替我们分配了内部的数据，packet.data指向它。后面我们会用av_free_packet()函数释放packet.data指针指向的数据。avcodec_decode_video()将packet转成frame，然而解码完一个packet之后我们得到的信息可能不够，所以avcodec_decode_video()设置我们传入的frameFinished来判定一帧是否结束。接下来我们通过sws_scale()函数来从原生格式(pCodecCtx->pix_fmt) 转为RGB。记得我们可以将AVFrame的指针cast为AVPicture指针。最后我们将帧和宽高的信息一起传入SaveFrame函数。

现在我们使用SaveFrame函数将RGB信息写入到ppm格式的文件，代码如下：
```c
void SaveFrame(AVFrame *pFrame, int width, int height, int iFrame) {
  FILE *pFile;
  char szFilename[32];
  int  y;
  
  // Open file
  sprintf(szFilename, "frame%d.ppm", iFrame);
  pFile=fopen(szFilename, "wb");
  if(pFile==NULL)
    return;
  
  // Write header
  fprintf(pFile, "P6\n%d %d\n255\n", width, height);
  
  // Write pixel data
  for(y=0; y<height; y++)
    fwrite(pFrame->data[0]+y*pFrame->linesize[0], 1, width*3, pFile);
  
  // Close file
  fclose(pFile);
}
```
我们打开了一个文件，然后一行一行的写入RGB数据，ppm文件的头是RGB信息的字符串，里面包含了图像的宽高以及RGB数值的最大长度。

在完成视频流的读取后，我们进行一系列的清理工作：
```c
// Free the RGB image
av_free(buffer);
av_free(pFrameRGB);

// Free the YUV frame
av_free(pFrame);

// Close the codecs
avcodec_close(pCodecCtx);
avcodec_close(pCodecCtxOrig);

// Close the video file
avformat_close_input(&pFormatCtx);

return 0;
```
我们使用av_free释放avcode_alloc_frame和av_malloc分配的内存。

## 输出到屏幕

### SDL与视频

为了在屏幕上绘制，我们使用SDL库，SDL是Simple Direct Layer，是一套出色的用户多媒体的跨平台的库。SDL有很多方法展示图片到屏幕，展示电影到屏幕最常用的是YUV overlay。YUV跟RGB相似，是一种存储图像原始数据的方式。YUV420P利用人类眼睛的特定，压缩了色度的带宽。SDL接收四种类型的YUV格式，但是YV12是最快的。ffmpeg可以将图片转为YUV420P，另外很多视频流本身就以这种存储的，或者可以很容易的转换为这种形式。YV12与YUV420的差异仅仅是UV数字位置互换。

所以我们现在需要一个函数替换掉SaveFrame()，来把帧输出到屏幕。但是首先我们来看一下怎么使用SDL库。首先我们引入头文件并且初始化SDL：
```c
#include <SDL.h>
#include <SDL_thread.h>

if(SDL_Init(SDL_INIT_VIDEO | SDL_INIT_AUDIO | SDL_INIT_TIMER)) {
  fprintf(stderr, "Could not initialize SDL - %s\n", SDL_GetError());
  exit(1);
}
```
SDL_Init()函数告诉库我们需要用哪些特性。SDL_GetError()是一个很好用的debug函数。

### 创建显示

我们需要屏幕上的一块区域用于防止内容。在SDL里展示图片的基本区域叫surface。

```c
SDL_Surface *screen;

screen = SDL_SetVideoMode(pCodecCtx->width, pCodecCtx->height, 0, 0);
if(!screen) {
  fprintf(stderr, "SDL: could not set video mode - exiting\n");
  exit(1);
}
```
这里就设置了屏幕的宽和高，接下来的参数设定了屏幕上的位深度，0代表与当前的显示器相同（在mac上不生效，参考源代码）。

接下来我们创建一个YUV overlay在屏幕上以便可以放置视频在上面，并且设置SWSContext来把图片数据转为YUV420：
```c
SDL_Overlay     *bmp = NULL;
struct SWSContext *sws_ctx = NULL;

bmp = SDL_CreateYUVOverlay(pCodecCtx->width, pCodecCtx->height,
                           SDL_YV12_OVERLAY, screen);

// initialize SWS context for software scaling
sws_ctx = sws_getContext(pCodecCtx->width,
                         pCodecCtx->height,
			 pCodecCtx->pix_fmt,
			 pCodecCtx->width,
			 pCodecCtx->height,
			 PIX_FMT_YUV420P,
			 SWS_BILINEAR,
			 NULL,
			 NULL,
			 NULL
			 );
```
按照我们之前说的，我们使用YV12来展示图片，但是我们从ffmpeg拿到的是YUV420格式的数据。

### 展示图片

接下来我们只需要展示图片，我们接着获取完整的帧之后继续。我们不再需要处理RGB帧相关的事务，用展示图片的代码替换掉SaveFrame()。为了展示图片，我们创建一个AVPicture的结构体并且设置其data指针和linesize指向YUV overlay内部的指针：
```c
  if(frameFinished) {
    SDL_LockYUVOverlay(bmp);

    AVPicture pict;
    pict.data[0] = bmp->pixels[0];
    pict.data[1] = bmp->pixels[2];
    pict.data[2] = bmp->pixels[1];

    pict.linesize[0] = bmp->pitches[0];
    pict.linesize[1] = bmp->pitches[2];
    pict.linesize[2] = bmp->pitches[1];

    // Convert the image into YUV format that SDL uses
    sws_scale(sws_ctx, (uint8_t const * const *)pFrame->data,
	      pFrame->linesize, 0, pCodecCtx->height,
	      pict.data, pict.linesize);
    
    SDL_UnlockYUVOverlay(bmp);
  }
```
首先，我们锁定overlay因为我们将要向其写入数据。在这里虽然没有影响，但是它是一个预防bug的好习惯。AVPicture结构体有一个名为data的指针的数组。因为我们正在处理YUV420P，所以我们只需要3个通道，也即三组数据。其它的格式可能需要第四个指针指向alpha通道之类的。linesize顾名思义。在与之相似的YUV overlay结构体里，它们分别叫pixels和pitches。pitches在SDL术语中指给定线的宽度。所以我们将data的三个数组指向overlay，我们想pict写入时，实际上改变的是overlay的值，它的空间已经被分配。类似的，我们从overlay中直接读取了linesize信息。我们将格式转为PIX_FMT_YUV420P，所以我们使用了sws_scale函数。

### 绘制图片
我们需要告知SDL来展示我们给它的数据，我们还需要传递一个矩形给函数来告知应该缩放到大多的宽和高。这样SDL就可以为我们完成缩放了，SDL可以使用显卡来加速缩放过程。
```c
SDL_Rect rect;

  if(frameFinished) {
    /* ... code ... */
    // Convert the image into YUV format that SDL uses
    sws_scale(sws_ctx, (uint8_t const * const *)pFrame->data,
              pFrame->linesize, 0, pCodecCtx->height,
	      pict.data, pict.linesize);
    
    SDL_UnlockYUVOverlay(bmp);
	rect.x = 0;
	rect.y = 0;
	rect.w = pCodecCtx->width;
	rect.h = pCodecCtx->height;
	SDL_DisplayYUVOverlay(bmp, &rect);
  }
```
现在视频就展示出来了。

接下来我们来看一下SDL的另一项特性：事件系统。SDL可以被设置为在有键盘或者鼠标输入或者向其发送一个信号时，产生一个事件。如果我们的程序想要处理这些用户输入，可以检测这些时间。我们的程序也可以产生时间并发送给SDL的时间系统。这才多线程编程中非常有用。在现在的程序里，我们在每一个packet处理完之后检查一下事件。现在我们只处理SDL_QUIT事件来退出程序。
```c
SDL_Event       event;

    av_free_packet(&packet);
    SDL_PollEvent(&event);
    switch(event.type) {
    case SDL_QUIT:
      SDL_Quit();
      exit(0);
      break;
    default:
      break;
    }
```
## 播放声音

### 音频

现在我们需要播放声音。SDL提供输出声音的函数。SDL_OpenAudio()被用于打开音频设备，它接收一个SDL_AudioSpec的结构体，结构体中包含了音频输出所需要的全部信息。

在开始之前，我们来解释一下计算机如何处理音频。数字音频包含了一个很长的采样流。每一个采样代表了声波的数值。声音被按照一个指定的采样频率存储，这告知了播放每个采样的速度，它是以每秒钟的采样数衡量的。例如可能的采样数有22,050 和44,100的采样每秒，分别用于广播和CD。另外，多数音频包含不止一个频道以便实现立体声或者环绕声，例如，如果采样立体声，每次就有两个采样。当我们拿到电影文件时，我们不知道能拿到几个采样，ffmpeg不会给我们分数个采样，它也不会把一个立体声的采样分开。

SDL函数播放音频的过程如下：我们设定音频的属性包括采样频率、频道书等等，并设定一个回调函数和回调函数的参数。当我们开始播放音频时，SDL会不断的调用callback函数让其填充给定数量的byte的音频缓存。当我们当这些信息放入SDL_AudioSpec之后调用SDL_OpenAudio()，它将打开音频设备并且给我们另一个AudioSpec结构体。

### 初始化音频

需要明白，我们现在还没有音频流的任何信息。我们回到找到视频流的代码的地方，找到哪一个流是音频流。
```c
// Find the first video stream
videoStream=-1;
audioStream=-1;
for(i=0; i < pFormatCtx->nb_streams; i++) {
  if(pFormatCtx->streams[i]->codec->codec_type==AVMEDIA_TYPE_VIDEO
     &&
       videoStream < 0) {
    videoStream=i;
  }
  if(pFormatCtx->streams[i]->codec->codec_type==AVMEDIA_TYPE_AUDIO &&
     audioStream < 0) {
    audioStream=i;
  }
}
if(videoStream==-1)
  return -1; // Didn't find a video stream
if(audioStream==-1)
  return -1;
```

然后我们从AVCodecContext中拿到我们需要的全部信息，跟视频流一样：
```c
AVCodecContext *aCodecCtxOrig;
AVCodecContext *aCodecCtx;

aCodecCtxOrig=pFormatCtx->streams[audioStream]->codec;
```

同样的我们需要打开音频的编解码器:
```c
AVCodec         *aCodec;

aCodec = avcodec_find_decoder(aCodecCtx->codec_id);
if(!aCodec) {
  fprintf(stderr, "Unsupported codec!\n");
  return -1;
}
// Copy context
aCodecCtx = avcodec_alloc_context3(aCodec);
if(avcodec_copy_context(aCodecCtx, aCodecCtxOrig) != 0) {
  fprintf(stderr, "Couldn't copy codec context");
  return -1; // Error copying codec context
}
/* set up SDL Audio here */

avcodec_open2(aCodecCtx, aCodec, NULL);
```
从编解码器的上下文中，我们可以获取初始化音频需要的全部信息：
```c
wanted_spec.freq = aCodecCtx->sample_rate;
wanted_spec.format = AUDIO_S16SYS;
wanted_spec.channels = aCodecCtx->channels;
wanted_spec.silence = 0;
wanted_spec.samples = SDL_AUDIO_BUFFER_SIZE;
wanted_spec.callback = audio_callback;
wanted_spec.userdata = aCodecCtx;

if(SDL_OpenAudio(&wanted_spec, &spec) < 0) {
  fprintf(stderr, "SDL_OpenAudio: %s\n", SDL_GetError());
  return -1;
}
```
参数的含义如下：
1. freq：采样率；
1. format：告诉SDL我们提供的是什么格式。S16SYS中的S代表signed，16代表每个采样16bit长，SYS代表字节次序与当前操作系统一致。这一格式是我们avcodec_decode_audio2将要给我们的音频的格式；
1. channels： 音频频道数；
1. silence：代表无声的数值。因为声音是有符号的，所以0是合适的；
1. samples：我们希望SDL问我们要更多的音频数据时，提供给我们的音频缓存的大小。比较好的数值在512到8192之间，ffplay使用1024；
1. callback：此处我们传入一个回调函数，后续我们会继续讨论这个回调函数；
1. userdata：SDL允许向callback函数提供一个void指针指向的任意用户数据。我们希望它知道编解码器的上下文。

然后我们调用了SDL_OpenAudio打开音频设备。

### 队列

我们开始从流中提取音频信息。我们怎么使用这些信息呢，我们将不断的从电影文件中读取packet，然而与此同时SDL将要调用callback函数。解决方式是创造一种全局的结构来放入音频的packet以便audio_callback能够从中获取音频的数据。我们将要创建一个packet的队列。ffmpeg提供了一个数据结构AVPacketList，它是一个packet的链表。我们使用队列结构如下：
```c
typedef struct PacketQueue {
  AVPacketList *first_pkt, *last_pkt;
  int nb_packets;
  int size;
  SDL_mutex *mutex;
  SDL_cond *cond;
} PacketQueue;
```
首先，我们需要指出，nb_packets与size不同，size是我们从packet->size获取的size。另外我们有mutex和condition变量。因为SDL在另一个线程运行音频的处理。如果我们不正确的锁定队列，我们可能会弄乱数据。我们接下来看一下这个队列的实现，以便增进对SDL函数的了解。

首先我们编写一个初始化队列的函数：
```c
void packet_queue_init(PacketQueue *q) {
  memset(q, 0, sizeof(PacketQueue));
  q->mutex = SDL_CreateMutex();
  q->cond = SDL_CreateCond();
}
```
接下来我们编写一个函数来往队列里插入
```c
int packet_queue_put(PacketQueue *q, AVPacket *pkt) {

  AVPacketList *pkt1;
  if(av_dup_packet(pkt) < 0) {
    return -1;
  }
  pkt1 = av_malloc(sizeof(AVPacketList));
  if (!pkt1)
    return -1;
  pkt1->pkt = *pkt;
  pkt1->next = NULL;
  
  
  SDL_LockMutex(q->mutex);
  
  if (!q->last_pkt)
    q->first_pkt = pkt1;
  else
    q->last_pkt->next = pkt1;
  q->last_pkt = pkt1;
  q->nb_packets++;
  q->size += pkt1->pkt.size;
  SDL_CondSignal(q->cond);
  
  SDL_UnlockMutex(q->mutex);
  return 0;
}
```
SDL_LockMutex()锁定了mutex以便往里面插入数据，然后SDL_CondSignal()通过condition变量发送信号给我们的get函数，告诉它有数据可以处理了，然后解锁mutex。

对应的get函数如下，注意SDL_CondWait()阻塞了函数直到我们告知其有数据可以处理了。
```c
int quit = 0;

static int packet_queue_get(PacketQueue *q, AVPacket *pkt, int block) {
  AVPacketList *pkt1;
  int ret;
  
  SDL_LockMutex(q->mutex);
  
  for(;;) {
    
    if(quit) {
      ret = -1;
      break;
    }

    pkt1 = q->first_pkt;
    if (pkt1) {
      q->first_pkt = pkt1->next;
      if (!q->first_pkt)
	q->last_pkt = NULL;
      q->nb_packets--;
      q->size -= pkt1->pkt.size;
      *pkt = pkt1->pkt;
      av_free(pkt1);
      ret = 1;
      break;
    } else if (!block) {
      ret = 0;
      break;
    } else {
      SDL_CondWait(q->cond, q->mutex);
    }
  }
  SDL_UnlockMutex(q->mutex);
  return ret;
}
```
可以看到，我们包装了一个有无限for循环的函数，以便我们在阻塞后总能拿到数据。我们通过使用SDL的SDL_CondWait()函数来避免无限循环。基本上，CondWait所做的是等待一个SDL_CondSignal()或者SDL_CondBroadcast()发出的信号，然后继续执行。然而，看起来我们一直持有mutex，我们获得了锁所以put函数无法往队列里写数据。实际上并不是，SDL_CondWait()会释放我们提供的mutex并在获取到信号之后尝试再次锁住它。

### 退出
我们需要注意到有一个全局的quit变量让我们检查以便确认我们是否向程序发送了退出信号。如果没有它的话，我们就只能kill -9杀掉程序了。
```c
  SDL_PollEvent(&event);
  switch(event.type) {
  case SDL_QUIT:
    quit = 1;
```
在收到退出信后后设置quit为1。

### 填充packet

接下来我们需要初始化队列
```c
PacketQueue audioq;
main() {
...
  avcodec_open2(aCodecCtx, aCodec, NULL);

  packet_queue_init(&audioq);
  SDL_PauseAudio(0);
```
SDL_PauseAudio()开启了音频设备，在拿到数据之前，它一直播放静音。接下来，我们初始化队列，并且开始向里面填充packet。我们的packet读取循环如下：
```c
while(av_read_frame(pFormatCtx, &packet)>=0) {
  // Is this a packet from the video stream?
  if(packet.stream_index==videoStream) {
    // Decode video frame
    ....
    }
  } else if(packet.stream_index==audioStream) {
    packet_queue_put(&audioq, &packet);
  } else {
    av_free_packet(&packet);
  }
```
需要注意的是，我们将packet放入队列后没有释放它，直到对其进行解码之后才释放。

### 拉取packet
现在我们可以使用audio_callback函数去从队列里拉取packet。这个callback函数的签名为void callback(void *userdata, Uint8 *stream, int len)，其中userdata是我们提供给SDL的指针，stream是我们将要往里面写入音频数据的缓存，len是缓存区域的大小。代码如下
```c
void audio_callback(void *userdata, Uint8 *stream, int len) {

  AVCodecContext *aCodecCtx = (AVCodecContext *)userdata;
  int len1, audio_size;

  static uint8_t audio_buf[(AVCODEC_MAX_AUDIO_FRAME_SIZE * 3) / 2];
  static unsigned int audio_buf_size = 0;
  static unsigned int audio_buf_index = 0;

  while(len > 0) {
    if(audio_buf_index >= audio_buf_size) {
      /* We have already sent all our data; get more */
      audio_size = audio_decode_frame(aCodecCtx, audio_buf,
                                      sizeof(audio_buf));
      if(audio_size < 0) {
	/* If error, output silence */
	audio_buf_size = 1024;
	memset(audio_buf, 0, audio_buf_size);
      } else {
	audio_buf_size = audio_size;
      }
      audio_buf_index = 0;
    }
    len1 = audio_buf_size - audio_buf_index;
    if(len1 > len)
      len1 = len;
    memcpy(stream, (uint8_t *)audio_buf + audio_buf_index, len1);
    len -= len1;
    stream += len1;
    audio_buf_index += len1;
  }
}
```

这是一个简单的循环，它将下述的audio_decode_frame函数的结果放入到一个中间的缓存中，以便向stream中写入len个byte，如果数据不够的话继续获取数据，多了的话存起来以便后续使用。audio_buf的大小是ffmpeg给我们的最大音频帧的1.5倍，起到很好的保护作用。

### 解码音频
解码的函数audio_decode_frame：
```c
int audio_decode_frame(AVCodecContext *aCodecCtx, uint8_t *audio_buf,
                       int buf_size) {

  static AVPacket pkt;
  static uint8_t *audio_pkt_data = NULL;
  static int audio_pkt_size = 0;
  static AVFrame frame;

  int len1, data_size = 0;

  for(;;) {
    while(audio_pkt_size > 0) {
      int got_frame = 0;
      len1 = avcodec_decode_audio4(aCodecCtx, &frame, &got_frame, &pkt);
      if(len1 < 0) {
	/* if error, skip frame */
	audio_pkt_size = 0;
	break;
      }
      audio_pkt_data += len1;
      audio_pkt_size -= len1;
      data_size = 0;
      if(got_frame) {
	data_size = av_samples_get_buffer_size(NULL, 
					       aCodecCtx->channels,
					       frame.nb_samples,
					       aCodecCtx->sample_fmt,
					       1);
	assert(data_size <= buf_size);
	memcpy(audio_buf, frame.data[0], data_size);
      }
      if(data_size <= 0) {
	/* No data yet, get more frames */
	continue;
      }
      /* We have data, return it and come back for more later */
      return data_size;
    }
    if(pkt.data)
      av_free_packet(&pkt);

    if(quit) {
      return -1;
    }

    if(packet_queue_get(&audioq, &pkt, 1) < 0) {
      return -1;
    }
    audio_pkt_data = pkt.data;
    audio_pkt_size = pkt.size;
  }
}
```
整个过程从函数的后半部分开始，调用packet_queue_get。从队列中拿到packet并保存它的信息。接着，一旦我们有了一个packet，我们调用avcodec_decode_audio4()，它与avcodec_decode_video()类似，唯一不同的是，一个packet可能包含一个或者多个帧，所以我们可能需要调用解码函数多次才能获取packet中的全部数据。一旦我们有了帧，我们只需要将其拷贝到音频缓存，确保data_size小于音频的buffer。另外，我们需要对audio_buf进行转换，因为SDL提供8bit整形的缓存，ffmpeg提供给我们的数据是16bit整形的缓存。我们需要注意到len1与data_size不同。len1是我们使用了多少packet，data_size是返回的原始数据的数量。

当我们获取到数据后，我们立即返回去查看我们是否还需要从队列中获取更多的数据，还是我们已经完成了。如果packet还没有处理完，将其存起来后续使用。如果我们处理完了一个packet，我们将其释放。

大功告成！我们从主循环中获取音频放入队列中，队列中的数据被audio_callback函数读取，它把数据提供给SDL，SDL把数据传给声卡。

## 多线程

### 总览
上一章我们使用SDL的音频功能增加了音频支持。SDL启动了一个新的线程在它需要音频数据的时候调用我们的callback函数。现在我们将要对视频进行同样的改造。这样代码会更模块化以便我们后续增加同步功能。

我们注意到main函数承载了太多的功能：它处理事件循环、读入packet、解码视频。所以我们需要将这些拆开：我们使用一个线程去解码packet，这些packet会被添加到队列然后被对应的音频和视频的线程读取。我们已经创建了我们想要的音频线程，视频的线程更复杂，因为我们需要自己去展示视频。我们在主循环中增加展示视频的代码。但是不再是每次循环都展示视频，我们将视频的展示合并到事件循环中。总的想法是，解码视频，将解码的结果帧放入另一个队列，然后创造一个自定义的事件FF_REFRESH_EVENT并将其添加到事件系统，然后当我们的事件循环看到这个事件，它从队列中拿到下一个帧并播放。整体的过程如下图所示

```
 ________ audio  _______      _____
|        | pkts |       |    |     | to spkr
| DECODE |----->| AUDIO |--->| SDL |-->
|________|      |_______|    |_____|
    |  video     _______
    |   pkts    |       |
    +---------->| VIDEO |
 ________       |_______|   _______
|       |          |       |       |
| EVENT |          +------>| VIDEO | to mon.
| LOOP  |----------------->| DISP. |-->
|_______|<---FF_REFRESH----|_______|
```
我们将视频展示的控制移入event loop的主要意图是，使用一个SDL_Delay线程，我们可以精确的控制下一帧展示在屏幕上的时间。在我们下一章同步视频的时候，增加计划下一帧刷新事件的代码，使得图片能在正确的时间展示在屏幕上的过程，会变得很简单。

### 简化代码

我们将要对代码进行清理，我们有了饮片和视频编解码器的全部信息，我们将要增加队列和缓存还有其它东西。所有这些可以放在一个逻辑模块中。所以我们可以做一个大的结构体VideoState来保存所有的信息。
```c
typedef struct VideoState {

  AVFormatContext *pFormatCtx;
  int             videoStream, audioStream;
  AVStream        *audio_st;
  AVCodecContext  *audio_ctx;
  PacketQueue     audioq;
  uint8_t         audio_buf[(AVCODEC_MAX_AUDIO_FRAME_SIZE * 3) / 2];
  unsigned int    audio_buf_size;
  unsigned int    audio_buf_index;
  AVPacket        audio_pkt;
  uint8_t         *audio_pkt_data;
  int             audio_pkt_size;
  AVStream        *video_st;
  AVCodecContext  *video_ctx;
  PacketQueue     videoq;

  VideoPicture    pictq[VIDEO_PICTURE_QUEUE_SIZE];
  int             pictq_size, pictq_rindex, pictq_windex;
  SDL_mutex       *pictq_mutex;
  SDL_cond        *pictq_cond;
  
  SDL_Thread      *parse_tid;
  SDL_Thread      *video_tid;

  char            filename[1024];
  int             quit;
} VideoState;
```
我们来看一下这个数据结构。首先我们看到了基本信息--格式的上下文和音频视频流的索引，以及对应的AVStream实例。然后我们可以看到我们将一些音频的缓存移入了结构体。这些，例如audio_buf, audio_buf_size，是之前乱放在别的地方的。我们曾杰了一个视频的队列，一个缓存（这个会被队列使用）用于存储解码后的帧（保存为一个overlay）。VideoPicture结构是我们自己创造的，后面会提及。我们注意到有指针指向我们创建的两个线程，还有一个quit标识以及电影的文件名。

接下来我们看一下该结构体的引入导致的main函数的变化。我们来初始化VideoState结构体：
```c
int main(int argc, char *argv[]) {

  SDL_Event       event;

  VideoState      *is;

  is = av_mallocz(sizeof(VideoState));
```
av_mallocz()是一个分配内存并全部置成0的函数。

接下来我们初始化展示缓存pictq的锁，因为每个事件循环我们的展示函数都从pictq中拉取已经解码过的帧。与此同时，我们的视频解码器将会往队列里添加帧。我们不知道哪个会先发生，这是一个典型的竞态条件。所以我们在启动任意线程之前初始化锁。然后我们将文件名拷贝到VideoState。
```c
av_strlcpy(is->filename, argv[1], sizeof(is->filename));

is->pictq_mutex = SDL_CreateMutex();
is->pictq_cond = SDL_CreateCond();
```
av_strlcpy是一个ffmpeg提供的函数，它与strncpy相比多了一些边界检查。

### 第一个线程

现在我们来启动线程来开展实际的工作
```c
schedule_refresh(is, 40);

is->parse_tid = SDL_CreateThread(decode_thread, is);
if(!is->parse_tid) {
  av_free(is);
  return -1;
}
```
schedule_refresh是一个后续会定义的函数。它的作用是告诉系统在指定的毫秒数之后推出一个FF_REFRESH_EVENT。这一事件在事件队列里会触发调取视频的刷新函数。现在我们来看SDL_CreateThread()。

SDL_CreateThread()启动一个新的线程，新线程可以访问原来的进程所有的内存，然后新的线程开始运行我们提供给它的新的函数，并传递给这个新函数一个自定义的数据。这里我们调用decode_thread()函数并提供VideoState结构体作为参数。decode_thread函数的前半部分只是打开文件并且找到音频和视频流的索引，然后将格式的上下文存到我们的大结构体中。在我们找到流的索引之后，我们调用我们定义的另一个函数，stream_component_open()。这种拆分的方式很自然，而且我们初始化音频和视频的解码器时做了一些相似的事，我们可以通过拆分函数达成重用。

stream_component_open()里我们找到解码器，设置音频属性，保存重要的信息到我们的大结构体，并启动音频和视频的线程。在这里我们可以插入一些其它的操作，例如强制指定解码器而不是自动检测等等。代码如下：
```c
int stream_component_open(VideoState *is, int stream_index) {

  AVFormatContext *pFormatCtx = is->pFormatCtx;
  AVCodecContext *codecCtx;
  AVCodec *codec;
  SDL_AudioSpec wanted_spec, spec;

  if(stream_index < 0 || stream_index >= pFormatCtx->nb_streams) {
    return -1;
  }

  codec = avcodec_find_decoder(pFormatCtx->streams[stream_index]->codec->codec_id);
  if(!codec) {
    fprintf(stderr, "Unsupported codec!\n");
    return -1;
  }

  codecCtx = avcodec_alloc_context3(codec);
  if(avcodec_copy_context(codecCtx, pFormatCtx->streams[stream_index]->codec) != 0) {
    fprintf(stderr, "Couldn't copy codec context");
    return -1; // Error copying codec context
  }


  if(codecCtx->codec_type == AVMEDIA_TYPE_AUDIO) {
    // Set audio settings from codec info
    wanted_spec.freq = codecCtx->sample_rate;
    /* ...etc... */
    wanted_spec.callback = audio_callback;
    wanted_spec.userdata = is;
    
    if(SDL_OpenAudio(&wanted_spec, &spec) < 0) {
      fprintf(stderr, "SDL_OpenAudio: %s\n", SDL_GetError());
      return -1;
    }
  }
  if(avcodec_open2(codecCtx, codec, NULL) < 0) {
    fprintf(stderr, "Unsupported codec!\n");
    return -1;
  }

  switch(codecCtx->codec_type) {
  case AVMEDIA_TYPE_AUDIO:
    is->audioStream = stream_index;
    is->audio_st = pFormatCtx->streams[stream_index];
    is->audio_ctx = codecCtx;
    is->audio_buf_size = 0;
    is->audio_buf_index = 0;
    memset(&is->audio_pkt, 0, sizeof(is->audio_pkt));
    packet_queue_init(&is->audioq);
    SDL_PauseAudio(0);
    break;
  case AVMEDIA_TYPE_VIDEO:
    is->videoStream = stream_index;
    is->video_st = pFormatCtx->streams[stream_index];
    is->video_ctx = codecCtx;
    
    packet_queue_init(&is->videoq);
    is->video_tid = SDL_CreateThread(video_thread, is);
    is->sws_ctx = sws_getContext(is->video_st->codec->width, is->video_st->codec->height,
				 is->video_st->codec->pix_fmt, is->video_st->codec->width,
				 is->video_st->codec->height, PIX_FMT_YUV420P,
				 SWS_BILINEAR, NULL, NULL, NULL
				 );
    break;
  default:
    break;
  }
}
```
这与我们之前写的差不多，除了我们对音频和视频的处理进行了泛化。注意不再使用aCodecCtx，而是初始化我们的大结构体以便后续提供给音频的callback。我们将流保存为audio_st和video_st。我们也增加了视频队列并将其与初始化音频队列同样的方法初始化。我们也将视频的队列添加到结构体，然后像初始化音频队列那样初始化它。这里的要点是启动视频和音频线程。对应的部分如下：
```c
    SDL_PauseAudio(0);
    break;

/* ...... */

    is->video_tid = SDL_CreateThread(video_thread, is);
```
这里的SDL_CreateThread()跟之前的SDL_PauseAudio()用法一样。我们回到video_thread()函数。

在此之前，我们回顾一下decode_thread()的后半部分。它基本上就是一个for循环，读入packet然后把packet放到正确的队列里。
```c
  for(;;) {
    if(is->quit) {
      break;
    }
    // seek stuff goes here
    if(is->audioq.size > MAX_AUDIOQ_SIZE ||
       is->videoq.size > MAX_VIDEOQ_SIZE) {
      SDL_Delay(10);
      continue;
    }
    if(av_read_frame(is->pFormatCtx, packet) < 0) {
      if((is->pFormatCtx->pb->error) == 0) {
	      SDL_Delay(100); /* no error; wait for user input */
	      continue;
      } else {
	      break;
      }
    }
    // Is this a packet from the video stream?
    if(packet->stream_index == is->videoStream) {
      packet_queue_put(&is->videoq, packet);
    } else if(packet->stream_index == is->audioStream) {
      packet_queue_put(&is->audioq, packet);
    } else {
      av_free_packet(packet);
    }
  }
```
这里没有新东西，除了我们设定了音频和视频队列的最大长度，并增加了读的时候的错误检查。格式的上下文包含一个名为pb的ByteIOContext结构体。ByteIOContext是一个保存所有底层的文件信息的结构体。


在for循环结束后，接下来的是等待程序退出或者通知我们它退出了的代码。代码很直观，它展示了如何插入事件，正如我们后续展示视频时的插入事件。
```c
  while(!is->quit) {
    SDL_Delay(100);
  }

 fail:
  if(1){
    SDL_Event event;
    event.type = FF_QUIT_EVENT;
    event.user.data1 = is;
    SDL_PushEvent(&event);
  }
  return 0;
```


我们通过使用SDL的常数SDL_USEREVENT来获取用户事件。第一个用户事件应该被赋予SDL_USEREVENT值，下一个是SDL_USEREVENT + 1，以此类推。在我们的程序里，将FF_QUIT_EVENT定义为SDL_USEREVENT + 1。我们还可以传入user data，此处我们传入大结构体。最终我们调用SDL_PushEvent()。在我们的事件循环中，之前我们是将SDL_QUIT_EVENT传入。后续我们会更详细的看一下这个事件循环，现在我们只需要知道放入FF_QUIT_EVENT，后续我们会捕获它并且修改quit标识。

### 获取帧：video_thread

在我们的编解码器准备好之后，我们启动视频线程。这个线程从视频队列中读入packet，解码视频到帧，然后调用queue_picture函数将处理后的帧放入图片队列：
```c
int video_thread(void *arg) {
  VideoState *is = (VideoState *)arg;
  AVPacket pkt1, *packet = &pkt1;
  int frameFinished;
  AVFrame *pFrame;

  pFrame = av_frame_alloc();

  for(;;) {
    if(packet_queue_get(&is->videoq, packet, 1) < 0) {
      // means we quit getting packets
      break;
    }
    // Decode video frame
    avcodec_decode_video2(is->video_st->codec, pFrame, &frameFinished, packet);

    // Did we get a video frame?
    if(frameFinished) {
      if(queue_picture(is, pFrame) < 0) {
	break;
      }
    }
    av_free_packet(packet);
  }
  av_free(pFrame);
  return 0;
}
```
这个函数里的多数代码看起来应该比较熟悉。我们将avcodec_decode_video2移动到这里，只是改了一些参数：例如，我们将AVStream存储到我们的大结构体，所以我们从那里获取编解码器。我们仅仅是不停的从视频队列中取出packet知道有人告诉我们要退出或者我们遇到了错误。

### 帧队列

我们来看一下存储解码后的帧的函数，pFrame是图片队列。因为我们的图片队列是一个SDL overlay（这样我们就可以让展示函数进行尽可能少的计算了），我们需要将帧转换成这种格式。我们存储图片队列的数据结构如下：
```c
typedef struct VideoPicture {
  SDL_Overlay *bmp;
  int width, height; /* source height & width */
  int allocated;
} VideoPicture;
```
我们的大结构体有一块缓存用于存储他们。然后我们需要手工分配SDL_Overlay的空间（注意allocated标识会展示我们是否已经完成分配）。

为了使用这个队列，我们有两个指针，正在写入的index和正在读出的index。我们还要记录缓存里有多少图片。为了向队列中写入，我们首先要等到缓存区域清空以便有地方存储VideoPicture。接下来我们检查我们写入的index是否已经分配了overlay。如果没有的话，我们将分配空间。如果窗口的大小改变了的话，我们需要重新分配缓存。
```c
int queue_picture(VideoState *is, AVFrame *pFrame) {

  VideoPicture *vp;
  int dst_pix_fmt;
  AVPicture pict;

  /* wait until we have space for a new pic */
  SDL_LockMutex(is->pictq_mutex);
  while(is->pictq_size >= VIDEO_PICTURE_QUEUE_SIZE &&
	!is->quit) {
    SDL_CondWait(is->pictq_cond, is->pictq_mutex);
  }
  SDL_UnlockMutex(is->pictq_mutex);

  if(is->quit)
    return -1;

  // windex is set to 0 initially
  vp = &is->pictq[is->pictq_windex];

  /* allocate or resize the buffer! */
  if(!vp->bmp ||
     vp->width != is->video_st->codec->width ||
     vp->height != is->video_st->codec->height) {
    SDL_Event event;

    vp->allocated = 0;
    alloc_picture(is);
    if(is->quit) {
      return -1;
    }
  }
```
其中alloc_picture()函数如下：
```c
void alloc_picture(void *userdata) {

  VideoState *is = (VideoState *)userdata;
  VideoPicture *vp;

  vp = &is->pictq[is->pictq_windex];
  if(vp->bmp) {
    // we already have one make another, bigger/smaller
    SDL_FreeYUVOverlay(vp->bmp);
  }
  // Allocate a place to put our YUV image on that screen
  SDL_LockMutex(screen_mutex);
  vp->bmp = SDL_CreateYUVOverlay(is->video_st->codec->width,
				 is->video_st->codec->height,
				 SDL_YV12_OVERLAY,
				 screen);
  SDL_UnlockMutex(screen_mutex);
  vp->width = is->video_st->codec->width;
  vp->height = is->video_st->codec->height;  
  vp->allocated = 1;
}
```
可以看到我们将主循环中的SDL_CreateYUVOverlay函数移动到了这里。需要注意的是，我们在这个函数周围有一个mutex锁，因为两个线程不能同时向屏幕写入信息。这将会避免alloc_picture忙于展示图片。我们在全局变量里创建了这个锁并且在main里面初始化了它。记住我们在VideoPicture结构体里保存了宽度和高度数据，因为基于某种原因，我们希望确保视频的尺寸不会改变。

好了，我们都设置好了，而且我们有一个已经分配好内存的YUV，并且准备好了接收图片。然后我们回到queue_picture函数来看一下将帧拷贝到overlay的代码。其中部分代码我们已经见过：
```c
int queue_picture(VideoState *is, AVFrame *pFrame) {

  /* Allocate a frame if we need it... */
  /* ... */
  /* We have a place to put our picture on the queue */

  if(vp->bmp) {

    SDL_LockYUVOverlay(vp->bmp);
    
    dst_pix_fmt = PIX_FMT_YUV420P;
    /* point pict at the queue */

    pict.data[0] = vp->bmp->pixels[0];
    pict.data[1] = vp->bmp->pixels[2];
    pict.data[2] = vp->bmp->pixels[1];
    
    pict.linesize[0] = vp->bmp->pitches[0];
    pict.linesize[1] = vp->bmp->pitches[2];
    pict.linesize[2] = vp->bmp->pitches[1];
    
    // Convert the image into YUV format that SDL uses
    sws_scale(is->sws_ctx, (uint8_t const * const *)pFrame->data,
	      pFrame->linesize, 0, is->video_st->codec->height,
	      pict.data, pict.linesize);
    
    SDL_UnlockYUVOverlay(vp->bmp);
    /* now we inform our display thread that we have a pic ready */
    if(++is->pictq_windex == VIDEO_PICTURE_QUEUE_SIZE) {
      is->pictq_windex = 0;
    }
    SDL_LockMutex(is->pictq_mutex);
    is->pictq_size++;
    SDL_UnlockMutex(is->pictq_mutex);
  }
  return 0;
}
```
这部分的主体是我们之前用到的将帧塞到YUV overlay的代码。最后一部分是将我们的值放进队列。这个队列的工作模式是，一直往里面里塞直到它满了，同时只要里面有值就从其中读入。然而一切都依赖于is->pictq_size的值，所以我们需要锁上它。所以这里我们是做的是增加写入指针的值（在必要的时候转回到初始位置），然后锁住队列并增大它的大小。现在读者应该知道队列有更多的信息，以便在队列满了的时候，写入者知道。

### 展示视频
这是我们的视频的线程，我们将除它之外所有其它线程封装了，还记得之前说过的schedule_refresh()函数吗？它的代码如下：
```c
/* schedule a video refresh in 'delay' ms */
static void schedule_refresh(VideoState *is, int delay) {
  SDL_AddTimer(delay, sdl_refresh_timer_cb, is);
}
```
SDL_AddTimer()是一个SDL函数，它在指定的毫秒之后调用传入的callback函数（还可以带有一些用户数据）。我们将使用这个函数去规划视频的更新，每次我们调用这个函数，它设置一个定时器，定时器会触发事件，事件接下来会让main函数调用一个函数去从我们的图片队列拉取一个帧并展示。

但是首先，我们需要触发一个事件。它通过这种方式发给我们：
```c
static Uint32 sdl_refresh_timer_cb(Uint32 interval, void *opaque) {
  SDL_Event event;
  event.type = FF_REFRESH_EVENT;
  event.user.data1 = opaque;
  SDL_PushEvent(&event);
  return 0; /* 0 means stop timer */
}
```
这是一个我们熟悉的事件插入，此处的FF_REFRESH_EVENT被定义为SDL_USEREVENT + 1。我们需要注意的是，当我们return 0，SDL会停止定时器所有callback就不再会被调用了。

我们插入一个FF_REFRESH_EVENT后，需要在事件循环中处理它：
```c
for(;;) {

  SDL_WaitEvent(&event);
  switch(event.type) {
  /* ... */
  case FF_REFRESH_EVENT:
    video_refresh_timer(event.user.data1);
    break;
```
上述代码调用了下述函数，它负责将数据从图片队列里面拿出来：
```c
void video_refresh_timer(void *userdata) {

  VideoState *is = (VideoState *)userdata;
  VideoPicture *vp;
  
  if(is->video_st) {
    if(is->pictq_size == 0) {
      schedule_refresh(is, 1);
    } else {
      vp = &is->pictq[is->pictq_rindex];
      /* Timing code goes here */

      schedule_refresh(is, 80);
      
      /* show the picture! */
      video_display(is);
      
      /* update queue for next picture! */
      if(++is->pictq_rindex == VIDEO_PICTURE_QUEUE_SIZE) {
	is->pictq_rindex = 0;
      }
      SDL_LockMutex(is->pictq_mutex);
      is->pictq_size--;
      SDL_CondSignal(is->pictq_cond);
      SDL_UnlockMutex(is->pictq_mutex);
    }
  } else {
    schedule_refresh(is, 100);
  }
}
```
到目前为止，函数很简单：在队列里有值的时候它将其取出，设置一个定时器来确定下一个帧被战士的时间，调用video_display来负责把视频展示到屏幕上，然后增加队列的计数器，减小队列的大小。我们需要注意在这里我们没有使用vp，后续在开始同步视频的时候，我们会使用它来获取时间信息。看到计时的代码了吗？在这里，我们需要指定下一个视频帧展示的时间，然后将它的值放入schedule_refresh()函数。目前我们设置为80，后面我们会优化它。

我们基本上完成了，现在做最后一件事：展示视频！video_display函数的代码如下：
```c
void video_display(VideoState *is) {

  SDL_Rect rect;
  VideoPicture *vp;
  float aspect_ratio;
  int w, h, x, y;
  int i;

  vp = &is->pictq[is->pictq_rindex];
  if(vp->bmp) {
    if(is->video_st->codec->sample_aspect_ratio.num == 0) {
      aspect_ratio = 0;
    } else {
      aspect_ratio = av_q2d(is->video_st->codec->sample_aspect_ratio) *
	is->video_st->codec->width / is->video_st->codec->height;
    }
    if(aspect_ratio <= 0.0) {
      aspect_ratio = (float)is->video_st->codec->width /
	(float)is->video_st->codec->height;
    }
    h = screen->h;
    w = ((int)rint(h * aspect_ratio)) & -3;
    if(w > screen->w) {
      w = screen->w;
      h = ((int)rint(w / aspect_ratio)) & -3;
    }
    x = (screen->w - w) / 2;
    y = (screen->h - h) / 2;
    
    rect.x = x;
    rect.y = y;
    rect.w = w;
    rect.h = h;
    SDL_LockMutex(screen_mutex);
    SDL_DisplayYUVOverlay(vp->bmp, &rect);
    SDL_UnlockMutex(screen_mutex);
  }
}
```
因为屏幕的大小可以是任意值（我们设置了640x480但是用户可以自行改变它），我们需要动态的弄清楚我们需要电影展示在多大的矩形中。所以我们需要找出电影的宽高比。有些编解码器有一个奇怪的采样宽高比，它是一个像素或者采样的宽高比。因为编解码器的宽高值是用像素衡量的，真实的宽高比等于宽高比乘以采样宽高比。有些编解码器的宽高比是0，这表明每个像素的大小都是1x1。接下来我们将电影缩放到屏幕大小。& -3位操作符将数值变为最近的4的倍数。接着我们可以进入电影，调用SDL_DisplayYUVOverlay()以便我们可以使用屏幕的锁去获取到overlay。

这就完了吗？当然，我们还需要重写音频的代码来使用VideoStruct，不过这都很简单，读者可以自己看一下代码。最后一件我们要做的事是将ffmpeg内置的quit的回调函数换成我们自己的：
```c
VideoState *global_video_state;

int decode_interrupt_cb(void) {
  return (global_video_state && global_video_state->quit);
}
```
我们在main函数里将global_video_state设置给大结构体。

## 同步视频

### 警告
在写本文的时候，所有的同步代码都来自ffplay.c。如今ffplay.c完全是另一个程序了，ffmpeg里面的一些改进是的某些策略发生了变化。现在的代码还能用，但是看起来不够好，还可以有很多改进。

### 视频怎么同步

直到现在，我们的播放器都没什么用，它播放音视频，但是播放的结果不能称之为一部电影。我们应该做点什么？

### PTS和DTS
幸运的是，音视频流都有播放速度的信息，其内部也有何时播放的信息。音频流有一个采样频率，视频流有一个帧率。然而如果我们仅仅根据数帧数乘以帧率来同步，我们可能无法与音频同步。因此，在packet中可能包含了解码时间戳（DTS）和一个展示时间戳（PTS）。为了理解这两个值，我们需要知道电影是怎么存储的。有些格式，比如MPEG，使用我们称之为B帧（B代表双向）。另外两种帧被称为I帧和P帧（I代表内部的，P代表预测）。I帧包含一副完成的图片，P帧基于之前的I帧和P帧，对比相同和差异。B帧与P帧相似，但是它基于它前后两个方向帧的信息。所以我们在调用了avcodec_decode_video2之后可能还是拿不到一个完整的帧。

假设我们有一个电影，它的帧序列是：I B B P。现在，我们需要知道P帧里面的信息以便展示两个B帧。因此，帧可能被存成：I P B B格式。所以我们有一个解码的时间戳和一个展示的时间戳。解码的时间戳告诉我们什么时候我们需要解码，展示的时间戳告诉我们什么时候我们需要展示。在这种情况下，我们的流看起来是这样的：
```
   PTS: 1 4 2 3
   DTS: 1 2 3 4
Stream: I P B B
```
总的来说，PTS和DTS只有在流里包含B帧的时候不同。

当我们从av_read_frame()获取到一个packet时，它包含了这个packet里面的PTS和DTS的值。但是我们需要的是我们解码出来的帧的PTS，以便知道什么时候播放它。

幸运的是，FFmpeg提供了一个best effort时间戳，我们可以通过av_frame_get_best_effort_timestamp()获取。

### 同步

现在，我们知道了什么时候展示特定的视频帧，然后我们该怎么做呢？现在的想法是：在展示完一个帧之后，我们看一下下一个帧应该什么时候展示。然后设置一个新的timeout来在一段时间之后刷新视频。可以想到，我们可以通过检查下一帧的PTS并与系统时钟对比去看timeout设置多大比较合适。这个方法可行，但是有两个问题需要处理。

第一个问题是如何知道下一个PTS的值。我们可以将视频的帧率加上当前的PTS，这样做几乎是对的了。但是有些视频会要求当前帧重复几次。这意味着我们应该将当前帧重复播放几次。这会导致程序太早的播放下一个帧，我们需要处理这个问题。

第二个问题是视频和音频持续的播放，相互之间同步没有处理。如果一切正常的话，我们不需要担心。但是电脑并不完美，很多电影文件也不完美。我们有三个选择：将音频与视频同步，将视频与音频同步，将两者与一个外部的时间（比如电脑）同步。目前我们将视频同步到音频。

### 获取帧的PTS

现在我们开始看完成这个功能的代码。我们需要根据我们的需求向我们的大结构体里面添加几个成员变量。首先我们看一下我们的视频线程。记住，展示我们获取解码线程放到队列里面的packet。这段代码需要做的是获取通过avcodec_decode_video2得到的帧的PTS值。我们之前讨论过的获取最新的处理过的packet的DTS值的第一种方法比较简单：
```c
  double pts;

  for(;;) {
    if(packet_queue_get(&is->videoq, packet, 1) < 0) {
      // means we quit getting packets
      break;
    }
    pts = 0;
    // Decode video frame
    len1 = avcodec_decode_video2(is->video_st->codec,
                                pFrame, &frameFinished, packet);
    if(packet->dts != AV_NOPTS_VALUE) {
      pts = av_frame_get_best_effort_timestamp(pFrame);
    } else {
      pts = 0;
    }
    pts *= av_q2d(is->video_st->time_base);
```
如果获取不到PTS我们将其设置为0。

一切都挺简单的，一个技术细节需要注意：我们应该注意到PTS的数值是int64。因为PTS被存储成一个integer。这个值是一个相对于流的时间基准（time_base）的时间戳。例如，一个流每秒钟24帧，PTS是42意味着1/24秒播放一帧的话这个是第42个帧。

我们可以将这个值除以帧率转为秒，流的时间基准的值是1/帧率（对于固定帧率的内容而言），所以想得到以秒计算的PTS，将其乘以时间基准即可。

### 使用PTS同步

我们现在得到了PTS，现在我们需要处理上述的同步问题。我们定义一个函数synchronize_video来更新PTS使其与其它的东西同步。这个函数也会处理我们获取不到PTS的情况。与此同时，我们跟踪下一帧将要出现的时间以便能正确的设置刷新率。我们可以通过使用一个内部的video_clock值来保存跟踪视频来看已经过去了多长时间。我们将这个值加到我们的大结构体
```c
typedef struct VideoState {
  double          video_clock; // pts of last decoded frame / predicted pts of next decoded frame
```
下面是synchronize_video函数，很容易看懂：
```c
double synchronize_video(VideoState *is, AVFrame *src_frame, double pts) {

  double frame_delay;

  if(pts != 0) {
    /* if we have pts, set video clock to it */
    is->video_clock = pts;
  } else {
    /* if we aren't given a pts, set it to the clock */
    pts = is->video_clock;
  }
  /* update the video clock */
  frame_delay = av_q2d(is->video_st->codec->time_base);
  /* if we are repeating a frame, adjust clock accordingly */
  frame_delay += src_frame->repeat_pict * (frame_delay * 0.5);
  is->video_clock += frame_delay;
  return pts;
}
```
需要注意这里我们处理了重复播放的帧。

现在我们获取到了正确的PTS并使用queue_picture函数建立了帧的队列，增加一个新的pts参数
```cpp
    // Did we get a video frame?
    if(frameFinished) {
      pts = synchronize_video(is, pFrame, pts);
      if(queue_picture(is, pFrame, pts) < 0) {
	      break;
      }
    }
```

我们需要对queue_picture函数进行修改，向添加进队列的VideoPicture增加pts值。所以我们向这个结构增加一个pts变量并增加一行代码
```cpp
typedef struct VideoPicture {
  ...
  double pts;
}
int queue_picture(VideoState *is, AVFrame *pFrame, double pts) {
  ... stuff ...
  if(vp->bmp) {
    ... convert picture ...
    vp->pts = pts;
    ... alert queue ...
  }
```
现在我们有了拥有正确的PTS数值的图片队列，接下来我们来改一下视频的刷新函数。之前的时候我们写的80ms刷新一次。现在我们来看看怎么处理。

我们的策略是通过测量上一个pts和这个来预测下一个pts的值。与此同时，我们需要将视频同步到音频，我们需要一个音频时钟：一个内部的值泳衣跟踪我们当前音乐播放的位置。它像MP3播放器的数字读数，因为我们是将视频同步到音频，视频的线程使用该值来确定我们是太靠前还是太靠后了。

我们后续再实现这一函数，现在我们假设get_audio_clock会提供给我们音频时钟的时间值。一旦我们有了这个值，当音视频不同步的时候我们该怎么做？通过seeking直接跳转到正确的packet不够合理。我们应该根据它调整我们计算出来的下一个刷新的时间间隔：如果PTS太落后于音频时钟，我们仅仅需要尽可能快的刷新，如果PTS太领先于音频时钟，我们将计算出来的延时乘以2。现在我们有了一个调整过的刷新时间，也即延时，我们需要把frame的计时器和计算机的时钟进行对比。这个frame的计时器将会把我们在播放电影的过程中计算出来的所有的延时求和。换句话说，frame_timer是展示下一个帧的时间，我们将新计算出来的延时加到上面，与计算机的时钟进行对比，然后使用比较的结果对下一个刷新进行规划。这个有点复杂，具体的代码实现如下
```cpp
void video_refresh_timer(void *userdata) {

  VideoState *is = (VideoState *)userdata;
  VideoPicture *vp;
  double actual_delay, delay, sync_threshold, ref_clock, diff;
  
  if(is->video_st) {
    if(is->pictq_size == 0) {
      schedule_refresh(is, 1);
    } else {
      vp = &is->pictq[is->pictq_rindex];

      delay = vp->pts - is->frame_last_pts; /* the pts from last time */
      if(delay <= 0 || delay >= 1.0) {
	/* if incorrect delay, use previous one */
	delay = is->frame_last_delay;
      }
      /* save for next time */
      is->frame_last_delay = delay;
      is->frame_last_pts = vp->pts;

      /* update delay to sync to audio */
      ref_clock = get_audio_clock(is);
      diff = vp->pts - ref_clock;

      /* Skip or repeat the frame. Take delay into account
	 FFPlay still doesn't "know if this is the best guess." */
      sync_threshold = (delay > AV_SYNC_THRESHOLD) ? delay : AV_SYNC_THRESHOLD;
      if(fabs(diff) < AV_NOSYNC_THRESHOLD) {
	if(diff <= -sync_threshold) {
	  delay = 0;
	} else if(diff >= sync_threshold) {
	  delay = 2 * delay;
	}
      }
      is->frame_timer += delay;
      /* computer the REAL delay */
      actual_delay = is->frame_timer - (av_gettime() / 1000000.0);
      if(actual_delay < 0.010) {
	/* Really it should skip the picture instead */
	actual_delay = 0.010;
      }
      schedule_refresh(is, (int)(actual_delay * 1000 + 0.5));
      /* show the picture! */
      video_display(is);
      
      /* update queue for next picture! */
      if(++is->pictq_rindex == VIDEO_PICTURE_QUEUE_SIZE) {
	is->pictq_rindex = 0;
      }
      SDL_LockMutex(is->pictq_mutex);
      is->pictq_size--;
      SDL_CondSignal(is->pictq_cond);
      SDL_UnlockMutex(is->pictq_mutex);
    }
  } else {
    schedule_refresh(is, 100);
  }
}
```
代码里做了一些检查：首先，我们确保当前的PTS与上一个PTS之间的延时有意义，如果没有的话我们使用上一个延时来替代。接下来，我们设定了一个同步的阈值，因为完美的同步永远都不可能发生，ffplay使用0.01作为阈值。我们需要确保同步的阈值不小于PTS之间的差值。最后，我们将最小的刷新时间设定为10ms（这时我们应该跳过这个帧的，不过我们就不处理了）。

我们向大结构体里面加入了很多个变量，记得看看代码回复一下。另外，我们需要在stream_component_open函数中初始化frame的计时器和上一帧的延时：
```cpp
    is->frame_timer = (double)av_gettime() / 1000000.0;
    is->frame_last_delay = 40e-3;
```

### 同步：音频时钟

我们接下来实现音频时钟，我们可以在解码音频的audio_decode_frame函数中更新音频时钟。我们并不是每次调用这个函数的时候都处理一个新的packet，所以我们有两个地方需要更新这个值。第一个地方时我们拿到一个新的packet的时候：我们将音频时钟设置为这个packet的PTS。如果一个packet包含多个音频帧，我们将时间更新为采样数乘以给定的采样率。所以一旦我们有了packet：
```cpp
    /* if update, update the audio clock w/pts */
    if(pkt->pts != AV_NOPTS_VALUE) {
      is->audio_clock = av_q2d(is->audio_st->time_base)*pkt->pts;
    }
```

之后在我们处理packet的时候：
```cpp
      /* Keep audio_clock up-to-date */
      pts = is->audio_clock;
      *pts_ptr = pts;
      n = 2 * is->audio_st->codec->channels;
      is->audio_clock += (double)data_size /
	(double)(n * is->audio_st->codec->sample_rate);
```

有一系列的细节需要注意：这个函数的模板改为包含pts_ptr，pts_ptr是一个用来告知音频audio_callback函数音频packet的pts。这个值未来会被用来将音频同步到视频。

现在我们来实现get_audio_clock函数，它并不是简单的获取is->audio_clock的值。虽然我们每次处理音频的时候都将其为设置音频的PTS，但是如果看一下audio_callback函数，将数据从音频的packet移动到输出的buffer需要时间。这就意味着我们音频时钟的值太超前了。所以我么你需要检查我们还有多少没有写入，完整的代码如下：
```cpp
double get_audio_clock(VideoState *is) {
  double pts;
  int hw_buf_size, bytes_per_sec, n;
  
  pts = is->audio_clock; /* maintained in the audio thread */
  hw_buf_size = is->audio_buf_size - is->audio_buf_index;
  bytes_per_sec = 0;
  n = is->audio_st->codec->channels * 2;
  if(is->audio_st) {
    bytes_per_sec = is->audio_st->codec->sample_rate * n;
  }
  if(bytes_per_sec) {
    pts -= (double)hw_buf_size / bytes_per_sec;
  }
  return pts;
}
```
代码的含义应该很容易看懂。

## 同步音频

现在我们有一个实现的相当好的播放器了，但是还有些地方处理的不好。上一章我们将视频同步到了音频。接下来我们对视频也增加一个内部的视频时钟来记录视频线程处理道德位置，并且与音频同步。再接着我们将音视频时钟同步到一个外部的时钟。

### 实现视频时钟

我们将实现一个与音频时钟很相似的视频时钟：带有一个内部的值来确定当前在播放的视频的时间偏移量。起初，这个看起来很简单，我们只需要将计时器的值更新为最新的帧的PTS。但是考虑到在毫秒尺度上，两个视频帧的时间间隔可能非常长。解决方案是记录另一个值，我们设定视频时钟为最新的帧的PTS时的时间。这样以来当前的视频时钟的值为PTS_of_last_frame + (current_time - time_elapsed_since_PTS_value_was_set)。这个解决方案跟我们处理get_audio_clock的方案非常类似。

所以，在我们的大结构体里，我们将放入double video_current_pts和int64_t video_current_pts_time。时钟在video_refresh_timer函数中更新：
```cpp
void video_refresh_timer(void *userdata) {

  /* ... */

  if(is->video_st) {
    if(is->pictq_size == 0) {
      schedule_refresh(is, 1);
    } else {
      vp = &is->pictq[is->pictq_rindex];

      is->video_current_pts = vp->pts;
      is->video_current_pts_time = av_gettime();
```

不要忘掉在stream_component_open中完成初始化：
```cpp
    is->video_current_pts_time = av_gettime();
```

接下来我们完成提供信息的函数即可：
```cpp
double get_video_clock(VideoState *is) {
  double delta;

  delta = (av_gettime() - is->video_current_pts_time) / 1000000.0;
  return is->video_current_pts + delta;
}
```


### 抽象时钟

但是我们为什么要用视频时钟呢？我们需要修改视频同步的代码，使得音频和视频不再相互同步。我们需要抽象出一个新的包装函数get_master_clock，它检查av_sync_type变量，然后调用get_audio_clock、get_video_clock，或者任何我们提供的其它时钟。我们还可以使用计算机的时钟，此时我们会调用get_external_clock：
```cpp
enum {
  AV_SYNC_AUDIO_MASTER,
  AV_SYNC_VIDEO_MASTER,
  AV_SYNC_EXTERNAL_MASTER,
};

#define DEFAULT_AV_SYNC_TYPE AV_SYNC_VIDEO_MASTER

double get_master_clock(VideoState *is) {
  if(is->av_sync_type == AV_SYNC_VIDEO_MASTER) {
    return get_video_clock(is);
  } else if(is->av_sync_type == AV_SYNC_AUDIO_MASTER) {
    return get_audio_clock(is);
  } else {
    return get_external_clock(is);
  }
}
main() {
...
  is->av_sync_type = DEFAULT_AV_SYNC_TYPE;
...
}
```


### 同步音频

接下来我们将音频同步到视频时钟。我们的策略是，确定音频播放到哪了，将它与视频时钟对比，确定我们需要调整多少采样，即，我们需要通过扔掉一些采样加速还是增加一些采样来降速？

我们将会处理魅族音频采样的时候运行synchronize_audio，来确定是否需要缩减或者展开它们。然而，我们并不像每次处理音频的时候都进行同步，因为音频比视频的packet要频繁的多。所以我们需要设定阈值，当连续多次不同步之后才去调用synchronize_audio函数。当然，跟之前的时候一样，不同步意味着音频时钟和视频时钟的差异大于某一个同步的阈值。

所以我们需要一个小数的系数c。我们假设我们有N个音频采样已经不同步了。不同步的音频数可能变化的很快。所以我们需要求不同步的音频采样数的平均值。举个例子，第一次调用函数的时候不同步的可能有40ms，接着是50ms，等等。但是我们并不是简单的取平均值，因为最近的数值比前面的数值更重要。我已我们使用一个小数的系数c，计算差异：diff_sum = new_diff + diff_sum*c。在我们求差异的平均值时，我们只需要计算avg_diff = diff_sum * (1-c)。

我们的函数看起来如下
```cpp
/* Add or subtract samples to get a better sync, return new
   audio buffer size */
int synchronize_audio(VideoState *is, short *samples,
		      int samples_size, double pts) {
  int n;
  double ref_clock;
  
  n = 2 * is->audio_st->codec->channels;
  
  if(is->av_sync_type != AV_SYNC_AUDIO_MASTER) {
    double diff, avg_diff;
    int wanted_size, min_size, max_size, nb_samples;
    
    ref_clock = get_master_clock(is);
    diff = get_audio_clock(is) - ref_clock;

    if(diff < AV_NOSYNC_THRESHOLD) {
      // accumulate the diffs
      is->audio_diff_cum = diff + is->audio_diff_avg_coef
	* is->audio_diff_cum;
      if(is->audio_diff_avg_count < AUDIO_DIFF_AVG_NB) {
	is->audio_diff_avg_count++;
      } else {
	avg_diff = is->audio_diff_cum * (1.0 - is->audio_diff_avg_coef);

       /* Shrinking/expanding buffer code.... */

      }
    } else {
      /* difference is TOO big; reset diff stuff */
      is->audio_diff_avg_count = 0;
      is->audio_diff_cum = 0;
    }
  }
  return samples_size;
}
```

目前为止我们做的不错，我们知道了音频距离视频或者我们用的任何时钟大约多远。接下来我们这段“缩短或者展长缓存”的代码来计算我们需要增加或者删掉多少采样：
```cpp
if(fabs(avg_diff) >= is->audio_diff_threshold) {
  wanted_size = samples_size + 
  ((int)(diff * is->audio_st->codec->sample_rate) * n);
  min_size = samples_size * ((100 - SAMPLE_CORRECTION_PERCENT_MAX)
                             / 100);
  max_size = samples_size * ((100 + SAMPLE_CORRECTION_PERCENT_MAX) 
                             / 100);
  if(wanted_size < min_size) {
    wanted_size = min_size;
  } else if (wanted_size > max_size) {
    wanted_size = max_size;
  }
```

记得audio_length * (sample_rate * # of channels * 2)是audio_length秒长的音频的采样数。所以，我们需要的样本数是是我们已经有的样本数加上或者减去已经播放过的音频的时间对应的音频数。我们还会设置一个最大或者最小的旧证书，以免我们一次改变了太多的buffer内容，用户听起来会比较刺耳。

### 纠正样本数

现在我们开始真正的音频纠正。应该已经注意到我们的synchronize_audio函数返回了一个采样数量，这会告诉我们我们往流里发送多少数据。所以我们只需要将采样数量调整为wanted_size。在将采样数量减小的情况下是没有问题的。但是如果我们想增大采样数量则不行，因为buffer中没有更多的数据。我们需要添加它，但是该添加哪些呢？填进去随机的数据显然很傻，我们使用最近的一个采样的数值进行填充。
``cpp
if(wanted_size < samples_size) {
  /* remove samples */
  samples_size = wanted_size;
} else if(wanted_size > samples_size) {
  uint8_t *samples_end, *q;
  int nb;

  /* add samples by copying final samples */
  nb = (samples_size - wanted_size);
  samples_end = (uint8_t *)samples + samples_size - n;
  q = samples_end + n;
  while(nb > 0) {
    memcpy(q, samples_end, n);
    q += n;
    nb -= n;
  }
  samples_size = wanted_size;
}
```

先我们返回了样本数量，并完成了相应的功能。我们所需要做的是使用下面的函数：
```cpp
void audio_callback(void *userdata, Uint8 *stream, int len) {

  VideoState *is = (VideoState *)userdata;
  int len1, audio_size;
  double pts;

  while(len > 0) {
    if(is->audio_buf_index >= is->audio_buf_size) {
      /* We have already sent all our data; get more */
      audio_size = audio_decode_frame(is, is->audio_buf, sizeof(is->audio_buf), &pts);
      if(audio_size < 0) {
	/* If error, output silence */
	is->audio_buf_size = 1024;
	memset(is->audio_buf, 0, is->audio_buf_size);
      } else {
	audio_size = synchronize_audio(is, (int16_t *)is->audio_buf,
				       audio_size, pts);
	is->audio_buf_size = audio_size;
```
我们所需要做的都在synchronize_audio中完成了。

最后一件要做的事：增加一个if语句确保在视频的计时器是主计时器时，不需要同步视频。
```cpp
if(is->av_sync_type != AV_SYNC_VIDEO_MASTER) {
  ref_clock = get_master_clock(is);
  diff = vp->pts - ref_clock;

  /* Skip or repeat the frame. Take delay into account
     FFPlay still doesn't "know if this is the best guess." */
  sync_threshold = (delay > AV_SYNC_THRESHOLD) ? delay :
                    AV_SYNC_THRESHOLD;
  if(fabs(diff) < AV_NOSYNC_THRESHOLD) {
    if(diff <= -sync_threshold) {
      delay = 0;
    } else if(diff >= sync_threshold) {
      delay = 2 * delay;
    }
  }
}
```

## 跳转

### 处理跳转函数

接下来我们往播放器里添加一些寻址的能力，因为看电影的时候不能回退用起来很不好。另外，添加这一功能可以展示一下av_seek_frame用起来很简单。

我们将增加一个向左的和一个向右的箭头代表往后或者往前一点，一个向上的和向下的箭头代表向后或者向前一段，此处一点代表10秒，一段代表60秒。我们需要设置主线程去捕获键盘事件。然而，当我们获得键盘事件后，我们不能直接调用av_seek_frame。我们需要在解码的线程decode_thread中调用。因此，我们向我们的大结构体里面增加一些变量来记录需要跳转的地址和一些跳转的标志：
```cpp
  int             seek_req;
  int             seek_flags;
  int64_t         seek_pos;
```

接下来我们在主线程中捕获键盘输入：
```cpp
  for(;;) {
    double incr, pos;

    SDL_WaitEvent(&event);
    switch(event.type) {
    case SDL_KEYDOWN:
      switch(event.key.keysym.sym) {
      case SDLK_LEFT:
	incr = -10.0;
	goto do_seek;
      case SDLK_RIGHT:
	incr = 10.0;
	goto do_seek;
      case SDLK_UP:
	incr = 60.0;
	goto do_seek;
      case SDLK_DOWN:
	incr = -60.0;
	goto do_seek;
      do_seek:
	if(global_video_state) {
	  pos = get_master_clock(global_video_state);
	  pos += incr;
	  stream_seek(global_video_state, 
                      (int64_t)(pos * AV_TIME_BASE), incr);
	}
	break;
      default:
	break;
      }
      break;
```

为了监控键盘输入，我们首先查看是否获取到一个SDL_KEYDOWN事件。然后我们使用event.key.keysym.sym函数检查哪个键被按下。一旦我们知道了如何跳转，我们将要增加的量加上get_master_clock函数的返回值作为新的事件。然后我们调用stream_seek去设置seek_pos等值。我们将新的事件转为avcodec内部的时间戳格式。记住流中的时间戳是以帧而不是秒衡量的，使用公式seconds = frames * time_base (fps)，avcodec中time_base的默认值是1,000,000 fps（所以两秒对应的时间戳是2000000）。

我们的stream_seek函数如下，注意如果后退的话我们设置一个标识：
```cpp
void stream_seek(VideoState *is, int64_t pos, int rel) {

  if(!is->seek_req) {
    is->seek_pos = pos;
    is->seek_flags = rel < 0 ? AVSEEK_FLAG_BACKWARD : 0;
    is->seek_req = 1;
  }
}
```

接下来我们修改decode_thread，它真正的处理跳转。源代码中跳转的部分在下面会引用。跳转主要依赖av_seek_frame函数。这个函数接收一个格式的上下文，一个流，一个时间戳和一系列的标识作为参数。函数将会跳转到传入的时间戳。时间戳的单位是传入给它的流的time_base。如果我们没有传入流（传入-1代表没有传入流），time_base将会取avcodec内部的时间戳单位，或者1,000,000 fps。所以上面我们将位置乘以AV_TIME_BASE作为seek_pos的参数。

然而，对于有些文件，传入-1给av_seek_frame可能会有问题，所以我们将文件的第一个流传入给av_seek_frame。注意到我们需要将时间戳转换一下单位。
```cpp
if(is->seek_req) {
  int stream_index= -1;
  int64_t seek_target = is->seek_pos;

  if     (is->videoStream >= 0) stream_index = is->videoStream;
  else if(is->audioStream >= 0) stream_index = is->audioStream;

  if(stream_index>=0){
    seek_target= av_rescale_q(seek_target, AV_TIME_BASE_Q,
                      pFormatCtx->streams[stream_index]->time_base);
  }
  if(av_seek_frame(is->pFormatCtx, stream_index, 
                    seek_target, is->seek_flags) < 0) {
    fprintf(stderr, "%s: error while seeking\n",
            is->pFormatCtx->filename);
  } else {
     /* handle packet queues... more later... */

```
av_rescale_q(a,b,c)是一个时间戳单位转换的函数，它的主要功能是计算a*b/c但是它处理了溢出之类的问题。AV_TIME_BASE_Q是AV_TIME_BASE的小数形式，AV_TIME_BASE * time_in_seconds = avcodec_timestamp而 AV_TIME_BASE_Q * avcodec_timestamp = time_in_seconds（需要注意AV_TIME_BASE_Q是一个AVRational对象，所以我们使用了avcodec的一个后缀是q的functions来处理）。

### 清空buffer
现在我们设置了跳转，但是并没有结束。我们之前设置了一个队列存储packet。现在我们跳转到了另一个地方，所以我们需要清空队列要不然电影不会跳转。不止如此，avcodec有它自己的缓存，所以每个线程都需要清空它。

为此，我们需要编写函数来清理packet的队列，然后我们需要使用某种机制通知音频和视频的线程，让它们清空avcodec的内部缓存。我们可以通过在清空队列后，向其中放一个特殊的packet来完成。在它们检测到特殊的包的时候，它们清空自己的缓存。

负责清空的函数代码如下：
```cpp
static void packet_queue_flush(PacketQueue *q) {
  AVPacketList *pkt, *pkt1;

  SDL_LockMutex(q->mutex);
  for(pkt = q->first_pkt; pkt != NULL; pkt = pkt1) {
    pkt1 = pkt->next;
    av_free_packet(&pkt->pkt);
    av_freep(&pkt);
  }
  q->last_pkt = NULL;
  q->first_pkt = NULL;
  q->nb_packets = 0;
  q->size = 0;
  SDL_UnlockMutex(q->mutex);
}
```

现在队列被清空了，我们放置一个“清空包”，不过首先我们需要定义它：
```cpp
AVPacket flush_pkt;

main() {
  ...
  av_init_packet(&flush_pkt);
  flush_pkt.data = "FLUSH";
  ...
}
```
然后将其放入队列
```cpp
  } else {
    if(is->audioStream >= 0) {
      packet_queue_flush(&is->audioq);
      packet_queue_put(&is->audioq, &flush_pkt);
    }
    if(is->videoStream >= 0) {
      packet_queue_flush(&is->videoq);
      packet_queue_put(&is->videoq, &flush_pkt);
    }
  }
  is->seek_req = 0;
}
```
我们还需要更改packet_queue_put函数来避免重复放入这个特殊的packet：
```cpp
int packet_queue_put(PacketQueue *q, AVPacket *pkt) {

  AVPacketList *pkt1;
  if(pkt != &flush_pkt && av_dup_packet(pkt) < 0) {
    return -1;
  }
```
然后在音视频的解码线程中，我们在packet_queue_get之后增加avcodec_flush_buffers：
```cpp
    if(packet_queue_get(&is->audioq, pkt, 1) < 0) {
      return -1;
    }
    if(pkt->data == flush_pkt.data) {
      avcodec_flush_buffers(is->audio_st->codec);
      continue;
    }
```
在视频线程中，代码也是一样的，只是audio换成video。


## 后记

现在我们有了一个可以用的播放器，但是有如下特性缺失：
1. 这个播放器不好用，因为它基于一个很老版本的ffplay.c，如果需要自己开发一件能用的播放器，建议查看最新版本的ffplay.c；
1. 错误处理。现在的代码几乎没有错误处理；
1. 暂停。我们现在不能暂停播放；
1. 硬件支持；
1. 按照bytes跳转，这在有些格式的文件中有用；
1. 丢弃帧，如果视频落后太多，我们需要丢弃帧而不是缩短刷新的间隔；
1. 网络支持，现在播放器不支持网络的视频流；
1. YUV格式的文件支持；
1. 全屏；
1. 其它的运行参数，可以参考ffplay.c的实现。