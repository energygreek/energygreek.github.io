---
date: 2024-01-29
tags: [ffmpeg]
---

## 笔记本的笔记整理和誊抄
主要是关于ffmpeg的知识总结

## FFmpeg
读法"F-F-M-派格", 由三个可执行文件组成"ffmpeg/ffplay/ffprobe"。

### ffmpeg

* 支持的容器 `ffmpeg -muxers` 分为Demuxing和Muxing（D/E）封装和解封装，支持Muxing肯定支持Demuxing。
* 支持的编码 `ffmpeg -codecs` 分为解码和编码(decoder/encoder)，支持编码一般能支持解码。
* 容器帮助信息 `ffmpeg -h muxer=mp4` `ffmpeg -h demuxer=mp4` 加上`-h full`查看更多信息
* 编码帮助信息 `ffmpeg -h encoder=h264` `ffmpeg -h decoder=h264` 加上`-h full`查看更多信息
* 支持的滤镜 `ffmpeg -filters` 这里的滤镜不是PS等图片编辑软件里面的滤镜，而应该称做特效，支持视频和音频还有字幕。

当转码或转容器格式时，-map参数用来手动选择流，参数格式 -map n:m:x 其中n表示选择第n个输入，m代表第n个输入中的第m个流，而x表示第n个输入中的第m个流的第x个通道。当没有-map参数时，ffmpeg会根据容器类型自动选择合适的流。-map还能用来选择filtergraph中的滤镜输出。

#### filter 滤镜
分为 filtergraph filterchain 和 filter
* filtergraph 是包含很多个filter的有向图，每两个滤镜之间都可以有多个连接。
* filter 分为source filter, sink filter, filter，其中source filter没有输入端，sink filter没有输出端。 

多个filter用","来连接，形成filterchain，而多个chain用";"来连接，形成fitergraph。

简单滤镜格式参数格式 -f:v 或 -f:a \[输入流或标记名]滤镜参数\[临时标记名];(重复n个)。复杂滤镜格式 -filter_complex
overlay滤镜用来设置显示层次，例如\[overlay]filter 将输入显示在另一个上面。

##### 加速
使用滤镜来加速播放
###### 加速视频
使用setpts来加速视频 -f:v setpts=0.5*PTS, 注意：1调整范围\[0.25,4]，越小越快，从加速4倍到减速4倍。2若只是调整视频则将音频关掉。3对视频加速时，如果不想丢帧，则使用-r参数调整输出的FPS。
###### 加速音频
最简单的方法是调整采样率，但这样会改变音色，一般使用对原音进行重采样。 -f:a atempo:2.0。范围 \[0.5,2.0]，若要4倍加速，使用滤镜组合 -f:a atempo=2.0,atempo=2

###### 同时加速音频和视频
可以用 -filter_complex "\[0:v]setpts=0.5*PTS\[v];\[0:a]atempo=2.0\[a]" -map \[v] -map \[a] 。也可以组合使用上面的两个滤镜。

##### 视频裁剪
-ss 选项seek，从开头跳过一段片长，参数两种形式 00:00:00或者秒数。-ss设置在输入对象和输出对象上有不同效果。
* -ss 指定在输入时，不仅可以用于复制，也可以用于转码。并且都是基于关键帧来寻找位置，默认启用选项“frame_accurate"
* -ss 指定在输出时，源文件依旧会每帧都要解码并丢弃，知道跳过指定的时间所以要等待。但是-SS用在输出时，最大的优点是当使用滤镜时，其timestamp不会重置0,这个在录制字幕时有用，不需要修改字幕的时间戳。
###### 在裁剪时有两个选项 frame_accurate_seek 和 -noaccurate_seek 区别, 后者会使用附近的关键帧。
###### -ss -t 与 -ss -to的区别，-ss 10 -t 20表示截取10s~30s的视频，而 -ss 10 -to 20 表示截取10s~20s的视频。

截取视频不准确问题：
* 当使用-ss和-c:copy时，由于ffmpeg强制使用I帧，所以可能调整起始时间到负值，即提前于ss设置的时间
* 例如 -ss 157 但直到159时才有I帧，它会有2秒的时间只有声音没有画面。 

#### 编译问题
* 编译ffmpeg支持h265, 因为h265库是用c++写的，所以编译ffmpeg时要加上额外库`-lstdc++`
* 如果pc文件不在标准路径，需要修改环境变量`export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:"pc文件路径"`
* 如果链接到了动态库，可以使用enable-rpath设置运行时找库路径

### ffplay 播放媒体文件
* 在视频中插入字幕 `ffplay -f:v "subtitle=input.srt" input.mp4`
* 音频可视化 `ffplay -showmode 1 input.mp3` 默认是0, 用傅立叶变换显示声音的频率频谱，处理时长差不多是1s。
* 视频显示运动方向 `ffplay -flags2 +export_mvs  -vf codecview=mv=pf -i input.mp4`

ffplay 指定视频显示大小，当播放原始视频时如input.h264，用-video_size来指定视频大小，而非设置播放器显示大小。如果要控制播放器显示大小用 scale/resize滤镜。

-s 选项，当用在输入选项时，可以替代video_size。当用在输出选项时，可以替代scale滤镜，但只能于filtergraph最后的filter，若想作用在filtergraph其它位置，则必须显示使用scale。


#### 时间戳 PTS/DTS
PTS是显示时间戳, presentation timestamp 指表示渲染的时间点，而DTS是Decode timestamp，解码时间戳，表示解码时间点。因为视频的B帧要等待下一个I帧，所以B帧会先保存在缓冲区。
在ffplay和ffmpeg 播放/转码视频时，控制台显示了tbn，tbc，tbr， 这些都是timebase，也叫时间精度。每次处理后当前的时间戳会增加一个timebase。
* tbn: timebase of AVstream, 表示从容器中读取的timestamp的增加一次timebase。 
* tbc: timebase of codec, 表示某个流的对应的编码采用的时间基，每次解码都增加一次timebase。 50tbc时，1S有50帧。
* tbr: 一帧的timebase，是预估值。 

### ffprobe 查看媒体文件信息

视频分析软件除了ffprobe还有 mp4info。


### 直播
视频直播技术里， HLS和DASH等分片技术比较流行。

### rtp流
rtp是通信语音行业的常用协议，可以发送视频和语音之一，不可以同时发送视频和语音，语音也只支持单通道（对于PCM类型的语言而言，而rtp支持OPUS发送多通道）。rtp有多流同步机制，接收终端可以将多个rtp流合并，支持类似webrtc的会议模式，rtp也能支持5G的视频通话。

#### rtp的sdp
在建立rtp连接之前，两个终端要协商一个两边都支持的视频和语音编码，否则无法建立。 sdp则描述了ip/端口/媒体编码相关的信息，这些不能通过rtp协议来传送（in-band）。

#### clockrate 时钟频率
sdp中的a属性有clok


### ffmpeg发送接收rtp
rtp对于ffmpeg而言是一种容器，与其它的mp4/avi类似，代码实现也同样在libavformat里。只是不同在于前着是写文件，rtp容器则是发送网络包。例如读取mp4文件并发送音频的命令`ffmpeg -i input.mp4 -vn -ac 1 -f rtp rtp://ip:port`，`-vn`表示不发送视频，`-ac 1`表示合并音频多通道为单通道。

## 常见分辨率
640x480（480p）分辨率比4:3
1280x720（720p HD）
1920x1080(1080p FHD）
3840x2160(4k UHD)

## h264
I/P/B 三种类型的帧:
* I帧, Index frame, 是完整的一幅图片，不依赖其它帧就能解码显示。
* P帧, Delta frame, 是向前帧，依赖I帧来解码。只有运动的对象，而没有背景信息。
* B帧， Bidirectional pridict picture, B帧则依赖P帧和下一个I帧来解码`I->P->B->I`，最节省字节。
所以视频种的B帧越多，视频文件的体积越小，而解码的复杂度越高。

ffmpeg转码h264的命令，`ffmpeg -i input.mp4 -av sample-rate -crf {17~23} -b:v video-bitrate -r frame-rate -profile {baseline|main|high}`
* -av 指定sample rate采样率
* -crf 指定视频质量
* -b:v video bitrate 视频的比特率
* -b:a audio bitrate 音频的比特率
* -r frame rate 指定帧率
* -profile 指定视频的档次
* -level 指定视频等级

使用baseline prfile时，不会包含B帧，当使用实时流媒体直播时，采用baseline编码相对main和high相对可靠，但加入B帧可以减小比特率。


### 视频格式P/I
p代表progressive代表逐行扫描，I代表interlaced 隔行扫描，但是一旦视频损坏时，视频几乎无法观看。

### 音视频同步
音频和视频都是独立的线程处理，主要通过各自PTS来同步，但实际上音视频大部分情况是不同步的，偶尔是同步的。当音频和视频时间误差超过阈值时，就会去重同步。重同步主要是调整视频，因为人眼对视频的敏感度不如对音频的敏感度。

### ffmpeg播放h264时，限制比特率（bps,bit per second)
视频的比特率分为两个码率，CBR恒定比特率，VBR波动比特率。互联网视频多为VBR。如果想用CBR，-b:v 设置编码比特率，但这里设定的平均值，不能很好控制最大和最小码率。要控制最大最小比特率，需要组合使用 -b:v，maxrate，minrate。另外还要设置buff打小`-bufsize`。例如:
```
ffmpeg -i input.avi -bo 15M -minrate 0.5M -maxrate 0.5M -bufsize 1M out.mkv
```
bufsize 说明，如果不使用bufsize，其变化范围将比我们的预期大很多。当设置bufsize很大时浮动范围比较大，设置太小时，会导致视频质量降低。最合适的大小时-b:v的大小一半，然后逐渐增加bufsize，直到bitrate变化比较明显时，这是质量最高而且比较恒定的大小。

### 通过比特率来计算文件大小：
视频大小 = 比特率 * time_in_second / 8
未压缩音频大小 = 采样率 * 采样深度 * channel数 * time_in_second / 8
压缩音频文件大小 = 比特率 * time_in_second /8


### 图片
alpha 通道是RGB通道外的一个通道，用来表示像素的透明度，是另一个维度。
* 当使用16bit的位图时，对于每个像素5bit表示红，5bite表示绿，5表示蓝，最后一个bit表示alpha。所以这时只有1/0选择，所以只有透明和不透明的。
* 当使用32bit的位图时，8bit来表示alpha通道，就有0～255个值表示不同程度的透明。而alpha的值不直接表示透明度而是通过与RGB三个通道的值相乘，这样得道了显示的值。

#### 高频洗劫低频轮廓 outline
在傅里叶边换中，用不同频率相位的正弦波，可以模拟出各种类型的信号。其中高频信号越高，还原效果越好。

#### Avframe 存储一帧解码后的像素数据，Avpacket 存储一帧压缩的数据。

### 音频pcm
* 采样：是将音频模拟信号定时取值，得到一个离散序列。为了能将离散序列恢复成为模拟信号供播放器播放，采样频率要是声音频率的至少两倍。所以若要采样1k频率的声音，需要用2k采样率。
* 采样深度：单个采样的储存长度。
* 音频帧：单位时间里的多个采样。

## 数学
### 差分和微分 difference equation / differential equation
差分是离散，各个连续项之间不存在其它元素。例如1,2,3,4,5 不存在1.1,1.2,1.3。而微分是连续的，1和2之间可以有1.1,1.11,1.12,1.2等。总结是差分是微分的离散化。

### 信号和系统
离散傅里叶边换 DFT discret fourier transform，傅里叶变换出来的结果是复数=X+YI，I^2=-1。 谐波是什么，为何傅里叶变换要复数，因为表示波的方法要用到频率和相位还有振幅。 
复数模等于振幅 `e^(jwb) = cos(wb) + isin(wb)`

波的表示方法： y=Asin(B(x+C)), A振幅，B频率，C相位。





