---
draft: true
comments: true
date: 2025-09-12
tags: []
---

# EVS_dec使用方法
转码器抓包，解析udp的payload 成 SLTAMRPEM，在phonely中导出evs的rtp成 rtpdump 格式，再用命令转成raw, 再用ffmpeg转成wav

```sh
EVS_dec  -VOIP_hf_only=1 16   ~/Downloads/wxDownloads/3602.rtpdump   /tmp/evs-output.16k
ffmpeg -f s16le -ar 16k  -i /tmp/evs-output.16k   /tmp/evs-output-3602.wav
```