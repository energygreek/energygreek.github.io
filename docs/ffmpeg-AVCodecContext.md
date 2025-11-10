---
date: 2023-08-02
tags: [ffmpeg,音视频]
---

# libavcodec使用
记录ffmpeg的学习系列，这是关于AVCodecContext的部分。

## sample_fmt

解码器avctx中的sample_fmt,是解码后的原始音频（PCM）采样格式，几乎所有的音频格式（amr,mp3）解码后都是PCM。但是采样格式可能不同，这个是由解码器来设定的。
具体在打开解码器的时候`avcodec_open2`, 需要传入解码解码avctx, 这个函数会调用解码器的`init`函数，从而将解码器的设定同步到avctx中。
对于编码器而言，这个值由是用户自己设置。如注释所说：

```
    /**
     * audio sample format
     * - encoding: Set by user.
     * - decoding: Set by libavcodec.
     */
    enum AVSampleFormat sample_fmt;  ///< sample format
```

所有的采样格式。因为工作需要解码amr, 先是自己基于opencore开发的，解码后是S16（即AV_SAMPLE_FMT_S16）, 而用ffmpeg解码后是AV_SAMPLE_FMT_FLTP, 这样的话重采样一次，效率低。后来发现ffmpeg有两套amr解码实现，选择opencore版本就是S16了。
```
enum AVSampleFormat {
    AV_SAMPLE_FMT_NONE = -1,
    AV_SAMPLE_FMT_U8,          ///< unsigned 8 bits
    AV_SAMPLE_FMT_S16,         ///< signed 16 bits
    AV_SAMPLE_FMT_S32,         ///< signed 32 bits
    AV_SAMPLE_FMT_FLT,         ///< float
    AV_SAMPLE_FMT_DBL,         ///< double

    AV_SAMPLE_FMT_U8P,         ///< unsigned 8 bits, planar
    AV_SAMPLE_FMT_S16P,        ///< signed 16 bits, planar
    AV_SAMPLE_FMT_S32P,        ///< signed 32 bits, planar
    AV_SAMPLE_FMT_FLTP,        ///< float, planar
    AV_SAMPLE_FMT_DBLP,        ///< double, planar
    AV_SAMPLE_FMT_S64,         ///< signed 64 bits
    AV_SAMPLE_FMT_S64P,        ///< signed 64 bits, planar

    AV_SAMPLE_FMT_NB           ///< Number of sample formats. DO NOT USE if linking dynamically
};
```
