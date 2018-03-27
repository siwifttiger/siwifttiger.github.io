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

```sequence

participant  CLIENT
participant  SERVER

note left of CLIENT : SYN_SENT
note right of SERVER: LISTEN
CLIENT->SERVER:SYN_SENT(SYN=1,seq=client_isn)
SERVER->CLIENT:SYN_RECV(SYN=1,seq=server_isn)  \n  ack=client_isn+1
note right of SERVER:SYN_RECV
CLIENT->SERVER:ESTABLISH(SYN=0,seq=client_isn+1) \n ack=server_isn+1
note left of CLIENT: ESTABLISH
note right of SERVER:ESTABLISH

```

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

### 简单的ROT13 客户端(译者加)
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

ssize_t sendn(int fd, void* buf, ssize_t len)
{   ssize_t remaining = len;
    ssize_t recv = len;
    char* bufp = (char*)buf;
    while(remaining)
    {
        recv = send(fd,(char*)bufp,remaining,0);
        if(recv == 0)
        {
            continue;
        }
        else if(recv < 0)
        {
            perror("send");
            return -1;
        }
        remaining -= recv;
        bufp += recv;
    }
    return len;
}

ssize_t readn(int fd, void* buf, ssize_t len)
{
    ssize_t nleft = len;
    ssize_t nread;
    char *bufp = (char*)buf;
    while(nleft)
    {
        if((nread = recv(fd,bufp,nleft,0)) < 0)
        {
            perror("recv");
            return -1;
        }
        else if(nread == 0)
        {
            return len - nleft;
        }
        nleft -= nread;
        bufp += nread;

    }
    return len;
}


int main()
{
    int fd;
    char buf[1024] = {0};
    char recvbuf[1024] = {0};
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
   
    if(connect(fd,(struct sockaddr *)&sin,sizeof(sin)) < 0)
    {
        perror("connect");
        close(fd);
        return 1;
    }

    while(fgets(buf,sizeof(buf),stdin) != NULL)
    {
        remaining = strlen(buf);
        n_written = sendn(fd,buf,remaining);
        if(n_written < 0)
        {
            perror("send");
            return 1;
        }
        if((readn(fd,recvbuf,remaining)) < 0)
        {
            perror("recv");
            return 1;
        }
        fwrite(recvbuf,1,remaining,stdout);
        memset(buf,0,sizeof(buf));
        memset(recvbuf,0,sizeof(recvbuf));
    }
    
}

    
```

到此为止，对于一次性处理多个连接的问题，我们有了完美的解决方案了吗？我们可以停止写这本书去干其它的事了吗？显然不是，首先，进程的创建(哪怕是线程的创建)开销在一些平台上可能都十分昂贵，在实际应用中，你可能会使用一个线程池而非创建新的进程。但是更重要的是，线程的规模不能如你所愿的扩张。当你的程序需要同时处理成千上万个连接的时候，处理成千上万个线程的效率肯定不必每个CPU只处理几个线程。  
如果多线程都处理不了多连接的话，那怎么办呢？在Unix环境中，你可以设置的你的socket为noblocking(非阻塞)，Unix系统调用如下  
> fcntl(fd,F_SETFL,O_NONBLOCK);

其中fd是socket的文件描述符，一旦你设置了fd是noblocking的，那么以后无论何时，当你的网络调用(译者加：例如send这种函数)作用在这个fd上时，函数要么立刻完成操作返回，要么返回一个特殊的错误码(error code)来表明"我现在无法取得任何进展，请再试一次"。所以我们可能会很天真地将上面处理两个socket连接的例子改写成下面这样：
### 糟糕的例子：忙轮询所有的socket套接字
```
/*This will work, but the performance will be unforgivably bad*/
int i, n;
char buf[1024];
for (i = 0; i < n_sockets, ++i)
    fcntl(fd[i],F_SETFL,O_NONBLOCK);

while(i_still_want_to_read())
{
    for(i = 0; i < n_sockets; ++i)
    {
        n = recv(fd[i],buf,sizeof(buf),0);
        if(n == 0)
        {
            handle_close(fd[i]);
        }
        else if(n < 0)
        {
            if(errno = EAGAIN) 
                ;                    //The kernel didn't have any data for us to read.
            else
                handle_error(fd[i],errno);
        }
        else
        {
            handle_input(fd[i],buf,n);
        }
}
```

现在我们使用非阻塞socket处理多连接的问题，上面的程序会起作用，但是效果微乎其微。性能会非常的差，原因有两点，第一：当任何一个连接上都没有数据的时候，该循环将不会停止，直至消耗完你的CPU时间。第二，当你使用这种方法处理一两个以上的连接的时候，你需要为每个连接都调用内核(译者加：recv)，不管这个连接上有没有数据。所以我们需要一种方法去告诉内核“必须等到其中一个socket准备好数据给我了才通知我，而且告诉我是哪个socket”，最早的解决办法包括现在也在使用的方法就是select()。select()调用需要三组fd集合(用bit数组实现):一个用来读，一个用来写，一个用来处理异常。该函数会一直等待直到某个集合中的socket准备就绪，然后修改包含这些可用socket的集合(告知我们可用)。  
下面依然是上面那个例子，不过这次我们使用select  
```
/*If you only have a couple dozen fds, this version */
fd_set readset;
int i, n;
char buf[1024];

while(i_still_want_to_read())
{
    int maxfd = -1;
    FD_ZERO(&readset);

    /*Add all of the interesting fds to readset*/
    for(i = 0; i < n_sockets; ++i)
    {
        if(fd[i] > maxfd) maxfd = fd[i];
        FD_SET(fd[i],&readset);
    }
    /*Wait util one or more fds are ready to read*/
    select(maxfd + 1, &readset,NULL,NULL,NULL);

    /*Process all of the fds that are still set in readset*/
    for(i = 0; i < n_sockets; ++i)
    {
        if(FD_ISSET(fd[i],&readset))   //if fd[i] ready for read 
        {
            n = recv(fd[i],buf,sizeof(buf),0);
            if(n == 0)
            {
                handle_close(fd[i]);
            }
            else if(n < 0)
            {
                if(errno == EAGAIN)
                    ;               /*if kernel didn't have any data for us to read*/
                else
                {
                    handle_error(fd[i],errno);
                }
            }
            else
            {
                handle_input(fd[i],buf,n);
            }
        }
    }
}

```
下面是基于select()重新实现的ROT13服务器  
### 例子：基于select()的ROT13服务器  
```
/*************************************************************************
	> File Name: select_rot13_server.c
	> Author: 
	> Mail: 
	> Created Time: Mon 12 Feb 2018 10:56:46 AM CST
 ************************************************************************/

#include <stdio.h>
/*For sockaddr_in*/
#include <netinet/in.h>
/*For socket functions*/
#include <sys/socket.h>
/*For fcntl*/
#include <fcntl.h>
/*For select*/
#include <sys/select.h>

#include <assert.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <errno.h>

#define MAX_LINE 16384

char
rot13_char(char c)
{
    /*We don't want to use isalpha here; setting the locale would change
     * which characters are considered alphabetical.*/
    if((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
    {
        return c+13;
    }
    else if((c >= 'n' && c <='z') || (c >= 'N' && c <= 'Z'))
    {
        return c-13;
    }
    else
        return c;
}

struct fd_state
{
    char buf[MAX_LINE];
    size_t buf_used;

    int writing;
    size_t n_written;
    size_t write_upto;
};

struct fd_state* alloc_fd_state()
{
    struct fd_state *state = malloc(sizeof(struct fd_state));
    if(!state)
        return NULL;
    state->buf_used = state->writing = state->n_written = state->write_upto = 0;
    return state;
}

void free_fd_state(struct fd_state *state)
{
    free(state);
}

void
make_nonblocking(int fd)
{
    fcntl(fd,F_SETFL,O_NONBLOCK);
}

int
do_read(int fd, struct fd_state* state)
{
    char buf[1024];
    ssize_t result;
    int i;
    while(1)
    {
        result = recv(fd,buf,sizeof(buf),0);
        if(result <= 0)
        {
            break;
        }
        for(i = 0; i < result; ++i)
        {
            if(state->buf_used < sizeof(state->buf))
            {
                 state->buf[state->buf_used++] = rot13_char(buf[i]);
            }
            if(buf[i] == '\n')
            {
                state->writing = 1;
                state->write_upto = state->buf_used;
            }
        }
    }
    if(result == 0)
    {
        return 1;
    }else if(result < 0)
    {
        if(errno == EAGAIN)
            return 0;
        return -1;
    }

    return 0;
}


int 
do_write(int fd, struct fd_state *state)
{
    while(state->n_written < state->write_upto)
    {
        ssize_t result = send(fd, state->buf + state->n_written,state->write_upto-state->n_written,0);
        if(result < 0)
        {
            if(errno == EAGAIN)
                return 0;
            return -1;
        }
        assert(result != 0);
        state->n_written += result;
    }

    if(state->n_written == state->buf_used)
    {
        state->write_upto = state->buf_used = state->n_written = 0;
    }
    state->writing = 0;
    return 0;
}

void
run(void)
{
    int listener;
    struct fd_state *state[FD_SETSIZE];
    struct sockaddr_in sin;
    int i,maxfd;
    fd_set readset,writeset,exset;

    sin.sin_family = AF_INET;
    sin.sin_port = htons(40713);
    sin.sin_addr.s_addr = 0;

    for(i = 0; i < FD_SETSIZE; ++i)
    {
        state[i] = NULL;
    }
    listener = socket(AF_INET,SOCK_STREAM,0);
    make_nonblocking(listener);

#ifdef WIN32
    {
        int one = 1;
        setsockopt(listener,SOL_SOCKET,SO_REUSEADDR,&one,sizeof(one));
    }
#endif

    if(bind(listener,(struct sockaddr*)&sin,sizeof(sin)) < 0)
    {
        perror("bind");
        return ;
    }

    if(listen(listener,16) < 0)
    {
        perror("listen");
        return;
    }
    
    FD_ZERO(&readset);
    FD_ZERO(&writeset);
    FD_ZERO(&exset);

    FD_SET(listener,&readset);

    for(i = 0; i < FD_SETSIZE; ++i)
    {
        if(state[i])
        {
            if(i > maxfd)
            {
                maxfd = i;
            }
            FD_SET(i,&readset);
            if(state[i]->writing)
            {
                FD_SET(i,&writeset);
            }
        }
    }

    if(select(maxfd+1,&readset,&writeset,&exset,NULL) < 0)
    {
        perror("select");
        return;
    }
    
    if (FD_ISSET(listener,&readset))
    {
        struct sockaddr_storage ss;
        socklen_t slen = sizeof(ss);
        int fd = accept(listener,(struct sockaddr*)&ss,&slen);
        if(fd < 0)
        {
            perror("accept");
        }
        else if(fd > FD_SETSIZE)
        {
            close(fd);
        }
        else{
            make_nonblocking(fd);
            state[fd] = alloc_fd_state();
            assert(state[fd]);
        }
    }

    for(i = 0 ; i < maxfd + 1; ++i)
    {
        int r = 0;
        if(i == listener)
        {
            continue;
        }
        if(FD_ISSET(i,&readset))
        {
            r = do_read(i,state[i]);
        }
        if(r == 0 && FD_ISSET(i,&writeset))
        {
            r = do_write(i,state[i]);
        }
        if(r)
        {
            free_fd_state(state[i]);
            state[i] = NULL;
            close(i);
        }
    }
}

int main()
{
    setvbuf(stdout,NULL,_IONBF,0);
    run();
    return 0;
}
```

但是这不算完, 因为生成和读取select()的二进制位花费的时间，与你需要提供给select的最大fd成正比。当sockets的数量非常大的时候，select的调用花费将会增加的非常恐怖。  
不同的操作系统提供了不同的函数替代select的功能。其中包含了poll(),epoll(),kqueue(),evports,和/dev/poll。所有这些给出的函数都比select()性能更好。除了poll()之外，所有的函数对于增加一个socket，删除一个socket，通知一个socket已经准备好IO的时间花费都是O(1).
遗憾的是，这些高效的接口并没有一个统一的标准。Linux有epoll，BSD系列(包括Darwin)有kqueue，Solaris有evports和/dev/poll...这些操作系统并不包含其他操作系统提供的接口。所以如果你想写一个可移植的高性能异步应用，你需要一个抽象的方法将所有这些接口封装好，并且提供其中最高效的一个。
这些就是Libevent API最底层的工作。它为多样的selec()替代方案提供了一个统一的接口，并且在运行它的电脑上使用了可用的最高效的版本。
下面给出另一个版本的异步ROT13服务器。这次，它使用Libevent 2 替代select()函数。注意，现在fd_sets已经消失了：我们使用一个结构体event_base来关联和分离事件，这可能由select(),poll(),epoll(),kqueue()等系列函数实现的。

### 例子：一个用Libevent实现的低级的ROT13服务器

```C
//for sockaddr_in
#include <netinet/in.h>

/*for socket functions*/
#include <sys/socket.h>
/*for fcntl*/
#include <event2/event.h>

#include <assert.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>

#define MAX_LINE  16384

void do_read(evutil_socket_t fd, short events, void *arg);
void do_write(evutil_socket_t fd, short events, void *arg);


char rot13_char(char c)
{
    if((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
        return c + 13;
    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
        return c - 13;
    else    
        return c;
}

struct fd_state
{
    char buffer[MAX_LINE + 1];
    size_t buffer_used;       

    size_t n_written;           
    size_t write_upto;

    struct event *read_event;
    struct event *write_event;
};

struct fd_state*
alloc_fd_state(struct  event_base *base, evutil_socket_t fd)
{
    struct fd_state *state = malloc(sizeof(struct fd_state));
    if(!state)
    {
        return NULL;
    }
    state->read_event = event_new(base,fd,EV_READ|EV_PERSIST,do_read,state);
    if(!state->read_event)
    {

        free(state);
        return NULL;
    }
    state->write_event = event_new(base,fd,EV_WRITE|EV_PERSIST,do_write,state);
    if(!state->write_event)
    {
        event_free(state->read_event);
        free(state);
        return NULL;
    }
    state->buffer_used = state->n_written = state->write_upto = 0;

    assert(state->write_event);
    return state;
}

void
free_fd_state(struct fd_state *state)
{
    event_free(state->read_event);
    event_free(state->write_event);
    free(state);
}

void
do_read(evutil_socket_t fd, short events, void *arg)
{
    struct fd_state *state = arg;
    char buf[1024];
    int i;
    ssize_t result;
    while(1)
    {
        assert(state->write_event);
        result = recv(fd,buf,sizeof(buf),0);
        if(result < 0)
        {
            break;
        }

        for(i = 0; i < result; ++i)
        {
            if(state->buffer_used < sizeof(state->buffer))
            {
                state->buffer[state->buffer_used++] = rot13_char(buf[i]);
            }
            if(buf[i] == '\n')
            {
                //断言，捕捉代码异常，如果条件为假，则终止程序
                assert(state->write_event);
                event_add(state->write_event,NULL);
                state->write_upto = state->buffer_used;
            }
        }
    }

    if(result == 0)
        free_fd_state(state);
    else if(result < 0)
    {
        if(errno == EAGAIN)
            return;
        perror("recv");
        free_fd_state(state);
    }
}

void do_write(evutil_socket_t fd, short events, void *arg)
{
    struct fd_state *state =  arg;

    while(state->n_written < state->write_upto)
    {
        ssize_t result = send(fd, state->buffer + state->n_written,
        state->write_upto - state->n_written,0);
        if(result < 0)
        {
            if(errno == EAGAIN)
                return;
            free_fd_state(state);
            return;
        }
        assert(result != 0);
        state->n_written  += result;
    }
    if(state->n_written == state->buffer_used)
        state->n_written = state->buffer_used = state->write_upto = 1;
    event_del(state->write_event);
}


void do_accept(evutil_socket_t listener,short event, void *arg)
{
    struct event_base *base = arg;
    struct sockaddr_storage ss;
    socklen_t s_len = sizeof(ss);
    int fd = accept(listener,(struct sockaddr*)&ss,&s_len);
    if(fd < 0) /*XXX eagain*/
    {
        perror("accept");
    }else if(fd > FD_SETSIZE)
    {
        close(fd);
    }
    else{
        struct fd_state *state;
        evutil_make_socket_nonblocking(fd);
        state = alloc_fd_state(base,fd);
        assert(state);
        assert(state->write_event);
        event_add(state->read_event,NULL);
    }
}

void run(void)
{
    evutil_socket_t listener;
    struct sockaddr_in sin;
    struct evnet_base *base;
    struct evnet      *listener_event;

    base = event_base_new();
    if(!base)
    {
        return;   /*err*/
    }

    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = 0;
    sin.sin_port = htons(40713);

    listener = socket(AF_INET,SOCK_STREAM,0);
    evutil_make_socket_nonblocking(listener);
#ifndef WIN32
    int one = 1;
    setsockopt(listener,SOL_SOCKET,SO_REUSEADDR,&one,sizeof(one));
#endif
    if(bind(listener,(struct sockaddr*)&sin,sizeof(sin)) < 0)
    {
        perror("bind");
        return;
    }

    if(listen(listener,16) < 0)
    {
        perror("listen");
        return ;
    }

    listener_event = event_new(base,listener,EV_READ|EV_PERSIST,do_accept,(void*)base);
    /*XXX checkt it*/
    event_add(listener_event,NULL);
    event_base_dispatch(base);
}




int main(int argc,char **argv)
{
    setvbuf(stdout,NULL,_IONBF,0);
    run();
    return 0;
}



```

(代码中需要注意的地方：sockets的定义我们使用evutil_socket_t类型来代替int类型，我们调用evutil_make_socket_nonblocking替代fcntl(O_NONBLOCK)来设置socket为非阻塞模式,这些改变使得我们的代码能够更好地兼容Win32网络API)


### 怎样才能更方便（在Windows下怎么工作？）

你可能也注意到了，当我们的代码更高效的同时，也更复杂了.回溯到我们还在使用forking方式的时候， 我们不需要为每个连接管理buffer,对于每个进程，我们有独自的栈分配的缓存。我们也无需显式地追踪那个socket可写或可读。这些都是自动隐含在我们代码中的。我们也不需要一个结构体去追踪每个操作有多少完成了，我们仅仅使用循环和栈变量就够了。
除此之外，如果你对windows网络编程非常有经验。你就会意识到，使用上面的例子，Libevent并不能得到最优的性能。在Windows上，你使用快速的IO异步端口并不是通过select()这样的接口，而是通过使用IOCP(IO完成端口)API，.与其他快速的网络API不同,当socket为操作准备好的时候,程序在即将执行操作之前IOCP是不会通知的.相反,程序会告诉Windows网络栈开始网络操作,IOCP会通知程序操作完成.

幸运的是,LibEvent2的bufferevents接口解决了这两个问题:首先使程序更加简单,其次提供了在Windows和Linux上高效运行的接口.  
下面是最终版本的ROT13服务器，使用了bufferevents API

### 例子：使用Libevent的更简单的ROT13服务器

```C
//for sockaddr_in
#include <netinet/in.h>

/*for socket functions*/
#include <sys/socket.h>
/*for fcntl*/
#include <event2/event.h>
#include <event2/buffer.h>
#include <event2/bufferevent.h>

#include <assert.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>

#define MAXLINE 16384 

void do_read(evutil_socket_t fd, short events, void *arg);
void do_write(evutil_socket_t fd, short events, void *arg);

char rot13_char(char c)
{
    if((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
        return c + 13;
    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
        return c - 13;
    else
        return c;
}

void readcb(struct bufferevent *bev, void *ctx)
{
    struct evbuffer *input, *output;
    char *line;
    size_t n;
    int i;
    input = bufferevent_get_input(bev);
    output = bufferevent_get_output(bev);

    while((line = evbuffer_readln(input, &n, EVBUFFER_EOL_LF)))
    {
        for(i = 0; i < n ; ++i)
        {
            line[i] = rot13_char(line[i]);
        }
        evbuffer_add(output,line,n);
        evbuffer_add(output,"\n",1);
        free(line);
    }
    if(evbuffer_get_length(input) >= MAXLINE)
    {
        /*Too long; just process what there is and go on so that the buffer
         *doesn't grow infinitely long*/
        char buf[1024];
        while(evbuffer_get_length(input))
        {
            int n = evbuffer_remove(input,buf,sizeof(buf));
            for(i = 0; i < n; ++i)
            {
                buf[i] = rot13_char(buf[i]);
            }
            evbuffer_add(output,buf,n);
        }
        evbuffer_add(output,'\n',1);
    }
}

void errorcb(struct bufferevent *bev, short error, void *ctx)
{
    bufferevent_free(bev);
}

void do_accept(evutil_socket_t listener, short event, void *arg)
{
        struct event_base *base = arg;
        struct sockaddr_storage ss;
        socklen_t slen = sizeof(ss);
        int fd = accept(listener,(struct sockaddr*)&ss,&slen);
        if(fd < 0)
        {
            perror("accept");
        }else if (fd > FD_SETSIZE)
        {
            close(fd);
        }else 
        {
            struct bufferevent *bev;
            evutil_make_socket_nonblocking(fd);
            bev = bufferevent_socket_new(base,fd,BEV_OPT_CLOSE_ON_FREE);
            bufferevent_setcb(bev,readcb,NULL,errorcb,NULL);
            bufferevent_setwatermark(bev,EV_READ,0,MAXLINE);
            bufferevent_enable(bev,EV_READ|EV_WRITE);
        }
}

void run(void)
{
    evutil_socket_t listener;
    struct sockaddr_in  sin;
    struct event_base *base;
    struct event *listener_event;

    base = event_base_new();
    if(!base)
        return;
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = 0;
    sin.sin_port = htons(40713);

    listener = socket(AF_INET,SOCK_STREAM,0);
    evutil_make_socket_nonblocking(listener);
#ifndef WIN32
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
    listener_event = event_new(base,listener,EV_READ|EV_PERSIST,do_accept,(void*)base);
    /*XXX check it*/
    event_add(listener_event,NULL);

    event_base_dispatch(base);
}

int main(int argc, char **argv)
{
    setvbuf(stdout,NULL,_IONBF,0);
    run();
    return 0;
}
``` 
### 这一切效率如何,当真？
在这里写了一个高效的代码.libevent页面上的基准是过时了

这些文件是版权(c)2009 - 2012年由尼克·马修森和可用创意下议院Attribution-Noncommercial-Share都许可,3.0版.未来版本

可能会用更少的限制性许可协议.

此外,这些文档的源代码示例都是基于"3条款"或"修改的"BSD许可,请参考license_bsd文件全部条款.

本文档的最新版本,请参阅[Libevent官网](http://libevent.org/)

本文档对应的最新版本代码,请安装git然后执行【git clone git://github.com/nmathewson/libevent-book.git】