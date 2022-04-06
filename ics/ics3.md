# 第3章 程序的机器级表示

按照课程顺序来

# 3.1 机器级编程-基础

## 3.1.1 C, assembly, machine code

重要概念:

- Architecture: 指令集架构, 主要描述指令集的元信息，以及一些规范。 比如界定包含的指令，指令的长度规范，操作数的规范，内存模型等等, 软硬件之间的"合同".
	- 软件与硬件之间的抽象层
	- 流行的: x86-64, arm, risc-v
- Microarchitecture: 指令级架构的具体实现方式(比如流水线级数，缓存大小等), 是可变的
- Machine code: 机器码, 机器直接执行的二进制指令
- Assembly code: 汇编码

### 机器码/汇编码

![image-20211122013059663](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013059663.png)

PC: Program counter, x86下是%rip, 给出下一条指令内存中的地址.

Register:(整数)寄存器, 16个, 存一些数据和关键信息

Condition codes: 保存最近的算术/逻辑运算的状态信息, 用于条件分支

其它内部信息比如CPU中的cache等等都是不可见的

Memory一般支持字节寻址的方式.绝大多数指令集架构的大小端模式是确定的



### 把c代码翻译成可执行程序

也就是 源代码 `->` 编译 `->` 汇编 `->` 链接 `->` 可执行文件 `->` 装载 

![image-20211122013121105](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013121105.png)

程序就是一系列（被编码了的）字节序列 （看上去和数据一模一样），这就是所谓的冯诺依曼结构计算机，即程序存储型计算机.

### 数据格式

汇编用后缀表示数据长度

Char -> b, short -> w, int -> l, long -> q

x86-64下整数有1byte(char),2byte(short), 4byte(int), 8byte(long), 汇编缩写是`b`, `w`, `l`, `q`

浮点数有4byte(float), 8byte(double), 10byte(long double)

没有像数组、结构这样的Aggregate types. 他们本质上是contiguously allocated bytes in memory

### 生成可执行程序

需要经过汇编与链接

汇编把`.s`变成`.o`, 把各条指令二进制编码, 链接解析文件间的引用, Combines with static run-time libraies(如malloc.o, printf.o).

有的库是动态链接的, 就是说当程序开始执行时才链接



## 3.1.2 汇编基础

### 寄存器

![image-20211122013138800](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013138800.png)

寄存器是8bytes的, 为了兼容性可以访问低位的4bytes、2bytes、1bytes.

### 数据传输指令

`movq Source, Dest` 把数据从Source移到Dest

`movb`, `movw`, `movl`, `movq`用来传送不同大小, 源不同大小时有形如`movzbw`(零扩展)和`movs`(符号扩展)

### 指令操作数格式

操作数可以是立即数, 寄存器或内存

![image-20211122013201499](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013201499.png)



括号的意思就是寻址, `Rb`的意思是`Reg[Rb]`, `(Rb)`的意思是`Mem[Reg[Rb]]`

通用的格式:

- `D(Rb, Ri, S)`:  `Mem[Reg[Rb]+S*Reg[Ri]+D]` -> Address mode expression
- `(Rb, Ri)`: `Mem[Reg[Rb]+Reg[Ri]]`

- `D(Rb, Ri)` -> `Mem[Reg[Rb]+Reg[Ri]+D]`

- `(Rb, Ri, S)` -> `Mem[Reg[Rb]+S*Reg[Ri]]`

## 3.1.3 算术与逻辑运算符

### Leaq - Address Computation Instruction

```
leaq Src Dst
```

Src是一个address mode expression, 算出其地址后不用这个地址访问内存, 而是直接把这个地址的值给Dst

如

```
leaq (%rdi, %rdi, 2), %rax 
```

 左边的数是(%rdi + 2 * %rdi), 然后把这个数写到%rax中

用途:

- 不寻址就计算出地址, 如`p = &x[i]`
- 算一些算术表达式

### 其它表达式

双目运算符

![image-20211122013241235](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013241235.png)

单目运算符

![image-20211122013259115](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013259115.png)

# 3.2 控制

不支持在 Docs 外粘贴 block

重点: 条件码, 分支, 循环

## 3.2.1 条件码

CPU中维护者一组单个位的条件码, 作用是**标记最近的算术或逻辑操作的属性, 它们implicitly set(think of it as side effect) by arithmetic operations.**

Example: add Src(a), Dest(b) (t = a + b)

- CF: 进位标志(Carry Flag)(针对无符号数), 最近的操作使最高位产生进位时置1(unsigned overflow)
- ZF: Zero flag, 最近的操作的结果是0时置1
- SF: Sign Flag(针对有符号数)最近的操作得到的结果是负数时, 置1
- OF: Overflow Flag, 最近的操作导致补码溢出时置1(a > 0 && b > 0 && t < 0) || (a < 0 && b < 0 && t > 0)



### 显式设置条件码



```
cmpq Src2(b), Src1(a)
```

相当于计算`a - b`, 然后利用结果进行条件代码设置.

- CF: 进位标志(Carry Flag)(针对无符号数), 最近的操作使最高位产生进位时置1
- ZF: a == b
- SF: a < b
- OF: 溢出(`(a > 0 && b > 0 && t <0) || (a<0 && b<0 && t>=0)`）

```
testq Src2(b), Src1(a)
```

相当于计算`a&b`

- ZF: a&b == 0
- SF: a&b < 0

### Reading Condition Codes

`SetX Instruction`: 根据条件码的组合把一个Byte设置成0或1(不会改变剩余的7个Byte)

![image-20211122013323238](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013323238.png)

![image-20211122013330922](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013330922.png)

因为它不改变高7位Byte, 所以一般配合`movezbl`再移动到别的地方完成任务(32-bit instructions set upper 32 bits to 0)

比如

```
setg %al
movzbl %al, %eax
ret
```

## 3.2.2 条件分支(if)

### Jumping

Jump to different part of code depending on condition codes

![image-20211122013342350](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013342350.png)

跳转指令分为直接跳转(跳转到指令中指定的label)和间接跳转(跳转目标从寄存器或内存中读出来), 这种情况下操作数前面加一个`*`

比如直接跳转: `jmp .L1`, 间接跳转: `jmp *%rax`, `jmp *(%rax)`

比如这个函数:

### Conditional Branch Example

```
gcc -Og -S -fon-if-conversion control.c
```

对于一个函数

```assembly
long absdiff(long x, long y){
    long result;
    if(x > y) result = x - y;
    else result = y - x;
    return result;
}
        .file        "absdiff.c"
        .text
        .globl        absdiff
        .type        absdiff, @function
absdiff:
.LFB0:
        .cfi_startproc
        cmpq        %rsi, %rdi
        jle        .L2
        movq        %rdi, %rax
        subq        %rsi, %rax
        ret
.L2:
        movq        %rsi, %rax
        subq        %rdi, %rax
        ret
        .cfi_endproc
.LFE0:
        .size        absdiff, .-absdiff
        .ident        "GCC: (Ubuntu 5.4.0-6ubuntu1~16.04.12) 5.4.0 20160609"
        .section        .note.GNU-stack,"",@progbits
```

翻译成c的goto就是

```c
long absdiff(long x, long y){
    long result;
    int ntest = x <= y;
    if(ntest) goto Else;
    result = x - y;
    goto Done;
Else:
    result = y - x;
Done:
    return result;
}
```

![image-20211122013425415](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013425415.png)

### General Conditional Expression Translation(Using Branches)

三目运算符 `Val = Test ? Then_Expr : Else_Expr`

```c
    ntesst = !Test;
    if(ntest) goto Else;
    val = Then_Expr;
    goto Done
Else:
    val = Else_Expr;
Done:
...
```

### Using Condition move

原来的分支语句很低效, 因为CPU是流水线工作的, 就是说要执行ABCDE, 那么执行A时就开始加载B所需的数据. 分支就可能导致加载错的情况. 用屁屁踢上的话叫"disruptive to instruction flow through pipelines"

解决方法: 计算一个操作的两个结果, 根据条件选取一个. 有一条指令叫条件传送(cmov)就是干这个的. 这条命令在条件满足时才执行.

```assembly
absdiff:
    movq    %rdi, %rax  # x
    subq    %rsi, %rax  # result = x-y
    movq    %rsi, %rdx
    subq    %rdi, %rdx  # eval = y-x
    cmpq    %rsi, %rdi  # x:y
    cmovle  %rdx, %rax  # if <=, result = eval
    ret
```

这个只适用于计算开销不大时, 且计算不能有副作用(比如`val = x > 0 ? x*=7 : x+=3`)

`cmov`从寄存器的名字判断出条件传送指令的操作数长度, 所以不区分数据长度后缀

![image-20211122013449157](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013449157.png)



## 3.2.3 循环

### Do-while loop

c语言:

```c
do
    Body
while(Test)
```

Goto Version

```c
loop:
    Body
if(Test)
    goto loop
```

例子: 计算`x`二进制表示中1的个数

```c
long pcount_do(unsigned long x)
{
    long result = 0;
    do {
        result += x & 0x1;
        x >>= 1;
    } while (x);
    return result;
}
  movl    $0, %eax    # result = 0
.L2:                    # loop:
    movq    %rdi, %rdx
    andl    $1, %edx    # t = x & 0x1
    addq    %rdx, %rax  # result += t
    shrq    %rdi        # x >>= 1
    jne     .L2         # if (x) goto loop
    rep; ret
```

### General while loop 

`-Og`优化时: initial goto starts loop at test

```c
while(Test)
    Body
goto test;
loop:
    Body
test:
    if(Test)
        goto loop;
```

`-O1`优化: 相当于转化成了do-while, 而且貌似这样利于编译器优化

```c
while(Test)
    Body
if(!Test)
    goto Done;
loop:
    Body
    if(Test)
        goto loop;
done:
```

### For loop

for循环可以转化成while循环

```c
for(Init; Test; Update)
    Body
Init;
while(Test){
    Body;
    Update;
}
```



## 3.2.4 Switch

显然switch可以转化成很多`if-else`来实现

当分支很多且条件值相差不大时可以用跳转表实现, 通过条件值为`index`访问跳转表, 读出对应条件下需执行的代码的位置

![image-20211122013534087](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013534087.png)



目标代码

```c
long switch_eg (long x, long y, long z){
    long w = 1;
    switch (x) {
        case 1:
            w = y*z;
            break;
        case 2:
            w = y/z;
            // fall through
        case 3:
            w += z;
            break;
        case 5:
        case 6:
            w -= z;
            break;
        default:
            w = 2;
    }
    return w;
}
```

![image-20211122013556576](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013556576.png)

![image-20211122013609555](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013609555.png)

相信编译器, 它肯定比你聪明多了.

# 3.3 过程

函数调用的过程 ：控制权转移 (含返回地址的保存)，参数传递，内存管理 (栈)，控制权返回

过程调用(理解成函数调用)的实现.

这个过程可以拆分成几个机制

- 传递控制: 如何开始执行过程代码, 如何返回到开始的地方
- 传递数据: 即提供参数与返回值
- 内存管理: 分配内存, 释放内存

就是调用者和被调用者的一种约定/规则, 注意不同的指令集架构可能有不同的行为

## 3.3.1 栈结构

栈从高地址向低地址增长, 栈顶用%rsp指示

Pushq Src: 把Src的数写到%rsp指的地方, rsp-=8

Popq Dest: 把rsp指的地方的值写到Dest, rsp += 8



## 3.3.2 传递控制

如何开始执行过程代码, 如何返回到开始的地方

执行`call label`时, call有两个效果, 把函数返回后要执行的命令地址入栈, 跳转到函数代码(设置PC). `ret`从栈中弹出那个地址然后跳转过去(设置PC)

## 3.3.3 传递数据

函数前六个参数放在寄存器里,剩下的放在栈里

![image-20211122013633314](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013633314.png)

## 3.3.4 Managing local Data (栈上的局部存储)

局部变量一般放在寄存器中, 因为访问寄存器比访问内存快得多, 但有时候必须放在内存中:

- 寄存器不足以存放所有本地数据
- 对一个局部变量使用地址运算符(所以必须给它产生一个地址)(啊这)
- 局部变量是数组或结构

把每个函数在栈上管理的空间叫做"栈帧", 大致分几部分: 保存局部寄存器和变量的部分, (如果它也要调用别的函数而且参数挺多的)参数部分. 然后注意到调用这个函数的函数(caller)里面有参数部分和返回地址

![image-20211122013647452](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013647452.png)

关于%rbp, 因为书上提到rbp是callee-saved register之一, 这个东西一开始是%rsp的快照, 用它配合偏移量访问栈上的东西, 后来编译器优化用rsp+偏移量访问东西, 所以就可以把rbp解放出来干别的了.(所以这里是optional(可选的)

### Register Saving Conventions

防止出现caller用某寄存器存一个临时值,结果在调用callee时值被改变了, 把寄存器分为Caller Saved和Callee Saved.

- Caller Saved: caller决定是否保存, 保存的话要自己放入栈中, callee可以随意改变
	- %rax (函数返回值)
	- %rdi, ..., %r9(函数参数)
	- %r10, %r11 (不知道有什么用)
- Callee Saved: callee不许改变.改变后在返回前要改回来
	- %rbx, %r12, %r13, %r14
	- %rbp: 可能被用为frame pointer
	- %rsp: 栈顶, 函数调用完后要释放内存(也就是置回原来的值了)(也就相当于callee saved了)



## 3.3.5 递归

根据条件判断是否跳到边界条件, 不然就直接调用自身就是了.

![image-20211122013701828](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013701828.png)

# 3.4 Data

即数组和指针的机器级表示



## 3.4.1 数组

### 一维数组

```
T A[L]
```

在内存中线性连续排布

![image-20211122013715909](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013715909.png)

`A`可以被用作指向数组首元素的指针, 访问时利用`A+index*sizeof(T)`

### 多维数组

```c
T A[R][C]`, 在内存中是row-major Ordering的. 访问`A[i][j]`时利用`A + i * (sizeof(T) * C) + j * sizeof(T)
```

![image-20211122013729443](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013729443.png)

### 嵌套数组

形如

![image-20211122013803229](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013803229.png)

就会变成真正的`int*`数组, 一维数组里放的是`int*`

![image-20211122013817743](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013817743.png)

访问者中数组需要做两次访存 -- 第一次找到`univ`中的对应指针, 第二次用这个指针去找对应的元素

```c
univ[index][digit] : Mem[Mem[univ+8*index]+4*digit]
```

与多维数组在抽象层上看似一致, 其实访存的区别很大

## 3.4 2 Struct

```c
struct rec{
    int a[4];
    size_t i;
    struct rec* next;
};
```

struct在内存中是一块连续的空间, 对于其子字段按照声明的顺序来以此分配(Even if another ordering could vield a more compact representation). 空间大小和字段具体位置(偏移量)通通由编译器来决定.

也就是说编译时就确定了Struct成员的偏移量. (机器代码不包含关于字段声明或字段名字的信息.)

所以要把大的元素对象放在前面以节省空间

![image-20211122013845183](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122013845183.png)



### 对齐

- 为什么要进行对齐: 大部分处理器是按照双字节/四字节/8字节/16字节/32字节为单位来取内存的, 这个值称为内存存取粒度. 比如四字节存取粒度的处理器要读取int, 只能从地址为4的倍数的内存开始读取数据. 内存对齐就是为了让处理器尽量一次把数据读出来, 不需要读好几次然后合并啥的.

> 以下信息来源于cmu15-213的屁屁踢, 不一定正确, 当然不同的ISA肯定也有不同的规则

对齐规则:

- k字节对象的地址必须是k的倍数(所以short对象的地址的最低位必须是0, int地址最低位必须00)
- Struct:
	- 每个元素的偏移量都要满足对齐的要求(不满足就空一块)
	- Struct起始地址 & Struct的长度必须是结构体中最大长度对象长度的倍数
		- 不够的话后面要补齐, 作用是方便结构数组中下一个元素地址也成倍数关系.





## 3.4.3 Union

根据最大的元素的大小来分配空间, 所有成员共用这块内存

# 3.5 Advanced topics

## 3.5.1 典型的内存布局

![image-20211122014030769](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122014030769.png)

典型的linux下的内存层次, 共享库、堆与栈、数据与代码区.值得一提的是内存会有两个指示位, 指示这一部分是否可读、是否可写.

另外堆貌似不是单向增长的, 而是双向, 然后比较大的内存放下面向上增长

![image-20211122014052818](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122014052818.png)

经过查阅 [[译\] malloc中的系统调用brk和mmap | yoko blog (pengrl.com)](https://www.pengrl.com/p/20032/)

向上增长的heap是系统调用brk()的结果, 中间这一块叫Memory Mapping Segment, 由mmap分配. malloc底层即会用brk也会用mmap





## 3.5.2 缓冲区溢出

课程里指栈溢出

```c
#include <stdio.h>
void echo(){
    char buf[4];
    gets(buf);
    puts(buf);
}

int main(){
    printf("Type a string:");
    echo();
    return 0;
}
```

![image-20211122014116154](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122014116154.png)

输入的string过长会导致段错误

同时还顺手编译器分配的局部变量的大小会比需要的多一点

![image-20211122014126889](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122014126889.png)

回到过程那一节的这幅图上: 

![image-20211122014138468](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122014138468.png)

如果把return Addr覆盖了, 函数就无法返回到正确的地方了.

代码注入攻击即能够精确的覆盖返回地址, 从而让程序跳到需要的地方.

避免缓冲区溢出:

- 用户层面: 不适用不安全的函数, 比如用`fgets`代替`gets`, `strncpy`代替`strcpy`

- os层面: ASLR(地址空间随机化), 每次运行随机分配栈,mmap 共享库等的地址.

	- gdb默认是关闭地址空间随机化的

- 硬件层面: 对栈区增加权限保护, 栈区不可执行. 

	```
	gcc -z noexecstack
	```

	来关闭

	- 解释一下(因为我第一次听课的时候没听明白233333), 就是我们一般是把代码也放在栈区的......
	- 绕过: ROP/ret2libc

- 编译器层面: 栈"金丝雀", `gcc -fsrack-protector`

![image-20211122014203186](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122014203186.png)在申请栈上缓存时在里面放个随机值, 执行函数后检查这个值有没有变(你如果想修改返回地址的话那一定会覆盖金丝雀值) 这是产生的汇编代码

![image-20211122014226641](https://gitee.com/oldataraxia/pic-bad/raw/master/img/image-20211122014226641.png)

### Return-Oriented Programming

一种对栈上返回地址的攻击方式, 把代码段里的多个片段拼凑成一段有效的逻辑,片段一般称为Gadget.

Gadget + retq

Attack lab就有针对这个的实验.