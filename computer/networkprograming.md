# 网络编程

# 客户端-服务端编程模型
**每个网络应用都是基于客户端-服务端模型的**。采用这种模型，一个应用是由一个服务器进程和一个或多个客户端进程组成。服务器管理某种资源，并且通过操作这种资源来为它的客户端提供某种服务。

客户端-服务器模型中的基本操作是**事务**（只是形容一些列步骤），一个事务由以下四步组成：
1. 当一个客户端需要服务时，它向服务器发送一个请求，发起一个事务。
2. 服务器收到请求后，解释它，并以适当的方式操作它的资源。
3. 服务端给客户端一个响应，并等待下一个请求。
4. 客户端收到响应并处理它。

![clientServer](/Users/liujie/Desktop/gitbook/image/computer/clientServer.png)

# 网络
客户端和服务器通常运行在不同的主机上，并且通过计算机网络的硬件和软件资源来通信。

对主机而言，**网络只是又一种I/O设备，是数据源和数据接收方**。而网络的实现来源于物理硬件和软件协议的共同作用。

## 物理硬件方面
**一个插到I/O总线扩展槽的适配器提供了到网络的物理接口**。从网络上接收到的数据从适配器经过I/O和内存总线复制到内存，通常是通过DMA发送。同理，数据也能从内存复制到网络。

![networkIO](/Users/liujie/Desktop/gitbook/image/computer/networkIO.png)

物理上而言，网络是一个按照地理远近组成的层次系统。最底层是LAN（Local Area Network）局域网，而**目前最流行的局域网技术是以太网（Ethernet）**。一个**以太网段**（Ethernet segment）**包括一些电缆（通常是双绞线）和一个叫做集线器的小盒子**。电缆的一端连接到主机的适配器，一端连接到集线器的一个端口上，集线器不加分辨的将从一个端口上收到的每个位复制到其他所有的端口上，因此每台主机都能看到每个位。

每个以太网适配器都有一个全球唯一的48位地址，它存储在这个适配器的非易失性存储器上。一台主机可以发送一段位（称为帧（frame））到这个网段内的其他任何主机。每个帧包含：
- 固定数量的头部位（header）：用以表示此帧的来源和目标地址以及此帧的长度；
- 数据位的有效载荷（payload）；

每个主机都能看到这个帧，但只有目标主机实际读取它。

使用一些电缆和叫做**网桥（bridge）**的小盒子，多个以太网段可以连接成较大的局域网，称为**桥接以太网（bridged Ethernet）**。

![networkBridge](/Users/liujie/Desktop/gitbook/image/computer/networkBridge.png)

在更高层次中，多个不兼容的局域网可以通过叫做**路由器（router）**的特殊计算机连接起来，组成一个internet（互联网络）。每台路由器对于它所连接到的每个网络都有一个适配器（端口）。路由器也能连接高速点到点的电话连接，这是称为WAN（Wide-Area Network，广域网）的网络示例。

> internet和Internet：我们一般用小写的internet描述一般概念，而用大写字母的Internet来描述一种具体的实现，也就是所谓的全球IP因特网。

## 软件协议方面
**互联网络至关重要的特性**是：它能由采用完全不同和不兼容技术的各种局域网和广域网组成。但是，如何能够让某台源主机跨过所有这些不兼容的网络发送数据位到另一台目的主机呢？

**解决办法就是一层运行在每台主机和路由器上的协议软件**，它消除了不同网络之间的差异。这个软件实现一种协议，这种协议控制主机和路由器如何协同工作来实现数据传输。这种协议必须提供两种基本能力：
- **命名机制**：不同的局域网技术由不同和不兼容的方式来为主机分配地址。**互联网络协议通过定义一种一致的主机地址格式来消除这种差异，这个地址唯一标识了这台主机**。
- **传送机制**：在电缆上编码位和将这些位封装成帧方面，不同的联网技术有不同和不兼容的方式。**互联网络协议通过定义一种把数据位捆扎成不连续的片（称为包）的统一方式，从而消除了这种差异**。一个包由包头和有效载荷组成。其中包头包含了包的源主机地址和目标主机地址以及包的大小，有效载荷包括从源主机发出的数据位。

下图展示了数据是如何从一台主机传送到另一台主句的（PH：互联网络包头；FH1：LAN1的帧头；FH2：LAN2的帧头）

![networkTransfer](/Users/liujie/Desktop/gitbook/image/computer/networkTransfer.png)

上图只是一个简单的示例，实际传输过程还会遇到很多问题：不同的网络有不同的帧大小的最大值怎么办？路由器如何判断向哪里转发帧？网络拓扑发生变化时如何通知路由器？包丢失了怎么办？但我们还是根据示例抓住了**互联网思想的精髓：封装是关键**。

# 全球IP因特网
**全球IP因特网是最著名和最成功的互联网络实现。**

下面展示一个因特网客户端-服务端应用程序的基本硬件和软件组织：

![networkBasic](/Users/liujie/Desktop/gitbook/image/computer/networkBasic.png)

## TCP/IP简单介绍
每台因特网主机都运行实现TCP/IP协议（Transmission Control Protocol/Internet Protocol，传输控制协议/互联网络协议）的软件。因特网的客户端和服务器混合使用套接字接口函数和Uninx I/O函数来进行通信。**通常套接字函数实现为系统调用，这些系统调用会陷入内核，并调用各种内核模式的TCP/IP函数**。

TCP/IP实际是一个协议族，其中每一个都提供不同的功能。例如：
- IP协议：提供基本的命名方法和递送机制，这种递送机制能够从一台因特网主机往其他主机发送包，也叫做**数据报（datagram）**。但IP机制某种意义上是不可靠的，因为，如果数据报在网络中丢失或重复，它并不会试图恢复。
- UDP（Unreliable Datagram Protocol，不可靠数据报协议）协议：稍微扩展了IP协议，这样一来包可以在进程间而不是主机间传送。
- TCP（Transmission Control Protocol，传输控制协议）协议：TCP是一个构建在IP之上的复杂协议，提供了进程间可靠的全双工连接。一般我们将TCP/IP看做是一个单独的整体协议。

在程序员的角度，我们可以将因特网看做是一个世界范围的主机集合，满足以下特性：
- 主机集合被映射为一组32位的IP地址；
- 这组IP地址被映射为一组称为因特网域名的标识符；
- 因特网主机上的进程能够通过连接（connection）和任何其他因特网主机上的进程通信。

> IPv4和IPv6：最初的因特网协议，使用32位地址，称为因特网协议版本4（IPv4），后面提出了一个新版本的IP，称为因特网协议版本6（IPv6），它使用128位地址，意在替代IPv4。

## IP地址
**一个IP地址就是一个32位无符号整数。**

TCP/IP为任意整数数据项定义了统一的**网络字节顺序（network byte order）（大端字节顺序）**，例如IP地址，它放在包头中跨过网络被携带。在IP地址结构中存放的地址总是以（大端法）网络字节顺序存放的，即使主机字节顺序（host byte order）是小端法。Unix提供了对应的函数实现在网络和主机字节顺序间转换。

IP地址通常是以一种称为**点分十进制表示法**来表示的。这里，每个字节由他的十进制表示，并且用句点和其他字节分开。例如：255.255.255.255就是地址0xffffffff的点分十进制表示（255转换成16进制：ff,得到，ff.ff.ff.ff，合并就是0xffffffff,转换成二进制就是32位）。

## 因特网域名
因特网客户端和服务器互相通信时使用的是IP地址，但对于人们而言IP太难记住，所以因特网定义了更加人性化的域名（domain name），以及一种将域名映射到IP地址的机制。

域名集合形成一个层次结构，每个域名编码了它在这个层次中的位置。

![internetStruct](/Users/liujie/Desktop/gitbook/image/computer/internetStruct.png)

因特网定义了域名集合和IP地址集合之间的映射。在1988年之前，这个映射都是通过一个叫做HOSTS的文本文件来手工维护的。从那之后，这个映射是通过分布在世界范围内的数据库（称为DNS（Domain Name System，域名系统））来维护的。

## 因特网连接
**因特网客户端和服务器通过在连接上发送和接收字节流来通信。** 从连接一对进程的意义上而言，连接是**点对点的**；从数据可以同时双向流动的角度来说，连接是**全双工的**。

一个**套接字**是连接的一个端点，每个套接字都有相应的套接字地址，是由一个因特网地址和一个16位的整数端口（软件端口和硬件不一样）组成的，用“**地址：端口**”来表示。

当客户端发起一个连接请求时，客户端套接字地址中的端口是内核自动分配的，称为**临时端口（ephemeral port）**。然而，服务器套接字地址中的端口通常是某个**知名端口**，是对应这个具体服务的。例如web服务器通常使用80端口，电子邮件服务器使用端口25。同时，每个知名的端口的服务都有一个对应的**知名的服务名**。例如，web服务器的知名名字是http，email的知名名字是smtp。

文件/etc/services包含一张这台机器提供的知名名字和知名端口之间的映射。

**一个连接是由它两端的套接字地址唯一确定的**。这对套接字地址叫做**套接字对（socket pair）**，由下列元组来表示：(cliaddr:cliport,seraddr:servport)

# 套接字接口
套接字接口是一组函数，它们和Unix I/O函数结合起来，用以创建网络应用。

下图是基于套接字接口的网络应用概述：

![socketfd](/Users/liujie/Desktop/gitbook/image/computer/socketfd.png)

## 套接字的地址结构
**从Linux内核的角度来看，一个套接字就是通信的一个端点；从Linux程序的角度来看，套接字就是一个由相应描述符的打开文件。**

因特网的套接字地址存放在下面所示的类型为socketaddr_in的16字节结构中。
```
/* IP socket address structure */
struct sockaddr_in  {
    uint16_t        sin_family;  /* 使用协议(总是：AF_INET) */
    uint16_t        sin_port;    /* 网络字节顺序的端口号 */
    struct in_addr  sin_addr;    /* 网络字节顺序的IP地址 */
    unsigned char   sin_zero[8]; /* Pad to sizeof(struct sockaddr) */
};

/* Generic socket address structure (for connect, bind, and accept) */
struct sockaddr {
    uint16_t  sa_family;    /* Protocol family */
    char      sa_data[14];  /* Address data  */
};	
```
## socket函数
客户端和服务器使用socket函数来创建一个**套接字描述符**（socket descriptor）。

```
int socket(int domain, int type, int protocol);
返回：若成功返回非负描述符，若出错则返回-1；
```
如果想使套接字称为连接的一个端点，就用如下硬编码的参数来调用socket函数：

```
clientfd = Socket(AF_INET, SOCK_STREAM, 0);
```
其中AF_INET表明我们正在使用32位IP地址，而SOCK_STREAM表示这个套接字是连接的一个端点。

socket返回的clientfd描述符仅是部分打开的，还不能用于读写。如何完成打开套接字的工作，取决于我们是客户端还是服务器。下面描述我们是客户端时如何完成打开套接字的工作。


## connect函数
**客户端通过调用connect函数来建立和服务器的连接。**

```
int connect(int clientfd, const struct sockaddr *addr, socklen_t addrlen);
返回：成功返回0，失败返回-1
```
connect函数试图与套接字地址为addr的服务器建立一个因特网连接，其中addrlen是sizeof(sockaddr_in)。connect函数会阻塞，一直到连接成功建立或发生错误。

## bind函数
剩下的套接字函数bind、listen和accept，服务器用它们来和客户端建立连接。

```
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
返回：成功返回0，失败返回-1
```
bind函数告诉内核将addr中的服务器套接字地址和套接字描述符sockfd联系起来。

## listen函数
客户端时发起请求的主动实体，服务器是等待来自客户端的连接请求的被动实体。默认情况下，内核会认为socket函数创建的描述符对应于**主动套接字**（active socket），它存在于一个连接的客户端。服务器调用listen函数告诉内核，描述符是被服务器而不是客户端使用的。

```
int listen(int sockfd, int backlog);
返回：成功返回0，失败返回-1
```
listen函数将sockfd从一个主动套接字转化为一个**监听套接字**（listening socket），该套接字可以接受来自客户端的连接请求。backlog参数暗示了内核在开始拒绝连接请求之前，队列中要排队的未完成的连接请求数量。（backlog参数确切的含义要求对TCP/IP协议的理解，这里不深入，通常我们会把它设置为一个较大的值，如1024）

## accept函数
服务器通过调用accept函数来等待来自客户端的连接请求。

```
int accept(int listenfd, struct sockaddr *addr, int *addrlen);
返回：成功返回非负连接描述符，出错返回-1
```
accept函数等待来自客户端的连接请求到达**监听描述符**listenfd，然后在addr中填写客户端的套接字地址，并返回一个**已连接描述符**，这个描述符可被用来利用Unix I/O函数与客户端通信。
- 监听描述符：作为客户端连接请求的一个端点，它通常被创建一次，并存在于服务器的整个生命周期。（如web服务器的80端口）
- 已连接描述符：是客户端和服务器之间已经建立起来了的连接的一个端点，服务器每次接受连接请求时都会创建一次，它只存在于服务器为一个客户端服务的生命周期内。

![socketConnect](/Users/liujie/Desktop/gitbook/image/computer/socketConnect.png)

> 为何要有监听描述符和已连接描述符之间的区别？ 区分这两者，使得我们可以建立并发服务器，它能够同时吹许多客户端连接。例如，每次一个连接请求到达监听描述符时，我们可以派生一个新的进程，它通过已连接描述符与客户端通信。

## 主机和服务的转换
**Linux提供了一些强大的函数（称为getaddrinfo和getnameinfo）实现二进制套接字地址结构和主机名、主机地址、服务名和端口号的字符串表示之间的相互转化。** 当和套接字接口一起使用时，这些函数能使我们编写独立于任何特定版本的IP协议的网络程序。

### getaddrinfo函数
getaddrinfo函数将主机名、主机地址、服务名和端口号的字符串表示转化成套接字地址结构。可重入和协议无关的。

```
#include<netdb.h>
int getaddrinfo( const char *hostname, const char *service, const struct addrinfo *hints, struct addrinfo **result );
//返回值：成功 返回 0；错误 返回非零错误码
```
参数：
- hostname:一个主机名或者地址串(IPv4的点分十进制串或者IPv6的16进制串)
- service：服务名可以是十进制的端口号，也可以是已定义的服务名称，如ftp、http等
- hints： hints是一个模板，它只使用ai_family，ai_flags,以及ai_socktype成员，用来过滤地址。剩下的整数成员必须被设置成0.
- result：本函数通过result指针参数返回一个指向addrinfo结构体链表的指针。

addrinfo 结构：

```
struct addrinfo {
  int               ai_flags;       /* customize behavior */
  int               ai_family;      /* address family */
  int               ai_socktype;    /* socket type */
  int               ai_protocol;    /* protocol */
  socklen_t         ai_addrlen;     /* length in bytes of address */
  struct sockaddr  *ai_addr;        /* address */
  char             *ai_canonname;   /* canonical name of host */
  struct addrinfo  *ai_next;        /* next in list */
 };
```
![addinfo](/Users/liujie/Desktop/gitbook/image/computer/addinfo.png)

如果本函数返回成功，那么由result参数指向的变量已被填入一个指针，它指向的是由其中的ai_next成员串联起来的addrinfo结构链表。可以导致返回多个addrinfo结构的情形有以下2个： 
1. 如果与hostname参数关联的地址有多个，那么适用于所请求地址簇的每个地址都返回一个对应的结构。 
2. 如果service参数指定的服务支持多个套接口类型，那么每个套接口类型都可能返回一个对应的结构，具体取决于hints结构的ai_socktype成员

### getnameinfo函数
getnameinfo函数和getaddrinfo是相反的，将一个套接字地址结构转换成相应的主机和服务名字字符串。可重入和协议无关的。

```
int getnameinfo(const struct sockaddr *sa, socklen_t salen, char *host, size_t hostlen, char *service, size_t servlen, int flags);
返回：成功返回0，错误返回失败代码
```
参数：
- sa:指向大小为salen字节的套接字地址结构
- host：指向大小为hostlen字节的缓冲区
- service：指向大小为servlen字节的缓冲区
- flags：NI_NUMERICHOST-getnameinfo默认试图返回host中的域名，设置该标识会使得函数返回一个数字地址字符串（IP）；NI_NUMERICSERV：getnameinfo默认检查/etc/services，如果肯会返回服务名而不是端口号，设置该标志会使函数跳过查找，简单返回端口号。


示例：实现一个解析域名的程序，效果：(类似nslookup)

```
linux> ./hostinfo www.baidu.com
199.16.23.123
196.231.23.12
112.23.23.1
```

```
/* $begin hostinfo */
#include "csapp.h"

int main(int argc, char **argv) 
{
    struct addrinfo *p, *listp, hints;
    char buf[MAXLINE];
    int rc, flags;

    if (argc != 2) {
	fprintf(stderr, "usage: %s <domain name>\n", argv[0]);
	exit(0);
    }

    /* 根据域名获取对应的addrinfo记录 */
    memset(&hints, 0, sizeof(struct addrinfo));                         
    hints.ai_family = AF_INET;       /* IPv4 only */        //line:netp:hostinfo:family
    hints.ai_socktype = SOCK_STREAM; /* Connections only */ //line:netp:hostinfo:socktype
    if ((rc = getaddrinfo(argv[1], NULL, &hints, &listp)) != 0) {
        fprintf(stderr, "getaddrinfo error: %s\n", gai_strerror(rc));
        exit(1);
    }

    /* 遍历展示getaddrinfo返回的listp链表中对应套接字地址对应的IP */
    flags = NI_NUMERICHOST; /* 展示IP地址而不是主机名 */
    for (p = listp; p; p = p->ai_next) {
        getnameinfo(p->ai_addr, p->ai_addrlen, buf, MAXLINE, NULL, 0, flags);
        printf("%s\n", buf);
    } 

    /* 释放getaddrinfo函数返回保存在内存中的链表 */
    Freeaddrinfo(listp);

    exit(0);
}
/* $end hostinfo */
```
## 套接字接口辅助函数
上面的套接字连接的建立还是很复杂，所以可以使用高级的辅助封装函数：open_clientfd和open_listenfd。

### open_clientfd
客户端调用open_clienfd与服务端建立连接。

```
int open_clientfd(char *hostname, char *port);
返回：成功则为描述符，失败-1
```
socket和connect过程的参数都是用getaddrinfo自动产生的，这使得我们代码干净可移植。

代码实现为：

```
/* $begin open_clientfd */
int open_clientfd(char *hostname, char *port) {
    int clientfd, rc;
    struct addrinfo hints, *listp, *p;

    /* 获取潜在服务器地址列表 */
    memset(&hints, 0, sizeof(struct addrinfo));
    hints.ai_socktype = SOCK_STREAM;  /* 面向连接 */
    hints.ai_flags = AI_NUMERICSERV;  /* 使用十进制数字端口 */
    hints.ai_flags |= AI_ADDRCONFIG;  /* Recommended for connections */
    if ((rc = getaddrinfo(hostname, port, &hints, &listp)) != 0) {
        fprintf(stderr, "getaddrinfo failed (%s:%s): %s\n", hostname, port, gai_strerror(rc));
        return -2;
    }
  
    /* 遍历列表直到找到可以成功连接的那一个 */
    for (p = listp; p; p = p->ai_next) {
        /* 使用socket函数建立一个套接字描述符 */
        if ((clientfd = socket(p->ai_family, p->ai_socktype, p->ai_protocol)) < 0) 
            continue; /* 创建失败遍历下一个地址 */

        /* 使用connect连接到服务器 */
        if (connect(clientfd, p->ai_addr, p->ai_addrlen) != -1) 
            break; /* 成功与服务器建立连接，跳出循环 */
        if (close(clientfd) < 0) { /* 连接服务器失败，关闭客户端描述符 */  //line:netp:openclientfd:closefd
            fprintf(stderr, "open_clientfd: close failed: %s\n", strerror(errno));
            return -1;
        } 
    } 

    /* 释放getaddrinfo返回的addrinfo列表，防止内存泄漏（因为是动态内存占用） */
    freeaddrinfo(listp);
    if (!p) /* 所有可连接列表都失败了 */
        return -1;
    else    /* 成功建立连接的客户端描述符 */
        return clientfd;
}
/* $end open_clientfd */
```

### open_listenfd
服务器调用open_listenfd创建一个监听描述符，准备好接收连接请求。

```
int open_listenfd(char *port);
返回：若成功返回描述符，出错为-1
```
open_listenfd函数打开和返回一个监听描述符，这个描述符准备好在端口port上接收连接请求。

代码实现为：

```
* $begin open_listenfd */
int open_listenfd(char *port) 
{
    struct addrinfo hints, *listp, *p;
    int listenfd, rc, optval=1;

    /* 获取潜在服务器地址列表，这里hostname为NULL */
    memset(&hints, 0, sizeof(struct addrinfo));
    hints.ai_socktype = SOCK_STREAM;             /* 接受连接 */
    hints.ai_flags = AI_PASSIVE | AI_ADDRCONFIG; /* 允许任何IP地址连接 */
    hints.ai_flags |= AI_NUMERICSERV;            /* 使用端口号 */
    if ((rc = getaddrinfo(NULL, port, &hints, &listp)) != 0) {
        fprintf(stderr, "getaddrinfo failed (port %s): %s\n", port, gai_strerror(rc));
        return -2;
    }

    /* 遍历地址列表，找到我们可以bind的那一个 */
    for (p = listp; p; p = p->ai_next) {
        /* Create a socket descriptor */
        if ((listenfd = socket(p->ai_family, p->ai_socktype, p->ai_protocol)) < 0) 
            continue;  /* Socket failed, try the next */

        /* 消除绑定中的“已使用的地址”错误，使用setsockopt配置服务器，使得服务器能够被终止、重启和立即开始接收连接请求，因为一个重启的服务器默认30秒内 拒绝客户端的连接请求*/
        setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR,    //line:netp:csapp:setsockopt
                   (const void *)&optval , sizeof(int));

        /* 将套接字描述符绑定到服务器套接字地址 */
        if (bind(listenfd, p->ai_addr, p->ai_addrlen) ** 0)
            break; /* 成功绑定，结束循环 */
        if (close(listenfd) < 0) { /* 绑定失败关闭套接字描述符 */
            fprintf(stderr, "open_listenfd close failed: %s\n", strerror(errno));
            return -1;
        }
    }


    /* Clean up */
    freeaddrinfo(listp);
    if (!p) /* No address worked */
        return -1;

    /* 使其成为一个准备接受连接请求的侦听套接字 */
    if (listen(listenfd, LISTENQ) < 0) {
        close(listenfd);
	    return -1;
    }
    return listenfd;
}
/* $end open_listenfd */
```
# Web服务器
略主要说明HTTP协议