* OS是硬件和应用之间的软件层, 对下管理硬件, 对象构筑应用生态

  ## OS为应用提供的服务

  OS为应用提供计算资源的抽象:

  * CPU: 进程/线程, 数量不受物理CPU的限制
  * 内存: 虚拟内存,大小不受物理内存的限制
  * IO设备: 把各种设备统一抽象为文件, 提供统一接口

  > 就是3 easy pieces中提到的虚拟化, 把有限的离散的资源抽象为无限的连续的

  OS为应用提供线程间的同步与隔离

  * 应用可以实现自己的同步原语
  * OS提供了更高效的同步原语(互斥锁, 条件变量, 信号量,读写锁等等)

  > 大致就是oestp的第二部分并发

  OS为应用提供了进程间的通信

  * 应用之间可以利用网络进行进程间通信
  * OS提供了更高效的本地通信机制,如pipe

  kernel就是为运行的程序提供服务的一种特殊的程序. 运行中的程序(进程)需要使用kernel的服务时会触发"系统调用", 内核执行相应的服务然后返回.

  ---

  **剩下的部分介绍的是对xv6服务的概述**

  ---

  

  ## OS与应用的交互: 系统调用

  应用调用系统与调用普通函数一样

  比如程序调用`printf`, `printf`中调用了标准库libc中的`write`, libc准备好参数之后执行`svc`指令(一条ARM指令)使得控制流从用户地址空间下陷到内核地址空间,os内核通过系统调用表调用对应函数.

  POSIX接口: Protable Operating System Interface foe Unix, 一套操作系统接口规范, 通常通过libc(C library)来实现

  

  xv6只有很少几个系统调用:

  | **系统调用**                            | **描述**                                                    |
  | --------------------------------------- | ----------------------------------------------------------- |
  | `int fork()`                            | 创建一个进程，返回子进程的PID                               |
  | `int exit(int status)`                  | 终止当前进程，并将状态报告给wait()函数。无返回              |
  | `int wait(int *status)`                 | 等待一个子进程退出; 将退出状态存入*status; 返回子进程PID。  |
  | `int kill(int pid)`                     | 终止对应PID的进程，返回0，或返回-1表示错误                  |
  | `int getpid()`                          | 返回当前进程的PID                                           |
  | `int sleep(int n)`                      | 暂停n个时钟节拍                                             |
  | `int exec(char *file, char *argv[])`    | 加载一个文件并使用参数执行它; 只有在出错时才返回            |
  | `char *sbrk(int n)`                     | 按n 字节增长进程的内存。返回新内存的开始                    |
  | `int open(char *file, int flags)`       | 打开一个文件；flags表示read/write；返回一个fd（文件描述符） |
  | `int write(int fd, char *buf, int n)`   | 从buf 写n 个字节到文件描述符fd; 返回n                       |
  | `int read(int fd, char *buf, int n)`    | 将n 个字节读入buf；返回读取的字节数；如果文件结束，返回0    |
  | `int close(int fd)`                     | 释放打开的文件fd                                            |
  | `int dup(int fd)`                       | 返回一个新的文件描述符，指向与fd 相同的文件                 |
  | `int pipe(int p[])`                     | 创建一个管道，把write/read文件描述符放在p[0]和p[1]中        |
  | `int chdir(char *dir)`                  | 改变当前的工作目录                                          |
  | `int mkdir(char *dir)`                  | 创建一个新目录                                              |
  | `int mknod(char *file, int, int)`       | 创建一个设备文件                                            |
  | `int fstat(int fd, struct stat *st)`    | 将打开文件fd的信息放入*st                                   |
  | `int stat(char *file, struct stat *st)` | 将指定名称的文件信息放入*st                                 |
  | `int link(char *file1, char *file2)`    | 为文件file1创建另一个名称(file2)                            |
  | `int unlink(char *file)`                | 删除一个文件                                                |

  ## 进程和内存

  每个xv6进程拥有自己的用户空间内存(存放指令、数据、栈等), 以及每个进程的内核空间状态(?

  相关的系统调用:

  * `fork`: 会拷贝当前进程的内存, 并创建一个新的进程. 在父进程中，`fork`的返回值是这个子进程的PID，在子进程中，返回值是0
  * `exit(int status)`: 0表示正常状态退出, 1表示非正常状态退出
  * `wait(int *status)`: 等待子进程退出，返回子进程PID，子进程的退出状态存储到`int *status`这个地址中。如果调用者没有子进程，`wait`将返回-1
  * `int exec(char* file, char* argv[])`: 加载一个可执行文件, 指定其参数, 加载到当前进程中并替换当前进程的内存(!), 执行新加载的指令. 因此要求文件必须是ELF格式(Linux的可执行文件格式, 里面会规定哪一部分是指定, 哪一部分是数据, 哪里是指定开始的地方). 执行错误会返回-1
    * exec会保存当前文件描述符表单, 比如之前的0, 1, 2
    * `argv`的最后一个元素需要是null
    * 通常来说exec系统调用不会返回, 因为exec会完全替换当前进程的内存


  xv6的shell就是使用以上4个系统调用来执行用户程序的. 在`user/sh.c`中. 

  ```c
  // Read and run input commands.
  while(getcmd(buf, sizeof(buf)) >= 0){
      if(fork1() == 0)
          runcmd(parsecmd(buf));
      wait(0);
  }
  ```

  子进程执行`runcmd`之后会调用`exec`来执行解析到的命令, 如输入的`echo`

  > xv6隐式地分配大多数用户地址空间: `fork`为子进程分配内存, `exec`为可执行文件的执行分配内存, 进程在运行时需要更多内存的话可以调用`sbrk(n)`来增长n bytes的data内存空间. `sbrk`返回新内存空间的位置.

  ## IO与文件描述符

  > 文件描述符接口将文件、管道和设备之间的差异抽象出来，使它们看起来都像字节流。

  文件描述符: 一个整数, 表示**进程可以读取或写入的一个由内核管理的对象**, 比如一个文件、目录、设备、管道等.

  每个进程有一个从0开始的文件描述符空间, 其中0是标准输入, 1是标准输出, 2是标准错误. 

  * `read(int fd, char *buf, int n)`: 文件描述符,读取数据的指针,想读取的最大长度, 返回成功读取数据的数量, 文件描述符不存在时返回-1
  * `write(int fd, char *buf, int n)`: 类似, 把`buf`中的n个字节写入`fd.

  * `close(int fd)`: 把打开的文件`fd`释放, 使得该文件描述符可以被后面的`open`, `pipe`等其它系统调用使用. 新分配的文件描述符总是当前进程中编号最小的未使用描述符。

  可以通过close实现I/O重定向

  ```java
  int main() {
      int pid;
  
      pid = fork();
      if(pid == 0) {
          // 把文件描述符1变成output.txt
          close(1);
          open("output.txt", O_WRONLY | O_CREATE); // open会返回未使用的最小文件描述符号
  
          char* argv[] = { "echo", "this", "is", "echo", 0}; // 0标记了数组的结尾...
     		exec("echo", argv); // echo根本不知道发生了什么
      	printf("exec failed!\n");
      	exit(1);
      } else {
          wait((int *) 0); 
      }
      exit(0);
  }
  ```

  * `open(filename, mode)`: 打开文件. 其中的mode: 定义在`kernel/fcntl.h`中了.

  | **宏定义** | **功能说明**             |
  | ---------- | ------------------------ |
  | `O_RDONLY` | 只读                     |
  | `O_WRONLY` | 只写                     |
  | `O_RDWR`   | 可读可写                 |
  | `O_CREATE` | 如果文件不存在则创建文件 |
  | `O_TRUNC`  | 将文件截断为零长度       |

  * `dup(int fd)`: 复制一个新的`fd`指向的I/O对象, 返回新的fd, 而且两个I/O对象的offset相同, 与open一样会使用未使用的最小文件描述符号

  ```c
  fd = dup(1);
  write(1, "hello", 6);
  write(fd, "world\n", 6);
  // 文件里是hello world
  ```

  由`fork`或`dup`得到的多个文件描述符时, 文件的偏移量是共享的

  > 每个文件描述符有一个offset，`read`会从这个offset开始读取内容，读完n个字节之后将这个offset后移n个字节，下一个`read`将从新的offset开始读取字节。`write`也有类似的offset

  ## 管道

  管道本质上就是一对文件描述符, 一个用来读一个用来写, 把数据写入管道的一端后可以使得数据可以从挂到的另一端读取

  * `pipe(int p[])`. `p[0]`为读取的文件描述符, `p[1]`为用来写的

  ```c
  int p[2];
  
  char* argv[2];
  argv[0] = "wc";
  argv[1] = 0;
  
  pipe(p); // 父进程和子进程都有自己的指向管道的文件描述符
  
  if(fork() == 0) {
      // 重定向p[0]到0(首先把0给close掉, 之后dup的时候就会分配给0), 让/bin/wc去pipe里读
      close(0);
      dup(p[0]);
      close(p[0]);
      close(p[1]);
      exec("bin/wc", argv); // exec从它的标准输入中读取的时候就是从管道来读取
  } else {
      // 关闭读端(0), 然后向写端(1)中写
      close(p[0]);
      write(p[1], "hello, world\n", 12);
      close(p[1]);
  }
  ```

  >  没有可用的数据时对管道读端的读会一直阻塞, 直到有数据了或者其它绑定在这个管道写端口的描述符都关闭了(这时候返回0, 就像是一份文件读到了最后). 
  >
  >  因此子进程中执行`wc`之前要**把自己的写端口给关闭**, 防止永远看不到EOF

  xv6中shell对管道的实现: 

  * 在子进程中关闭1(stdout), 将其重定向到写端并执行`runcmd`来执行`pcmd->left`

  * 在另一个子进程中关闭0(stdin), 将其定向到读端并执行`runcmd`来执行`pcmd->right`

  ```c
  case PIPE:
      pcmd = (struct pipecmd*)cmd;
      if(pipe(p) < 0)
        panic("pipe");
      if(fork1() == 0){
        close(1);
        dup(p[1]);
        close(p[0]);
        close(p[1]);
        runcmd(pcmd->left);
      }
      if(fork1() == 0){
        close(0);
        dup(p[0]);
        close(p[0]);
        close(p[1]);
        runcmd(pcmd->right);
      }
      close(p[0]);
      close(p[1]);
      wait(0);
      wait(0);
      break;
  
  ```

  ## 文件系统

  * 文件: 就是一个简单的字节数组
  * 目录: 包含指向文件和其它目录的引用, 也是一种特殊的文件. 目录形成一颗树, 根目录即`/`

  

  相关系统调用:

  * `chdir("/a")`: 把当前目录切换到/a
  * `mkdir("/dir")` 创建新目录
  * `mknod("/console", 1, 1)` 创建一个设备文件, 记录主设备号和辅设备号, 它们唯一地标识了一个内核设备. 进程打开设备文件时kernel会使用内核设备而非文件系统来实现read和write
  * `fstat`: 获取一个文件描述符指向的文件的信息, 定义在`kernel/stat.h`中

  ```c
  #define T_DIR     1   // Directory
  #define T_FILE    2   // File
  #define T_DEVICE  3   // Device
  
  struct stat {
    int dev;     // File system's disk device
    uint ino;    // Inode 编号
    short type;  // 文件类型
    short nlink; // 指向文件的链接数
    uint64 size; // 文件字节数
  };
  ```

  * `link`: 为文件创建链接. 创建一个即叫a又叫b的新文件:

  ```c
  open("a", O_CREATE|O_WRONGLY);
  link("a", "b");
  ```

  * `unlink("a")`: 去掉链接a, 如果a是文件的最后一个链接就删除文件

  Unix以用户级程序的形式提供了可以从shell调用的实用的与文件系统相关的程序, 如`mkdir`, `ln`, `rm`等. 但`cd`是内置在`shell`中的, 因为要更改shell本身的工作目录, 不能像其它命令那样由shell来fork一个子进程出来运行`cd`, 否则`cd`只会更改子进程的工作目录, 父进程(`shell`)的工作目录不会改变.

  