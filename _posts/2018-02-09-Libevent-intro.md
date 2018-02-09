---
layout:     post
title:      Libevent 帮助文档
subtitle:    "\"libevent 介绍\""
date:       2018-02-09
author:     Wangsigui
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - libevent,网络编程
---

> These documents are Copyright (c) 2009-2012 by Nick Mathewson, and are made available under the Creative Commons Attribution-Noncommercial-Share Alike license, version 3.0. Future versions may be made available under a less restrictive license.
> Additionally, the source code examples in these documents are also licensed under the so-called "3-Clause" or "Modified" BSD license. See the [license_bsd file](http://www.wangsigui.top/2018/02/09/license-bsd/) distributed with these documents for the full terms.
>
>For the latest version of this document, see http://www.wangafu.net/~nickm/libevent-book/TOC.html
>
>To get the source for the latest version of this document, install git and run "git clone git://github.com/nmathewson/libevent-book.git"

## 关于异步IO的简单介绍  
大多数初学者都是从阻塞式IO开始接触网络编程的。同步IO的定义是，当你调用它的时候，它不会立刻返回结果，而是直到IO上的所有操作都被完成，或者直到时间长得网络协议栈放弃了这次操作，它才会返回。举个例子，当你调用"connect()"函数主动发起一次TCP连接时，你的操作系统将SYN包排队发送至TCP连接另一端的主机(也即服务器)，**(译者注：此时应用程序阻塞在IO操作上)**除非你的操作系统从对面收到了SYN ACK包，或者经过足够的时间(超时)，操作系统决定放弃这次连接请求，否则操作是不会返回给应用程序的。  
> TCP三次握手  
>第一次握手：建立连接时，客户机发送SYN包到服务器(syn = j)，进入SYN_SENT状态，等待服务器确认  
>第二次握手：服务器收到客户机发来的SYN包，必须确认客户的SYN(ack = j + 1),同时自己也发送一个SYN(syn = i)包，以及一个ACK包。此时服务器进入SYN_RECV状态  
>第三次握手：客户机收到服务器发送的SYN+ACK包，向服务器发送ACK(ack=i+1)包，此包发送完毕，客户机进入ESTABLISH，服务器收到ACK包，也会进入ESTABLISH状态，此时TCP连接建立成功.  
>　　　　　　　　　　　　CLIENT　　　　　　　　　　SERVER  
>　　　　　　　SYN_SENT　　|　　　　　　　　　　　　|　　LISTEN  
>　　　　　　　　　　　　　|　　　　　　　　　　　　|  
>　　　　　　　　　　　　　|　　　　　　　　　　　　|    SYN_RECV  
>　　　　　　　　　　　　　|　　　　　　　　　　　　|  
>　　　　　　ESTABLISH　　　|　　　　　　　　　　　　|  
>　　　　　　　　　　　　　|　　　　　　　　　　　　|　　ESTABLISH  

下面是一个使用阻塞网络调用的简单的例子。它向 www.baidu.com(原文是www.google.com)打开一个连接，发送一个简单的HTTP请求，然后将response打印到标准输出(stdout)。
### 例子: 一个简单的阻塞HTTP client　
```
/*************************************************************************
	> File Name: block_http_client.c
	> Author: 
	> Mail: 
	> Created Time: Fri 09 Feb 2018 01:29:01 PM CST
 ************************************************************************/
/*For sockaddr_in*/
#include <netinet/in.h>
/*For socket functions*/
#include <sys/socket.h>
/*For gethostbyname*/
#include <netdb.h>

#include <unistd.h>
#include <string.h>
#include <stdio.h>

int main(int c, char **v)
{
    const char query[] =
        "GET / HTTP/1.0\r\n"
        "Host: www.baidu.com\r\n"
        "\r\n";
    const char hostname[] = "www.baidu.com";
    struct sockaddr_in sin;
    struct hostent *h;
    const char *cp;
    int fd;
    ssize_t n_written, remaining;
    char buf[1024];

    /*Look up for IP address fot the hostname. Watch out; this isn't threadsafe on most platforms.*/
    h = gethostbyname(hostname);
    if(!h)
    {
        fprintf(stderr,"Couldn't lookup %s: %s",hostname,hstrerror(h_errno));
        return 1;
    }
    if(h->h_addrtype != AF_INET)
    {
        fprintf(stderr,"No ipv6 support,sorry.");
        return 1;
    }

    /* Allocate a new socket */
    fd = socket(AF_INET,SOCK_STREAM,0);
    if(fd < 0)
    {
        perror("socket");
        return 1;
    }

    /*Connect to the remote host.*/
    sin.sin_family = AF_INET;
    sin.sin_port  =  htons(80);
    sin.sin_addr = *(struct in_addr*)h->h_addr;
    if(connect(fd,(struct sockaddr*) &sin,sizeof(sin)))
    {
        perror("connect");
        close(fd);
        return -1;
    }

    /*Write Query*/
    /* XXX can send succeed partially?*/
    cp = query;
    remaining = strlen(query);
    while(remaining)
    {
        n_written = send(fd,cp,remaining,0);
        if(n_written < 0)
        {
            perror("send");
            close(fd);
            return 1;
        }
        remaining -= n_written;
        cp += n_written;
    }

    /*Get an answer back*/
    while(1)
    {
        ssize_t result = recv(fd,buf,sizeof(buf),0);
        if(result == 0)
            break;
        else if(result < 0)
        {
            perror("receive");
            close(fd);
            return 1;
        }
        fwrite(buf,1,result,stdout);
    }
    close(fd);
    return 0;
}
```
上述代码中的网络调用全是阻塞的：gethostbyname在成功地或者失败地解析了www.baidu.com之前不会返回；connect在它建立连接(成功或失败)之前不会返回；recv在它接受数据或者关闭之前不会返回；send至少要把它待发送区间的数据刷新一次到内核的写缓冲区，否则它不会返回。  
到现在为止，阻塞IO看起来还没什么大问题。如果你的程序在处理网络IO的同时不会再做其他事情了，阻塞IO可以为你工作得很好。但是假设你要写个程序一次性去处理多个连接。比方说：你需要同时从两个连接中读取输入数据，你就无法得知到底哪个连接先来的数据。  
### 糟糕的例子
```
/*This won't work*/
char buf[1024];
int i, n;
while(i_still_want_to_read())
{
    for(i=0; i < n_sockets; ++i)
    {
        n = recv(fd[i],buf,sizeof(buf),0);
        if(n == 0)
            handle_close(fd[i]);
        else if(n < 0)
            handle_error(fd[i],errno);
        else
            handle_input(fd[i],buf,n);
    }
}
```
假设数据先到达了**fd[2]**这个socket，上面的程序在**fd[0]**和**fd[1]**获取数据结束之前是不会尝试先读取**fd[2]**上的数据的。  
有时候人们用多线程或者多进程服务器的方式来解决这个问题。多线程一种最简单的方式就是让每个线程或者进程单独处理一个连接。因为每一个连接都有自己的处理进程，所以某一个连接上引发等待的阻塞IO调用不会引起任何其他连接进程的等待。  
下面是一个很简单的服务器，它在40713端口上监听TCP连接，每次读取一行输入数据，然后使用ROT13加密方法对每行数据进行加密输出。它使用Unix fork()调用为每个新来的连接创建新的进程。

### 采用fork的ROT13服务器
```

/*************************************************************************
	> File Name: fork_rot13_server.c
	> Author: 
	> Mail: 
	> Created Time: Fri 09 Feb 2018 03:14:44 PM CST
 ************************************************************************/

#include<stdio.h>
#include <netinet/in.h>
#include <sys/socket.h>

#include <unistd.h>
#include <string.h>
#include <stdlib.h>

#define MAX_LINE 16384                   //max characters for each line

char
rot13_char(char c)
{
    /* We don't want to use isalpha here; setting the locale would change
     * which characters are considered alphabetical.*/
    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
        return c + 13;
    else if((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
        return c - 13;
    else 
        return c;
} 

void 
child(int fd)
{
    char out_buf[MAX_LINE + 1];
    size_t outbuf_used = 0;
    ssize_t result;
    ssize_t nwrite;
    while(1)
    {
        char ch;
        result = recv(fd,&ch,1,0);
        if (result == 0)
        {
            break;
        }
        else if( result == -1 )
        {
            perror("recv");
            break;
        }

        if(outbuf_used < sizeof(out_buf))
        {
            out_buf[outbuf_used++] = rot13_char(ch);
        }
        if(ch == '\n')
        {
            send(fd,out_buf,outbuf_used,0);
            outbuf_used = 0;
            continue;
        }
    }
}

void 
run(void)
{
    int listener;
    struct sockaddr_in sin;

    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = 0;
    sin.sin_port = htons(40713);

    listener = socket(AF_INET, SOCK_STREAM,0);

#ifdef WIN32
    {
        int one = 1;
        setsockopt(listener,SOL_SOCKET,SO_REUSEADDR,&one,sizeof(one));
    }
#endif

    if(bind(listener,(struct sockaddr*)&sin,sizeof(sin)) < 0)
    {
        perror("bind");
        return;
    }

    if(listen(listener,16) < 0)
    {
         perror("listen");
         return;
    }

    //循环，接收连接
    while(1)
    {
        struct sockaddr_storage ss;
        socklen_t slen = sizeof(ss);
        int fd = accept(listener,(struct sockaddr*)&ss,&slen);
        if(fd < 0)
        {
            perror("accept");
        }
        if(fork() == 0)
        {
            child(fd);
            exit(0);
        }
    }
}


int main()
{
    run();
    return 0;
}


```

### ROT13 客户端
```

/*************************************************************************
	> File Name: rot13_client.c
	> Author: 
	> Mail: 
	> Created Time: Fri 09 Feb 2018 05:27:19 PM CST
 ************************************************************************/

#include <stdio.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <string.h>
#include <stdlib.h>


int main()
{
    int fd;
    char c;
    struct sockaddr_in sin;
    ssize_t n_written, remaining;

    fd = socket(AF_INET,SOCK_STREAM,0);
    if(fd < 0)
    {
        perror("socket");
        return 1;
    }
    /*connect to server*/
    sin.sin_family = AF_INET;
    sin.sin_port = htons(40713);
    sin.sin_addr.s_addr = inet_addr("127.0.0.1");
    
}
```
