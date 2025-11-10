---
date: 2021-01-15
tags: [socket, udp, socketopt]
---

# udp 的端口复用实现负载均衡

## 前言

偶尔看到 python 3.9 的release note 里面提到一个bug
```
asyncio¶

出于重要的安全性考量，asyncio.loop.create_datagram_endpoint() 的 reuse_address 形参不再被支持。 这是由 UDP 中的套接字选项 SO_REUSEADDR 的行为导致的。 更多细节请参阅 loop.create_datagram_endpoint() 的文档。 （由 Kyle Stanley, Antoine Pitrou 和 Yury Selivanov 在 bpo-37228 中贡献。。）
```
意思是tcp的socket option:SO_REUSEADDR不适用于udp：
在tcp中这个选项表示立即回收端口，减少 time_wait 的时间。而在udp中，这个选项表示多个socket可以绑定一个端口， 由内核来分发请求。

所以看到此功能，自己试了一下，确实如此， 顺便回顾一下知识

## 主要代码

```

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8888);
    inet_pton(AF_INET,"127.0.0.1",(void*)&addr.sin_addr);
    // inet_pton 支持ipv4和ipv6,是比较新的转换函数

    sfd = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if(sfd == -1)
    {
        perror("socket");
        exit(1);
    }
    int val = 1;
    if(0 != setsockopt(sfd, SOL_SOCKET, SO_REUSEPORT,&val, sizeof(val))){
        perror("setsockopt");
        exit(1);
    }
    /* sockaddr 和 sockaddr_in 有什么区别？
       struct sockaddr {  
     	sa_family_t sin_family;//地址族
　　  	char sa_data[14]; //14字节，包含套接字中的目标地址和端口信息               
　　    }; 
　　    struct sockaddr_in {
　　      sa_family_t sin_family;//地址族
　　      uint16_t sin_port;
　　      struct in_addr sin_addr;	// 32 位地址
　　      char	sin_zero[8];	// reserve
　　    };
　　    struct in_addr {
　　    	In_addr_t	s_addr;  //32位
　　    };
　　    
　　    sockaddr_in 和 sockaddr 长度相同，都 sin_family + 14 个字节，但是前者显式划分了
　　 */ 
    if(bind(sfd, (struct  sockaddr*) &addr, sizeof(addr)) != 0)
    {
        perror("bind");
        exit(1);
    }

```

## 效果

先启动两个服务端，再使用ncat 来模拟请求, 

```
ncat -uv 0.0.0.0 8888
```

启动ncat时，系统会分配给一个服务端处理。 但是重启ncat时， 会切换到另一个服务端处理



