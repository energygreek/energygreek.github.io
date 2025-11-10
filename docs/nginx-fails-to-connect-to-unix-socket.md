---
date: 2020-07-25
tags: [flask,webserver]
---

# 记录在部署一个nginx fastcgi 项目时失败， fastcgi 通过unix socket 通讯

这是一个flask 项目， 使用flup创建的fastcgi

```python
#!/usr/bin/env python
from flup.server.fcgi import WSGIServer
from main import app

if __name__ == '__main__':
    # for apache
    # WSGIServer(app).run()
    # for nginx, need to set conmunication sock between nginx and the cgiserver
    WSGIServer(app, bindAddress='/tmp/myflask-fcgi.sock').run()
```

这里直接说明结果， socket 文件不能创建在/tmp/ 或者 /vat/tmp 目录。

nginx 错误配置
```
    location / { try_files $uri @myflask; }
    location @myflask {
        include fastcgi_params;
        fastcgi_param PATH_INFO $fastcgi_script_name;
        fastcgi_param SCRIPT_NAME "";
        fastcgi_pass unix:/tmp/fcgi.sock;
    }
```	

nginx 错误提示
```
 connect() to unix:/tmp/fcgi.sock failed (2: No such file or directory) while connecting to upstream
```



## 原因

如果socket创建在/tmp 或者 /var/tmp 目录， 只能被socket的进程发现。 意思是只有cgi程序能发现，所以nginx 无法连接了

正确配置是创建到/var/sockets 目录，并注意权限问题
```
    location / { try_files $uri @myflask; }
    location @myflask {
        include fastcgi_params;
        fastcgi_param PATH_INFO $fastcgi_script_name;
        fastcgi_param SCRIPT_NAME "";
        fastcgi_pass unix:/var/sockets/myflask-fcgi.sock;
    }
```	

## 关于unix socket

以上是一种解决办法，当然使用ip来连接nginx和cgi也是可以的，但是unix socket 有不少好处。 

## 关于cgi

上面无论使用unix socket 还是 ip 来访问cgi， 这两种方式都是通过cgi来提供web服务。  
而nginx 不支持cgi， 所以需要自己来通过python的 flup 来实现fastcgi。 nginx 只不过是一个代理的作用。而apache和tomcat 都能支持cgi

```
cgi 的定义是 common gateway interface， 是一个协议。 fastcgi，uwcgi都是升级版或者变种。  
flup实现了cgi的服务程序，也就是可以用来解析脚本（python php 等等），提供web服务。
```


另外除了以上方式， nginx也可以使用pypass proxy 来转发请求，由应用自己来实现网络处理。 

```
location / {
        proxy_pass https://localhost:8080
        proxy_set_header Host            $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP       $remote_addr;
        proxy_set_header X-Scheme        $scheme;
        access_log /var/log/nginx/access.log my_tracking;
    }

```

如过是python 就需要监听某个端口，这种是不稳定的。 不适合生产模式下使用。 
