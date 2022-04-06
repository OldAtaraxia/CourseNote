# Network Programming

* Networks
* Global IP internet

* Sockets Interface

前面的内容其实都挺简单的, 后面则是讲里Socket编程

# IP地址有关的库函数

TCP/IP统一规定是大端字节顺序, 但大部分主机都是小端的, 所以Unix提供函数来进行网络字节顺序(大端)和主机字节顺序(一般是小端)的转化.

```c
#include <arpa/inet.h>

// 返回按照网络字节顺序的值
uint32_t htonl(uint32_t hostlong);
uint16_t htons(uint16_t hostshort);

// 返回按照主机字节顺序的值
uint32_t ntohl(uint32_t netlong);
uint32_t ntohs(uint16_t netshort);
```

IP地址一般用淀粉十进制法, 比如地址0x8002c2f2的点分十进制表示为128.2.194.242

还提供了`inet_pton`和`inet_ntop`来实现IP地址和淀粉十进制传的转换

```c
#include <arpa/inet.h>

// 若成功返回1, 若src为非法点分十进制地址则为0, 出错则为-1
int inet_pton(AP_INET, const char* src, void* dst);

// 若成功则返回指向点分十进制表示字符串的指针, 并把字符串的最多size个字节赋值给dst, 
// 出错则为NULL
const char* inet_ntop(AF_INET, const void* src, char* dst, socklen_t size);

```

> 使用例子

```c
#include <stdio.h>
#include <stdlib.h>
#include <arpa/inet.h>

int main() {
    const char* s = "128.2.194.242";
    int dst = 0;
    int res = inet_pton(AF_INET, s, &dst);
    printf("%d 0x%X\n", res, dst); // 1 0xF2C20280
}
```

![image-20211117133559262](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211117133559262.png)



# Socket编程

> 看了那个geektime的智障课程

Unix一切皆文件, Socket也是文件

网络编程中客户端和服务器工作的核心逻辑

![image-20211115192406410](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211115192406410.png)

Socket编程常用的头文件

> sys/types.h：数据类型定义
>
> sys/socket.h：提供socket函数及数据结构
>
> netinet/in.h：定义数据结构sockaddr_in
>
> arpa/inet.h：提供IP地址转换函数
>
> netdb.h：提供设置及获取域名的函数
>
> sys/ioctl.h：提供对I/O控制的函数
>
> sys/poll.h：提供socket等待测试机制的函数

## Socket地址格式

Socket地址大概可以理解成(地址: 端口), 一个连接由它两端的套接字地址唯一确定, 这对套接字地址称为套接字对, 记作`(cliaddr: cliport, servaddr:servport)`

### 通用地址结构

套接字的通用地址结构 ,可以适用于多种总地址族:

```C
typedef unsigned short int sa_family_t;

struct sockaddr {
    sa_family_t sa_family;
    char sa_data[14];
}
```

第一个字段是地址族, 表示使用什么方式对地址进行解释和保存, 常见的:

* AF_LOCAL: 表示本地地址, 一般用于本地socket通信(?)
* AF_INET: IPv4地址
* AF_INET6: IPv6地址

`AF`指`Address Family`, 这种值用来初始化Socket地址. 还有`PF`开头的宏, 表示`Protocol Family`, 那种纸用来初始化socket. 他们本质上是一样的, 在`<sys/socket.h>`中定义的

![image-20211115193928593](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211115193928593.png)

### IPv4套接字格式地址

```c
typedef uint32_t in_addr_t;
struct in_addr {
    in_addr_t s_addr;
}

struct sockaddr_in {
    sa_family_t sin_family; // AF_INET
    in_port_t sin_port; // 端口号 16-bit(65536)
    struct in_addr sin_addr; // 32-bit
    
    unsigned char sin_zero[8]; // 占位符, 没有用处
}
```

ipv4地址的`sin_family`是`AF_INET`, 端口到是16bit, 即65536, 点口号最多是65535, 分为保留端口(比如http80, ftp21...)与自由使用的端口

### IPv6套接字格式地址

```c
struct sockaddr_in6 {
    sa_family_t sin6_family; //16-bit
    in_port_t sin6_port; // 传输端口号
    uint32_t sin6_flowinfo; // IPv6流控信息 32-bit
    struct in6_addr sin6_addr; // IPv6地址 128-bit
    uint32_t sin6_scope_id; // IPv6域ID 32-bit
};
```

地址族是`AF_INET6`, 地址是128位.

### 本地套接字格式

```c
struct sockaddr_un {
    unsigned short sun_family; // AF_LOCAL
    char sun_path[108]; // socket文件路径名
    // 比如/var/a.sock, var/lib/a.sock
};
```

### 比较

![image-20211115195904983](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211115195904983.png)

## 套接字建立连接

### 服务端

#### 创建套接字

用这个函数 : 

```c
int socket(int domain, int type, int protocol);
```

* `domain`: 指`PF_INET`、`PF_INET6`, `PF_LOCAL`等, 表示什么样的套接字
* `type`: 可用的值
	* `SOCK_STREAM`: 表示字节流, 对应TCP
	* `SOCK_DGRAM`: 表示数据报, 对应UDP
	* `SOCK_RAW`: 表示原始套接字(?)

* `protocol`现在已经废弃, 写成`0`就行

#### bind

想使用套接字的话需要先用`bind`函数**把套接字和套接字地址绑定到一起**

```c
bind(int fd, sockaddr* addr, socklen_t len);
```

第二个参数是通用socket地址, 第三个参数是`len(第二个参数)`. 第二个参数也可以传入IPv4、IPv6或本地套接字格式, **函数会根据传入结构的前两个字节(`sa_family_t`)和第三个参数`len`字段判断你传的啥, 该怎么解析.**

> 设计这个函数的时候(1982)C语言还不支持`void *`

所以使用时需要把IPv4, IPv6或者本地套接字格式转换为通用套接字.

```c
struct sockaddr_in name;
bind(sock, (struct sockaddr *) &name, sizeof(name));
```

其中ip地址显然应该是本机地址, 相当于告诉kernel只对目标IP是本地IP的IP包进行处理, 但显然`ip`不能硬编码到代码里. 这是可以用`INADDR_ANY`完成统配地址的设置(IPv6采用`IN6ADDR_ANY`)

```c
struct sockaddr_in name;
name.sin_addr.s_addr = htonl (INADDR_ANY);
```

> htonl: 将主机数转换成无符号长整型的网络字节顺序。本函数将一个32位数从主机字节顺序转换成网络字节顺序。

#### 实例: 初始化IPv4 TCP套接字

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <sys/socket.h>
#include <netinet/in.h>

int make_socket (uint16_t port) {
    int sock;
    struct sockaddr_in name;
    /* 创建字节流类型的IPv4 socket */
    sock = socket (PF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        perror ("socket");
        exit(EXIT_FAILURE);
    }

    /* 绑定到port和ip */
    name.sin_family = AF_INET;
    name.sin_port = htons(port); // 指定地址
    name.sin_addr.s_addr = htonl(INADDR_ANY); // 统配地址
    if (bind(sock, (struct sockaddr *) &name, sizeof(name)) < 0) {
        perror("bind");
        exit(EXIT_FAILURE);
    }
    return sock;
}
```

#### listen

初始化创建的套接字是一个"主动"套接字, 其目的是之后主动发起请求.(通过调用`connect`函数). `listen`函数则是把主动套接字转换为"被动"套接字, 调速kernel本套接字是等到用户请求的.

```c
int listen (int socketfd, int backlog);
```

第一个参数即套接字描述符, 第二个参数backlog是"未完成连接队列的大小", 决定了1可以接受的并发数目. 但Linux不允许对这个参数进行改变.

#### accept

客户端的连接请求到达, 连接建立后操作系统内核和应用程序之间的桥梁.

```c
int accept(int listensockfd, struct sockaddr *cliaddr, socklen_t *addrlen);
```

第一个参数就是listen套接字, 是前面通过`bind`, `listen`得到的套接字. `cliaddr`是通过指针方式获取的客户端的地址, `addrlen`是其大小.

函数的返回值是一个全新的套接字, 可以称为"已连接套接字". 之后服务器就可以使用这个已连接套接字和客户进行通信处理.完成服务后会关闭已连接套接字从而完成TCP连接的释放.

> 监听套接字只有一个, 连接套接字则是每个连接对应一个

### 客户端

首先建立套接字, 这一点与服务端是一样的, 然后通过`connect`发起请求

```c
int connect(int sockfd, const struct socladdr *servaddr, socklen_t addrlen);
```

* 第一个参数: 连接套接字, 通过`socket`函数创建
* 第二三个参数: 指向套接字地址结构的指针和结构大小

> 调用connect之前不需要用bind, socket执行体为你的程序自动选择一个未被占用的端口，并通知你的程序数据什么时候打开端口。

对于TCP套接字, connect函数会触发TCP的三次握手过程, 在连接成功/出错时返回. 可能出错的情况:

* 三次握手无法建立, 客户端的SYN包没有响应
* 客户端收到了RST回答, 常见于客户端发送连接请求时端口写错
* 客户端的SYN包引起了"destination unreachable".

### TCP三次握手

![image-20211115223036739](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211115223036739.png)

这里假设了所有的函数的调用都是阻塞式的, 非阻塞式的下次再说

具体的过程:

* 客户端协议栈向服务器发送SYN包, 并告诉服务器端当前发送序列号`j`, 客户端进入SYN_SENT状态
* 服务端协议栈收到这个包之后和客户端进行ACK应答, 应答值为`j+1`, 表示对SYN包`j`的确认. 同时服务器发送SYN包告诉客户端发送序列号为`k`. 服务端进入SYNC_RCVD状态
* 客户端协议栈收到ACK后, 应用程序从`connect`调用返回, 表示客户端到服务端的单向连接建立成功, 客户端的状态为ESTABLISHED, 客户端协议栈会对服务端的SYN包进行应答, 应答数据为k+1
* 应答包到达服务器端后, 服务器端协议栈使得accept阻塞调用返回, 这是服务器端到客户端的单向连接也建立成功, 服务器端进入ESTABLISHED状态

## 使用套接字进行读写

建立连接的根本目的就是为了数据的收发.

### 发送数据

常用函数: `write`, `send`, `sendmsg`

```c
ssize_t write(int socketfd, const void* buffer, size_t size);
ssize_t send(int socketfd, const void* buffer, size_t size, int flags);
ssize_t sendmsg(int sockfd, const struct msghdr* msg, int flags);
```

* `write`就是文件写函数, 只不过把文件描述符换成socket文件描述符
* `send`可以指定选项, 发送带外数据(一种基于TCP协议的紧急数据, 用于客户端 - 服务器在特定场景下的紧急处理)
* `sendmsg`可以指定多重缓冲区传输数据, 以结构体`msghdr`发送数据

### 发送缓冲区

os内核会在tcp连接建立后创建发送缓冲区, 之后会一直从里面取数据, 包装秤TCP的MSS包, 以及IP的MTU包, 最后走数据链路层把数据发送出去.

调用`write`函数是回答数据从应用程序中拷贝到os内核的发送缓冲区中, 若数据多于缓冲区则函数阻塞. 

![image-20211116123218031](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211116123218031.png)

> 发送成功近近表示**数据被拷贝到发送缓冲区中**, 不意味着连接对端已经收到所有的数据

### 读取数据

最多读取size个字节

```c
ssize_t read(int sockerfd, void* buffer, size_t size);
```

返回值为0表示EOF, 返回值为-1表示出错

一个确确实实读取size个字节的函数(跟csapp第十章那个一样)

```c
ssize_t readn(int fd, void* vptr, size_t size) {
    size_t nleft;
    ssize_t nread;
    char* ptr;

    ptr = vptr;
    nleft = size;

    while(nleft > 0) {
        if((nread = read(fd, ptr, nleft)) < 0) {
            if(errno == EINTR) {
                nread = 0; // 再次调用nread
            } else {
                return -1;
            }
        } else if(nread == 0) {
            break; // EOF
        }
        nleft -= nread;
        ptr += nread;
    }
    return (n - nleft);
}
```

# UDP编程

UDP是无上下文的...

创建UDP Socket要用SOCK_DrEAM类型

UDP编程主要用到`recvfrom`和`sendto`

![image-20211117142228034](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211117142228034.png)

服务端创建socket之后绑定到本地端口, 调用`revcfrom`等待客户端的报文发送; 客户端创建套接字之后调用`sendto`函数网目标地址和端口发送UDP报文.

```c
#include <sys/socket.h>

ssize_t recvfrom(int sockfd, void* buff, size_t nbytes, int flags, struct sockaddr* from, socklen_t* addrlen);

ssize_t sendto(int sockfd, const void* buff, size_t nbytes, int flags, const struct sockaddr* to, socklen_t* addrlen);
```

* `recvfrom`: `sockfd`即本地的套接字描述符, `buff`指向本地缓存, `nbytes`表示最大接受数据字节, `flags`是和I/O相关的参数, `from`和`addrlen`是用来存发送方的信息和长度的(不需药自己传). 返回值为实际接收的字节数

* `sendto`: 前三个参数是socket描述符, 要发送的东西, 发送字节数, 第四个`flags`也还是I/O参数, 后面`to`和`addrlen`则是接收方信息. 返回值为实际接受的字节数.

> udp的`recvfrom`连不上会阻塞, 而tcp的`connect`会直接返回.
>
> 为了防止一直阻塞可以考虑添加超时处理.

# 主机和服务的转换

> 欢迎回到csapp

## getaddrinfo

Linux提供一些函数实现二进制套接字地址结构和主机名、主机地址、服务名和端口号的字符串表示之间的相互转化.

```c
#include <sys.types.h>
#include <sys/socket.h>
#include <netdb.h>

int getaddrinfo(const char* host, const char* service, 
                const struct addrinfo *hints, 
                struct addrinfo **result);
```

给定`host`(可以是域名, 可以是点分十进制地址)和`service`(可以是端口号, 可以直接写服务名(比如HTTP,然后会找80端口))(强啊), `result`会"变成"指向`addrinfo`结构的链表, 每个结构指向一个套接字地址结构.

> 具体result里有啥会在`hints`里说

而`addrinfo`:

```c
typedef struct addrinfo {
    int        　　　　    　　ai_flags;//指示在getaddrinfo函数中使用的选项的标志。
    int        　　　　   　　 ai_family;
    int        　　　　    　　ai_socktype;
    int        　　　　　 　　ai_protocol;
    size_t       　　　　 　　ai_addrlen;
    char       　　　　 　　*ai_canonname;
    struct sockaddr    　　*ai_addr;
    struct addrinfo     　　*ai_next;//指向链表中下一个结构的指针。
    //此参数在链接列表的最后一个addrinfo结构中设置为NULL。
} ADDRINFOA, *PADDRINFOA;
```

![image-20211117145903326](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211117145903326.png)

> `ai_addr`字段指向一个套接字地址结构, `ai_addrlen`字段是地址结构的大小, `ai_next`字段指向列表中下一个`addrinfo`结构

可选参数`hints`也是`addrinfo`, 相当于一个控制参数. 可以设置`ai_family`、`ai_socktype`, `ai_protocol`和`ai_flag`字段, 其它字段都应该是0

* `ai_family`: 默认`getaddrinfo`可以返回IPv4和IPv6地址, 设置为`AF_INET`可以限制为IPv4地址, 设置为`AF_INET6`就是限制为IPv6地址
* 对于host关联的每个地址, `getaddrinfo`函数默认返回三个`addrinfo`结构, 其`ai_socktype`字段不同. 分别是连接、数据报、原始套接字. 把hint的`ai_socktype`设置成`SOCK_STREAM`就可以限制为每个地址只有一个`addrinfo`结构, 其对应连接(TCP)套接字.

## getnameinfo

把一个套接字地址结构转换成响应的主机和服务名字符串.

```c
int getnameinfo(const struct sockaddr* sa, socklen_t salen,
                char* host, size_t hostlen,
                char* service, size_t servlen, int flags);
```

`sa`指向大小为`salen`字节的套接字地址结构, `host`指向大小为`hostlen`字节的缓冲区, `server`指向大小为`servlen`字节的缓冲区. 

`getnameinfo`函数把套接字地址结构`sa`转换成对应的主机和服务名字符串, 并把它们复制到host和service缓冲区.

如果不想要主机名, 可以把host设置为NULL, 把hostlen设置为0, 服务字段也是这样. 不过两个必须设置其一.

## freeaddrinfo

清理列表

```c
freeaddrinfo(list);
```

## 示例程序

> todo, 在csapp第659页

# 套接字接口的辅助函数

`csapp.h`实现的一些包装了上述函数的高级辅助函数

## open_clientfd

客户端通过`open_clientfd`建立与服务器的连接

````c
int open_clientfd(char* hostname, char *port);
````

返回一个socket描述符, 可以用Unix I/O函数做输入输出.

实现:

```c
int open_clientfd(char* hostname, char* port) {
    int clientfd;
    struct addrinfo hints, *listp, *p;

    /* 得到potential server addresses的列表 */
    memset(&hints, 0, sizeof(struct addrinfo));
    hints.ai_socktype = SOCK_STREAM;
    hint.ai_flags = AI_ANUMERICSErV | AI_ADDRCONFIG;
    getaddrinfo(hostname, port, &hints, &listp);

    for(p = listp; p; p = p -> ai_next) {
        if((clientfd = socket(p ->ai_family, p -> ai_socktype, p -> ai_protocol)) < 0) {
            continue;
        }

        if(connect(clientfd, p -> ai_addr, p -> ai_addrlen) != -1) {
            // connect successfully
            break;
        }
        close(clientfd);
    }
    freeaddrinfo(listp);
    if(!p) {
        // connect failed
        return -1;
    } else {
        return clientfd;
    }
}
```

## open_listenfd

服务器调用一个监听描述符, 准备好接收连接请求

```c
int open_listenfd(char *port);
```

实现: 使用来`setsocket`函数来配置服务器, 使得服务器能够被终止,重启和立即开始接受连接请求.

```c
int open_listenfd(char* port) {
    struct addrinfo hints, *listp, *p;
    int listenfd, optval = 1;

    memset(&hints, 0, sizeof(struct addrinfo));
    hints.ad_socktype = SOCK_STREAM;
    hints.ai_flags = AI_PASSIVE | AI_ADDRCONFIG;
    hints.ai_flags |= AI_NUMERICHOST;
    getaddrinfo(NULL, port, &hints, &listp);

    for(p = listp, p; o = p -> ai_next) {
        if((listenfd = socket(p ->ai_family, p -> ai_socktype, p ->ai_protocol)) < 0) {
            continue; /*socket failed, try the next*/
        }
        setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, (const void*)&optval, sizeof(int));
        if(bind(listenfd, p -> ai_addr, p -> ai_addrlen) == 0) break; /* successful */
        close(listenfd); /*bind failed, try the next*/
    }
    freeaddrinfo(listp);
    if(!p) return -1;
    if(listen(listenfd, LISTENQ) < 0) {
        close(listenfd);
        return -1;
    }
    return listenfd;
}
```



# Web服务器

