# 异常控制流

1. 明白什么是异常控制流？
2. 计算机各层对于异常控制流的使用
3. 了解进程以及内核对于进程的控制管理
4. 信号
# 什么是异常控制流
异常控制流，其实拆分为就是：异常+控制流，从Java的角度出发很容易联想到try-catch。但是，操作系统的异常控制流指的什么？

**什么是控制流？**

当处理器启动开始，直到系统断点，程序计数器中假设一个值的序列为：</br>
A0,A1,A2,......,An-1 <br />
其中Ak是某个相应的指令Ik的地址。程序计数器指令执行从Ak到Ak+1的过渡称为**控制转移（control transfer）**。这样的控制转移序列叫做处理器的**控制流（flow of control）**。

**什么是异常控制流？**

根据上面控制流的叙述，最简单的一种控制流是一个“平滑的”序列，其中指令Ik和Ik+1在内存中都是相邻的。但是这种**平滑流的突变**（也就是Ik+1和Ik不相邻）通常是由诸如跳转、调用和返回这样一些熟悉的指令造成的。

上述平滑流的突变在系统运行中也是必要的，比如硬件定时器产生的处理信号必须要被处理、包到达网络适配器之后必须放到内存中、程序向磁盘请求数据然后休眠直到被通知说数据已就绪、子进程终止时父进程必须得到通知等等。

所以，现代系统通过使控制流发生突变来对这些情况做出反应，这些突变就称之为**异常控制流**（Exception Control Flow，ECF）。

**ECF是操作系统用来实现I/O、进程和虚拟内存的基本机制。**

**异常控制流一般发生在哪？**

异常控制流发生在系统各个层次：
- 硬件层：硬件检测到的事件会触发控制突然转移到**异常处理程序**；
- 操作系统层：内核通过**上下文切换**将控制从一个用户进程转移到另一个用户进程；
- 应用层：一个进程可以发**信号**到另一个进程，而接收者会将控制突然转移到它的一个**信号处理程序**。

上述异常控制流在各层的体现会逐个分析。**从异常开始**，异常位于硬件和操作系统的交界处；**再到系统调用**，它们是为应用程序提供到操作系统的入口点的异常；**继续抽象到进程和信号**，它们位于应用和操作系统的交界之处；**最后探讨非本地跳转**，这是ECF的一种应用层形式。

# 异常-硬件层和操作系统的交互
**异常分属异常控制流的一种形式（不同于我们在Java中说的异常），它一部分由硬件实现，一部分由操作系统实现。**

异常就是控制流中的突变，用来响应处理器状态的某种变化。基本思想如下图：

![exception-b](/Users/liujie/Desktop/gitbook/image/computer/exception-b.png)

其中事件有可能是虚拟内存缺页、算术溢出，也有可能是系统定时器信号或I/O请求的完成等等。

但是，异常处理程序完成后，会发生以下三种情况：
1. 处理程序将控制返回给事件发生时正在执行的指令Icurr；
2. 处理程序将控制返回给Inext，即没有发生异常将会执行的下一条指令；
3. 处理程序将终止被中断的程序。

那么从知道事件发生到进行异常处理程序是怎样的一个异常处理过程？

## 异常处理
操作系统为每种可能的异常类型都分配了一个**唯一的非负整数的异常号（exception number）**。其中一部分是由处理器的设计者分配，一部分是由操作系统内核（操作系统常驻内存的部分）设计者分配的。

**处理器异常号**：包括被零除、缺页、内存访问违例、断点以及算术溢出等。

**内核异常号**：包括系统调用和来自外部I/O设备的信号。

在操作系统启动时，操作系统分配和初始化一张称为异常表的跳转表，使得表目k包含异常k的处理程序地址，如下图：

![exception-table](/Users/liujie/Desktop/gitbook/image/computer/exception-table.png)

**处理器检测到事件-->确定异常号k-->处理器执行间接过程调用根据异常表的表目k转到相应的处理程序。**

下图展示处理器如何通过异常号索引到异常处理程序的地址。

![exception-deal](/Users/liujie/Desktop/gitbook/image/computer/exception-deal.png)

发生的异常基本分类是怎么样的？
## 异常类别
**异常基本可以分为四类：中断（interrupt）、陷阱（trap）、故障（fault）和终止（abort）**。由于陷阱、故障和终止是同步发生的，是执行当前指令的结果，所以这类指令叫做**故障指令**。

![exception-type](/Users/liujie/Desktop/gitbook/image/computer/exception-type.png)

现在来详细的介绍几种异常

### 中断（interrupt）
**中断是异步发生的，是来自处理器外部的I/O设备的信号的结果。** 中断一般不是程序执行本身引发的，而是来自外在（可以说是硬件）的影响导致不得不中断当前执行程序来调用适当的**中断处理程序**。一般中断处理程序执行完毕也会将控制返回给被中断程序的下一条指令让程序继续执行。

下图概述了一个中断的处理。I/O设备，例如网络适配器、磁盘控制器和定时器芯片，通过向处理器芯片上的一个引脚发信号，并将异常号放到总线上，来触发中断，这个异常号标识了引起中断的设备。

![Interrupt](/Users/liujie/Desktop/gitbook/image/computer/Interrupt.png)

### 陷阱和系统调用
**陷阱是一种故意为之的异常，就是执行一条指令的结果**。与中断处理程序一样，陷阱处理程序执行完将控制返回到下一条指令，是程序运行中基于某种目的的积极触发。

**陷阱最重要的用途**是在用户程序和内核之间提供一个像过程一样的接口，叫做**系统调用（system call）**。

因为**普通的函数运行在用户模式中**，用户模式限制了函数可以执行的指令类型，并且只能访问与调用函数相同的栈；**系统调用运行在内核模式中**，内核模式允许系统调用执行特权指令，并访问定义在内核中的栈。（用户模式和内核模式如何出现后面做分析）

陷阱使用广泛，因为用户程序经常要向内核请求服务，如：
- 读文件（read）
- 创建新进程（fork）
- 加载新程序（execve）
- 终止当前进程（exit）
- 等等

**一般通过什么指令触发陷阱从而进行系统调用呢？**

处理器提供一个特殊的 **“syscall n”指令** ：当用户程序想要请求服务n时，可以执行这条指令。**执行syscall指令会导致一个异常处理程序的陷阱**，这个处理程序解析参数，并调用适当的内核程序。下图概述了一个**系统调用的过程**：

![trap](/Users/liujie/Desktop/gitbook/image/computer/trap.png)

### 故障
**故障时程序运行过程中由错误情况引起的，它是有可能被错误处理程序修正。**

**简单说就是错误可以被修正就可以返回程序继续执行，不能被修复就终止程序**。故障处理的流程如下图：

![fault](/Users/liujie/Desktop/gitbook/image/computer/fault.png)

经典的故障例子是：**缺页异常**，当指令引用一个虚拟地址，而与该地址相对应的物理页面不在内存中，因此必须从磁盘中取出时，就会发生故障。故障处理流程就会从磁盘中加载适当的页面（页面会在后面详细分析，一个页面就是虚拟内存一个连续的块，典型的是4kb），然后将控制返回给引起故障的指令。指令继续执行时对应的物理页面已经在内存中了，程序正常运行。

### 终止
终止是不可恢复的致命错误造成的后果，通常是硬件错误，终止处理程序直接终止，从不将控制返回给应用程序。

![aboart](/Users/liujie/Desktop/gitbook/image/computer/aboart.png)

## Linux/x86-64系统中的异常
这里通过我们最常用的服务器操作系统来展示具体的异常。x86-64系统有高达256种不同的异常类型。0~31对应的是Intel架构师定义的异常，对任何x86-64系统通用；32~255对应的是操作系统定义的中断和陷阱。

### Linux/x86-64 故障和终止
- 除法错误：当应用试图除以0，或者是除法指令结果对于目标操作数来说太大的时候就会发生除法错误。Unix选择终止程序，一般错误报告为“浮点异常（Floating Exception）”。
- 一般保护故障：通常是因为一个程序引用了一个未定义的虚拟内存故障，或者是因为程序试图写入一个只读的文本段。Linux不会修复这类故障，故障报告为“段故障（Segmentation fault）”。
- 缺页：是会重新执行产生故障指令的一个异常示例。
- 机器检查：机器检查是导致故障的指令执行中检测到致命的硬件错误时发生的，机器检查处理程序从不返回控制给应用程序。

### Linux/x86-64 系统调用
Linux提供几百种系统调用，用来支持应用程序请求内核服务。

**怎么发起系统调用？**

**每一个系统调用都对应一个唯一的整数号，对应一个内核中跳转表的偏移量**。（跳转表不同于异常表）

在x86-64系统中，系统调用通过一个syscall的陷阱指令发起。同时，所有到Linux系统调用的参数都是通过通用寄存器而不是栈传递的。对应常用的系统调用指令有：

| 指令调用整数号 | 名字   | 描述                 |
| -------------- | ------ | -------------------- |
| 0              | read   | 读文件               |
| 1              | write  | 写文件               |
| 2              | open   | 打开文件             |
| 9              | mmap   | 将内存页映射到文件   |
| 33             | pause  | 挂起进程直到信号到达 |
| 37             | alarm  | 调度告警信号的传送   |
| 39             | getpid | 获得进程号           |
| 57             | fork   | 创建子进程           |
| 59             | execve | 执行一个程序         |
| 60             | _exit  | 终止进程（原子性）   |
| 57             | kill   | 发送信号到进程       |

如常用的hello程序使用系统级函数write：
```
int main(){
    write(1, "Hello,world", 13);
    _exit(0);
}
```
write函数第一个参数代表将输出发送到stdout，第二个参数是要写的字节序列，第三个参数是要写的字节数。这个wirte函数编译变成汇编字节指令执行会变为：

```
movq $1, %rax     write的系统指令编号是1（%rax为对应的寄存器位置）
movq $1, %rdi       stdout的描述符是1
movq $string, %rsi    输出的字节序列hello world
movq $len, %rdx       字节数
syscall     执行系统调用
```

# 进程
**异常是允许操作系统内核提供进程（process）概念的基本构造块，进程是计算机科学中最深刻、最成功的概念之一**。那么异常是怎么跟进程关联的？

**进程的经典定义：一个执行中程序的实例**。系统中的每个程序都运行在某个进程的**上下文**中。**上下文是由程序正确运行所需的状态组成**的，状态都包括：
存放在内存中的程序的代码和数据，它的栈、通用目的寄存器的内容、程序计数器、环境变量以及打开文件描述符的集合。

如**执行shell脚本**：输入可执行目标文件的名字--->shell创建一个新进程--->在新进程上下文中运行这个可执行文件。

**那么进程对于应用程序的意义是什么？**

- 一个独立的逻辑控制流：它提供了一个假象，好像我们的程序独占地使用处理器。
- 一个私有的地址空间：它提供了一个假象，好像我们的程序独占地使用内存系统。

## 逻辑控制流
程序计数器（PC）值的序列叫做逻辑控制流，简称逻辑流。（这些值唯一对应与包含在程序的可执行目标文件中的指令，或是包含在运行时动态链接到程序的共享对象中的指令）

**为什么会说是一个独立的逻辑控制流？**

如一个系统运行着三个进程，那么处理器的物理控制流就会被分成三个逻辑流，每个进程一个：

![logicFlow](/Users/liujie/Desktop/gitbook/image/computer/logicFlow.png)

上面的例子可以看到进程是轮流使用处理器的，每个进程执行它的逻辑流的一部分，然后被抢占（preempted）进入暂时挂起状态，然后轮到其他进程（抢占式控制分配是绝大多数系统默认的）。对于一个运行在这些进程之一的上下文的程序，进程的切换对它是透明的，它看上去就像是在独占地使用处理器。

**逻辑流都有哪些存在形式呢？**
异常处理程序、进程、信号处理程序、线程和Java进程都是逻辑流的例子。

### 并发流和并行流
假设上图描述了三个进程从开始到终止并回收这个周期，可以看到进程A开始后到结束前，B和C进程在这期间启动了，**进程A在执行的时间上会跟其他进程B和C有重叠**，我们可以称AB、AC为**并行流**。这两个流称为**并发地运行**。但是BC不是并行流，因为进程C是在进程B结束之后才创建运行的。

**所以**，定义多个流并发地执行的一般现象称为**并发（concurrency）**。一个进程和其他进程轮流运行的概念称为**多任务（Multitasking）**。一个进程执行他的控制流的一部分的每一时间段叫做**时间片（time slice）**。所以，**多任务也叫做时间分片**。

**并发流的思想跟具体的处理器核数或者计算机数无关**。只要两个流在执行的时间上有重叠就是并发流。但确认并行流会对处理思想有很大帮助，并行流是并发流的一个真子集。如果两个流并发地运行在不同的处理器核或计算机上，那么我们称之为**并行流（parallel flow）**，它们**并行地运行（running in parallel）且并行地执行（parallel execution）**。

**并行流主要体现在并行地执行！**

## 私有地址空间
在一台计算机中，我们假定整个操作系统的地址空间是2^n个可能地址的集合。**操作系统为每一个进程都分配一个只属于它们自己的地址空间段（私有地址空间），进程间互相隔离禁止互相读或写，而进程为运行在其上下文中的程序开发自己的私有地址空间，所以程序看起来就是自己独占系统的地址空间**。（如Java内存模型，线程间私有的地址空间不可见，数据共享和传递只能通过公共的主存，也因此带来线程安全问题）

虽然每个进程的私有地址空间相关联的内存的内容一般是不同的，但私有地址空间的结构都是通用的。地址空间顶部留给内核（操作系统常驻内存部分），底部留给用户程序，通常包括代码、数据、堆和栈段，而中间段部分包含内核在代表进程执行指令时（如应用程序执行系统调用时）使用的代码、数据和栈。

## 用户模式和内核模式
上面提到了很多关于用户模式和内核模式，为什么要这样区分？这样是为了帮助操作系统更好的抽象进程，限制一个应用进程可以执行的指令和可以访问的地址空间范围，并且需要的话可以向操作系统内核申请限制指令操作。这样的分层保证了操作系统底层的稳健。

**怎么样标识进程归属内核模式和用户模式？**

处理器通常是用某个控制寄存器中的一个**模式位**（mode bit）来提供这种功能，该寄存器描述了进程当前享有的特权。**进程设置了模式位就运行在内核模式中，没有设置模式位就是运行在用户模式中**。

运行在内核模式中的进程可以执行指令集中的任何指令，并且可以访问系统中的任意内存位置。

**用户模式和内核模式怎么切换？**

**进程由用户模式变为内核模式的唯一方法就是通过中断、故障或者陷入系统调用这样的异常。**

进程运行模式切换：

![runModeChange](/Users/liujie/Desktop/gitbook/image/computer/runModeChange.png)

频繁的模式切换会导致性能降低，特别是用户模式的进程只需要访问内核数据结构的内容时，Linux提供了一种优化机制，/proc文件系统将许多内核数据结构的内容输出为一个用户程序可以读的文本文件的层次结构。2.6版本的Linux内核引入/sys文件系统，输出关于系统总线和设备的额外的低层信息。

## 上下文切换
**一个进程对应一个上下文，上下文由内核维持**。上面说到并发流的时候有定义多任务的概念，其实内核就是使用称为**上下文切换（context switch**）的**较高层形式的异常控制流**来实现多任务，也就是经常说的并发。

**上下文切换机制是建立在上述的较低层的异常机制之上的。简单的理解就是，较低层提供了异常机制这种工具，我们怎么在较上层中更聪明的使用这种机制，通过上下文切换带来更好的并发执行效果就是一种聪明的使用。**

**那么上下文是什么？**
简单理解就是：**上下文切换机制可以使得内核控制自由切换进程执行，上下文就是保证进程在被“执行-被抢占后挂起-继续执行”这个过程中正常的执行**。

上下文就是内核重新启动一个被抢占进程所需的状态。它由一些对象的值组成，包括统一目的寄存器、浮点寄存器、程序计数器、用户栈、状态寄存器、内核栈和各种内核数据结构（比如描述地址空间的页表、包含当前进程信息的进程表以及进程已打开文件的信息的文件表）。

### 上下文切换具体过程
当进程执行的某些时刻，内核可以决定抢占当前进程，并重新开始一个先前被抢占了的进程。这种决策就叫做**调度（scheduling）**，是由内核中称为**调度器（scheduler）**的代码处理的。

内核调度了一个新的进程运行，它就会抢占当前进程，并使用上下文切换的机制来将控制转移到新的进程。

上下文切换过程：
1. 保存当前进程的上下文；
2. 恢复某个先前被抢占的进程被保存的上下文；
3. 将控制传递给这个新恢复的进程。

### 有哪些情况会发生上下文切换？
**当内核代表用户执行上下文切换时，可能会发生上下文切换**。如果系统调用发生阻塞，那么内核可以让当前进程休眠，切换到另一个进程，如read系统调用，或者sleep会显示地请求让调用进程休眠。一般，即使系统调用没有阻塞，内核亦可以决定上下文切换，而不是将控制返回给调用进程。

**中断也可能引发上下文切换**。典型的就是时间片轮转，所有系统都有某种产生周期性定时器中断的机制，通常为每1毫秒或每10毫秒，每次发生定时器中断，内核就能判断当前进程已经运行了足够久的时间，并切换到一个新的进程。

下图是一个I/O读数据进程切换流程：



# ![processChange](/Users/liujie/Desktop/gitbook/image/computer/processChange.png)调用错误处理
当Unix系统级函数遇到错误时，它们通常会返回-1，并设置全局整数变量errno来表示什么出错了。

# 进程控制管理
上面说到了进程对于操作系统的重要性，那么操作系统是怎么对进程进行控制管理的呢？

## 获取进程ID（getpid、getppid）
**每个进程都有一个唯一的整数（非零）进程ID（PID）。**

getpid函数返回调用进程的PID。getppid函数返回它的父进程的PID（创建调用进程的进程）

```
#include <sys/types.h>
#include <unistd.h>

pid_t getpid(void);
pid_t getppid(void);

返回：调用者或其父进程的PID
```

## 创建和终止进程（fork、exit/_exit）
从程序员的角度，可以认为进程总是处于以下三种状态之一：
- 运行。进程要么在CPU上执行，要么在等待被执行且最终会被内核调度。
- 停止。进程的执行被挂起（suspended），且不会被调度，除非接收到重新运行信号。
- 终止。进程永远的停止了。进程会因为三种原因终止：1、收到一个信号，信号的默认行为是终止进程；2、从主程序返回；3、调用exit函数。
- 回收。被父进程或init进程回收。

**exit函数以status退出状态来终止进程**（另一种设置退出状态的方法是从主程序中返回一个整数值）

```
#include <stdlib.h>

void exit(int status)
```
**父进程调用fork函数来创建子进程**：

```
#include <sys/types.h>
#include <unistd.h>

pid_t fork(void);

返回：子进程返回0，父进程返回子进程的PID，如果出错返回-1。
```
**fork函数被调用一次，却会返回两次：一次是在调用的进程（父进程）中，一次是在新创建的子进程中。根据返回值确定当前的执行进程**。

**父进程和子进程最大的区别在于它们有不同的PID。** 子进程得到与父进程用户级虚拟地址空间相同但独立的副本，包括代码和数据段、堆、共享库以及用户栈，还会继承父进程任何打开文件描述符相同的副本（意味着子进程可以读写父进程在调用fork时打开的任何文件）。

如下面的例子：

```
int main(){
    pid_t pid;
    int x= 1;
    pid = Fork(); //创建子进程，子进程从此处开始执行，父程序中pid=子进程的PID，子进程pid=0
    if(pid ** 0){
        printf("child : x = %d\n", ++x);
        exit(0);
    }
    
    printf("parent: x = %d\n", --x);
    exit(0);
}
```
会输出：(因为子进程会复制父进程的虚拟地址空间，输出顺序在该程序中是无法确定的)
```
parent: x = 0
child: x= 2
```
总结**fork函数的特征**：
- 调用一次，返回两次。
- 并发执行。父子进程是并发运行的独立进程。
- 相同但是独立的地址空间。虽然地址空间是相同的，但是他们是独立的进程，是各自的私有地址空间。
- 共享文件。子进程继承父进程所有打开了的文件。

## 回收子进程
我们在进程创建的时候会为其分配资源，但进程执行完程序终止，我们需要回收其占用的资源，以便于提高资源的利用。
### 僵死进程、孤儿进程
**当一个进程由于某种原因终止时，内核并不是立即把它从系统中清除。相反，进程保持在一种已终止的状态中，直到被它的父进程回收（reaped）**。当父进程回收已终止的子进程时，内核将子进程的退出状态传递给父进程，然后抛弃已终止的进程，此时开始，该进程就不存在了。

根据上面回收流程，我们将一个**终止了但还未被回收的进程**称为**僵死进程**。但如果父进程终止了，下面的子进程就会变成**孤儿进程**，内核会**安排init（PID = 1）进程成为孤儿进程的养父**，回收那些僵死进程。**init进程是在系统启动时内核创建的，它不会终止，是所有进程的祖先。**

僵死进程可以通过以下命令查看，僵死进程后面会用 **"<defunct>"** 标识：

```
> ps t
```


父进程怎么监控子进程是否终止？一种是通过调用**waitpid函数**主动监控，一种是根据信号通知（后续信号会讲解）：

```
#include <sys/types.h>
#include <sys/wait.h>

pid_t waitpid(pid_t pid, int *statusp, int options);

返回：如果成功返回子进程的PID，如果options=WNOHANG，则为0，如果其他错误则为-1
```
### waitpid函数说明
waitpid函数有点复杂，默认地（当options=0时），waitpid挂起调用进程的执行，知道它的等待集合中的一个子进程终止。
1. 判定等待集合的成员，通过参数pid判定：
- 如果pid > 0，那么等待集合就是一个单独的子进程，它的进程ID等于pid。
- 如果pid = -1，那么等待集合就是由父进程所有的子进程组成的。
2. 修改默认行为，通过将option设置为常量WNOHANG、WUNTRACED和WCONTINUED的各种组合来修改默认行为
3. 检查已回收子进程的退出状态，如果statusp参数是非空的，那么waitpid就会在status中放上关于导致返回的子进程的状态信息，status是statusp指向的值。
4. 错误条件，如果调用进程**没有子进程**，那么waitpid返回-1，并且设置**errno=ECHILD**。如果waitpid**被一个信号中断**了，那么它返回-1，并设置**errno=EINTR**。

这里有个简单例子说明：

使用waitpid函数不按照特定顺序回收僵死子进程
```
int main() 
{
    int status, i;
    pid_t pid;

    /* Parent creates N children */
    for (i = 0; i < N; i++)                       //line:ecf:waitpid1:for
	if ((pid = Fork()) ** 0)  /* Child */     //循环创建N个子进程
	    exit(100+i);                          //子进程终止

    /* Parent reaps N children in no particular order */
    while ((pid = waitpid(-1, &status, 0)) > 0) { //父进程等待回收子进程
	if (WIFEXITED(status))          //WIFEXITED(status)=true代表子进程正常终止（通过EXIT或是RETURN）
	    printf("child %d terminated normally with exit status=%d\n",
		   pid, WEXITSTATUS(status));     //WEXITSTATUS(status)返回正常终止子进程退出状态，即上面的100+i，需要WIFEXITED为true
	else
	    printf("child %d terminated abnormally\n", pid);
    }

    /* The only normal termination is if there are no more children */
    if (errno != ECHILD)   //当没有子进程时，errno=ECHILD，waitpid中断时errno=EINTR
	unix_error("waitpid error");

    exit(0);
}
```
使用waitpid函数按照子进程创建的顺序回收僵死子进程：

```
int main() 
{
    int status, i;
    pid_t pid[N], retpid;  //pid[N]，按照创建顺序存储子进程的PID

    /* Parent creates N children */
    for (i = 0; i < N; i++) 
	if ((pid[i] = Fork()) ** 0 
	    exit(100+i);

    /* Parent reaps N children in order */
    i = 0;
    while ((retpid = waitpid(pid[i++], &status, 0)) > 0) { //按照子进程创建顺序监控回收
	if (WIFEXITED(status))  
	    printf("child %d terminated normally with exit status=%d\n",
		   retpid, WEXITSTATUS(status));
	else
	    printf("child %d terminated abnormally\n", retpid);
    }
    
    /* The only normal termination is if there are no more children */
    if (errno != ECHILD) 
	unix_error("waitpid error");

    exit(0);
}
```


### wait函数
wait函数是waitpid函数的简单版本：

```
#include <sys/types.h>
#include <sys/wait.h>

pid_t wait(int *statusp);

返回：成功返回子进程PID，出错返回-1
```
调用wait(&status)等价于调用waitpid(-1, &status, 0)

## 进程休眠（sleep、pause）
**sleep函数**将一个进程挂起一段指定的时间：

```
unsigned int sleep(unsigned int secs);
返回：还要休眠的秒数
```

**pause函数**让调用进程休眠，直到该进程收到一个信号：

```
int pause(void);
返回：总是返回-1
```
## 加载并运行程序（execve）
**execve函数在当前进程的上下文中加载并运行一个新程序**。

```
int execve(const char *filename, const char *argv[], const char *envp[]);
返回：成功不返回，失败返回-1。
```
execve函数加载并运行可执行文件filename，且带参数argv和环境变量列表envp。只有当出现错误时，例如找不到filename，execve才会返回调用程序。所以，与fork一次调用两次返回不同，**execve调用一次从不返回**。

参数列表的组织结构，argv指向一个以null结尾的指针数组，且**argv[0]惯例保存filename**：

![execve1](/Users/liujie/Desktop/gitbook/image/computer/execve1.png)

环境变量列表的组织结构，envp指向一个以null结尾的指针数组：

![execve2](/Users/liujie/Desktop/gitbook/image/computer/execve2.png)

execve成功加载filename之后，它会调用链接阶段的启动代码，将控制传递给新程序的主函数。运行的新的程序依然是基于之前的进程，即PID不会改变，但会覆盖当前进程的地址空间并继承调用execve函数时已打开的所有文件描述符。

**进程是执行中程序的一个具体的实例；程序总是运行在某个进程的上下文中**。这也是fork和execve的本质差别。

## 利用fork和execve运行程序（如shell、web服务器）
**web服务器和Shell大量使用了fork和execve函数。**

### shell的执行过程
下面针对Shell来说明：

最早的Shell是sh程序，后面出现了：csh、tcsh、ksh和bash等变种。

**Shell的执行过程**可以总结为：执行一系列的读/求值（read/evaluate）步骤，然后终止。读步骤读取来自用户的一个命令行；求值步骤解析命令行，并代表用户运行程序。

下面展示了一个简单shell的main例程：shell打印一个命令行提示符，等待用户在stdin输入命令行，然后对命令行求值
```
/* $begin shellmain */
#include "csapp.h"
#define MAXARGS   128

/* 函数声明 */
void eval(char *cmdline); //对命令行求值
int parseline(char *buf, char **argv); //解析以空格分隔的命令行参数，最终传递给execve函数的argv[]
int builtin_command(char **argv); //判断命令行参数是否是一个内置的shell命令

int main() 
{
    char cmdline[MAXLINE]; //存储用户输入的命令行

    while (1) {
	/* Read */
	printf("> ");   //shell打印的命令行提示符                
	Fgets(cmdline, MAXLINE, stdin);  //得到stdin输入流
	if (feof(stdin)) //feof()是检测流上的文件结束符的函数，如果文件结束，则返回非0值，否则返回0
	    exit(0);

	/* Evaluate */
	eval(cmdline);//解析命令行并代表用户运行
    } 
}
```
看一下具体的求值过程：

```
/* $begin eval */
/* eval - Evaluate a command line */
void eval(char *cmdline) 
{
    char *argv[MAXARGS]; /* Argument list execve() */
    char buf[MAXLINE];   /* Holds modified command line */
    int bg;              /* Should the job run in bg or fg? */
    pid_t pid;           /* Process id */
    
    strcpy(buf, cmdline);
    bg = parseline(buf, argv);   //解析命令行，并判断设置是否需要后台运行
    if (argv[0] ** NULL)  //判断是否是空命令行
	    return;   

    if (!builtin_command(argv)) {  //判断是否是内置的shell命令
        if ((pid = Fork()) ** 0) {   //创建一个子进程用来运行程序
            if (execve(argv[0], argv, environ) < 0) {
                printf("%s: Command not found.\n", argv[0]);
                exit(0);
            }
        }

	/* Parent waits for foreground job to terminate */
	if (!bg) {  //判断是否是以“&”结尾的后台运行程序
	    int status;
	    if (waitpid(pid, &status, 0) < 0) //非后台程序执行则阻塞等待子进程执行完毕
		    unix_error("waitfg: waitpid error");
	}
	else
	    printf("%d %s", pid, cmdline);
    }
    return;
}

/* 如果argv[0]，即命令行第一个参数是内置的shell命令，返回 true */
int builtin_command(char **argv) 
{
    if (!strcmp(argv[0], "quit")) /* quit command */
	exit(0);  
    if (!strcmp(argv[0], "&"))    /* Ignore singleton & */
	return 1;
    return 0;                     /* Not a builtin command */
}
/* $end eval */
```

```
/* $begin parseline */
/* parseline - Parse the command line and build the argv array */
int parseline(char *buf, char **argv) 
{
    char *delim;         /* Points to first space delimiter */
    int argc;            /* Number of args */
    int bg;              /* Background job? */

    buf[strlen(buf)-1] = ' ';  /* Replace trailing '\n' with space */
    while (*buf && (*buf ** ' ')) /* Ignore leading spaces */
	buf++;

    /* Build the argv list */
    argc = 0;
    while ((delim = strchr(buf, ' '))) {
	argv[argc++] = buf;
	*delim = '\0';
	buf = delim + 1;
	while (*buf && (*buf ** ' ')) /* Ignore spaces */
            buf++;
    }
    argv[argc] = NULL;
    
    if (argc ** 0)  /* Ignore blank line */
	return 1;

    /* Should the job run in the background? */
    if ((bg = (*argv[argc-1] ** '&')) != 0)
	    argv[--argc] = NULL;

    return bg;
}
/* $end parseline */
```
但是上述程序有很大的问题是，他不回收后台子进程，修复这个缺陷需要用到信号。

# 信号
我们在上面叙述了软硬件是如何合作以提供基本的低层异常机制的，也看到了操作系统如何利用异常来支撑进程上下文切换的异常控制流形式。这里我们继续研究一种更高层的软件形式的异常，称为**Linux信号，它允许进程和内核中断其他进程。**

**一个信号就是一个小消息，它来告诉进程系统当前发生了一种某种类型的事件。**Linux支持了30种不同类型的信号，**每种信号类型都对应某种系统事件**。

常见的信号类型有：

| 名称    | 默认行为 | 相应事件                 |
| ------- | -------- | ------------------------ |
| SIGINT  | 终止     | 来自键盘的中断，如CTRL+C |
| SIGCHID | 忽略     | 一个子进程停止或者终止   |

## 信号流程
传送一个信号是由两个步骤组成的：发送信号、接收信号。

### 发送信号
**怎么样触发和发送信号？**

**内核通过更新目的进程上下文的某个状态，发送一个信号给目的进程（可以是自己）。**

一般触发发送信号有两种原因：
1. 内核检测到一个系统事件，如除零错误或者子进程终止；
2. 一个进程调用了kill函数，显式的要求内核发送一个信号给目的进程。

### 接收信号
**目的进程怎么接收信号并怎么反应呢？**

**当目的进程被内核强迫以某种方式对信号的发送做出反应时，他就接受了信号。**

**进程可以忽略这个信号，终止或通过执行一个称为信号处理程序的用户层函数来捕获这个信号**。下面是具体信号处理流程：

![reciveSign](/Users/liujie/Desktop/gitbook/image/computer/reciveSign.png)

### 信号的状态
- **待处理信号（pending signal）：一个发出而没有被接收的信号**。在任何时刻，一种信号类型至多只会有一个待处理信号，多出来的同一类型信号只会被丢弃。内核为每个进程在pending位向量中维护着待处理信号的集合。
- **被阻塞的信号：进程可以选择的阻塞接收某种信号**，当一种信号被阻塞，它仍可以发送但是产生的待处理信号不会被接收，直到取消阻塞。内核为每个进程在blocked位向量中维护着被阻塞信号的集合。

## 发送信号-具体描述
上面有说到发送信号的原因，但具体是怎么操作或实际我们的应用场景是什么呢？

### 1.进程组（批量发送信号的基础）
大量向进程发送信号的机制----进程组。

**每个进程都只属于一个进程组，进程组是由一个正整数进程组ID来标识的。** 同时，**默认子进程和父进程同属于一个进程组**。

getpgrp函数返回当前进程的进程组：

```
include <unistd.h>
pid_t getpgrp(void);
返回：调用进程的进程组id
```
setpgid函数改变自己或其他进程的进程组：
```
include <unistd.h>
int setpgid(pid_t pid, pid_t pgid);
返回：成功为0，失败为-1
```
### 2.用/bin/kill程序发送信号
**/bin/kill程序可以向另外的进程发送任意的信号**，如：

发送信号9（SIGKILL）给进程1111：
```
> /bin/kill -9 1111
```
发送信号9（SIGKILL）给进程组1111中的每个进程：(**为负的PID表示进程组**)

```
> /bin/kill -9 -1111
```
### 3.从键盘发送信号
**Unix shell使用作业（job）这个抽象概念来对一条命令行求值而创建的进程**。在任何时刻，至多会有一个前台作业和0个到多个后台作业。

举例，假设我们查看一个文件夹下的文件并对其排序可以使用命令：

```
> ls | sort
```
**这个命令其实会创建一个由两个进程组成的前台作业，这两个进程通过Unix管道连接起来：一个运行ls程序，一个运行sort程序**。shell会为每个作业都创建一个独立的进程组，进程组的ID通常会取自作业中父进程中的一个的PID。

![processGroup](/Users/liujie/Desktop/gitbook/image/computer/processGroup.png)

### 4.用kill函数发送信号
**进程通过调用kill函数发送信号给其他进程（包括自己）。**

kill函数：

```
/**
 * pid 目标
 * sig 信号号码
 */
int kill(pid_t pid, int sig);
返回：成功返回0，错误返回-1
```
参数pid有三种取值：
- pid > 0：发送信号sig给进程pid；
- pid = 0：发送信号sig给调用进程所在的进程组中的每个成员；
- pid < 0：发送信号sig给进程组|pid|中的每个成员。

### 5.用alarm函数发送信号
进程可以调用alarm给自己发送SIGALRM信号。

```
unsigned int alarm(unsigned int secs);
返回：前一次闹钟剩余的秒数，若以前没有定义则为0
```
**alarm函数安排内核在secs秒后发送一个SIGALRM信号给调用进程（secs不为0）。** alarm函数会覆盖上一次的alarm函数调用闹钟。

## 接收信号-具体描述
内核对于信号的接收处理是有一套流程的。

### 内核如何强制让目的进程接收信号

当内核把进程P从内核模式切换到用户模式时（系统调用返回或是完成一次上下文切换），它会检查进程P的未被阻塞的待处理的信号集合（pending & ~blocked），如果集合不为空，那么内核会选择集合中的某个信号k（通常是最小的信号码），并且强制P接收信号k。收到这个信号的进程会采取某种行为，一旦进程完成了这种行为那么控制就传递回P的逻辑控制流中的下一条指令。

### 信号的默认行为
- 进程终止
- 进程终止并转储内存
- 进程停止（挂起）直到被SIGCONT信号重启
- 进程忽略信号

### 自定义信号行为
我们可以使用signal函数修改用户默认行为，但SIGSTOP和SIGKILL的默认行为不给更改。

```
typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);
返回：若成功则为指向前次处理程序的指针，出错返回SIG_ERR（不设置errno）
```

**调用信号处理程序称为捕获信号，执行信号处理程序称为处理信号。** 处理程序一般使用“return 0；”将控制传递回控制流中进程被信号接收中断位置处的指令。

下面是一个用新号处理程序捕获SIGINT信号的程序：
```
/* $begin sigint */
#include "csapp.h"

void sigint_handler(int sig) /* SIGINT handler */   //line:ecf:sigint:beginhandler
{
    printf("Caught SIGINT!\n");    //line:ecf:sigint:printhandler
    exit(0);                      //line:ecf:sigint:exithandler
}                                              //line:ecf:sigint:endhandler

int main() 
{
    /* Install the SIGINT handler */         
    if (signal(SIGINT, sigint_handler) ** SIG_ERR)  // 重新定义信号行为
	    unix_error("signal error");                 //line:ecf:sigint:endinstall
    
    pause(); /* 等待信号处理返回 */  //line:ecf:sigint:pause
    
    return 0;
}
/* $end sigint */
```


信号处理程序可以被其他信号处理程序中断

![signProcess](/Users/liujie/Desktop/gitbook/image/computer/signProcess.png)

## 阻塞和解除阻塞信号
Linux提供阻塞信号的显式和隐式机制。

### 隐式阻塞机制
内核默认阻塞任何当前处理程序正在处理信号类型的待处理的信号。

### 显式阻塞机制
应用程序可以使用sigprocmask函数和它的辅助函数，明确的阻塞和接触阻塞选定的信号。

## 如何编写好的信号处理程序
信号处理是Linux系统编程最棘手的一个问题。需要考虑：
- 处理程序与主程序并发运行，共享同样的全局变量，因此可能与主程序和其他处理程序互相干扰；
- 如何以及何时接收信号的规则常常有违人的直觉；
- 不同的系统有不同的信号语义。

### 安全的信号处理：解决并发运行
1. 处理程序尽可能的简单；
2. 在处理程序中只调用异步信号安全的函数（要么具备原子性不会被中断（如_exit函数），要么具备可重入性）；
3. 保存和恢复errno（进入处理程序时用局部变量保存当前的errno，返回时恢复）；
4. 阻塞所有的信号，保护对共享全局数据结构的访问；
5. 用volatile声明全局变量；
6. 用sig_atomic_c声明标志。

### 正确的信号处理
信号有阻塞和丢弃机制，如果存在一个未处理的信号就表明至少有一个信号到达了，同时信号不可作为事件计数。

### 可移植的信号处理
信号在不同系统中的语义不一致。

解决这个问题Posix标准定义了sigaction函数，它允许用户在设置信号处理时明确它们想要的信号处理语义。

# 非本地跳转
**C程序可以使用非本地跳转来规避正常的调用/返回栈规则，并且从一个函数的分支到另一个函数。**
# 操作进程的工具
- STRACE：打印一个正在运行的程序和它的子进程调用的每个系统调用的轨迹；
- PS：列出当前系统中的进程（包含僵死进程）；
- TOP：打印出关于当前进程资源使用的信息；
- PMAP：显示进程的内存映射;

/proc:一个虚拟文件系统，以ASCII文本格式输出大量内核数据结构的内容，用户程序可以读取这些内容。比如，查看系统的平均负载（系统平均负载被定义为在特定时间间隔内运行队列中的平均进程数）：

```
root@card-web:~# cat /proc/loadavg
0.13 0.05 0.01 1/801 17901
```
0.13（1分钟平均负载），0.05（5分钟平均负载），0.01（15分钟平均负载），1/801（分子是当前正在运行的进程数，分母是总的进程数），17901（最近运行进程的PID）