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
这与我们之前的
