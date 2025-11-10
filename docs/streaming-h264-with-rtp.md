---
date: 2024-02-07
tags: [ffmpeg,rtp,音视频]
---

# streaming h264 with rtp
这里记录了使用ffmpeg来发送h264的rtp流，主要问题是处理pps和sps的发送，看了非常多的文档和例子包括gptchat，直到用gdb跟ffmpeg才找到解决办法。

## 背景
公司的有个发送视频彩铃的业务，需要向终端发送h264。开始想法是创建ffmpeg进程来发送，不过发现进程太好资源并发上不去。后来用写代码来多线程发送。

### sdp协商转码问题
sdp协商结果会有不同的分辨率、等级、质量之类的参数(pps/sps)，为了避免转码，提前制作了不同参数的视频。不过后来发现只要正确发送pps/sps，终端都能正确解码，不是必须按sdp里的视频参数。

### 用ffmpeg发送
开始直接使用命令ffmpeg发送，发现终端不能解码。对比正常的rtp流发现缺少了pps/sps。
```
ffmpeg -i video.mp4 -an -c:v copy -f rtp rtp://ip:port
```
一番搜索发现ffmpeg将pps/sps等参数写到sdp中(out-of-band)，还用base64编码了。
```
a=fmtp:96 packetization-mode=1; sprop-parameter-sets=Z2QAKKzRAHgCJ+XAWoCAgKAAAAMAIAAAB4HjBiJA,aOvvLA==; profile-level-id=640028
```

ffplay等播放软件会解析sdp，载入pps/sps，所以正确解码，但是sip场景下只能通过rtp来发送sps/pps(in-band)。 后来发现用ffmpeg的'bit stream filter'能解决问题

```
ffmpeg -i video.mp4 -an -c:v copy -bsf h264_mp4toannexb -f rtp rtp://ip:port

```

即使不转码，使用ffmpeg进程发送视频，在两核的系统中大概只能发送十几路。

### 代码实现发送
基于ffmpeg的示例代码'doc/example/remux.c', 将原来写文件改为rtp即可。因为还没有用到'h264_mp4toannexb'，还不会发送pps和sps。

#### 关键问题就是如何使用这个bsf
网上看到的例子包括用gptchat生成的例子，都是类似下面的步骤。

```
# 搜索bsf
av_bsf_get_by_name("h264_mp4toannexb")
# 创建bsf的上下文
av_bsf_alloc(bsf, &bsf_ctx)
# 从输入的format上下文中复制编码参数
avcodec_parameters_copy(bsf_ctx->par_in, input_ctx->streams[video_stream_idx]->codecpar)
# 初始化bsf上下文
av_bsf_init(bsf_ctx)
# 读入packet
av_read_frame(input_ctx, &pkt)
# 送到bsf中处理
av_bsf_send_packet(bsf_ctx, pkt)
# 取出处理后的pkt
av_bsf_receive_packet(bsf_ctx, pkt)
# 发送rtp
```

抓包发现还是没有发送PPS/SPS, 而且第一个NALU是SEI，并且是坏的(Malformed)。调试发现bsf确实成功将SEI从AVCC转换成了AnnexB形式，也在SEI后追加了PPS和SPS。
```
(gdb) x/150bx pkt->data
0x5555556da310: 0x00    0x00    0x00    0x01    0x06    0x05    0x2e    0xdc
0x5555556da318: 0x45    0xe9    0xbd    0xe6    0xd9    0x48    0xb7    0x96
0x5555556da320: 0x2c    0xd8    0x20    0xd9    0x23    0xee    0xef    0x78
0x5555556da328: 0x32    0x36    0x34    0x20    0x2d    0x20    0x63    0x6f
0x5555556da330: 0x72    0x65    0x20    0x31    0x35    0x35    0x20    0x72
0x5555556da338: 0x32    0x39    0x30    0x31    0x20    0x37    0x64    0x30
0x5555556da340: 0x66    0x66    0x32    0x32    0x00    0x80    0x00    0x00
0x5555556da348: 0x00    0x01    0x67    0x64    0x00    0x28    0xac    0xd1
0x5555556da350: 0x00    0x78    0x02    0x27    0xe5    0xc0    0x5a    0x80
0x5555556da358: 0x80    0x80    0xa0    0x00    0x00    0x03    0x00    0x20
0x5555556da360: 0x00    0x00    0x07    0x81    0xe3    0x06    0x22    0x40
0x5555556da368: 0x00    0x00    0x00    0x01    0x68    0xeb    0xef    0x2c
0x5555556da370: 0x00    0x00    0x01    0x65    0x88    0x84    0x02    0xff
0x5555556da378: 0x91    0x3c    0x4a    0x51    0x5b    0xfd    0x02    0x3f
0x5555556da380: 0xc1    0x67    0x8d    0xc0    0x94    0x98    0xee    0x7d
0x5555556da388: 0x43    0x23    0xc0    0x4f    0xf7    0x56    0x37    0xfc
0x5555556da390: 0xf1    0xf3    0xd3    0x83    0x03    0xa9    0x6d    0xd2
0x5555556da398: 0x07    0xcf    0x19    0xa2    0x1e    0x29    0x64    0xfe
0x5555556da3a0: 0x1f    0x8e    0xd6    0x71    0x5f    0x33
```
`0x00 0x00 0x00 0x01`是annexb格式的起始码，第一个NALU是0x06(SEI)，第二个NALU是0x07(SPS)，第三个是0x08(PPS)。问题出在rtp打包的`ff_rtp_send_h264_hevc`。这里判断出`s->nal_length_size`不是0而是4, 所以还是以AVCC格式的首四个字节代表长度来解析pkt，而这是pkt是annexB格式了，前四个字节就是0x00, 0x00, 0x00, 0x01。所以打包错误。 问题怎么使rtp按annexB来打包，为什么`nal_length_size`是4不是0。

```
void ff_rtp_send_h264_hevc(AVFormatContext *s1, const uint8_t *buf1, int size)
{
    const uint8_t *r, *end = buf1 + size;
    RTPMuxContext *s = s1->priv_data;

    s->timestamp = s->cur_timestamp;
    s->buf_ptr   = s->buf;
    if (s->nal_length_size)
        r = ff_avc_mp4_find_startcode(buf1, end, s->nal_length_size) ? buf1 : end;
    else
        r = ff_avc_find_startcode(buf1, end);
```

找到初始化rtp的初始化函数`rtp_write_header`，发现设置`nal_length_size`的地方，原来判断extradata，如果第一个字节是1,则按avcc打包。
```
    case AV_CODEC_ID_H264:
        /* check for H.264 MP4 syntax */
        if (st->codecpar->extradata_size > 4 && st->codecpar->extradata[0] == 1) {
            s->nal_length_size = (st->codecpar->extradata[4] & 0x03) + 1;
        }
        break;
```
而当前的extradata是类似avcc的（又不同于AVCC, 因为多一个header)，第一个字节等于1，而通过调试ffmpeg_g到这里时，extradata是annexB格式，即首4个字节是0x00,0x00,0x00,0x01。关于extradata格式的问题，从开始我就发现刚初始化时，extradata就是0x01开头，那么问题ffmpeg_g的extradata什么时候变的，再次调试发现，bsf初始化跟上面的不同。在ffmpeg_g中bsf初始化在fftools/ffmpeg_mux.c文件中:
```
# fftools/ffmpeg_mux.c
static int bsf_init(MuxStream *ms)
{
    OutputStream *ost = &ms->ost;
    AVBSFContext *ctx = ms->bsf_ctx;
    int ret;

    if (!ctx)
        return avcodec_parameters_copy(ost->st->codecpar, ost->par_in);

    ret = avcodec_parameters_copy(ctx->par_in, ost->par_in);
    if (ret < 0)
        return ret;

    ctx->time_base_in = ost->st->time_base;

    ret = av_bsf_init(ctx);
    if (ret < 0) {
        av_log(ms, AV_LOG_ERROR, "Error initializing bitstream filter: %s\n",
               ctx->filter->name);
        return ret;
    }

    ret = avcodec_parameters_copy(ost->st->codecpar, ctx->par_out);
    if (ret < 0)
        return ret;
    ost->st->time_base = ctx->time_base_out;

    ms->bsf_pkt = av_packet_alloc();
    if (!ms->bsf_pkt)
        return AVERROR(ENOMEM);

    return 0;
}
```
发现，ffmpeg的bsf初始化多一个步骤，将bsf的par_out拷贝到输出AVstream中。而这par_out中的extradata就是我们要的annexB格式！
```
ret = avcodec_parameters_copy(ost->st->codecpar, ctx->par_out);
```

问题找到答案了，1是初始化rtp muxer前先初始化bsf，2是初始化bsf后将par_out拷贝回rtp muxer，再初始化rtp muxer。

#### 正确例子
基于ffmpeg的doc/example/remux.c 删除了错误处理。只关心h264，所以输入输出都只处理index=0的包。
```
int main(int argc, char **argv) {
  const AVOutputFormat *ofmt = NULL;
  AVFormatContext *ifmt_ctx = NULL, *ofmt_ctx = NULL;
  AVPacket *pkt = NULL;
  const char *in_filename, *out_filename;
  int ret = 0;
  in_filename = "video.mp4";
  out_filename = "rtp://127.0.0.1:10020";
  pkt = av_packet_alloc();
  ret = avformat_open_input(&ifmt_ctx, in_filename, 0, 0);
  ret = avformat_find_stream_info(ifmt_ctx, 0);
  av_dump_format(ifmt_ctx, 0, in_filename, 0);

  // 创建输出rtp上下文，不初始化
  avformat_alloc_output_context2(&ofmt_ctx, NULL, "rtp", out_filename);
  // 初始化bsf
  const AVBitStreamFilter *bsf_stream_filter =
      av_bsf_get_by_name("h264_mp4toannexb");
  AVBSFContext *bsf_ctx = NULL;
  ret = av_bsf_alloc(bsf_stream_filter, &bsf_ctx);
  ret =
      avcodec_parameters_copy(bsf_ctx->par_in, ifmt_ctx->streams[0]->codecpar);
  ret = av_bsf_init(bsf_ctx);

  ofmt = ofmt_ctx->oformat;
  AVStream *out_stream = avformat_new_stream(ofmt_ctx, NULL);
  // 关键在这！！ 原来是从ifmt_ctx的stream中拷贝codecpar，改成从bsf中拷贝
  // ret = avcodec_parameters_copy(out_stream->codecpar, in_codecpar);
  ret = avcodec_parameters_copy(out_stream->codecpar, bsf_ctx->par_out);
  out_stream->codecpar->codec_tag = 0;
  av_dump_format(ofmt_ctx, 0, out_filename, 1);
  if (!(ofmt->flags & AVFMT_NOFILE)) {
    ret = avio_open(&ofmt_ctx->pb, out_filename, AVIO_FLAG_WRITE);
  }
  // 初始化bsf后再初始化rtp
  ret = avformat_write_header(ofmt_ctx, NULL);

  while (1) {
    AVStream *in_stream, *out_stream;
    ret = av_read_frame(ifmt_ctx, pkt);
    if (pkt->stream_index != 0) {
      continue;
    }
    in_stream = ifmt_ctx->streams[pkt->stream_index];
    out_stream = ofmt_ctx->streams[pkt->stream_index];
    log_packet(ifmt_ctx, pkt, "in");

    av_bsf_send_packet(bsf_ctx, pkt);
    av_bsf_receive_packet(bsf_ctx, pkt);
    /* copy packet */
    av_packet_rescale_ts(pkt, in_stream->time_base, out_stream->time_base);
    pkt->pos = -1;
    log_packet(ofmt_ctx, pkt, "out");
    ret = av_interleaved_write_frame(ofmt_ctx, pkt);
  }

  av_write_trailer(ofmt_ctx);
}

```

### h264_mp4toannexb
这个bsf将从mp4文件中读取的avcc格式的packet转换成annexb格式的packet。调试发现第一个读出来的包是SEI一个I帧，通过bsf处理后，会在SEI后面追加上PPS和SPS信息。而读取mp4文件时，sps/pps在extradata中。

### 相关知识
这些知识反复看来看去，总也不能贯穿起来，直到问题解决才算明白。

#### NALU
NALU是真正用来保存h264视频信息的，包括I帧，P/B帧，PPS，SEI，SPS等。NALU由两部分组成：头（1字节）和payload，头中包含nalu的类型。h264规范只定义了NALU本身单元，但没有定义怎么保存NALU单元，所以有了两种格式保存NALU，AVCC和AnnexB。

##### NALU类型
NAL unit type的值和说明，类型后面跟payload。详细参见在Rec. ITU-T H.264文件的63页，这里展示常用的
* 5,Coded slice of an IDR picture (I帧)
* 6,Supplemental enhancement information (SEI)
* 7,Sequence parameter set (SPS) 
* 8,Picture parameter set (PPS)
* 24,Single-Time Aggregation Packet(STAP-A)

从抓包看到SPS算上payload的长度为30，PPS算上payload的长度为4。不知道长度是不是固定的。

##### STAP-A
STAP-A是多个NALU的聚合(Aggregation)，即这个NALU的payload里是多个NALU。STAP-A类型的NAL用来发送PPS/SPS/SEI等多种聚合。因为这些单元都很小。STRAP-A类型的header也是一个字节，但是payload里面有多个NALU，并且每个NALU前面用2字节来表示这个NALU的大小。
```
|STAP-A header|NALU-1 size|NALU-1|NALU-2 size|NALU-2|
```

NALU-1中又有header和payload。

#### AVCC和AnnexB
上面说了规范没有定义怎么保存NALU，所以有了这两个格式，他们两是平等关系，只有保存的格式不同而已。AVCC用来保存，annexB用来流传输。
* AVCC用1~4个字节来表示NALU的长度，长度后面是NALU。读取方法是先读长度，再读取NALU。再读下一个长度，再读下一个NALU...
* 而annexb用`0x00,0x00,0x00,0x01`或者`0x00,0x00,0x01`的起始码(start code)来分隔不同的NALU，所以方法是先读起始码，再一直读，直到发现下一个起始码，表示这个NALU结束，下一个NALU开始。

ffmpeg使用中发现，AVCC一般用4字节表示NALU的长度，具体多少字节，在ffmpeg的extradata中有定义。annexB也是用4字节的起始码，也就是0x00,0x00,0x00,0x01。

#### Fragmentation Units (FUs) 分片
FU就是网络分片，因为I帧是一个完整的图片，所以非常大，为了保证udp不丢包，所以要分次发送。 第一个分片的FU的头设置了Start bit， 最后一个分片的FU头设置了END bit。分片是在rtp muxer中完成，注意ffmpeg中一个packet可以包含多个音频帧，但是只包含一个帧，直到发送rtp之前，一个packet总是完整的一帧视频（I/P/B）。 对于比较小的packet，例如聚合了PPS/SPS等信息的STAP-A包，不需要分片。


#### extradata
上面很多次提到ffmpeg的extradata, 就是AVCodecParameters.extradata，它的长度是AVCodecParameters.extradata_size。在读取mp4文件的时候，ffmpeg会自动填充，在解码rtp的时候可能就需要手动填充了。extradata的比特位如下，首先是6字节的头，然后是多个SPS类型的NALU（2字节的长度分割多个NALU），再然后是PPS类型的NALU个数，最后是PPS类型的多个NALU(2字节的长度分割多个NALU）。
```
bits    
8   version ( always 0x01 )
8   avc profile ( sps[0][1] )
8   avc compatibility ( sps[0][2] )
8   avc level ( sps[0][3] )
6   reserved ( all bits on )
2   NALULengthSizeMinusOne
3   reserved ( all bits on )
5   number of SPS NALUs (usually 1)

repeated once per SPS:
  16         SPS size
  variable   SPS NALU data

8   number of PPS NALUs (usually 1)

repeated once per PPS:
  16       PPS size
  variable PPS NALU data
```

里面包含了一个或多个PPS/SPS(NALU)，但保存的格式，既不是AVCC也不是AnnexB。因为上面可知AVCC用1~4个字节表示NALU的长度，AnnexB用3~4字节的起始码，而extradata是有一个6字节的header，里面有个字段叫`NALULengthSizeMinusOne`就是定义了AVCC使用多少个字节来表示NALU的长度。如果`NALULengthSizeMinusOne`等于0，那么AVCC用1字节表示NALU的长度。通常就是用4字节。

##### extradata例子

```
(gdb) x /150bx fmt_ctx->streams[0]->codecpar->extradata
0x55555571d9c0: 0x01    0x64    0x00    0x28    0xff    0xe1    0x00    0x1e
0x55555571d9c8: 0x67    0x64    0x00    0x28    0xac    0xd1    0x00    0x78
0x55555571d9d0: 0x02    0x27    0xe5    0xc0    0x5a    0x80    0x80    0x80
0x55555571d9d8: 0xa0    0x00    0x00    0x03    0x00    0x20    0x00    0x00
0x55555571d9e0: 0x07    0x81    0xe3    0x06    0x22    0x40    0x01    0x00
0x55555571d9e8: 0x04    0x68    0xeb    0xef    0x2c    0x00    0x00    0x00
0x55555571d9f0: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55555571d9f8: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55555571da00: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55555571da08: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55555571da10: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55555571da18: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55555571da20: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55555571da28: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55555571da30: 0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55555571da38: 0xc1    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55555571da40: 0x9d    0xad    0x00    0x00    0x50    0x55    0x00    0x00
0x55555571da48: 0xfa    0x19    0xc4    0x92    0x40    0xa0    0xdc    0xba
0x55555571da50: 0x00    0x00    0x00    0x00    0x00    0x00
```

* 第5字节0xff,二进制为:1111 1111, 后2位表示`NALULengthSizeMinusOne`=3，所以ffmpeg用4字节表示NALU的大小（AVCC格式）。
* 第6字节0xe1,二进制为:1110 0001，后5位表示SPS的个数=1,所以只有一个SPS。
* 第7，8字节表示SPS的长度，0x00，0x1e，二进制为:0000 0000, 0001 1110，所以SPS长度为30。
* 跳过30个字节，0x01，二进制为:0000 0001, 表示有1个PPS。
* 后面的2个字节，0x00,0x04,二进制为:0000,0004,表示PPS的长度为4个字节。

##### NALU头的解析
NALU头是1字节，第一位F bit, 后两位NRI bit, 后五位表示NALU的type。

SPS的头是0x67，而进制为: 01100111，所以type值正好是7。
PPS的头是0x68，而进制为: 01101000，所以type值正好是8。

对比wireshark解析结果可以确认上面的理解正确，可见extradata就是6个字节的header加多个AVCC格式的NALU。

##### 创建extradata
rtp解码时，需要手动生成extradata。创建AVCC格式的extradata
```
write(0x1);  // version
write(sps[0].data[1]); // profile
write(sps[0].data[2]); // compatibility
write(sps[0].data[3]); // level
write(0xFC | 3); // reserved (6 bits), NULA length size - 1 (2 bits)
write(0xE0 | 1); // reserved (3 bits), num of SPS (5 bits)
write_word(sps[0].size); // 2 bytes for length of SPS
for(size_t i=0 ; i < sps[0].size ; ++i)
  write(sps[0].data[i]); // data of SPS

write(&b, pps.size());  // num of PPS
for(size_t i=0 ; i < pps.size() ; ++i) {
  write_word(pps[i].size);  // 2 bytes for length of PPS
  for(size_t j=0 ; j < pps[i].size ; ++j)
    write(pps[i].data[j]);  // data of PPS
}
```

创建annexB格式的extradata
```
write(0x00)
write(0x00)
write(0x00)
write(0x01)
for each byte b in SPS
  write(b)

for each PPS p in PPS_array
  write(0x00)
  write(0x00)
  write(0x00)
  write(0x01)
  for each byte b in p
    write(b)
```

## wireshark解析
从wiresshark对比用bsf和不用bsf的抓包发现，SEI包的内容没有变化，是不是可以不需要转成annexb，直接将PPS/SPS直接拷贝到packet中发出去呢？

## 参考
https://membrane.stream/learn/h264/3
https://github.com/cisco/openh264/issues/2501#issuecomment-231340268
https://stackoverflow.com/questions/17667002/how-to-add-sps-pps-read-from-mp4-file-information-to-every-idr-frame
https://stackoverflow.com/questions/24884827/possible-locations-for-sequence-picture-parameter-sets-for-h-264-stream
https://aviadr1.blogspot.com/2010/05/h264-extradata-partially-explained-for.html


