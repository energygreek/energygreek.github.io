---
title: yt-dlp downloads qq mv
date:
  created: 2025-05-07 
  updated: 2025-06-14
tags:
  - yt-dlp
  - tool
---

# 使用yt-dlp工具下载QQ音乐的MV

## 获取m3u8文件
- Right click -> Inspect -> Network tab.
- Enter m3u in the filter box.
- Refresh the page, then click play on the video.
- Several results will pop up in the network tab, in this case 15Mps.m3u8 was the 4k version. There's lower quality (10/5Mps) ones as well.
- Right click on the 15Mps.m3u8 line, copy the url.

## 下载

```
yt-dlp  https://apd-e884a42ebcb1819c8e185bdd22d6b32d.v.smtcdns.com/mv.music.tc.qq.com/AI85i7JvkmyF4poFqhirDtuEnNIwyQjQKiMDd0TFbAjw/6371017979FE2EAD13F0216756673D8F64FF084085A30F1293F38C7A31471243BB1F9358EDD23833D48FCB2D2AC4B515ZZqqmusic_default/qmmv_0b6biyaaeaaapqaa3pcdpjrfirqaajdaaasa.f9944.m3u8
```

## 提取音频
`-ss 14s` 跳过了MV开头的剧情
```
ffmpeg -i video_audio.mp4  -map 0:1 -ss 14s   -c:a copy  audio.m4a
```
使用mediainfo查看音频质量，比特率192kbps，对比其它音乐文件，压缩率挺高的。
```
mediainfo output.m4a | grep "Bit rate" 
Bit rate mode                            : Constant
Bit rate                                 : 192 kb/s
```

## 参考
https://www.reddit.com/r/DataHoarder/comments/1ab8812/how_to_download_blob_embedded_video_on_a_website/