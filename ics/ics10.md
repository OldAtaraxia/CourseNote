# 第10章 系统级I/O

从第十章开始是第三部分" 程序间的交互和通信" 

I/O就是在主存和I/O设备(磁盘驱动器, 终端,网络)之间复制数据的过程, Linux内核提供了系统级Unix I/O函数, 其它语言通过这些来实现了较高级别的I/O函数, 比如`printf`, `scanf`

文件IO/标准IO的区别: 文件I/O不包含缓存, 

## 10.1 Unix I/O

### Overview

- A linux file is a sequence of m bytes
- All I/O devices are represented as files, even the kernel
- 这种把文件和设备做映射的方法允许kernel实现简单的Unix I/O接口:
	- `open`, `close`
	- `read`, `write`
	- `lseek`: 设置current file position, which indicates next offset into file to read or write

![image-20211122023413512](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023413512.png)

- 文件类型: 
	- regular files(怎么翻译啊...), 具体分为文本文件和二进制文件, 文本文件就是由ASCII或Unicode字符组成的文件
		- 文本文件是一些text line terminated by new line char '\n'
		- EOL: 在Linux和Mac OS上是'\n'(0xa), 在Windows和网络协议中是`\r\n'
		- 当然两种文件对kernel来说没啥区别就是了
	- 目录: 目录其实是an array of links, each link maps a filename to a file
		- 每个目录都之至少有两条, `..`(指向指向其父目录)和`.` (指向它自身)
		- 每个进程都有一个当前工作目录(current working directory, cwd)

![image-20211122023426528](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023426528.png)

### 开关文件

```c
int fd;
int retval;

if((fd = open("etc/hosts", O_RDONLY) < 0)) {
    // fd == -1 indicates that an error occured
    perror("open");
    exit(1);
}


if((retval = close(fd)) < 0) {
    perror("close");
    exit(1);
}
```

`open`打开文件, 返回file descriptor, 具体值指定为当前进程没有用过的最小的数

每个进程被shell启动会呕都会默认打开三个文件:

- 0: stdin
- 1: stdout
- 2: stderr

关闭文件: 关闭已经关闭的文件is a recipt for disaster. Always check return codes, even for seemingly benign functions

### 读写文件

```c
int fd;
char buf[512];
ssize_t nbytes;

if((fd = open("etc/hosts", O_RDONLY) < 0)) {
    perror("open");
    exit(1);
}

if((nbytes = read(fd, buf, sizeof(buf))) < 0) {
    perror("read");
    exit(1);
}
```

参数:文件描述符, 读到哪里, 读多少

返回类型是`ssize_t`, 是有符号数, `size_t`是无符号数(啊, 我的json库...)

返回值nbytes表示读了多少字符, 如果 < 0说明发生了错误, 一般来说因为各种原因nbytes都会少于第三个参数(sizeof(buf))的

一般来说少于的原因:

- read时遇到了EOF
- Reading text lines from a terminal
- Reading and writing network sockets

Writing to disk files永远不会产生少于的情况

### Simple unix I/O example

Copying files to stdout, BUFSIZE bytes at a time

```c
#define BUFSIZE 64

int main(int argc, char* argv[]) {
    char c[BUFSIZE];
    int infd = STDIN_FILENO;
    if (argc == 2) {
        infd = open(argv[1], O_RDONLY, 0);
    }
    while(read(infd, &c, 1) != 0) {
        write(STDOUT_FILENO, &c, 1);
    }
    exit(0);
}
```

## 10.2 Metadata, sharing, and redirection



### metadata

kernel会保存文件的metadata, 可以通过`stst`和`fstat`得到

```c
int stat(const char* filename, struct stat* buf);
int fstat(int fd, struct stat* buf);
```

![image-20211122023454934](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023454934.png)

### How the unix kernel represents open files

内核用三个相关的数据结构来表示打开的文件:

- 描述符表: 由进程打开的文件描述符索引, 每个描述符指向文件表中的一个表项
- 文件表: 相当于打开文件的集合, 由所有进程共享, 每一项由当前文件位置, 引用计数(指向当前标枪的描述符表项数), 指向v-node表的指针. 关闭描述符会减少响应的文件表表项的引用计数. 引用计数到0就会关闭它
- v-node表: 每个表项包括文件的元数据, 由所有进程共享

![image-20211122023503959](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023503959.png)

### 文件共享

简单来说就是Two distinct descriptors sharing the same disk file through two distinct open file table entries, 多个描述符通过不同的文件表表项引用相同的文件. 

要点在于每个表舒服有会保存自己的文件位置, 可以分别从文件的不同位置获取数据.

![image-20211122023513651](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023513651.png)

如果是多个进程, 子进程会继承父进程的打开的文件

Before fork:

![image-20211122023521713](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023521713.png)

After fork

![image-20211122023528901](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023528901.png)

### IO重定向

常用于管道通信

```
ls > foo.txt
```

具体执行

```c
#include <unistd.h>

int dup2(int oldfd, int newfd);
```

具体做的事是把oldfd的描述符表复制给newfd

![image-20211122023540590](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023540590.png)

### IO重定向例子

```c
int main(int argc, char* argv[]) {
    int fd1, fd2, fd3;
    char c1, c2, c3;
    char* fname = argv[1];
    fd1 = open(fname, O_RDONLY, 0);
    fd2 = open(fname, O_RDONLY, 0);
    fd3 = open(fname, O_RDONLY, 0);
    
    dup2(fd1, fd2);
    read(fd2, &c1, 1);
    read(fd2, &c2, 1);
    read(fd3, &c3, 1);
    printf("c1 = %c, c2 = %c, c3 = %c\n", c1, c2, c3);
    return 0;
}
```

## Standard I/O

C语言定义的标准io库, 相当于Unix I/O的较高级别的替代, 包括常用的fopen, fclose, fread, fwrite, fgets, fputs, fscanf, fprintf

### Standard I/O Streams

标准I/O把文件视为`streams`, 相当于文件描述符和内存中的缓冲区的抽象概念

标准C程序启动时就自带三个流, stdin, stdout和stderr

### Buffer I/O

加buffer的原因: Unix I/O calls expensive, read和write是Unix Kernel调用, > 10000clock cycles

所以采用`buffer read`: 每次用Unix read读一块字节, 然后用一个输入函数每次从buffer里读一个byte进来. 当buffer空了之后再重新去读buffer

![image-20211122023554316](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023554316.png)

![image-20211122023603440](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023603440.png)



输出`\n`或者调用`fflush`会刷新缓冲区.

```c
int main(int argc, char* argv[]) {
    char buf[MLINE];
    FILE *infile = stdin;
    if(argc == 2) {
        infile = fopen(argv[1], "r");
        if(!infile) exit(1);
    }
    while(fgets(buf, MLINE, infile) != NULL) {
        fprintf(stdout, buf);
    }
    exit(0);
}
```

## RIO package

cmu 15-213写的一个特定的包

![image-20211122023614231](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023614231.png)

读入的数据不足("Best practice is to always allow for short counts")的情况:

- 遇到EOF
- 从terminal读一行文本
- 读写sockets

Best practice is to always for short counts

RIO提供了两种不同种类的函数:

- 不带缓冲, 用于读取二进制数据, 比如`rio_readn`和`rio_writen`

- 带缓冲, 用于文本数据和二进制数据, 比如`rio_readlineb`和`rio_readnb`

	- 带buffer的是线程安全的, 并且可以在同一描述符上任意交错使用。

### 不带缓冲的RIO Input和Output

```c
ssize_t rio_readn(int fd, void *usrbuf, size_t n);
ssize_t rio_writen(int fd, void *userbuf, size_t n);
```

rio_readn只有在遇到EOF时才会返回short count

对rio_readn和rio_writen的调用可以在同一描述符上任意交错使用。

实现:

```c
ssize_t rio_readn(int fd, void* usrbuf, size_t n) {
    size_t nleft = n;
    ssize_t nread;
    char* bufp = usrbuf;

    while(nleft > 0) {
        if((nread = read(fd, bufp, nleft)) < 0) {
            if(errno == EINTR) {
                /* Interrupted by sig handler -> call read() again */
                nread = 0;
            }else {
                return -1;
            }
        }else if(nread == 0) {
            break;
        }
        nleft -= nread;
        bufp += nread;
    }
    return (n - nleft);
}
```

### 带缓冲的函数

```c
void rio_readinitb(rio_t* rp, int fd);

ssize_t rio_readlineb(rio_t* rp, void* usrbuf, size_t maxlen);
ssize_t rio_readnb(rio_t* rp, void* usrbuf, size_t n);
```

`rio_readlineb`从文件`fd`中读取最多`maxlen`的文件并存在usrbuf里

```c
typedef struct {
    int rio_fd;
    int rio_cnt;
    char* rio_bufptr;
    char rio_buf[RIO_BUFSIZE];
} rio_t;
```

停止的条件:

- 读了`maxlen`的bytes
- 遇到EOF
- 遇到了`\n`



## Standard I/O的利弊

利弊用的词是Pros and Cons

利:

- Buffer让read和write的调用降低了, 提高了效率
- 自动处理了Short counts

弊端:

- 没有提供访问元数据的能力
- 不是线程安全的, 也不适合信号处理函数
- 标准IO不适合网络sockets读写

## 选择合适的I/O函数

尽可能的用高层次的I/O函数. 在信号处理函数里用Unix I/O, 因为它是线程安全的. 其他情况就用Standard I/O就够了.

对于二进制文件不要使用`Text-oriented I/O`, 比如`fgets`, `scanf`, 他们会被EOF打断

![image-20211122023656084](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023656084.png)