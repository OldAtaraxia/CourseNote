*并发的概念是 "逻辑控制流在时间上重叠", 实现手段:*

* *进程*
* *I/O多路复用*
* *线程*

## *进程*

*由进程来调度*

*就是用像`fork`, `exec`, `waitpid`等等*

* *优点: 父子进程共享文件表,但是不共享用户地址空间. 每个进程有独立的地址空间, 不需要考虑覆盖了啥的*
* *缺点: 通信不方便*

> *Unix IPC: 进程间通信技术, 包括管道, 先进先出, 共享内存, 信号量*

## *I/O多路复用*

*后面再说*

## *线程*

线程也是由内核调度, 每个线程有自己的线程上下文(thread context), 包括一个唯一的线程id(tid), 栈, 栈指针, 程序计数器, 通用目的寄存器和条件码. 而所有线程时共享进程的虚拟地址空间的所有内容, 包括代码, 数据, 堆. 共享库和打开的文件

线程切换比进程快得多

*![image-20211119104649376](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211119104649376.png)*

### *Posix线程*

```c
#include <pthread.h>
#include <stdlib.h>
#include <stdio.h>

void *thread(void *vargp);

int main() {
    pthread_t tid;
    pthread_create(&tid, NULL, thread, NULL);
    pthread_join(tid, NULL);
    exit(0);
}

void* thread(void *vargp) {
    printf("Hello, world\n");
    return NULL;
}
```

编译时要加`-lpthread`选项

* *创建线程: `pthread_create`, 用输入变量`arg`在新的线程上下文运行`f`. 返回创建线程的`tid`*

```c
#include <pthread.h>
typedef void* (func) (void *);
int pthread_create(pthread_t *tid, pthread_attr_t *attr, func *f, void *arg);
```

* *得到当前线程的id*

```c
#include <pthread.h>
pthread_t pthread_self(void);
```

* 终止线程. 线程终止的情况:*
	* 线程例程返回时线程会隐式的终止*
	* 显式调用函数.* 
		* 主线程调用`pthread_exit`函数时会先等待所有其它对等限额和国内终止, 然后终止主线程和整个进程. 返回值为`thread_return`*
		* 对等线程调用`exit`会终止该进程与所有与该进程相关的线程*
		* `pthread_cancel(pthread_t tid);`终止当前线程*

```c
#include <pthread.h>

void pthread_exit(void *thread_return);
```

* *等待其它线程终止: `pthread_join`函数会阻塞, 直到线程`tid`终止, 把函数返回的指针赋给`thread_return`指向的位置, 然后**回收已终止线程占用的所有内存资源***

```c
#include <pthread.h>

int pthread_join(pthread_t tid, void **thread_return)
```

#### 分离线程

线程是joinable的或者detached的,

* joinable的线程能被其它线程收回和杀死, 在被其它线程回收之前其内存资源时不释放的
* detached的线程时不能被其它线程回收或杀死的, 它的内存志愿在它终止时由系统自动释放

默认线程时joinable的, 可以通过`int pthread_detach(pthread_t tid)`函数分离`tid`线程. 线程可以通过`pthread_self()`作为参数来分离自己

比如Web Server, 每个连接由一个单独的线程独立处理的话没有必要显式等待每个对等线程终止

#### 初始化线程

用来初始化与线程例程相关的状态.

```c
#include <pthread.h>

pthread_once_t once_control = PTHREAD_ONCE_INIT;

int pthread_once(pthread_once_t* once_control, void (*init_routine)(void));
```

`once_control`应该是全局变量, `init_routine`是个没有参数也不返回的函数

### 多线程程序中的共享变量

示例程序:

```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>

#define N 2
void* thread(void* vargp);

char** ptr; 

int main() {
    int i;
    pthread_t tid;
    char* msgs[N] = {
        "Hello from foo",
        "Hello from bar"
    };

    ptr = msgs;
    for (i = 0; i < N; i++) pthread_create(&tid, NULL, thread, (void *)i);
    pthread_exit(NULL);
}

void* thread(void *vargp) {
    int myid = (int)vargp;
    static int cnt = 0;
    printf("[%d]: %s (cnt = %d)\n", myid, ptr[myid], ++cnt);
    return NULL;
}
```

*![image-20211119150314527](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211119150314527.png)*

#### 线程内存模型

每个线程的独立的线程上下文包括线程ID, 栈, 栈指针, 程序计数器, 条件码, 通用目的寄存器.

线程共享进程上下文的剩余部分, 包括整个用户虚拟地址空间, 读/写数据, 堆, 共享库代码, 数据区域, 打开文件的集合

进程的栈是分别保存在虚拟地址空间中的, 理论上应该是线程访问各自的栈, 但栈对其它线程时不设防的...比如示例程序中`thread`函数通过全局变量`ptr`访问到了主线程栈上的内容(`msg`在主线程的栈上).

#### 将变量映射到内存

多线程C程序变量根据其类型被映射到虚拟内存

* *全局变量: 虚拟内存读/写区含有去阿奴变量的一个实例*
* 本地自动变量: 值定义在函数内部但是没有`static`属性的变量. 每个线程的栈都包含它自己的本地自动变量的实例
* 本地静态变量: 定义在函数内部的`static`变量, 与全局变量一样, 全局只有一个实例, 比如示例程序的`cnt`

> 把由一个以上的线程引用的变量叫做共享变量, 比如示例程序的`cnt`、`msgs`.
>
> `msgs`是本地自动变量, 但它也能被共享

### *用信号量同步线程*

老生常谈的badcnt例子

```c
#include <pthread.h>
#include <stdlib.h>
#include <stdio.h>

void* thread(void *vargp);

volatile long cnt = 0;

int main(int argc, char** argv) {
    long niters;
    pthread_t tid1, tid2;
    if(argc != 2) {
        exit(0);
    }
    niters = atoi(argv[1]);
    pthread_create(&tid1, NULL, thread, &niters);
    pthread_create(&tid2, NULL, thread, &niters);
    pthread_join(tid1, NULL);
    pthread_join(tid2, NULL);

    if(cnt != (2 * niters)) {
        printf("BOOM! cnt=%ld\n", cnt);
    } else {
        printf("OK cnt=%ld\n", cnt);
    }
    exit(0);
}

void* thread(void* vargp) {
    long i, niters = *((long *)vargp);

    for(i = 0; i < niters; i++) cnt++;
    return NULL;
}
```

*![image-20211119154333651](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211119154333651.png)*

简单的`cnt++`翻译成汇编:

```assembly
movq	cnt(%rip), %rdx
addq	%eax
movq	%eax, cnt(%rip)
```

*是分为三个部分(加载`cnt`,更新`cnt`, 储存`cnt`)的.*

> *这里其实我一开始有个疑惑就是明明把`cnt`定义成了`volatile`为什么还会发生并发问题. 写一篇文章放在"小坑"里吧*
>

#### *信号量操作*

信号量s是具有肺腑整数值的全局变量, 只能有P/V操作来改变

* *P(s) : 若`s != 0`则`s--,` 若`s == 0`则挂起当前进程直到s变为0*
* *V(s) : `s++`, 如果有线程阻塞在P操作等待s变成非0那么V操作会重启这些线程中的一个(至于重启哪一个是不可预测的)*

*其中P的减1, V的加1操作都是原子性的*

> *因此正确初始化的信号量是不可能变为负值的, 这个性质叫做信号量不变性*

```c
#include <semaphore.h>

int sem_init(sem_t* sem, int _pshared, unsigned int value); /*将信号量sem初始化为value */
int sem_wait(sem_t* s); /*P(s)*/
int sem_post(sem_t* s); /*V(s)*/
```

#### *使用信号量来实现互斥*

> *基本思想 :  将共享变量与一个信号量s(初始值为1)联系起来, 用P(s)和V(s)操作将响应的临界区联系起来*

*这种信号量一般被叫做"二元信号量"(值总是0/1), 或者直接一点, "互斥锁"(mutex), P操作相当于加锁, V相当于解锁.*

*用`csapp`书上的"进度图"解释*

*![image-20211121032535496](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211121032535496.png)*

*具体实现*

```c
volatile long cnt = 0;
sem_t mutex;

sem_init(&mutex, 0, 1);

for(int i = 0; i < niters; i++) {
	P(&mutex);
    cnt++;
    V(&mutex);
}
```

## *使用信号量来调度共享资源*

#### *生产者消费者问题*

*![image-20211121133452382](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211121133452382.png)*

*解决这个问题的框架:*

*sbuf.h:*

```c
#include  <semaphore.h>

typedef struct{
    int *buf; // buffer array
    int n; // maximum number of buffer slots
    int front; 
    int rear; // 可用buf的开始和结束
    sem_t mutex; // protects assesses to buf
    sem_t slots; // counts available buffer slots
    sem_t items; // counts available items
} sbuf_t;

void sbuf_init(sbuf_t *sp, int n);
void sbuf_deinit(sbuf_t *sp);
void sbuf_insert(sbuf_t *sp, int item);
int sbuf_remove(sbuf_t *sp);
```

*sbuf.c*

```c
#include "sbuf.h"

void sbuf_init(sbuf_t *sp, int n) {
    sp -> buf = calloc(n, sizeof(int));
    sp -> n = n;
    sp -> front = sp -> rear = 0;
    sem_init(*sp -> mutex, 0, 1);
    sem_init(&sp -> slots, 0, n);
    sem _init(&sp -> items, 0, 0);
}

void sbuf_deinit(sbuf_t *sp) {
    free(sp -> buf);
}

void sbuf_insert(sbuf_t *sp, int item) {
    sem_wait(&sp -> slots);
    sem_wait(sp -> mutex);
    sp -> buf[(++sp -> rear) % sp -> n] = item;
    sem_post(sp -> mutex);
    sem_post(&sp -> items);
}

int sbuf_remove(sbuf_t *sp) {
    int item;
    sem_wait(&sp -> items);
    sem_wait(sp -> mutex);
    item = sp -> buf[(++sp -> front) % sp -> n];
    sem_post(sp -> mutex);
    sem_post(&sp -> slots);
    return item;
}

```

> *web server可以借用生产者/消费者问题, server没街道一个请求就往缓冲区加一个slot(一个socket描述符), 消费者则是从solt里拿socket然后凯新线程去处理.*

#### *读者-写者问题*

*一组并发的线程要访问一格共享对象, 例如一个主存中的数据结构/磁盘上的数据库. 有的线程制度对象, 而其他的线程只修改对象. 修改对象的线程叫做写者, 制度对象的线程叫做读者. 写者必须拥有对对象的督战队访问1,读者则可以和无限多个其他的读者共享对象. 一般来说, 可能有无限多个并发的读者和写者*

*第一类读者写者问题: 读者优先, 不要让读者等待, 除非已经把使用对象的权限给了写者*

*第二类: 写者优先, 一旦一个写者准备好开始写, 他就会进口可能快地完成它的写操作*

*第一类读者问题的解决方案:*

```c
#include <stdio.h>
#include <stdlib.h>
#include <semaphore.h>

int readcnt = 0;
sem_t mutex, w; // both initially = 1

void reader(void) {
    while(1) {
        sem_wait(&mutex);
        readcnt++;
        if (readcnt == 1)
            sem_wait(&w);
        sem_post(&mutex);

        printf("Reader\n");
        sleep(1);

        sem_wait(&mutex);
        readcnt--;
        if (readcnt == 0)
            sem_post(&w);
        sem_post(&mutex);
    }
    
}

void writer(void) {
    while(1) {
        sem_wait(&w);
        printf("Writer\n");
        sleep(1);
        sem_post(&w);
    }
}
```

*信号量`w`控制对共享变量临界区的访问, `mutex`控制对`readcnt`的访问, `readcnt`表示当前在临界区的`reader`数量. 只有第一个进入临界区的reader对w加锁, 当reader进入和林凯临界区时如果有其它reader还在临界区中, 那么它会忽略互斥锁`w`*

*而每个writer进入临界区的时候都对`w`加锁.*

> *这种解决方案可能造成饥饿问题*

#### *Java monitor*

*Java monitor是用信号量实现的, 提供了对信号量互斥和调度能力的更高级别的抽象.* 

#### *使用线程提高并行性*

*大多数现代机器具有多核处理器, 并发程序在这样的机器上运行的更快, 因为os kernel在多个核上并行地调度这些并发线程.*

*但是并行程序真的是线程数越多性能越好吗 ? 书上给出了并行计算求和的函数, 具体每个线程求和时都是P(mutex) -> 全局变量 ++ -> v(mutex), 统计性能结果:*

*![image-20211121163135702](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211121163135702.png)*

*线程数越多性能越差的原因: 相对于内存更新操作的开销, 同步操作(P和V)代价太大.* 

> *同步开销巨大, 要尽可能避免, 如果无可避免, 必须要用尽可能多的有用计算弥补这个开销*

*所以后面的改进是每个进程在它的私有变量中计算它自己的部分和,  最后主线程再把所有线程的和累加起来*

*![image-20211121163530054](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211121163530054.png)*

> *写并行程序相当棘手, 对代码很小的改动可能会对性能有很大的影响*

## *其它并发问题*

> *感觉不是很重要... 不过解释了很多之前见过的概念*

### *线程安全*

*一个函数被称为线程安全的(thread-safe), 当且仅当**被多个线程并发调用时会一直产生正确的结果***

*常见的线程不安全函数类*

* *不保护共享变量的函数. 挺常见的...一般的处理方式是给共享变量 加上PV操作*
* *保持跨越多个调用的状态的函数, 比如`rand`随机函数. 这种函数除了重写没有别的用户态的让它线程安全的方法*

```c
unsigned rand(void) {
    next_seed = next_seed * 1103515245 + 12345;
    return (unsigned)(next_seed/65536) % 32768;
}
```

* *返回指向静态变量的指针的函数. 有的函数把计算结果放在一个static变量中, 然后返回一个指向这个变量的指针. 可能会导致**一个线程的结果被另一个线程给覆盖***
	* *可以每次调用函数时加锁, 定义一个包装函数*

```c
char* ctime_ts(const time_t *timep, char* privatep) {
    char* sharedp;

    sem_wait(&mutex);
    sharep = ctime(timep);
    strcpy(privatep, sharep);
    sem_post(&mutex);
    return privatep;
}
```

* *调用线程不安全函数的函数. 但不是所有这种函数都是线程不安全的, 比如对第一种/第三种函数加锁*

### *可重入性*

*可重入函数是一类重要的**线程安全函数**, 特点: 被多个线程调用时不会引用任何共享变量.* 

*![image-20211121165739292](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211121165739292.png)*

*例子: 改写之后的`rand`(通过参数得到next变量)*

```c
unsigned rand(unsigned int*nextp) {
    *nextp = *nextp * 1103515245 + 12345;
    return (unsigned)(*nextp/65536) % 32768;
}
```

*如果所有函数参数都是值传递的, 并且所有数据引用都是本地自动栈变量(不访问静态/全局变量), 那么函数是**显式可重入**的*

### *竞争*

> *一个程序的正确性依赖于一个线程要在另一个线程到达y点之前到达它的控制流中的x点时就会发生竞争(race)*

*比如这个示例:*

```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>

#define N 4

void* thread(void* vargp);

int main() {
    pthread_t tid[N];
    int i;

    for (i = 0; i < N; i++) {
        pthread_create(&tid[i], NULL, thread, &i);
    }
    for (i = 0; i < N; i++) {
        pthread_join(tid[i], NULL);
    }
    exit(0);
}

void* thread(void *vargp) {
    int myid = *((int *)vargp);
    printf("Hello from thread %d\n", myid);
    return NULL;
}
```

*![image-20211122012051468](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122012051468.png)*

*可能的原因是线程在执行23行之前执行了`i++`*

### *死锁*

*使用二元信号量实现互斥时, 为了避免死锁可以给定所有互斥操作一个顺序, 如果每个线程都是以一种顺序获得互斥锁并以相反的顺序释放, 那么这个程序就是无死锁的.*
