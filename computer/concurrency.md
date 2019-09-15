# 操作系统的并发编程

## 什么是并发？
**如果逻辑控制流在时间上重叠，那么它们就是并发的**，像硬件异常处理程序、进程和Linux信号处理都是很熟悉的例子。

并发在操作系统内核中的主要体现是用来运行多个应用程序的机制。但是，并发不仅仅局限于内核，它也能在应用程序中扮演重要的角色。

**使用应用级并发的应用程序称为并发程序（concurrent program）。**

并发程序基于现代操作系统提供的三种基本机制：
- **进程**。使用这种方式，**每个逻辑控制流都是一个进程，由内核来调度和维护**。但进程有独立的虚拟地址空间，所以进程间的通信必须使用某种显式的进程间通信（interprocess communication，**IPC**）机制。
- **I/O多路复用**。在这种形式的并发编程中，应用程序在一个进程的上下文中显式调度它们自己的逻辑流。逻辑流被模型化为状态机，数据到达文件描述符（发起文件读写请求成功后会返回一个文件描述符）后，主程序显式地从一个状态转换到另一个状态。因为程序是一个单独的进程，所以所有的流都共享一个地址空间。
- **线程**。**线程是运行在一个单一进程上下文的逻辑流，由内核调度**。线程是上面两种机制的有机结合体，像进程流一样由内核调度，而向I/O多路复用流一样共享同一个虚拟地址空间。

## 基于进程的并发编程
基于进程的并发编程是最简单的也是效率最低的。使用很熟悉的函数类似fork、exec和waitpid。

一个构造并发服务器的自然方法是：**在父进程中接受客户端的连接请求，然后创建一个新的子进程来为每个新客户端提供服务**。

以下有一个很简单的例子：

1、服务器正在监听一个监听描述符（listenfd(3)）的连接请求，client1发起一个连接请求：

![connRequest](/Users/liujie/Desktop/gitbook/image/computer/connRequest.png)

2、服务器接受连接请求后派生了一个子进程，这个子进程获得服务器的描述符表的完整副本。**子进程关闭它的副本中的监听描述符listenfd3，父进程关闭它的已连接描述符connfd4，这样可以释放对于的文件表条目，否则会引起内存泄漏。**

![connOther](/Users/liujie/Desktop/gitbook/image/computer/connOther.png)

3、同理，client2发起连接：

![connRequest2](/Users/liujie/Desktop/gitbook/image/computer/connRequest2.png)



![connOther2](/Users/liujie/Desktop/gitbook/image/computer/connOther2.png)

两个子进程并发的为客户端提供服务。

### 基于进程的并发服务器实现
并发服务器注意的几点：
- 需要进行回收僵死子进程的资源，防止资源浪费。
- 父子进程必须关闭它们各自的connfd（已连接描述符），对父进程尤为重要，以避免内存泄漏。
- 最后因为套接字的文件表表项中的引用计数，直到父子进程的connfd都关闭了，到客户端的连接才会终止。

### 进程实现并发的优劣势
对于父子进程间共享状态信息，进程有一个非常清晰的模型：共享文件表，但是不共享用户地址空间。

**优势：**
进程之间的虚拟地址空间互相独立，保证了数据的独立性。

**劣势：**
1. 独立的地址空间使得进程共享状态信息变得更加困难。为了共享信息，它们必须使用显式的IPC（进程间通信）机制。
2. 性能开销大，因为进程控制和IPC的开销很高。

### IPC进程间通信机制
IPC有很多种实现形式：
- waitpid函数和信号是基本的IPC机制，它们允许进程发送小消息到同一主机的其他进程；
- 套接字接口是一种重要且特殊的IPC形式，它允许不同主机上的进程交换任意的字节流；（Unix IPC通常指同一主机）
- Unix IPC包括：**管道、先进先出（FIFO）、系统V共享内存（通过共享内存块交换信息，最高效）、系统V信号量（semaphore）**。（后面会单独分析）

## 基于I/O多路复用的并发编程
**I/O多路复用技术（I/O multiplexing）基本思路是使用select函数，要求内核挂起进程，只有在一个或多个I/O事件发生后，才将控制返回给应用程序。**

select是一个复杂的函数，有许多不同的使用场景，这里只讨论：等待一组描述符准备好读：

```
#include <sys/select.h>
int select(int n, fd_set *fdset, NULL, NUll, NULL);
返回：返回已经准备好的描述符的非零个数，若出错则为-1。

处理描述符集合fd_set的宏：
FD_ZERO(fd_set *fdset); //清空描述符集合
FD_CLR(int fd, fd_set *fdset); //清理一个指定位上的描述符
FD_SET(int fd, fd_set *fdset); //设置一个位上的描述符
FD_ISSET(int fd, fd_set *fdset); //判断一个位上的描述符是否开启
```
**select函数处理类型为fd_set的集合**，也叫做**描述符集合**。逻辑上我们将描述符集合看成一个大小为n的位向量：Bn-1,...，B1，B0，每个位Bk对应于描述符k。当且仅当Bk=1，描述符k才表明是描述符集合的一个元素，同时只允许对描述符集合操作三件事：



1. 分配它们；

2. 将一个此种类型的变量赋值给另一个变量；

3. 用FD_ZERO、FD_SET、FD_CLR和FD_ISSET宏来修改和检查它们。

   

**select函数会一直阻塞，直到读集合中至少有一个描述符准备好可以读。**

下面有一个理解select函数的例子：**使用I/O多路复用同时处理等待监听描述符的连接请求和标准输入上的命令**

```
#include "csapp.h"
void echo(int connfd);
void command(void);

int main(int argc, char **argv) 
{
    int listenfd, connfd; //设置监听描述符和已连接描述符
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;
    fd_set read_set, ready_set; //设置I/O多路复用的描述符集合和准备好集合

    if (argc != 2) {
	    fprintf(stderr, "usage: %s <port>\n", argv[0]);
	    exit(0);
    }
    listenfd = Open_listenfd(argv[1]);  //开启一个监听描述符

    FD_ZERO(&read_set);              //清空描述符集合 如read_set({})
    FD_SET(STDIN_FILENO, &read_set); //将输入流加入到描述符集合的位上 如read_set({0})
    FD_SET(listenfd, &read_set);     //将监听描述符加入到描述符集合的位上 如read_set({0,3})

    while (1) {
    	ready_set = read_set; //重新将描述符集合赋值给准备好集合 如ready_set({0,3})
    	Select(listenfd+1, &ready_set, NULL, NULL, NULL); //一直阻塞，直到有可以读的描述符，然后返回对应刻度的准备好集合 如ready_set({0})
    	if (FD_ISSET(STDIN_FILENO, &ready_set)) //判断标准输入是否准备好可以读
    	    command();
    	if (FD_ISSET(listenfd, &ready_set)) { //判断监听描述符是否准备好可以读
                clientlen = sizeof(struct sockaddr_storage); 
    	    connfd = Accept(listenfd, (SA *)&clientaddr, &clientlen);
    	    echo(connfd); 
    	    Close(connfd);
    	}
    }
}
//标准输入流的命令执行过程
void command(void) {
    char buf[MAXLINE];
    if (!Fgets(buf, MAXLINE, stdin))
	    exit(0); /* EOF */
    printf("%s", buf); /* Process the input command */
}
/* $end select */
```
### 基于I/O多路复用的并发事件驱动器
**I/O多路复用可以用作并发事件驱动（event-driven）程序的基础，在事件驱动程序中，某些事件会导致流向前推进**。一般的思路就是将**逻辑流模型化为状态机**。

**状态机**就是一组状态（state）、输入事件和转移，其中转移是将状态和输入事件映射到状态（简单说就是另一个结果状态或事件）。每个转移是将一个（输入状态、输入事件）对映射到一个输出状态。

![IOMultiState](/Users/liujie/Desktop/gitbook/image/computer/IOMultiState.png)

## 基于线程的并发编程
**线程（Thread）就是运行在进程上下文中的逻辑流。** 线程由内核自动调度，每个线程都有它自己的**线程上下文，包括唯一的整数线程ID（Thread ID，TID）、栈、栈指针、程序计数器、通用目的寄存器和条件码**。

所有的运行在一个进程上下文中的**线程共享该进程的整个虚拟地址空间**。包括它的**代码、数据、堆、共享库和打开的文件**。

### 线程的执行模型
多线程的执行模型在某些方面和多进程的执行模型是相似的。但在一些重要的方面，线程执行是不同与进程的。因为一个线程的上下文要比一个进程的上下文小得多，**线程的上下文切换要比进程的上下文切换快得多**。另一个方面就是**线程不像进程那样严格按照父子层次来组织**，和一个进程相关的线程组成一个对等（线程）池，独立于其他线程创建的线程。

主线程和其他线程的区别仅在于它总是进程中第一个运行的线程。对等（线程）池概念主要影响是，一个线程可以杀死它的任何对等线程，或者等待它的任意对等线程终止，每个对等线程都能读写相同的共享数据。

每个进程开始生命周期时都是一个单一线程，这个线程称之为**主线程**（main thread）。在某一时刻，主线程创建一个**对等线程**（peer thread），从这个时间点开始两个线程就开始并发地运行。

![multiThread](/Users/liujie/Desktop/gitbook/image/computer/multiThread.png)

### Posix线程
Posix线程（Pthreads）是在C语言程序中处理线程的一个标准接口，适用于所有linux系统。定义了大约60个函数允许程序创建、杀死和回收线程，与对等线程安全地共享数据，还可以通知对等线程系统状态的变化。

下面展示了一个简单的Pthreads程序，主线程创建对等线程，然后等待它的终止，主线程检测到对等线程终止后通过调用exit终止该进程：

```
#include "csapp.h"
void *thread(void *vargp);                    //line:conc:hello:prototype

int main()                                    //line:conc:hello:main
{
    pthread_t tid;                            //声明本地变量存放对等线程ID
    Pthread_create(&tid, NULL, thread, NULL); //创建一个新的对等线程，成功返回后主线程和对等线程同时运行
    Pthread_join(tid, NULL);                  //主线程等待对等线程终止
    exit(0);                                  //终止该进程中的所有线程，终止进程
}

void *thread(void *vargp) 
{
    printf("Hello, world!\n");                 
    return NULL;                               //终止对等线程
}                                           
/* $end hello */
```
### 创建线程
**线程通过调用pthread_create函数来创建其他线程。**

```
typedef void *(func)(void *);
int pthread_create(pthread_t *tid, pthread_attr_t *attr, func *f, void *arg);
返回：成功返回0，出错返回非零
```
pthread_create函数创建一个新的线程，并带着一个输入变量arg，在新线程的上下文中运行线程例程f，可以用attr参数来改变新创建线程的默认属性。返回时tid包含新创建线程的ID。

新线程可以**使用pthread_self函数来获得自己的线程ID**：

```
pthread_t pthread_self(void);
返回：调用者的线程ID
```

### 终止线程
一个线程通过以下的方式之一来终止：
- 当顶层的线程例程返回时，线程会隐式地终止；（即顶层线程逻辑流执行完毕）
- 通过调用pthread_exit函数，线程会显式地终止。如果主线程调用pthread_exit，它会等待所有其他对等线程终止，然后再终止主线程和整个进程，返回值为thread_return。
- 某个对等线程调用Linux的exit函数，该函数终止进程以及所有与该进程相关的线程。
- 另一个对等线程通过以当前线程ID作为参数调用pthread_cancel函数来终止当前线程。


```
void pthread_exit(void *thread_return);
```

```
int pthread_cancel(pthread_t tid);
返回：成功返回0，失败返回非零
```
### 回收已终止线程的资源
线程通过调用pthread_join函数等待其他线程终止。

```
int pthread_join(pthread_t tid, void *thread_return);
返回：成功返回0，失败返回非零
```
**pthread_join会一直阻塞，直到线程tid终止，将线程例程返回的通用（void*）指针赋值为thread_return指向的位置，然后回收已终止线程占用的所有内存资源**。

### 分离线程
**在任何一个时间点上，线程是可结合的（joinable）或者是分离的（detached）。**

**可结合的的线程**可以被其他线程回收和杀死，并且在被其他线程回收之前，它的内存资源（例如栈）是不释放的。

**分离的线程**是不能被其他线程回收和杀死的，它的内存资源在它终止时由系统自动释放。

为了减少显式的回收线程和防止内存泄漏，最好使用分离的线程。

```
int pthread_detach(pthread_t tid);
返回：成功返回0，失败返回非零
```
### 初始化线程
**pthread_once函数允许你初始化与线程例程相关的状态**。

```
pthread_once_t once_control = PTHREAD_ONCE_INIT;
int pthread_once(pthread_once_t *once_control, void (*init_routine)(void));
总是返回0
```

### 基于线程的并发服务器
基于线程的并发echo服务器（类似socket服务器，主线程不断等待连接请求，然后创建一个对等线程处理该请求）：

```
#include "csapp.h"

void echo(int connfd);
void *thread(void *vargp);

int main(int argc, char **argv) 
{
    int listenfd, *connfdp;
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;
    pthread_t tid; 

    if (argc != 2) {
    	fprintf(stderr, "usage: %s <port>\n", argv[0]);
    	exit(0);
    }
    listenfd = Open_listenfd(argv[1]); //打开监听连接描述符

    while (1) {
        clientlen=sizeof(struct sockaddr_storage);
    	connfdp = Malloc(sizeof(int));
    	*connfdp = Accept(listenfd, (SA *) &clientaddr, &clientlen); //获取已连接描述符的指针，并在下面创建对等线程的时候将已连接描述符传递给它
    	Pthread_create(&tid, NULL, thread, connfdp);//创建对等线程处理连接请求
    }
}

/* Thread routine */
void *thread(void *vargp) 
{  
    int connfd = *((int *)vargp);//对等线程间接引用已连接描述符指针，并赋值给局部变量
    Pthread_detach(pthread_self()); //分离当前的对等线程，以保证终止时资源被内核自动回收
    Free(vargp);                    //line:conc:echoservert:free
    echo(connfd);
    Close(connfd);
    return NULL;
}
/* $end echoservertmain */
```
## 多线程程序中的共享变量
线程很有吸引力的一个点在于是多个线程很容易共享相同的程序变量。

**我们在理解一个变量是否是共享的时候，需要先理解以下问题**：
1. 线程的基础内存模型是什么？
2. 根据这个模型，变量实例是如何映射到内存的？
3. 那么有多少线程会引用这些实例？

一个变量是共享的，当且仅当多个线程引用这个变量的某个实例。

### 线程的内存模型
一组并发线程运行在一个进程的上下文中。

**每个线程都有自己独立的线程上下文，包括线程ID、栈、栈指针、程序计数器、条件码和通用目的寄存器。每个线程和其他线程一起共享进程上下文的剩余部分。这部分包括整个用户虚拟地址空间，它是由只读文本（代码）、读/写数据、堆以及所有的共享库代码和数据区域组成。线程也共享相同的打开文件集合。**

### 将变量映射到内存
多线程C语言中根据他们的存储类型被映射到虚拟内存：
- 全局变量。
- 本地自动变量。
- 本地静态变量。（Java中不允许定义局部静态变量）

### 共享变量
一个变量是共享的，当且仅当多个线程引用这个变量的某个实例。

## 用信号量同步线程
**共享变量十分便利，但也带来了同步错误（synchronization error）的可能性。**（多线程操作同一个共享变量的值）

### 进度图
略

### 信号量
**无论哪种并发机制，同步对共享数据的并发访问都是一个困难的问题，提出信号量的P和V操作就是为了帮助解决这个问题。**

**信号量操作可以用来提供对共享数据的互斥访问，也对诸如生产者-消费者程序中有限缓冲区和读者-写者系统中的共享对象这样的资源访问进行调度。**

信号量s是具有非负整数值的全局变量，只能由两种特殊的操作来处理，这两种操作称为P和V：
- P(s):如果s是非零的，那么P将s减1，并立即返回。如果s为零，那么久挂起这个线程，直到s变为非零，而一个V操作会重启这个线程。在重启之后，P操作将s减1，并将控制返回给调用者。
- V(s):V操作将s加1。如果有任何线程阻塞在P操作等待s变为非零，那么V操作会重启这些线程中的一个，然后该线程将s减1，完成它的P操作。

P中的判断和减1操作是原子的不可分割，即一旦s为非零，那么一定会将s减1，不可中断。V中的加1操作也是不可分割的，也就是加载、加1和存储信号量的过程没有中断。V重启的一个线程不可预测。

P和V的操作保证了一个正在运行中的程序的信号量决不可能是负值，这个属性称为**信号量的不变性**。

#### 使用信号量来实现互斥加锁
通过信号量pv机制可以使用很简单的方式来确保对共享变量的互斥访问。以这种方式保护共享变量的信号量叫做二元信号量，因为它的值总是0或1，以提供互斥为目的的二元信号量常常也称为**互斥锁（mutex）**。

P操作称为对互斥锁加锁，V操作称为对互斥锁解锁。如以下操作：

```
for(int i = 0; i < niters; i++){
    P(&mutex);
    cnt++;
    V(&mutex);
}
```
#### 利用信号量来调度共享资源
**除了提供互斥之外，信号量的另一个重要的作用是调度对共享资源的访问**。这种场景中，一个线程用信号量操作来通知另一个线程，程序中的某个条件已经为真了。

1. 生产者-消费者问题
![producer-customer](/Users/liujie/Desktop/gitbook/image/computer/producer-customer.png)
生产者和消费者共享一个由n个槽的有限缓冲区。生产者反复生成新的项目，并把他们插入到缓冲区。消费者不断地从缓冲区中取出项目并消费。

**因为插入和取出都涉及到对共享变量，所以必须保证对缓冲区的访问时互斥的，但因为缓冲区是有限的，所以还需要调度对缓冲区的访问。**（现实系统中很普遍，如一个多媒体系统，生产者编码视频帧，而消费者解码并在屏幕上展示，缓冲区的目的是为了减少视频流的抖动，这种抖动是由各个帧的编码和解码时与数据相关的差异引起的）

2. 读者-写者问题
读者-写者问题是对于互斥问题的概括。在多线程下，有些线程只读对象，而有些线程只修改对象。修改对象的线程叫做写者，只读对象的线程叫做读者。

写者必须拥有对对象的独占的访问，而读者可以和无限多个其他的读者共享对象。

#### 其他同步机制
Java线程使用一种叫做**Java监控器（Java Monitor）**的机制来同步的，它提供了对信号量互斥和调度能力的更高级别的抽象；实际上，监控器可以用信号量来实现。

#### 基于线程的事件驱动程序
![eventThread](/Users/liujie/Desktop/gitbook/image/computer/eventThread.png)

上面是一个简单的并发服务器

从上面可以看出，I/O多路复用不是编写事件驱动程序的唯一方法。上面是一个事件驱动服务器，带有主线程和工作者线程的简单状态机。

主线程：
- 两种状态：等待连接状态、等待可用缓冲区槽位
- 两个I/O事件：连接请求到达、缓冲区槽位变为可用
- 两个转换：接收连接请求、插入缓冲区项目

工作者线程：
- 一个状态：等待可用的缓冲区项目
- 一个I/O事件：缓冲区项目变为可用
- 一个转换：取出缓冲区项目