# Linux Page Cache和pdflush
当你写出最终用于磁盘的数据时，Linux会将此信息缓存在称为**页面缓存**的内存区域中。 你可以使用free，vmstat或top等工具找到有关页面缓存的基本信息。

有关页面缓存的完整信息仅通过查看/proc/meminfo显示。 以下是来自具有16GB RAM的系统的示例：

```
root@iZwz9b1cixx5jxvmsukqtzZ:~# cat /proc/meminfo
MemTotal:       16432348 kB
MemFree:         3753016 kB
MemAvailable:   10900912 kB
Buffers:          267156 kB
Cached:          6935608 kB
SwapCached:            0 kB
Active:         11564336 kB
Inactive:         728824 kB
Active(anon):    5091008 kB
Inactive(anon):     2720 kB
Active(file):    6473328 kB
Inactive(file):   726104 kB
Unevictable:           0 kB
Mlocked:               0 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:               844 kB
Writeback:             0 kB
AnonPages:       5090396 kB
Mapped:            87604 kB
Shmem:              3332 kB
Slab:             310364 kB
SReclaimable:     286800 kB
SUnreclaim:        23564 kB
KernelStack:       10992 kB
PageTables:        17604 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     8216172 kB
Committed_AS:    5337144 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
HardwareCorrupted:     0 kB
AnonHugePages:         0 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:       59264 kB
DirectMap2M:     3086336 kB
DirectMap1G:    15728640 kB
```
**页面缓存本身的大小是这里的“Cached”数字**，在这个例子中它是6.9GB。 在写入页面时，“Dirty”部分的大小将增加。 一旦写入磁盘已开始，您将看到“Writeback”数字上升，直到写入完成。 实际上很难实现Writeback值高，因为它的值非常短暂，并且只在I/O排队但尚未写入的短暂时间内增加。

**Linux通常使用名为pdflush的进程将数据写入Page cache。** 在任何时候，系统上都会运行2到8个pdflush线程。您可以通过查看/proc/sys/vm/nr_pdflush_threads来监视活动的数量。
```
root@iZwz9b1cixx5jxvmsukqtzZ:~# cat /proc/sys/vm/nr_pdflush_threads
0
```
每当所有现有的pdflush线程都忙碌至少一秒钟时，就会产生一个额外的pdflush守护进程。 新的尝试将数据写回非拥塞的设备队列，旨在让每个活动的设备获得自己的线程刷新数据到该设备。 每次传递一秒而没有任何pdflush活动，其中一个线程将被删除。有可调参数用于调整pdflush进程的最小和最大数量，但很少需要调整它们。

## pdflush可调整参数
确切地说，每个pdflush线程的作用是由/proc/sys/vm中的一系列参数控制的：
- /proc/sys/vm/dirty_writeback_centisecs (default 500): 在百分之一秒内，这就是pdflush唤醒将数据写入磁盘的频率。 默认值每五秒唤醒两个（或更多）活动线程。
- /proc/sys/vm/dirty_expire_centiseconds (default 3000): 在百分之一秒内，数据在被认为过期之前可以在页面缓存中存在多长时间，并且必须在下一次机会时写入。 请注意，此默认值很长：整整30秒。 这意味着在正常情况下，除非你写得足以触发其他pdflush方法，否则Linux实际上不会提交任何你写的东西，直到30秒之后。
- /proc/sys/vm/dirty_background_ratio (default 10): 在pdflush开始编写之前可以使用脏页填充的最大活动百分比

## 针对大量写操作调整建议
正在编写大量内容的人通常的问题是Linux一次性缓冲太多信息，以提高效率。对于需要使用fsync等系统调用同步文件系统的操作而言，这尤其麻烦。如果在进行此调用时缓冲区中有大量数据，系统可能会冻结相当长的一段时间来处理同步。

另一个常见问题是，因为在任何物理写入开始之前必须编写这么多内容，所以I/O看起来比看起来最优的更突发。您将有很长时间没有物理写入发生，因为大页面缓存已填满，然后一旦pdflush触发器被触发，设备可以以最高速度写入。

- dirty_background_ratio：主要可调整，可能向下调整。如果您的目标是减少Linux在内存中缓存的数据量，以便将其更一致地写入磁盘而不是批量写入，则降低dirty_background_ratio是最有效的方法。在系统具有大量内存和/或物理I / O较慢的情况下，默认值更可能太大。
- dirty_ratio：辅助可调参数，仅针对某些工作负载进行调整。能够完全阻止其写入的应用程序可能会从大幅降低此值中受益。调整前请参阅下面的“警告”。
- dirty_expire_centisecs：测试降低，但不是极低水平。试图加速页面在内存中存放多长时间可以在这里完成，但这会大大降低平均I / O速度，因为它的效率会降低多少。对于磁盘物理I / O较慢的系统尤其如此。由于脏页面编写机制的工作方式，尝试将此值降低到非常快（少于几秒）不太可能正常工作。不断尝试将脏页写出来只会更频繁地触发I / O拥塞代码。
- dirty_writeback_centisecs：别管。这个参数设置的pdflush线程的时间因内核代码中的规则而变得如此复杂，例如写入拥塞，调整这个可调参数不太可能产生任何实际效果。通常建议将其保持为默认值，以便此内部时序调整与pdflush运行的频率相匹配。