---
draft: true
comments: true
date: 2025-06-16
tags: [tool]
---

# 网络文件浏览器
相比基于ftp, nfs，samba等传输共享方式，基于http的网络共享最简单了。因为终端方面，浏览器普及高，终端里面也能用curl等工具很方便使用。
服务端选择有： nginx（autoindex） 和 miniserve

## miniserve
支持 url_prefix

## nginx 

启用autoindex
```
location /myfolder {  # new url path
   alias /home/username/myfolder/; # directory to list
   autoindex on;
}
```

## 文件列表

1. 浏览器使用
2. cli使用，例如显示文件列表
```bash
curl -s 'http://user:password@ip:port/miniserve/?raw=true'  | w3m -dump -T text/html
```

## 


