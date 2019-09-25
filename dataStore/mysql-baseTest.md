# 基准测试

## 测试策略

整体系统测试、mysql单独测试

## 测试目标

- 吞吐量：单位时间内的事务处理数
- 响应时间/延迟
- 并发性：与web服务器相关，同时测试并发量增加，吞吐量的变化
- 可扩展性：如增加双倍CPU核数，指标的变化

## 记录数据

- CPU使用率
- 磁盘I/O
- 网络流量
- SHOW GLOBAL STATUS计数器

具体也要根据业务：I/O密集型应用，CPU密集型应用制定测试策略

## 基准测试工具

集成工具：

- ab：简单，针对单个URL，测试HTTP服务器每秒处理请求数
- http_load：类似ab，单策略更加灵活，支持多个URL
- JMeter：较复杂，但可以根据真实环境设置策略，如预热策略等

单组件式工具：

- mysqllap
- sql-bench
- super smack
- sysbench

# 使用sysbench进行基准测试
## 概述

## sysbench的CPU基准测试
CPU的基准测试，简单例子测试计算素数直到某个最大值所需要的时间。

1. 查看CPU配置：**cat /proc/cpuinfo**

```
root@iZwz9c3wp0ipx17m7be3s6Z:/mnt/mysql# cat /proc/cpuinfo
...
model name	: Intel(R) Xeon(R) CPU E5-2682 v4 @ 2.50GHz
stepping	: 1
microcode	: 0x1
cpu MHz		: 2494.220
cache size	: 40960 KB
...
```
2. 在这台服务器进行以下测试:**sysbench --test=cpu --cpu-max-prime=20000 run**

```
root@iZwz9c3wp0ipx17m7be3s6Z:/mnt/mysql# sysbench --test=cpu --cpu-max-prime=20000 run
WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
sysbench 1.0.11 (using system LuaJIT 2.1.0-beta3)
...
CPU speed:
    events per second:   377.61

General statistics:
    total time:                          10.0004s
    total number of events:              3777

Latency (ms):
         min:                                  2.61
         avg:                                  2.65
         max:                                  3.07
         95th percentile:                      2.71
         sum:                               9993.62

Threads fairness:
    events (avg/stddev):           3777.0000/0.00
    execution time (avg/stddev):   9.9936/0.00

```

3. 另外一个4核8G的服务器执行：

```
root@card-web:~# cat /proc/cpuinfo
...
model name	: Intel(R) Xeon(R) Platinum 8163 CPU @ 2.50GHz
stepping	: 4
microcode	: 0x1
cpu MHz		: 2500.006
...
```

4. 测试结果：

```
root@card-web:~# sysbench --test=cpu --cpu-max-prime=20000 run
sysbench 0.4.12:  multi-threaded system evaluation benchmark
...
Test execution summary:
    total time:                          27.7776s
    total number of events:              10000
    total time taken by event execution: 27.7762
    per-request statistics:
         min:                                  2.76ms
         avg:                                  2.78ms
         max:                                  6.36ms
         approx.  95 percentile:               2.79ms

Threads fairness:
    events (avg/stddev):           10000.0000/0.00
    execution time (avg/stddev):   27.7762/0.00

```

## sysbench的文件I/O基准测试
文件I/O基准测试可以测试系统在不同I/O负载下的性能。对于比较不同的磁盘驱动器、不同的RAID卡、不同的RAID模式都很有帮助。可以根据测试结果调整I/O子系统。文件I/O基准测试模拟了很多InnoDB的I/O特性。

步骤：

准备数据阶段（Prepare），生成用于测试的文件，生成的数据文件至少要比内存大。如果文件数据能够完全放入内存中，则操作系统会缓存大部分的数据，导致测试结果无法体现I/O密集型的工作负载。

```
$ sysbench --test=fileio --file-total-size=15G prepare

root@iZwz9c3wp0ipx17m7be3s6Z:/mnt/fileio# sysbench --test=fileio --file-total-size=15G prepare
WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
sysbench 1.0.11 (using system LuaJIT 2.1.0-beta3)

128 files, 122880Kb each, 15360Mb total
Creating files for the test...
Extra file open flags: 0
Creating file test_file.0
Creating file test_file.1
Creating file test_file.2
Creating file test_file.3
Creating file test_file.4
...

```
上述命令会创建测试文件，后续run阶段会通过读写这些文件进行测试。

run阶段，针对不停的I/O类型有不同的测试选项：
- seqwr：顺序写入
- seqrewr：顺序重写
- seqrd：顺序读取
- rndrd：随机读取
- rndwr：随机写入
- rndrwL：混合随机读/写

下面命令运行文件I/O混合随机读/写基准测试：

```
root@iZwz9c3wp0ipx17m7be3s6Z:/mnt/fileio# sysbench --test=fileio --file-total-size=15G --file-test-mode=rndrw --max-time=300 --max-requests=0 run
...
Running the test with following options:
Number of threads: 1
Initializing random number generator from current time

Extra file open flags: 0
128 files, 120MiB each
15GiB total file size
Block size 16KiB
Number of IO requests: 0
Read/Write ratio for combined random IO test: 1.50
Periodic FSYNC enabled, calling fsync() each 100 requests.
Calling fsync() at the end of test, Enabled.
Using synchronous I/O mode
Doing random r/w test
Initializing worker threads...

Threads started!


File operations:
    reads/s:                      1749.75
    writes/s:                     1166.50
    fsyncs/s:                     3732.39

Throughput:
    read, MiB/s:                  27.34
    written, MiB/s:               18.23

General statistics:
    total time:                          300.0052s
    total number of events:              1994639

Latency (ms):
         min:                                  0.00
         avg:                                  0.15
         max:                                241.42
         95th percentile:                      0.37
         sum:                             298296.11

Threads fairness:
    events (avg/stddev):           1994639.0000/0.00
    execution time (avg/stddev):   298.2961/0.00

```
输出包含了大量信息，和I/O相关的包括每秒请求数和总吞吐量。

测试完成后运行清除（cleanup）操作删除第一步生成的测试文件：

```
$ sysbench --test=fileio --file-total-size=15G cleanup
```
## sysbench 的OLTP基准测试
OLTP基准测试模拟了一个简单的事务处理系统的工作负载。下面的例子使用的是一张超过百万行记录的表，第一步是先生成这张表：(不同的sysbench版本，参数不一样，这里的版本是1.0.11)

```
$ sysbench --test=/usr/share/sysbench/oltp_read_write.lua --table-size=1000000 --tables=1 --db-driver=mysql --mysql-db=test --mysql-user=root --mysql-password=1qazXSW@ prepare

root@iZwz9c3wp0ipx17m7be3s6Z:/usr/share/sysbench# sysbench --test=/usr/share/sysbench/oltp_read_write.lua --table-size=1000000 --tables=1 --mysql-db=test --mysql-user=root --mysql-password=1qazXSW@ prepare
WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
sysbench 1.0.11 (using system LuaJIT 2.1.0-beta3)

FATAL: Multiple DB drivers are available. Use --db-driver=name to specify which one to use
FATAL: `sysbench.cmdline.call_command' function failed: ./oltp_common.lua:82: failed to initialize the DB driver
root@iZwz9c3wp0ipx17m7be3s6Z:/usr/share/sysbench# sysbench --test=/usr/share/sysbench/oltp_read_write.lua --table-size=1000000 --tables=1 --db-driver=mysql --mysql-db=test --mysql-user=root --mysql-password=1qazXSW@ prepare
WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
sysbench 1.0.11 (using system LuaJIT 2.1.0-beta3)

Creating table 'sbtest1'...
Inserting 1000000 records into 'sbtest1'
Creating a secondary index on 'sbtest1'...
```
接下来可以进行测试，这个例子使用8个并发线程，只读模式，测试60秒：

```
$ sysbench --test=/usr/share/sysbench/oltp_read_only.lua --table-size=1000000 --tables=1 --db-driver=mysql --mysql-db=test --mysql-user=root --mysql-password=1qazXSW@ --max-requests=0 --num-threads=8 --max-time=60 run

root@iZwz9c3wp0ipx17m7be3s6Z:/usr/share/sysbench# sysbench --test=/usr/share/sysbench/oltp_read_only.lua --table-size=1000000 --tables=1 --db-driver=mysql --mysql-db=test --mysql-user=root --mysql-password=1qazXSW@ --max-requests=0 --num-threads=8 --max-time=60 run
WARNING: the --test option is deprecated. You can pass a script name or path on the command line without any options.
WARNING: --num-threads is deprecated, use --threads instead
WARNING: --max-time is deprecated, use --time instead
sysbench 1.0.11 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 8
Initializing random number generator from current time


Initializing worker threads...

Threads started!

SQL statistics:
    queries performed:
        read:                            418306
        write:                           0
        other:                           59758
        total:                           478064
    transactions:                        29879  (497.88 per sec.)
    queries:                             478064 (7966.09 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0104s
    total number of events:              29879

Latency (ms):
         min:                                 11.20
         avg:                                 16.06
         max:                                 22.49
         95th percentile:                     17.63
         sum:                             479971.64

Threads fairness:
    events (avg/stddev):           3734.8750/10.83
    execution time (avg/stddev):   59.9965/0.00

```
上面输出了很多信息，最有价值的是：
- 总事务数：29879
- 每秒事务数：497.88 per sec
- 时间统计信息（最小、平均、最大响应）：Latency (ms)
- 线程公平性统计信息（Threads fairness），用于模拟负载的公平性。