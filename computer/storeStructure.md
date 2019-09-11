# 现代计算机存储层次结构

我们可以对系统有一个简单的模型认知，CPU执行指令，而存储系统为COU存放指令和数据。这个简单模型中，存储器系统是一个线性的字节数组，而CPU能够在一个常数时间内访问每个存储器的位置。但是，现代计算机系统的实际工作模式更为复杂。

实际上，**存储器系统（memory system）**是一个具有不同容量、成本和访问时间的存储设备的层次结构。类似一个金字塔形的层次：
- CPU寄存器：保存着最常用的数据，也是容量最小的；（指令执行期间，0个周期就能访问到对应数据）
- 高速缓存存储器（cache memory）（L1，L2，L3）：作为一部分存储在相对慢速的主存储器（main memory）中的数据和指令的缓冲区域；（指令执行期间，4~75个周期就能访问到对应数据）
- 主存储器（main memory）：作为一部分存储在慢速的磁盘中的数据的缓冲区域；（指令执行期间，几百个周期就能访问到对应数据）
- 磁盘（Disk）：作为存储在通过网络连接的其他机器的磁盘或磁带上的数据的缓冲区域。（指令执行期间，大约几千万个周期才能访问到对应数据）

对于访问性能的这些思想，围绕着计算机程序的一个称为**局部性（locality）** 的基本属性。具有良好局部性的程序倾向于一次又一次的访问相同的数据项集合，或是倾向于访问邻近的数据项集合。具有良好局部性的程序比局部性差的程序更多地倾向于从存储器层次结构中较高层次处访问数据项，因此运行更快。

这里我们需要知道基本的存储技术：SRAM存储器、DRAM存储器、ROM存储器以及旋转的核固态的硬盘。特别是高速缓存存储器，它是作为CPU和主存之间的缓存区域，它对应用程序性能的影响最大。

# 存储技术

## 随机访问存储器
**随机访问存储器（Random-Access Memory，RAM）分为两类：静态的和动态的。静态RAM（SRAM）比动态RAM（DRAM）更快，但也更贵**。
- SRAM用来作为高速缓存存储器，既可以在CPU芯片上，也可以在片下。
- DRAM用来作为主存以及图形系统的帧缓冲区。

一个典型的桌面系统的SRAM不会超过几兆字节，但是DRAM却有几百或几千兆字节。

### 静态RAM：SRAM
**SRAM将每个位存储在一个双稳态的（bistable）存储器单元里。** 每个单元是用一个六晶体管电路来实现的。这个电路有一个这样的属性，**它可以无限期地保持在两个不同的电压配置（configuration）或状态（state）之一**。其他任何状态都是不稳定的-----从不稳定状态开始，电路会迅速地转移到两个稳定状态中的一个。

这里有个简单的钟摆示例，同SRAM单元一样，只有两个稳定的配置和状态：

![dianlu](/Users/liujie/Desktop/gitbook/image/computer/dianlu.png)

由于SRAM存储器单元的双稳态特性，只要有电，它就会永远的保持它的值，即使有干扰来扰乱电压，干扰消除时，电路会立刻恢复到稳定值。

### 动态RAM：DRAM
**DRAM将每个位存储对一个电容的充电。** DRAM存储器可以制造得非常密集---每个单元由一个电容和一个访问晶体管组成。但是与SRAM不同，DRAM存储器对于干肉非常敏感。当电容的电压被扰乱后它就永远不会恢复了，暴露在光线下会导致电容电压改变。（实际上，数码相机和摄像机中的传感器本质上就是DRAM单元的阵列）

只要有供电SRAM就会保持不变，而且与DRAM不同，SRAM不需要刷新。SRAM存取比DRAM快，SRAM对干扰不敏感。代价是SRAM单元比DRAM单元使用更多的晶体管，因而更密集，而且更贵，功耗更大。

### 传统的DRAM
DRAM芯片中的单元（位）被分成d个超单元（supercell），每个超单元都由w个DRAM单元组成。一个d*w的DRAM总共存储了dw位信息。

超单元被组织成一个r行c列的长方形阵列，这里rc=d。每个超单元有形如（i，j）的地址，这里i表示行，j表示列。

下面展示了一个128位16*8的DRAM芯片的高级视图：有d=16个超单元，每个超单元有w=8位，r=4行，c=4列。信息通过称为引脚（pin）的外部连接器流入和流出芯片。每个引脚携带一个1位的信号。下面给出了两组引脚：8个data引脚，他们能传送一个字节到芯片或从芯片传出一个字节；以及2个addr引脚，他们携带2位的行和列超单元地址。其他的携带控制信息的引脚没有显示。

![xinpian](/Users/liujie/Desktop/gitbook/image/computer/xinpian.png)

每个DRAM芯片被连接到某个称为**内存控制器（memory controller）** 的电路，这个电路可以一次传送w位到每个DRAM芯片或一次从每个DRAM芯片传出w位。读取超单元(i,j)内容的流程如下：
- 内存控制器将行地址i发送到DRAM，然后是列地址j。DRAM把超单元(i,j)的内容发回给控制器作为响应。
- 行地址i称为RAS（Row Access Strobe，行访问选通脉冲），列地址j称为CAS（Column Access Strobe，列访问选通脉冲），RAS和CAS共享相同的DRAM地址引脚。

如下，读取（2，1）超单元的内容：

![duqu01](/Users/liujie/Desktop/gitbook/image/computer/duqu01.png)

![duqu02](/Users/liujie/Desktop/gitbook/image/computer/duqu02.png)

DRAM组织成二维阵列而不是线性数组的好处和缺点：

- 二维阵列可以降低芯片上地址引脚的数量。
- 缺点是分两步发送地址，增加了访问时间。

### 内存模块
**DRAM芯片封装在内存模块（memory module）中，它插到主板的扩展槽上。** （core i7系统使用的240个引脚的双列直插内存模块（DIMM），它以64位为块传送数据到内存控制器和从内存控制器传出数据）

**通过将多个内存模块连接到内存控制器，能够聚合成主存。** 这种情况下，当控制器收到一个地址A时，控制器选择包含A的模块k，将A转换成（i，j）的形式，并将（i，j）发送到模块k。

要取出内存地址为A的一个字，内存控制器将A转换成一个超单元地址（i，j），并将它发送到内存模块，然后内存模块再将i和j广播到每个DRAM。作为响应，每个DRAM芯片输出它的（i，j）超单元的8位内容。模块中的电路收集这些输出，并把它们合并成一个64位字，再返回给内存控制器。下面是一个8M*8的DRAM芯片，共存储64MB，这8个芯片编号是0-7，每个芯片的同一位置存储主存的一个字节，8个超单元的字节表示了主存中字节地址A的64位字。

![storelists](/Users/liujie/Desktop/gitbook/image/computer/storelists.png)

### 增强的DRAM
现在已经有很多种DRAM存储器了，但基本上都是基于传统的DRAM单元进行一些优化，提高访问基本DRAM单元的速度。

### 非易失性存储器
如果断电，DRAM和SRAM会丢失它们的信息，从这个意义上来说，它们是易失的。而非易失性存储器就是即使在断电后，仍然保存着它们的信息。如：
- ROM设备是一种非易失性的存储器，一般我们把存储在ROM设备中的程序称为固件，当一个计算机系统通电后，它会运行存储在ROM中的固件。（如PC的BIOS例程）
- 闪存（flash memory）是一类非易失性存储器。（固态硬盘就是一种基于闪存的磁盘驱动器）

### 访问主存
**数据流通过称为总线（bus）的共享电子电路在处理器和DRAM主存之间来来回回。** 每次CPU到主存之间的数据传送都是经过一系列步走来完成的，这些步骤称为**总线事务**。读事务是指从主存传输数据到CPU，写事务是从CPU传送数据到主存。

总线是一组并行的导线，能携带地址、数据和控制信号。具体的传输规则取决于总线的设计。总线设计是计算机系统一个复杂且变化迅速的方面。下面是一个连接CPU和主存的总线结构示例：

![visitMainStore](/Users/liujie/Desktop/gitbook/image/computer/visitMainStore.png)

当CPU执行一个加载数据操作时的流程是什么：movq A，%rax 将地址A的内存加载到寄存器%rax中，CPU芯片上称为总线接口的电路在总线上发起读事务，该事务由以下三个步骤组成：
1. CPU将地址A放到系统总线上，I/O桥将信号传递到内存总线；
2. 主存感觉到内存总线上的地址信号，从内存总线读取地址，从DRAM取出数据字，并将数据写到内存总线；
3. I/O桥将内存总线信号翻译成系统总线信号，然后沿着系统总线传递；
4. CPU感觉到系统总线上的数据，从总线上读数据，并将数据复制到寄存器%rax上。

## 磁盘存储
**磁盘是广为应用的保存大量数据的存储设备，存储数据的数量级可以达到几百到几千千兆字节，而基于RAM的存储器只能与几百到几千兆字节。不过，从磁盘上读信息的时间为毫秒级，比从DRAM读满了10万倍，比从SRAM读满了100万倍。**

### 磁盘构造、磁盘容量、磁盘操作、逻辑磁盘块、连接I/O设备
//略

### 访问磁盘
CPU使用一种称为**内存映射I/O**（memory-mapped I/O）的技术来向I/O设备发射命令。在使用内存映射I/O的系统中，地址空间中有一块地址是为与I/O设备通信保留的。每个这样的地址称为一个I/O端口，当一个设备连接到总线时，它与一个或多个端口相关联（或它被映射到一个或多个端口）。

下图总结了当CPU从磁盘读数据时发生的步骤：

1、cpu通过将命令、逻辑块号和目的内存地址写到磁盘相关联的内存映射地址，发起一个磁盘读；

![cpuRead1](/Users/liujie/Desktop/gitbook/image/computer/cpuRead1.png)

2、磁盘控制器读扇区，并执行到主存的DMA传送；

![cpuRead2](/Users/liujie/Desktop/gitbook/image/computer/cpuRead2.png)

3、当DMA传送完成时，磁盘控制器用中断的方式通知CPU。

![cpuRead3](/Users/liujie/Desktop/gitbook/image/computer/cpuRead3.png)

我们来详细分析一下：

假设磁盘控制器映射到端口0xa0，随后CPU可能通过执行三个对地址0xa0的存储指令，发起磁盘读：
1. 第一个指令是发送一个命令字，告诉磁盘发起一个读，同时还发送了其他的参数，例如当读完成，是否中断CPU；
2. 第二条指令指明应该读的逻辑块号；
3. 第三条指令指明应该存储磁盘扇区内容的主存地址。

当CPU发出了请求后，在磁盘执行读的时候，它通常会做其他的工作。因为一个1GHz的处理器时钟周期为1ns，在用来读磁盘的16ms时间里，它理论上可以处理1600万条指令，所以在传输的时间里等待是对资源的极大地浪费。

在磁盘控制器收到来之CPU的读命令之后，它将逻辑块号翻译成一个扇区地址，读取该扇区的内容，然后将这些内容直接传送到主存，不需要CPU的干涉。**设备可以自己执行读或者写总线事务而不需要CPU干涉的过程，称为直接内存访问（Direct Memory Access，DMA）。这种数据传送称为DMA传送。**

在DMA传送完成后，磁盘扇区的内容被安全地存储在主存中之后，磁盘控制器通过给CPU发送一个中断信号来通知CPU。（中断会发信号到CPU芯片的一个外部引脚上，会导致CPU暂停当前正在做的工作，跳转到一个操作系统的例程，这个程序会记录下I/O已经完成，然后返回CPU中断的地方）

## 固态硬盘
**固态硬盘（Solid State Disk，SSD）是一种基于闪存的存储技术**，在某些情况下是传统旋转磁盘的吸引力的替代产品。

一个SSD封装由一个或多个**闪存芯片和闪存翻译层**组成，闪存芯片替代传统旋转磁盘中的机械驱动器，而闪存翻译层是一个硬件/固件设备，扮演与磁盘控制器相同的角色，将对逻辑块的请求翻译成对底层物理设备的访问。

![ssd](/Users/liujie/Desktop/gitbook/image/computer/ssd.png)

优点：SSD由半导体存储器构成，没有移动的部件，因而随机访问时间比旋转磁盘要快，能耗更低，同时也更结实。

缺点：因为反复写会对闪存有磨损，所以SSD也很容易磨损。价格较贵

# 局部性
是否有良好的**局部性（locality）**是评估一个计算机程序的关键因素。局部性也就是倾向于引用邻近于其他最近引用过的数据项的数据项，或者最近引用过的数据项本身，这种倾向性，被称为**局部性原理**。

局部性通常有两种不同的形式：
- 时间局部性（temporal locality）：一个具有良好时间局部性的程序中，被引用过一次的内存位置很可能在不远的将来再被多次引用；
- 空间局部性（spatial locality）：一个具有良好空间局部性的程序中，如果一个内存位置被引用了一次，那么程序很可能在不远的将来引用附近的一个内存位置。

一般而言，具有良好局部性的程序比局部性差的程序运行得更快。
- 硬件层：局部性原理允许计算机设计者通过引入称为高速缓存存储器的小而快速的存储器来保存最近被引用的指令和数据项，从而提高对主存的访问速度；
- 操作系统级：局部性原理允许系统使用主存作为虚拟地址空间最近被引用块的高速缓存，类似地，操作系统用主存来缓存磁盘文件系统中最近被使用的磁盘块；
- 应用程序层：例如web浏览器将最近被引用的文档放在本地磁盘上，利用的就是时间局部性。大容量的web服务器将最近被请求的文档放在前端磁盘高速缓存中，这些缓存能满足对这些文档的请求，而不需要服务器的干预。

## 对程序数据引用的局部性
我们来举个例子：

```
int sum(int v[N]){
    int i,sum = 0;
    for(i = 0;i<N;i++){
        sum += v[i];
    }
    return sum;
}
```
我们可以看到，变量sum在每次循环中都会被引用一次，所以对于sum来说有好的时间局部性，但是由于sum是标量，即没有空间局部性。同时，数组v的元素是被顺序读取的，一个接一个，按照它们存储在内存中的孙旭。因此对于变量v，函数有很好的空间局部性，但每个元素只被访问一次，所以时间局部性较差。但对于sum函数而言，里面的变量要么有时间局部性，要么有空间局部性，所以可以说sum函数有良好的局部性

## 取指令的局部性
因为程序指令是存放在内存中的，CPU必须取出这些指令，所以我们也能够评价一个程序关于取指令的局部性。

代码区别于程序数据的一个重要属性是在运行时它是不能被修改的。当程序正在执行时，CPU只能从内存中读出它的指令。CPU很少会重写或修改这些指令。

## 小结
局部性的一些简单原则：
- 重复引用相同变量的程序有良好的时间局部性；
- 对于具有步长为k的引用模式的程序（如for循环，以1自增或自减），步长越小，空间局部性越好；
- 对于取指令来说，循环有好的时间和空间局部性。循环体越小，循环迭代次数越多，局部性越好。

# 存储器层次结构
**存储器层次结构**是硬件和软件相互补充的良好成果。

![storeStruct](/Users/liujie/Desktop/gitbook/image/computer/storeStruct.png)

## 存储器层次结构中的缓存
**一般而言，高速缓存是一个小而快速的存储设备，它作为存储在更大、也更慢的设备中的数据对象的缓冲区域。**

**存储器层次结构的中心思想是：对于每个存储层次k，位于k层的更快更小的存储设备作为k+1层的更大更慢的存储设备的缓存**。以此类推，到最小的缓存--CPU寄存器组。

下图展示了存储器层次结构中缓存的一般性概念。第K+1层的存储器被划分成连续的数据对象块，称为**块（block）**。每个块都有一个唯一的地址或名字，使之区别于其他的块。块的大小可以是固定的，也可以是可变大小的。

![transferUint](/Users/liujie/Desktop/gitbook/image/computer/transferUint.png)

数据总是以块大小为**传送单元（transfer unit）** 在第k层和第k+1层之间来回复制。虽然在层次结构中任何一对相邻的层次时间的块大小是固定的，但是其他的层次对之间可以有不同的块大小。如
- L1和L0之间的传送通常使用1个字大小的块；
- L2和L1之间（以及L3和L2之间、L4和L3之间）的传送通常使用的是几十个字节的块；
- L5和L4之间（磁盘和主存）的传送用的是大小为几百或几千字节的块。

一般层次结构中较低层（离CPU较远）的设备访问时间较长，所以为了补偿这些较长的访问时间，倾向于使用较大的块。

### 缓存命中
当程序需要在第k+1层的某个数据对象d时，它首先在当前存储在第k层中的一个块中查找d。如果d刚好缓存在第k层，那么就是我们所说的缓存命中。

### 缓存不命中
根据上面的缓存命中，如果第k层没有d就是缓存不命中。

当发生缓存不命中时，第k层的缓存从第k+1层缓存中取出包含d的那个块，如果第k层缓存已经满了，那么就需要覆盖现存的一个块。

覆盖一个现存的块的过程称为**替换或驱逐这个块**。被驱逐的这个块也称为**牺牲块**。

决定该替换哪个块是由缓存的**替换策略**来控制的。例如：
- 一个具有**随机替换策略**的缓存会随机选择一个牺牲块；
- 一个具有最少被使用**（LRU）替换策略**的缓存会选择那个最后被访问的时间距现在最远的块。

### 缓存不命中的种类
如果第k层是空的，那么对任何数据对象的访问都是不命中的，一个空的缓存称为**冷缓存** ，此类不命中称为**强制不命中或冷不命中**。但是冷不命中通常是短暂事件，不会在反复访问存储器使得缓存暖身之后的稳定状态出现。

只要发生了不命中，第k层的缓存就必须执行某个**放置策略**，确定把它从第k+1层中取出的块放在哪里。最灵活的替换策略是允许来自第k+1层的任何块放在第k层的任何块中，但这个策略通常是昂贵的，因为**随机的放置块，定位起来代价很高**。

所以，硬件缓存通常使用的是更严格的放置策略，这个策略将第k+1层的某个块限制放置在第k层块的一个小的子集中（有时只是一个块）。例如，第k+1层由1、3、5、7这四个块，但只能映射到对应第k层的块0上。这种限制性的放置会引起一种不命中，称为**冲突不命中**，在这种情况中，缓存足够大，能够保存被引用的数据对象，但是因为这些对象会映射到同一个缓存块，缓存会一直不命中。

**程序通常是按照一系列阶段（如循环）来运行的，每个阶段访问缓存块的某个相对稳定不变的集合。** 例如，一个嵌套的循环可能会反复地访问同一个数组的元素。这个块的集合称为这个阶段的**工作集**。当工作集的大小超过缓存的大小时，缓存会经历**容量不命中**，意思就是缓存太小了，不能处理这个工作集。

### 缓存管理
管理缓存的逻辑可以是硬件、软件，或是两者的结合，它将缓存划分成块，在不同的层次将传送块，判定命中还是不命中，并处理它们。

如在一个由虚拟内存的系统中，DRAM主存作为存储在磁盘上的数据块的缓存，是由操作系统软件和CPU上的地址翻译硬件共同管理的。

# 高速缓存存储器
早期的计算机系统存储器层次结构只有三层：CPU寄存器、DRAM主存和磁盘存储。但由于CPU和主存之间逐渐增大的差距，系统设计者被迫在CPU寄存器文件和主存之间插入了一个小的SRAM高速缓存存储器，称为L1高速缓存（一级缓存）。L1高速缓存的访问速度集合和寄存器一样块，典型地大约是4个时钟周期。（后续又陆续在CPU和主存加了L2、L3高速缓存）

## 通用的高速缓存存储器组织结构
考虑一个计算机系统，其中每个存储器地址有m位，形成M = 2^m 个不同的地址。
如下图，
- 这样一个机器的高速缓存被组织成一个有S = 2^s 个**高速缓存组（cache set）** 的数组。
- 每个组包含E个**高速缓存行（cache line）**。
- 每个行是由一个B = 2^b 字节的数组**块（block）**组成的，一个**有效位（valid bit）**指明这个行是否包含有意义的信息，还有t=m-(b+s)个**标记位（tag bit）**，它们唯一地表示存储在这个高速缓存行中的块。

![BES](/Users/liujie/Desktop/gitbook/image/computer/BES.png)

一般而言，高速缓存的结构可以用元组（S，E，B，m）来描述。高速缓存的大小C指的是所有块的大小的和。标记为和有效位不包括，因此 C=S*E*B。

当一条加载指令指示CPU从主存地址A读取一个字时，它将地址A先发送到高速缓存中，那么在高速缓存中怎么判定地址A是否有缓存呢？
这里介绍高速缓存如何实现：
- 参数S和B将m个地址位分为了三段，如下图。
- A中**s个组索引位**是一个到S个组的数组的索引。组索引位被解释为一个无符号的整数，他告诉我们这个字必须存储在哪个组中。
- 我们通过s个组索引位确定了具体的组，A中的**t个标记位**就告诉我们这个组中的哪一行包含这个字（如果有的话）。当且仅当设置了有效位且改行的标记位与地址A中的标记位相匹配时，组中这一行才包含这个字。
- 一旦我们在由组索引标识的组中定位了由标号所标识的行，那么**b个块偏移位**给出了在B个字节的数据块总共的字偏移。

![BES-01](/Users/liujie/Desktop/gitbook/image/computer/BES-01.png)

## 高速缓存行、组和块的区别
- 块：块是一个固定大小的信息包（不同层块大小可以不一致），在高速缓存和主存（或下一层高速缓存）之间来回传送；
- 行：行是高速缓存中的一个容器，存储块以及其他信息（例如有效位和标记位）；
- 组：组是一个或多个行的集合。

## 直接映射到高速缓存
根据每个组的高速缓存行数E，高速缓存被分为不同的类。每个组只有一行（E=1）的高速缓存称为**直接映射高速缓存（direct-mapped cache）**。

![directMap](/Users/liujie/Desktop/gitbook/image/computer/directMap.png)

高速缓存确定一个请求是否命中，然后抽取出被请求的字的过程，分为三步：
1. 组选择
2. 行匹配
3. 字抽取

### 1、直接映射高速缓存中的组选择
在这一步中，高速缓存从w的地址中间抽取出s个组索引位。这些位被解释成一个对应与一个组号的无符号整数。

下面展示了直接映射高速缓存的组选择是如何工作的，组索引位0001被解释为一个选择组1的整数索引。

![directGroup](/Users/liujie/Desktop/gitbook/image/computer/directGroup.png)

### 2、直接映射高速缓存中的行匹配
在上一步中我们已经选择了某个组i，接下来就是确定是否有字w的一个副本存储在组i包含的一个高速缓存行中。在直接映射高速缓存中很容易，因为每个组只有一个行。当且仅当设置了有效位，而且高速缓存行中的标记与w地址中的标记相匹配时，才确认这一行包含w的一个副本。

![directRow](/Users/liujie/Desktop/gitbook/image/computer/directRow.png)

### 3、直接映射高速缓存中的字选择
一旦缓存命中，我们就知道w就在这个块中的某个位置。这一步就是要确认所需要的字在块中是从哪开始的。**块偏移位** 提供了所需要的字的第一个字节的偏移。

### 4、直接映射高速缓存中不命中时的行替换
**如果缓存不命中，那么它需要从存储器层次结构中的下一层取出被请求的块，然后蒋欣的块存储在组索引位只是的组中的一个高速缓存行中。** 对于直接映射高速缓存来说，每个组只有一行，替换策略就是替换这一行。

## 组相联高速缓存
组相联高速缓存放松了直接映射高速缓存中的额每个组只有一行的限制，所以每个组都保存有多于一个的高速缓存行。

缓存操作类似上面的直接映射高速缓存，只不过是在行匹配的时候需要通过标记位去匹配对应的行。

## 全相联高速缓存
全相联高速缓存是一个有包含所有高速缓存行的组组成的，即只有一个高速缓存组，地址中也没有了组索引位，只有标记和块偏移。

全相联高速缓存中的行匹配和子选择与组相联高速缓存中是一样的，它们之间的区别主要是规模大小的问题。

因为高速缓存电路必须并行地搜索许多相匹配的标记，构造一个又打又快的相联高速缓存很困难，而且很昂贵。因此，全相联高速缓存只适合做小的高速缓存，例如虚拟内存系统中的**翻译备用缓冲器（TLB）**，用来缓存页表项。

## 高速缓存有关写的问题--直写/写回
第一个问题是，我们在高速缓存中更新了w数据，怎么更新w在层次结构中紧接着低一层中的副本？
- 直写（write-throuth）：就是立即将w的高速缓存块写回到紧接着的低一层中。虽然简单，但直写的缺点是每次写都会引起总线流量。
- 写回（write-back）：尽可能的推迟更新，只有当替换算法要驱逐这个更新过的块时，才把它写回到紧接着的低一层中。由于局部性，写回能显著的减少总线流量，但它的缺点是增加了复杂性。钙素缓存必须为每个高速缓存行维护一个额外的修改位，表明这个高速缓存块是否被修改过。

还有一个问题是如何处理写不命中（更新数据项的时候高速缓存中没有对应的缓存块）：
- 写分配（write-allocate）：加载响应的低一层中的块到高速缓存中，然后更新这个高速缓存块。写分配试图利用写的空间局部性，但缺点是每次不命中都会导致一个块从低一层传送到高速缓存。
- 非写分配（not-write-allocate）：避开高速缓存，直接吧这个字写到低一层中。直写高速缓存通常是非写分配的。写回高速缓存通常是写分配的。

比较推荐的模式是写回和写分配的高速缓存模型。因为通常，由于较长的传送时间，存储器层次结构中较低层的缓存更可能使用写回，而不是直写。例如虚拟内存系统只使用写回。同时写回写分配试图利用局部性。