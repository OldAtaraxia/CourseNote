# 第9章 虚拟内存

## 9.1 物理和虚拟寻址

- 物理寻址: 也就是拿着物理地址去寻址. 一些简单的嵌入式的系统啥的是这样的。

![image-20211122022949868](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122022949868.png)

- 虚拟寻址: CPU生成一个虚拟地址来访问主存, 这个寻你地址首先要通过"内存管理单元"(MMU)转换成适当的物理地址。现代系统都是用的这个

![image-20211122022957503](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122022957503.png)



MMU通过主存里的一个查询表来动态翻译虚拟内存, 这个表的内容是操作系统管理的.

### 虚拟内存的意义:

- 把主存看做了一种磁盘的缓存, 在组村中只保存活跃的区域.对主存的利用更高效
- 为每个进程提供了一致的数据空间, 简化了内存管理, 提供了一种"抽象", Each process gets the same uniform linear address space
- 保护了每个进程的地址空间不被其它进程破坏,一个进程不能访问其它进程的memory, 用户程序不能访问kernel的数据和代码



## 9.2 虚拟内存作为缓存的工具

虚拟内存可以抽象成**存放在磁盘上的** N个连许的字节大小的单元(毕竟是字节寻址)组成的数组. 而其中的内容被缓存在**物理主存.**

![image-20211122023006578](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023006578.png)

每字节有一个唯一的虚拟地址, 数组的内容被缓存在主存中.VM系统把虚拟内存划分成称为"虚拟页"(VP)的大小固定的块来解决这个问题. 物理内存也被分成物理页(PP), 大小与虚拟页一样. 有的虚拟页会被缓存到物理内存中.

- Any vp can be placed in any pp
- Write-back rather than write through

### 9.2.1 页表

A **page table** is an array of page table entries(called PTES) that maps virtual pages to physical pages. 属于kernel data.

![image-20211122023020756](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023020756.png)



负责把虚拟页映射到物理页, os负责维护.  页表是由页表条目组成的数组, 每个条目由一个有效位和n位地址字段组成, 有效位设置了的情况下地址字段即表示对应物理页的起始地址,  

CPU想读的东西通过页表找到了在主存中就叫命中(hit),

![image-20211122023030949](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023030949.png)



不命中就叫缺页(page fault). 缺页是一种异常, 会调用内核的缺页异常处理程序

![image-20211122023042287](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023042287.png)

Handling Page Fault:

- Page miss  causes page fault
- Page fault handler selects a victim to be evicted
- Offending instruction is restarted: page hit!

![image-20211122023051306](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023051306.png)

可以看到是返回到了引起异常的指令(movl)

### 9.2.2 抖动(thrashing)

根据局部性原理, 一般来说程序会趋向于在一个小的page集合上工作. 一般叫做工作集(working set). 然后如果工作集的大小超出了物理内存的大小, 页面会不断地换进换出, 这就叫抖动

## 9.3 虚拟内存作为内存管理的工具

Key idea: each process has its own virtual address space.

- 简化链接: 虚拟内存营造了每个进程独占内存的假象, 所以each program has similiar virtual address space, 比如Linux64位地址空间下程序代码总是从0x400000开始, 方便了链接器的实现

![image-20211122023103970](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023103970.png)

- 简化加载: loader不用去从磁盘到内存复制数据(具体是`.data`和`.text`), 虚拟内存系统给你做
- 简化共享: 便于管理进程共享, 因为每个进程都有自己私有的代码数据堆和栈区, 这时候os可以把相应的虚拟内存映射到不连续的物理界面防止干扰. 需要共享的时候, 比如`printf`, os把不同进程中适当的虚拟页面映射到相同的物理页面, where是需要共享的代码的副本. 这样就不需要每个进程包含单独的内核和C标准库的脚本了(这一部分可能等学了链接那一章就能搞清楚了吧)

![image-20211122023121010](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023121010.png)

- 简化内存分配: 比如`malloc`, os可以把分配的空间映射到不同的物理页面.

## 9.4 虚拟内存作为内存保护的工具

在PTE(页表的一行/一个元素)上添加控制位, MMU check these bits on each access.

![image-20211122023132837](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023132837.png)

SUP表示在内核态下才能访问这一页, READ和WRITE控制用户态下读写权限.

如果违反了这些许可条例CPU就会触发异常, 即"段错误".

## 9.5 地址翻译

记虚拟地址空间的大小为N=2^n, 物理地址空间的大小为M=2^m, Page size P= 2^p

### 9.5.1 MMU映射

`n`位的虚拟地址被分为p位的虚拟页面偏移量(页的大小为2^p字节)(vpo, offert)和(n-p)的虚拟页号(Vpn, number). MMU用vpn选择适当的pte. 把物理页号和vpo连起来就是物理地址

![image-20211122023143627](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023143627.png)

![image-20211122023159177](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023159177.png)

 可以看到有两次访存, 第一次通过PTE地址的得到物理地址, 第二次通过物理地址访问主存.

### 9.5.2 结合高速缓存和虚拟内存

上面的过程中MMU需要两次访问内存, 这不好, 应该在中间加缓存, 两次访存都要缓存

![image-20211122023210482](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023210482.png)

### 9.5.3 利用TLB加速地址翻译

Tlb, "translation lookaside buffer", mmu里的缓存, 缓存页表的项(pte),希望能省去一次访存.

假设TLB有T=2^t项, 需要访问TLB的时候可以把虚拟地址看做p位VPO, t位TLB index和n-p-t为TLB tag, 通过TLB index来选择集合, 通过TLB tab来matches tag of line within set. 感觉类似第六章讲的cache机制...

TLB hit eliminates a cache/memory access. 可以减少一次访存, 而且命中率极高.

![image-20211122023221208](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023221208.png)

### 9.5.4 多级页表

页表很大

![image-20211122023232484](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023232484.png)



多级页表可以让它小一点......一级页表的每个pte映射的是二级页表中的一个片.没有分配的片可以为空, 大大节省了空间

![image-20211122023243050](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023243050.png)

这时候对地址的处理: 地址分为vpn1, vpn2, ... Vpnk, 分别用来访问各级页表

![image-20211122023259548](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023259548.png)

这样要访问k个pte,乍一见很慢, 这时候tlb的作用就出来了, tlb能把不同层次上页表的pte缓存起来, 最后不会慢太多.

## 9.6 内存映射

内存映射: 将一个虚拟内存区域与一个磁盘上的对象关联起来, 就是把文件来映射进内存

可以映射的部分:

- linux文件系统中的普通文件. 文件被分成页大小的片, 按需进行页面调度. 通过文件内容初始化虚拟页面的初始内容
- 匿名文件: 匿名文件是内核创建的, 全是`0`. 如果映射到这种文件, CPU才不会傻乎乎的从磁盘里读数据传输数据呢, 他会直接用0覆盖内存中的页面.

被映射之后的页会在memory和一个内核维护的`swap file`之间相互交换.

### 9.6.1 共享对象和私有对象

各个进程有自己的虚拟地址空间, 但很多进程也有相同的代码区域(比如linux bash), 许多程序需要访问只读运行代码库的相同副本, 如printf. 

映射到虚拟内存的对象分为"共享对象"和"私有对象". 对共享对象的写操作会反映到磁盘上, 当然其它的进程也会感知到.私有对象的修改对其它进程则是不可见的, 也不会反映在磁盘上.

映射到共享对象的虚拟内存区域叫做共享区域, 对应的也有私有区域.

对于共享对象, 无论多少个进程来映射它, 在物理内存中只需要存放共享对象的一个副本就可以了

![image-20211122023313701](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023313701.png)

如果修改这个对象, 对应磁盘上的源对象也会被修改.

私有对象映射到虚拟内存是通过"写时复制"技术实现的, 一开始物理内存中也只存一个副本, 所有的进程都指向它,PTE里把这个地方也被标识为只读. 直到有个进程想写它,会触发一个"protection fault", 故障处理程序在物理内存中创建这个页面的新副本, 更新页表条目指向新副本, 恢复这个页面的可写权限, 故障处理程序返回时CPU重新执行写操作, 就写在新创建的页面上了. 

![image-20211122023321570](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023321570.png)

OS会扫描一遍物理内存,找重复的页,找到之后merge它们, 标记位写时复制, 

### 9.6.2 fork

当 fork 函数被当前进程调用时，内核为新进程创建各种数据结构，并分配给它一个唯一的 PID。为了给这个新进程创建虚拟内存，它创建了当前进程的 mm_struct、区域结构和页表的原样副本。它将两个进程中的每个页面都标记为只读，并将两个进程中的每个区域结构都标记为私有的写时复制。

当 fork 在新进程中返回时，新进程现在的虚拟内存刚好和调用 fork 时存在的虚拟内存相同。当这两个进程中的任一个后来进行写操作时，写时复制机制就会创建新页面，因此，也就为每个进程保持了私有地址空间的抽象概念。

### 9.6.3 User-Level Memory Mapping

```c
void *mmap(void* start, int len, int prot, int flag, int fd, int offset);
```

![image-20211122023336437](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023336437.png)

返回指向映射区域开始的指针(可能不是`start` , 由kernel决定)

![image-20211122023346561](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122023346561.png)