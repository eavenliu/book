# 目的
虚拟内存（VM）是现代内存对于主存的抽象概念。它为每个进程提供了一个大的、一致的和私有的地址空间，并且可以更加有效的管理内存和减少出错，大大简化了内存管理。

**虚拟内存的三个重要的能力：高效使用主存、简化内存管理、进程地址空间隔离**；

- 它将主存看成是一个存储在磁盘上的地址空间的高速缓存，在主存中只保存活动区域，并根据需要在磁盘和主存之间来回传递数据，通过这种方式，它高效的使用了主存；
- 它为每个进程提供了一致的地址空间，从而简化了内存管理；
- 它保护了每个进程的地址空间不被其他进程破坏。

为什么要理解虚拟内存？
- 虚拟内存是计算机系统的核心概念。它遍及计算机系统的所有层面。
- 虚拟内存的功能是强大的。虚拟内存赋予了应用程序强大的能力，可以创建和销毁内存片（chunk）、将内存片映射到磁盘文件的某个部分，以及与其他进程共享内存。
- 虚拟内存的使用是危险的。每次应用程序引用变量、间接引用指针等，它就会和虚拟内存发生交互。

我们的核心是需要要理解：
1. **虚拟内存是如何工作的？**
2. **应用程序如何使用和管理虚拟内存？**

# 物理和虚拟寻址
## 物理寻址
**计算机系统的主存被组织成一个由M个连续的字节单元组成的数组。每字节单元都有一个唯一的物理地址（Physical Address，PA）**。第一个字节的地址为0，接下来的字节地址为1，以此类推，CPU访问内存的最自然的方式就是使用物理地址，这种方式就称为**物理寻址**。


一个物理寻址的流程：上下文加载一个指令，读取从物理地址4处开始的4字节字。

![pyhsicalAddress](/Users/liujie/Desktop/gitbook/image/computer/pyhsicalAddress.png)
## 虚拟寻址
**虚拟寻址是现代计算机的寻址形式，CPU通过生成一个虚拟地址（Virtual Address，VA）来访问主存，这个虚拟地址在被传送到内存之前先转换成适当的物理地址**。转换的过程叫做地址翻译，一般通过CPU芯片上内存管理单元（MMU）的专用硬件，利用存放在主存中的查询表来动态翻译地址实现。

一个虚拟寻址流程：
![virtualAddress](/Users/liujie/Desktop/gitbook/image/computer/virtualAddress.png)

# 地址空间
**地址空间（Address Space）** 是一个非负整数的集合：{0，1，2，3，....}

如果地址空间中的整数是连续的，那么我们称它为**线性地址空间**。

在一个带虚拟内存的系统中，CPU从一个有N=2^n个地址的地址空间中生成虚拟地址，这个地址空间称为**虚拟地址空间**：{0,1,2,...，N-1}。

一个地址空间的大小是由表示最大地址所需要的位数来描述的，如一个N=2^n个地址的虚拟空间地址就叫做一个n位地址空间。（现代系统通常支持32位和64位）

同理，**物理地址空间**也类似虚拟地址空间的定义。

所以，主存中的每字节都有一个选自虚拟地址空间的虚拟地址和一个选自物理地址空间的物理地址。每个字节有多个独立的地址，其中每个地址都选自不同的地址空间。

# 虚拟内存作为缓存的工具
**目标：了解虚拟内存是如何提供一种机制，利用DRAM缓存来自通常更大的虚拟地址空间的页面。**

和存储器层次结构中的其他缓存一样，磁盘（较低层）上的数据被分割成块，这些块作为磁盘和主存（较高层）之间的传输单元。

**VM系统通过将虚拟内存分割为称为虚拟页（Virtual Page，VP）的大小固定的块来处理这个问题。**
**每个虚拟页的大小为P=2^p字节**。类似的物理内存被分割为**物理页**（Physical Page，PP），大小也为P字节（物理页也称为**页帧（Page Frame）**）

任意时刻，虚拟页面的集合都分为三个不相交的子集：
- 未分配的：VM系统还未分配（或创建）的页。未分配的块没有任何数据和它们关联，因此也就不占任何磁盘空间。
- 已缓存的：当前已缓存在物理内存中的已分配页。
- 未缓存的：当前未缓存在物理内存中的已分配页。

一个VM系统是如何使用主存作为缓存的：

![VMAddress](/Users/liujie/Desktop/gitbook/image/computer/VMAddress.png)

## DRAM缓存的组织结构
存储层次中有不同额缓存概念：
- DRAM缓存：表示虚拟内存系统的缓存，它在主存中缓存虚拟页。
- SRAM缓存：表示位于CPU和主存之间的L1、L2和L3高速缓存。

**因为DRAM在存储层次结构中的位置，所以对其组织结构有很大的影响。** 

因为DRAM要比SRAM慢大约10倍，而磁盘要比DRAM慢大约100000多倍，所以DRAM不命中的成本要比SRAM不命中的成本昂贵太多。而且，从磁盘的一个扇区读取第一个字节的时间开销比起读这个扇区中连续的字节要慢大约100000倍。总结就是，**DRAM缓存的组织结构完全是由巨大的缓存不命中开销驱动的。**

因为DRAM缓存不命中的巨大成本和访问第一个字节的开销，所以DRAM组织结构有以下特点：
- 虚拟页往往很大，通常是4kb~2MB；
- DRAM缓存是全相联的，即**任何虚拟页都可以放置在任何的物理页中**；
- 不命中时的替换算法也很精细（暂不深入扩展）；
- 因为对磁盘的访问时间很长，**DRAM缓存总是使用写回，而不是直写**。

> 写回和直写：参考“现代计算机存储结构”

## 页表（page table）：将虚拟页映射到物理页
上面分析了基于主存的虚拟内存，以及虚拟内存使用的缓存DRAM，跟其他任何缓存一样，**虚拟内存系统必须要有某种方法来判定一个虚拟页是否缓存在DRAM中的某个地方，如果在DRAM还要确定这个虚拟页存放在哪个物理页中；如果不命中，系统还必须判断这个虚拟页存放在磁盘的哪个位置，在物理内存中选择一个牺牲页，并将虚拟页从磁盘复制到DRAM中替换掉这个牺牲页**。

实现上面的目标需要软硬件联合，包括：**操作系统软件、MMU（内存管理单元）中的地址翻译硬件、一个存放在物理内存中叫做页表（page table）的数据结构**，页表将虚拟页映射到物理页。

每次地址翻译硬件将一个虚拟地址转换为物理地址时，都会读取页表。操作系统负责维护页表的内容，以及在磁盘和DRAM之间来回传送页。

**页表就是一个页表条目（Page Table Entry，PTE）的数组**，页表的基本组织结构如下：每个PTE由两部分组成（具体的页表实现可能还会分得更细，如判断读写的访问位、是否被修改过的修改位、是否被访问过的访问位（用于页面置换算法））：
- 有效位（valid bit）：表明该虚拟页是否被缓存在DRAM中（0/1），如果设置了有效位（1），那么地址字段就表示DRAM中相应物理页的起始位置，这个物理页中缓存了该虚拟页；
- n位地址字段：没有设置有效位时，空地址表示该虚拟页还未被分配，否则这个地址就指向该虚拟页在磁盘上的起始位置。

![PageMapping](/Users/liujie/Desktop/gitbook/image/computer/PageMapping.png)

## 页命中：DRAM缓存命中
当CPU通过虚拟地址读取VP2中的虚拟内存的一个字时：（具体定位技术后面叙述）

地址翻译硬件将虚拟地址作为一个索引来定位PTE2，并从内存中读取它，判断有效位知道VP2是缓存在DRAM中的，所以使用PTE中对应的地址得出这个字的物理地址。

![DramCacheShot](/Users/liujie/Desktop/gitbook/image/computer/DramCacheShot.png)	

## 缺页（page fault）：DRAM缓存不命中
**在虚拟内存的习惯说法中，DRAM缓存不命中称为缺页（page fault）。** 

缺页的触发和处理流程：
1. CPU的一个指令需要读取某个虚拟地址对应的虚拟页VP；
2. 地址翻译硬件根据虚拟地址对应到具体的PTE，发现PTE有效位未设置，即目标VP未被DRAM缓存，这时触发一个缺页异常；
3. 缺页异常会调用内核中的缺页异常处理程序，如果DRAM内存中有空闲的物理页面则直接分配一个物理页帧，否则该程序会在DRAM中选择一个牺牲页VPs（某种页面置换算法），如果VPs已经被修改了，那么内核就会将它复制回磁盘，最终内核会修改VPs的页表条目PTE，反映出VPs不再缓存在主存中；
4. 接下来内核将指定VP从磁盘复制到内存中之前VPs所在的物理页，然后更新虚拟地址对应的PTE，然后返回；
5. 缺页异常处理流程返回时，它会重新启动导致缺页的指令，该指令会把导致缺页的虚拟地址重发送到地址翻译硬件，从而顺利执行下去。

回顾之前的概念，**虚拟内存中，块被称之为页**，这里有几个概念：
- 在磁盘和内存之间传送页的活动叫做**交换（swapping）或页面调度（paging）**；
- 页从磁盘换入（或者页面调入）DRAM和从DRAM换出（或者页面调出）磁盘；
- 当有DRAM不命中发生时，才换入页面的这种策略称为**按需页面调度**（demand paging），所有现代系统都使用这种方式。

> 可以利用Linux的getrusage函数检测缺页的数量

## 分配页面
分配一个新的虚拟页面（如调用malloc的结果），内核在磁盘上分配了VP5，并将PTE5指向这个新的位置：

![loadPage](/Users/liujie/Desktop/gitbook/image/computer/loadPage.png)

## 实际运行中的局部性原则
在上面分析了虚拟内存的基本运转流程，可以发现其实虚拟内存存在性能问题，特别是频繁发生DRAM不命中的情况，需要进行大量的页面调度。但在实际应用中，虚拟内存其实运行得很好，因为这里使用了局部性（locality）原则。

**局部性原则**：指程序在执行过程中的一个较短时期，所执行的指令地址和指令的操作地址，分别局限于一定区域。表现为：
- 时间局部性：一条指令的一次执行和下次执行，一个数据的一次访问和下次访问都集中在一个较短的时期内。
- 空间局部性：当前指令和邻近的几条指令，当前访问的数据和邻近的几个数据都集中在一个较小区域内。

尽管一般我们在程序运行中引用的不同页面的总数可能会超过物理内存总的大小，但是局部性原则保证了在任意时刻，程序都将趋向于在一个较小的活动页面（active page）集合上工作，这个集合叫做**工作集（working set）**或者**常驻集合（resident set）**。

初始开销时，就是将工作集页面调度到内存中之后，接下来对这个工作集的引用将导致命中，而不产生额外的磁盘流量。只要程序有很好的时间局部性，那么虚拟内存就会工作得非常好。但是没有保证好导致工作集的大小超出了物理内存的大小，程序就会发生**抖动**，这时页面将不断地换进换出，程序性能将急剧降低。

# 虚拟内存作为内存管理的工具
虚拟地址大大的简化了内存管理，并提供了一种自然的保护内存的方法。

**上面说到了页表（Page Table）将一个虚拟地址空间映射到物理地址空间，在应用中操作系统为每个进程提供了一个独立的页表，因而也就是一个独立的虚拟地址空间。**

VM如何为进程提供独立的地址空间，操作系统为系统中的每个进程都维护一个独立的页表，多个虚拟页面可以映射到同一共享物理页面上：

![pagess](/Users/liujie/Desktop/gitbook/image/computer/pagess.png)\

**按需页面调度和独立的虚拟地址空间的结合，对系统中内存的使用和管理造成了深远地影响。特别地，VM简化了链接和加载、代码和数据共享，以及应用程序的内存分配**。
- **简化链接**。**独立的虚拟地址空间允许每个进程的内存映像使用相同的基本格式，而不用考虑代码和数据实际存放在物理内存的何处**。如linux系统规定的进程的虚拟地址空间结构，这样的一致性极大地简化了链接器的设计和实现，允许链接器生成完全链接的可执行文件，这些可执行文件是独立于物理内存中代码和数据的最终位置的。
- **简化加载**。**虚拟内存还使得容易向内存中加载可执行文件和共享数据文件**。例如要把目标文件中.text和.data节加载到一个新创建的进程中，Linux加载器为代码和数据段分配虚拟页，并把它们标记为未被缓存的，将页表条目指向目标文件中适当的位置。特别的，加载器从不从磁盘到内存实际复制任何数据，都是指令执行时虚拟内存系统按需页面调度的。

> 将一组连续的虚拟页映射到一个任意文件中的任意位置的表示法称作内存映射（Memory Mapping），后续会做分析。

- **简化共享**。独立的地址空间为操作系统提供了一个管理用户进程和操作系统自身之间共享的一致机制。一般来说，每个进程都有自己私有的代码、数据、堆以及栈区域，是不和其他进程共享的。这种情况下操作系统创建页表，将对应的虚拟页映射到不连续的物理页面。但有些需要共享的代码和数据，如公共库，需要将不同进程中的虚拟页面映射到相同的物理页面，从而达到共享的目的，而不是每个进程都创建自己的副本。
- **简化内存分配**。**虚拟内存为向用户进程提供一个简单的分配额外内存的机制**。当一个运行在用户进程中的程序需要额外的堆空间时（如调用malloc的结果），操作系统分配一个适当的数字（例如k）个连续的虚拟内存页面，并将它们映射到物理内存中任意位置的k个任意的物理页面。

# 虚拟内存作为保护内存的工具：页面级的内存保护
计算机系统必须为操作系统提供手段来控制对内存系统的访问。如不应该允许一个用户进程修改它的只读代码段、不应该允许它读或修改任何内核中的代码和数据结构。

通过上面我们可以知道，为进程提供独立的地址空间使得区分不同进程的私有内存变得容易。但是，**地址翻译机制可以以一种自然优雅的方式扩展到提供更好的访问控制，因为每次CPU生成一个地址时，地址翻译硬件都会读一个PTE，所以我们通过在PTE上添加额外的许可位来控制对一个虚拟页面内容的访问就很简单**。如下图：

![pageProtect](/Users/liujie/Desktop/gitbook/image/computer/pageProtect.png)

每个PTE中增加了三个许可位：（不同操作系统应用还可以继续扩展许可位）
- SUP：进程是否必须运行在内核模式下才能访问该页；
- READ：是否可以读页面
- WRITE：是否可以写页面

如果操作违反了许可位，那么CPU救护触发一个一般保护故障，将控制传递给一个内核中的异常处理程序，这种错误一般称之为：**段错误（segmentation fault）**。

# 地址翻译
这里主要是分析地址翻译的基础知识点。

地址翻译符号小结：

**基本参数**：
- N=2^n：虚拟地址空间中的地址数量
- M=2^m：物理地址空间中的地址数量
- P=2^p：页的大小（字节）

**虚拟地址（VA）的组成部分**
- VPO：虚拟页面偏移量（字节）
- VPN：虚拟页号
- TLBI：TLB索引
- TLBT：TLB标记

**物理地址（PA）的组成部分**
- PPO：物理页面偏移量（字节）
- PPN：物理页号
- CO：缓冲块内的字节偏移量
- CI：高速缓存索引
- CT：高速缓存标记

**形式上来说，地址翻译是一个N元素的虚拟地址空间中的元素和一个M元素的物理地址空间中元素之间的映射。**

下图展示了MMU如何利用页表来实现这种映射：
1. CPU中的一个控制寄存器，页表基址寄存器（PTBR）指向当前页表；
2. 一个n为的虚拟地址由两部分组成：一个p位的虚拟页面偏移（Virtual Page Offset，VPO）、一个n-p位的虚拟页号（Virtual Page Number，VPN），MMU使用VPN翻译到适当的PTE；
3. 通过定位的PTE将页表条目中的物理页号和虚拟地址中的VPO串联起来，就得到对应的物理地址（因为物理和虚拟页面都是P字节的，所以物理页面偏移（PPO）和VPO是相同的）；

![addressMapping](/Users/liujie/Desktop/gitbook/image/computer/addressMapping.png)

根据映射的逻辑，实际CPU执行过程中，页面命中和不命中会有稍显不同的逻辑：

**页面命中**：
1. 处理器引用一个虚拟地址，并把它传送给MMU；
2. MMU通过地址翻译生成PTE地址（PTEA），并从高速缓存/主存请求得到它；
3. 高速缓存/主存向MMU返回对于的PTE；
4. MMU构造物理地址，并把它传递回高速缓存/主存；
5. 高速缓存/主存返回锁请求的数据字给处理器。

![memoryShot](/Users/liujie/Desktop/gitbook/image/computer/memoryShot.png)

**页面未命中**：
1. 前面三步同上面的页面命中；
2. PTE有效位是零，所以MMU触发了一个缺页异常；
3. 缺页处理程序判断有无空闲物理页，没有则根据某种替换算法选择一个对应的牺牲页，如果这个页面被修改过则要将它换出到磁盘；
4. 缺页处理程序调入新的页面，并更新内存中的PTE；
5. 缺页处理程序返回到庲的进程，重新执行导致缺页的指令。

![memoryUnShot](/Users/liujie/Desktop/gitbook/image/computer/memoryUnShot.png)

## 结合高速缓存和虚拟内存
在任何使用虚拟内存又使用SRAM高速缓存的系统中，都有是要用虚拟地址还是用物理地址来访问SRAM高速缓存的问题。大部分系统的使用的物理寻址SRAM，具体这里不做分析。使用物理寻址，多个进程同时在高速缓存中有存储块和共享来自相同虚拟页面的块成为很简单的事。

高速缓存不须做保护问题，因为访问权限的检查是地址翻译过程的一部分。

**将VM与物理寻址的高速缓存结合起来**（VA:虚拟地址、PTEA：页表条目地址、PTE：页表条目、PA：物理地址），注意，**页面条目可以缓存，就像其他的数据一样**。

![togetherAddress](/Users/liujie/Desktop/gitbook/image/computer/togetherAddress.png)

## 利用TLB加速地址翻译
页表一般都很大，并且存放在内存中，所以处理器引入MMU后，读取指令、数据需要访问两次内存：首先通过查询页表得到物理地址，然后访问该物理地址读取指令、数据。提高访存性能的关键在于依靠页表的访问局部性。当一个转换的虚拟页号被使用时，它可能在不久的将来再次被使用到。

为了减少因为MMU导致的处理器性能下降，引入了TLB，**TLB是Translation Lookaside Buffer的简称，可翻译为“地址转换后援缓冲器”，也可简称为“快表”**。简单地说，TLB就是页表的Cache，其中存储了当前最可能被访问到的页表项，其内容是部分页表项的一个副本。只有在TLB无法完成地址翻译任务时，才会到内存中查询页表，这样就减少了页表查询导致的处理器性能下降。

**TLB是一种高速缓存，内存管理硬件使用它来改善虚拟地址到物理地址的转换速度**。当前所有的个人桌面，笔记本和服务器处理器都使用TLB来进行虚拟地址到物理地址的映射。使用TLB内核可以快速的找到虚拟地址指向物理地址，而不需要请求RAM内存获取虚拟地址到物理地址的映射关系。这与data cache和instruction caches有很大的相似之处。

TLB是一个小的、虚拟寻址的缓存，其中每一行都保存一个由单个PTE组成的块。TLB通常有高度的关联度。

虚拟地址中用以访问TLB的组成部分：

![TLB](/Users/liujie/Desktop/gitbook/image/computer/TLB.png)

下面是TLB命中和不命中的操作图：
1. 第1步CPU产生一个虚拟地址；
2. 第2、3、4步是MMU从TLB中命中时取出对应的PTE，不命中是从L1缓存中取出相应的PTE；
3. 第5步，MMU将这个虚拟地址翻译成一个物理地址，并且将它发送到高速缓存/主存；
4. 第6步，高速缓存/主存将所请求的数据字返回给CPU。

![TLBShot](/Users/liujie/Desktop/gitbook/image/computer/TLBShot.png)

## 多级页表
**多级页表是对于页表的优化，主要体现在节约内存和进行离散存储方面**。

（1）**使用多级页表可以使得页表在内存中离散存储**。多级页表实际上是增加了索引，有了索引就可以定位到具体的项。举个例子：比如虚拟地址空间大小为4G，每个页大小依然为4K，如果使用一级页表的话，共有2^20个页表项，如果每一个页表项占4B，那么存放所有页表项需要4M，为了能够随机访问，那么就需要连续4M的内存空间来存放所有的页表项。随着虚拟地址空间的增大，存放页表所需要的连续空间也会增大，在操作系统内存紧张或者内存碎片较多时，这无疑会带来额外的开销。但是如果使用多级页表，我们可以使用一页来存放页目录项，页表项存放在内存中的其他位置，不用保证页目录项和页表项连续。

（2）**使用多级页表可以节省页表内存**。使用一级页表，需要连续的内存空间来存放所有的页表项。多级页表通过只为进程实际使用的那些虚拟地址内存区请求页表来减少内存使用量。
- 如果一级页表中的一个PTE是空的，那么二级页表根本不会存在，这代表了一种巨大的节约；
- 只有一级页表才需要总是在主存中，虚拟内存系统可以在需要时创建、页面调入或调出二级页表，这样减少了主存的压力，只有经常使用的二级页表才需要缓存在主存中。

举个例子：比如一个进程只是用4MB内存空间。对于一级页表，我们需要4M空间来存放页表，然后可以找到进程真正使用的4M内存空间。但是如果使用二级页表的话，一个页目录项可以定位4M内存空间，存放一个页目录项占4K，还需要一页用于存放进程使用的4M（4M=1024*4K，也就是用1024个页表项可以映射4M内存空间）内存空间对应的页表，总共需要4K+4K=8K来存放进程使用的这4M内存空间对应页表和页目录项，这比使用一级页表节省了很多内存空间。

一个两级页表层次结构：

![mutliPageTable](/Users/liujie/Desktop/gitbook/image/computer/mutliPageTable.png)

下图描述了使用k级页表层次结构的地址翻译：

![KsPageTable](/Users/liujie/Desktop/gitbook/image/computer/KsPageTable.png)

# 现代系统案例
## Core i7地址翻译

core i7的主要地址翻译情况：CR3控制寄存器指向第一级页表（L1）的起始位置。cr3的值是每个进程上下文的一部分，每次上下文切换时，CR3的值都会被恢复。

![i7Address](/Users/liujie/Desktop/gitbook/image/computer/i7Address.png)

## Linux虚拟内存系统
一个虚拟内存系统要求硬件和内核软件之间的紧密协作。

Linux为每一个进程维护了一个单独的虚拟地址空间，如下图：

![LinuxVM](/Users/liujie/Desktop/gitbook/image/computer/LinuxVM.png)

### Linux虚拟内存优化：区域
**Linux将虚拟内存组织成一些区域（也叫做段）的集合**。一个区域（area）就是已经存在着的（已分配的）虚拟内存的连续片（chunk），这些页是以某种方式相关联的。例如，代码段、数据段、堆、共享库段，以及用户栈都是不同的区域。

**每个存在的虚拟页面都保存在某个区域中，而不属于某个区域的虚拟页是不存在的，并且不能被进程引用**。区域的概念很重要，因为它允许虚拟地址空间有间隙，内核不需要记录那些不存在的虚拟页，而这样的页也不占内存、磁盘或者内核本身中的任何额外资源。

下图记录可一个进程中虚拟内存区域的内核数据结构：

![linuxVMAddress](/Users/liujie/Desktop/gitbook/image/computer/linuxVMAddress.png)

内核为系统中的每个进程维护一个单独的任务结构（源代码中的task_struct）。任务结构中的元素包含或者指向内核运行该进程所需要的所有信息：例如PID、指向用户栈的指针、可执行目标文件的名字、以及程序计数器。

任务结构中的一个条目指向**mm_struct**，他描述可虚拟内存的当前状态，其中两个字段：
- pgd：指向第一级页表（页全局目录）的基址；
- mmap：指向一个vm_area_structs（区域结构）的链表，其中每个vm_area_structs都描述了当前虚拟地址空间的一个区域。

当内核运行这个进程时，就把pgd存放在CR3控制寄存器中。

一个具体**区域的区域结构**包含下面的字段：
- vm_start:指向这个区域的起始处；
- vm_end:指向这个区域的结束处；
- vm_prot:描述这个区域内包含的所有页的读写许可权限；
- vm_flags:描述这个区域内的页面是与其他进程共享的，还是这个进程私有的（还描述了其他一些信息）；
- vm_next:指向链表中下一个区域结构。

### Linux缺页异常处理
linux的访问异常一般包括三种形式：
1. 虚拟地址是合法的么？即虚拟地址在某个区域结构定义的区域内吗？
2. 试图进行的内存访问是偶发合法？即进程是否有读写这个区域页面的权利？
3. 正常的缺页交换。

![lessPage](/Users/liujie/Desktop/gitbook/image/computer/lessPage.png)


# 内存映射
Linux通过将一个虚拟内存区域与一个磁盘上的对象（object）关联起来，以初始化这个虚拟内存区域的内容，这个过程称为**内存映射（memory mapping）**。

虚拟内存区域可以映射到两种类型的对象中的一种：
- **Linux文件系统中的普通文件**：一个区域可以映射到一个普通磁盘文件的连续部分，例如一个可执行文件。文件区（section）被分成页大小的片，每一片包含一个虚拟页面的初始内容。这些虚拟页面根据按需页面调度进入物理内存。如果区域比文件区要大，那么就用零来填充这个区域的余下部分。
- **匿名文件**：一个区域也可以映射到一个匿名文件，匿名文件是由内核创建的，包含的全是二进制零。CPU第一次引用这样一个区域内的虚拟页面时，内核就在物理内存中找到一个合适的牺牲页面，如果该页面被修改过，就将这个页面换出来，用二进制零覆盖牺牲页面并更新页表，将这个页面标记为驻留在内存中的。这里磁盘和内存之间并没有实际的数据传送。所以，映射到匿名文件的区域中的页面有时也叫做**请求二进制零的页**（demand-zero page）

无论是哪种情况，一旦一个虚拟页面被初始化了，它就在一个由内核维护的专门的**交换文件（swap file）**之间换来换去。**交换文件也叫做交换空间（swap space）或者交换区域（swap area）**。
在任何时候，**交换空间都限制着当前运行着的进程能够分配的虚拟页面的总数**。

## 再看共享对象
**一个对象可以被映射到虚拟内存的一个区域，要么作为共享对象，要么作为私有对象**。

如果一个进程将一个共享对象映射到它的虚拟地址空间的一个区域内，那么这个进程对这个区域的任何写操作，对于那些也把这个共享对象映射到它们虚拟地址空间的其他进程而言，也是可见的。同时这个变化也会反映在磁盘上的原始对象中。

另一方面，对于一个映射到私有对象的区域做的改变，对于其他进程来说是不可见的，并且进程对这个区域做的任何写操作都不会反映在磁盘上的对象中。一个映射到共享对象的虚拟内存区域叫做**共享区域**，类似地，也有**私有区域**。

### 写时复制：最充分的使用物理内存
私有对象使用一种叫做写时复制（copy-on-write）的巧妙技术被映射到进程虚拟内存中。

私有对象开始生命周期的方式基本上与共享对象的一样，在物理内存中只保存有私有对象的一份副本。会出现一种情况，**有两个进程将一个私有对象映射到它们虚拟内存的不同区域，但是共享这个对象的同一个物理内存副本。对于每个映射私有对象的进程，相应私有区域的页表条目都被标记为只读，并且区域结构被标记为私有的写时复制**。只要没有进程试图写它自己的私有区域，它们就可以继续共享物理内存中对象的一个单独副本。

当有一个进程试图写私有区域的某个页面，那么这个写操作就会触发一个保护故障，**当故障处理程序注意到保护异常是由于进程试图写私有的写时复制区域中的一个页面而引起的，它就会在物理内存中创建这个页面的一个副本，更新页表条目指向这个新的副本，然后恢复这个页面的可写权限**。故障控制返回重新执行写即可正常。

![COW](/Users/liujie/Desktop/gitbook/image/computer/COW.png)

**通过延迟私有对象中的副本直到最后可能的时刻，写时复制最充分地使用了稀有的物理内存。**

## 再看fork函数-创建新进程
我们知道，**当fork函数被当前进程调用时**，内核为新进程创建各种数据结构，并分配给它一个唯一的PID。**为了给这个新进程创建虚拟内存，它创建了当前进程的mm_struct、区域结构和页表的原样副本。它将两个进程中的每个页面都标记为只读，并将两个进程中的每个区域结构都标记为私有的写时复制。**

所以，**当fork在新进程中返回时，新进程现在的虚拟内存刚好和调用fork时存在的虚拟内存相同。当后续两个进程中的任一个进行写操作时，写时复制机制就会创建新页面，因此，也就为每个进程保持了私有地址空间的抽象概念。**

## 再看execve函数-执行程序
虚拟内存和内存映射在将程序加载到内存的过程中也扮演着关键的角色。

假设在当前进程中执行如下的execve调用：execve("a.out", NULL, NULL);这时execve函数在当前进程中加载并运行包含在可执行目标文件a.out中的程序，用a.out程序有效的替代了当前程序。有以下步骤：
1. 删除已存在的用户区域。删除当前进程虚拟地址的用户部分中的已存在的区域结构。
2. 映射私有区域。为新程序的代码、数据、bss和栈区域创建新的区域结构。所有新的区域都是私有的、写时复制的。bss区域是请求二进制零的，大小包含在a.out中，栈和堆区域也是请求二进制零的，初始长度为零。
3. 映射共享区域。如果a.out程序与共享对象（或目标）链接，比如标准C库libc.so，那么这些对象都是动态链接到这个程序的，然后再映射到用户虚拟地址空间中的共享区域内。
4. 设置程序计数器。设置当前进程上下文中的程序计数器，使之指向代码区域的入口点。

下图展示加载器是如何映射用户地址空间区域的：

![mapUserAddress](/Users/liujie/Desktop/gitbook/image/computer/mapUserAddress.png)

## 使用mmap函数的用户级内存映射
Linux进程可以使用mmap函数来创建新的虚拟内存系统，并将对象映射到这些区域中。

```
void *mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset);
返回：成功则为指向映射区域的指针
```
mmap函数要求内核创建一个新的虚拟内存区域，最好是从地址start（仅仅是一个暗示，一般为NULL）开始的一个区域，并将文件描述符fd指定的对象的一个连续的片（chunk）映射到这个新的区域。连续的对象片大小为length字节，从局文件开始处偏移量为offset字节的地方开始。

![mmapFunction](/Users/liujie/Desktop/gitbook/image/computer/mmapFunction.png)

参数prot包含描述新映射的虚拟内存区域的访问权限位。

munmap函数删除虚拟内存区域：

```
int munmap(void *start, size_t length)
返回：成功返回0
```

# 动态分配内存-堆
虽然可以使用低级的mmap和munmap函数来创建和删除虚拟内存的区域，但高级语言还是觉得当运行时需要额外虚拟内存时，用**动态内存分配器**（dynamic memory allocator）更方便，移植性也更强。

**动态内存分配器为何着一个进程的虚拟内存区域，称为堆（heap）。** 系统之间实现细节不同，但是通用性是一致的，假设堆是一个请求二进制零的区域，它紧接着在未初始化的数据区域后开始，并向上生长。对于每个进程，内核维护这一个堆顶指针（brk）。

![heapStruct](/Users/liujie/Desktop/gitbook/image/computer/heapStruct.png)

**分配器将堆视为一组不同大小的块（block）的集合来维护。每个块就是一个连续的虚拟内存片（chunk），要么是已分配的，要么是空闲的。** 已分配的块显式地保留为供应用程序使用，空闲块可以用来分配。一个已分配的块保持已分配状态，直到它被释放，这种释放要么是应用程序显式执行的，要么是内存分配器自身隐式执行的。

分配器有两种基本风格。两种风格都要求应用程序显式分配块，不同之处在于由哪个实体来释放已分配的块：
- 显式分配器：要求应用程序显式地释放任何已分配的块。例如c语言通过malloc韩式分配一个块，并通过free释放一个块。
- 隐式分配器：也叫做**垃圾回收器**。要求分配器检测一个已分配块何时不再被程序所使用，那么就释放这个块，自动释放未使用的已分配的块的过程叫做**垃圾收集**。诸如lisp、Java（new分配块，gc回收块）之类的高级语言就依赖垃圾收集来释放已分配的块。

举例，malloc和free分配和释放块。每个方框代表一个字，每个粗线标出的矩形相当于一个块。堆地址从左向右增加：

![dynMollocMemory](/Users/liujie/Desktop/gitbook/image/computer/dynMollocMemory.png)

## 为什么要动态内存分配
程序使用动态分配内存的最重要的原因是经常直到程序运行时，才知道某些数据结构的大小。

## 分配器的要求和目标
显式分配器必须在一些相当严格的约束条件下工作：
- 处理任意请求序列。应用可以有任意的分配请求和释放请求序列，只要满足：每个释放请求必须对应一个当前已分配块。
- 立即响应请求。分配器必须立即响应分配请求。因此不允许分配器为了性能重新排列或缓冲请求。
- 只使用堆。为了保证分配器是扩展的，分配器使用的任何非标量数据结构都必须保存在堆里。
- 对齐块（对齐要求）。分配器必须对齐块，使得他们可以保存任何类型的数据对象。
- 不修改已分配的块。分配器只能操作或者改变空闲块。

分配器的目标：
- 最大化吞吐率：吞吐率定义为每个单位时间里完成的请求数。我们要求一个分配器要有合理性能，指一个分配请求的最糟运行时间与空闲块的数量成线性关系，而一个人释放请求的运行时间是个常数。
- 最大化内存利用率：一个系统中被所有进程分配的虚拟内存的全部数量是受磁盘上交换空间的数量限制的。（使用峰值利用率（peak utilization）来描述分配器使用堆的效率）

## 堆内存碎片化的由来
造成堆利用率很低的主要原因是一种称为**碎片**的现象，当虽然有未使用的内存但不能用来满足分配请求时，就发生这种现象。

**碎片有两种形式**：
- **内部碎片：由一个已分配块比有效载荷大时发生的**。很多原因会导致这个情况发生，例如分配器的实现可能对已分配块有最小值设定、分配器增加块大小保证对齐约束条件。所以，内部碎片的数量只取决于分配器的实现模式。
- **外部碎片：是当空闲内存合计起来足够满足一个分配请求，但是没有一个单独的空闲块足够大可以来处理这个请求时发生的**。

> 有效载荷（payload）：如果一个应用程序请求一个P字节的块，那么得到的已分配的块的有效载荷就是P字节。多个已分配的块的有效载荷之和称为聚集有效载荷。

因为外部碎片难以量化且不可预测，所以分配器通常采用启发式策略来试图维持少量的大空闲块，而不是维持大量的小空闲块。

## 实现一个分配器应该考虑的问题
一个很简单的分配器就是把堆组织成一个大的字节数组，还有一个指针p，初始指向这个数组的第一个字节。为了分配n个字节，将p的当前值保存在栈里，将p增加n，并将p的旧值返回到调用函数。但这个简单的分配器从不重复使用任何块，内存利用率极差。

一个好的分配器需要考虑：
1. 空闲块组织：如何记录空闲块；
2. 放置：如何选择一个合适的空闲块来放置一个新分配的块；
3. 分割：将一个新分配的块放置到某个空闲块之后，如何处理这个空闲块中的剩余部分？
4. 合并：如何处理一个刚刚被释放的块？

## 空闲链表
任何实际的分配器都需要一些数据结构，允许它来区别块边界，以及区别已分配块和空闲块。大多数分配器将这些信息嵌入块本身。一个简单的堆块格式如下：

![freeLinkList](/Users/liujie/Desktop/gitbook/image/computer/freeLinkList.png)

上面这种结构称为隐式空闲列表是因为空闲块是通过头部中的大小字段隐含地连接着的，一个块是由一个字的头部、有效载荷，以及可能的一些额外的填充组成的。头部编码了这个块的大小（包括头部和所有的填充），以及这个块是已分配的还是空闲的。

我们可以将堆组织为一个连续的已分配块和空闲块的序列：阴影部分是已分配块，头部标记为（大小（字节）/已分配位）

![freeUser](/Users/liujie/Desktop/gitbook/image/computer/freeUser.png)

**但是隐式空闲链表块分配和堆块总数呈线性关系，所以对于通用分配器，不是很适合，所以基于这个有显式空闲链表、分离的空闲链表等方案。**

# 垃圾收集
**垃圾收集器（garbage collector）是一种动态内存分配器，它自动释放程序不再需要的已分配块。自动回收堆存储的过程就做垃圾收集（garbage collection）。** 在一个支持垃圾收集的系统中，应用显式分配堆块，但是从不显式地释放它们。

## 垃圾收集器中的有向可达图
垃圾收集器将内存视为一张**有向可达图**，图中的节点分为一组根节点（root node）和一组堆节点（heap node）。

**每个堆节点**对应于堆中的一个已分配块。

**根节点**对应于这样一种不在堆中的位置，它们中包含指向堆中的指针。这些位置**可以是寄存器。栈里的变量，或者是虚拟内存中读写数据区域内的全局变量。**

![gcArrive](/Users/liujie/Desktop/gitbook/image/computer/gcArrive.png)

当存在一条从任意根节点触发并到达p的有向路径时，我们说节点p是可达的。在任意时刻，不可达节点对应于垃圾，是不能被应用再次使用的。垃圾收集器的角色是维护可达图的某种表示，并通过释放不可达节点且将它们返回给空闲链表来定期回收它们。

像Java这样的语言的垃圾收集器，对应用如何创建和使用指针有很严格的控制，能够维护可达图的一种精确的便是，因此也就能够回收所有垃圾。

## Mark & Sweep垃圾收集器
**Mark&Sweep垃圾收集由标记（mark）和清除（sweep）阶段组成。** 标记阶段标记出根节点的所有可达的和已分配的后继，而后面的清除阶段释放每个未被标记的已分配块。块头部中空闲的低位中的一位通常来表示这个块是否被标记了。

下面有一个标记清除示例，箭头表示内存引用：初始情况下，下面的堆由六个已分配块组成，其中每个块都是未标记的。根节点指向的位置位于第4块。

![mark&Sweep](/Users/liujie/Desktop/gitbook/image/computer/mark&Sweep.png)