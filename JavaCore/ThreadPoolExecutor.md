# 深入剖析线程池

# 概述
Java中的线程池适用场景广泛，几乎所有需要异步或并发执行任务的程序都可以使用线程池。

使用线程池的主要优点是：
- 在执行大量的异步任务的时候，线程池能够提供较好的处理性能，因为线程池通过重复利用已创建的线程降低线程创建和销毁造成的消耗，当任务到达时，任务可以不需要等到线程创建就能立即执行。
- 线程池提供了一种资源限制和管理的手段，线程作为系统稀缺资源，如果无限制地创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一分配、调优和监控。

线程池的最佳适用场景：
- 单个任务处理时间短；
- 需要处理的任务量大。

这里我们通过深入理解ThreadPoolExecutor来探究Java的线程池。

# 从ThreadPoolExecutor分析线程池原理
## 类图结构
当前JDK版本为1.8，以下是ThreadPoolExecutor的继承情况

![threadpool](https://github.com/eavenliu/book/blob/master/image/javacore/threadpool.png)

### Executor-线程池顶级接口类
Executor接口只拥有一个方法：

```
/**
 * 在将来的某个时间点执行传入的command。
 * 这个command可能会在一个新创建线程中执行、一个线程池已有的线程中执行、或者是在调用线程中执行，具体由Executor的实现决定。
 * @param command the runnable task
 * @throws RejectedExecutionException if this task cannot be
 * accepted for execution
 * @throws NullPointerException if command is null
 */
void execute(Runnable command);
```
该接口将任务提交给线程池，由线程池为该任务创建线程并启动。注意这个方法没有返回值，获取不到线程执行结果。

### ExecutorService-线程池扩展基础接口
ExecutorService提供用于管理终止的方法如 shutDown()和shutDownNow()：

```
void shutdown();

List<Runnable> shutdownNow();
```
用于关闭线程池的方法以及判断线程池是否关闭的方法如，isShutdown()，isTerminated()的方法：

```
/**
 * 如果当前的executor被shutdown则返回true
 * @return {@code true} if this executor has been shut down
 */
boolean isShutdown();

/**
 * 如果所有任务在关闭后都已完成，则返回{@code true}。
 * 请注意，除非先调用{@codeshutdown}或{@code shutdownNow}，否则{@code isTerminated}永远不会是{@code true}。
 * @return {@code true} if all tasks have completed following shut down
 */
boolean isTerminated();
```
提供了可以生成用于跟踪一个或多个异步任务进度的方法如，invokeAll()，submit()。这些方法的返回值都是Future类型，可以获取线程的执行结果：

```
//提交要执行的返回值任务，并返回表示任务的返回结果的Future。 其中Future的{@code get}方法将在成功完成后返回任务的结果。
//如果要立即阻塞并等待任务，则可以使用类似这样的构造：{@code result = exec.submit(aCallable).get();}
//注意：{@link Executors}类包含一组方法，这些方法可以将其他一些类似闭包的常见对象转换为{@link Callable}形式，例如将{@link java.security.PrivilegedAction}转换为{@link Callable}形式。
<T> Future<T> submit(Callable<T> task);

//提交一个Runnable任务以执行并返回一个表示该任务的Future。 Future的{@code get}方法将在成功完成后返回给定的结果。
<T> Future<T> submit(Runnable task, T result);

//提交一个Runnable任务以执行并返回一个表示该任务的Future。 <em>successful</em>完成时，Future的{@code get}方法将返回{@code null}。
Future<?> submit(Runnable task);

//执行给定的任务，并在所有任务完成时返回保存其状态和结果的Futures列表。
//对于返回列表的每个元素，{@ link Future＃isDone}为{@code true}。 
//请注意，<em> completed </ em>任务可能已正常终止或引发了异常。如果在进行此操作时修改了给定的集合，则此方法的结果不确定。
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

//执行给定的任务，如果成功，则返回成功完成任务（即不引发异常）的结果。 
//在正常或异常返回时，尚未完成的任务将被取消。 如果在进行此操作时修改了给定的集合，则此方法的结果不确定。
<T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
<T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
```

### AbstractExecutorService
AbstractExecutorService抽象类实现ExecutorService接口的以下方法：

![image](http://note.youdao.com/yws/res/143205/9C9C39F4877C4509AA7BB78247843313)

### ThreadPoolExecutor
ThreadPoolExecutor继承了AbstractExecutorService，ThreadPoolExecutor是Java线程池的核心实现。

ThreadPoolExecutor的成员变量有：

```
/**
 * 该变量是对线程池的运行状态和线程池中有效线程的数量进行控制的一个字段。
 * ctl是一个AtomicInteger, 它包含两部分的信息: 
 * runState:高三位表示线程池的运行状态 
 * workerCount:低29位表示线程池内有效线程的数量
 *
 * 默认是RUNNING状态，线程个数为0
 **/
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
//线程个数掩码位数
private static final int COUNT_BITS = Integer.SIZE - 3;
//线程最大个数(低29位)00011111111111111111111111111111
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
// runState存储在高位，线程池的生命周期，总共有五种状态
//（高3位）：11100000000000000000000000000000
private static final int RUNNING    = -1 << COUNT_BITS;
//（高3位）：00000000000000000000000000000000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
//（高3位）：00100000000000000000000000000000
private static final int STOP       =  1 << COUNT_BITS;
//（高3位）：01000000000000000000000000000000
private static final int TIDYING    =  2 << COUNT_BITS;
//（高3位）：01100000000000000000000000000000
private static final int TERMINATED =  3 << COUNT_BITS;

// 获取高三位 运行状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 获取低29位 线程个数
private static int workerCountOf(int c)  { return c & CAPACITY; }
// 计算ctl新值，线程状态 与 线程个数
private static int ctlOf(int rs, int wc) { return rs | wc; }

/**
 * 尝试CAS递增ctl的workerCount字段。
 */
private boolean compareAndIncrementWorkerCount(int expect) {
    return ctl.compareAndSet(expect, expect + 1);
}
/**
 * 尝试CAS递减ctl的workerCount字段。
 */
private boolean compareAndDecrementWorkerCount(int expect) {
    return ctl.compareAndSet(expect, expect - 1);
}

/**
 * 减少ctl的workerCount字段。 仅在线程突然终止时调用此方法（请参阅processWorkerExit）。 其他减量在getTask中执行。
 */
private void decrementWorkerCount() {
    do {} while (! compareAndDecrementWorkerCount(ctl.get()));
}

/**
 * 用于持有任务并移交给工作线程的队列。
 * 我们不要求当workQueue.poll()返回null必然意味着workQueue.isEmpty()，因此仅依靠isEmpty来查看队列是否为空（例如，在决定是否从SHUTDOWN过渡到TIDYING时必须这样做）。。
 * 这可容纳诸如DelayQueues之类的专用队列，这些队列允许poll()返回null，即使它可能在延迟到期后稍后返回non-null。
 */
private final BlockingQueue<Runnable> workQueue;

/**
 * Set包含线程池中的所有工作线程。仅在持有mainLock锁（final ReentrantLock mainLock）时访问。
 */
private final HashSet<Worker> workers = new HashSet<Worker>();

/**
 * 跟踪达到的最大线程池大小。 仅在持有mainLock下访问。
 */
private int largestPoolSize;

/**
 * 计数器完成的任务。 仅在终止工作线程时更新。 仅在持有mainLock下访问。
 */
private long completedTaskCount;
/**
 * 创建新线程的工厂类。 
 * 所有线程都是使用此工厂创建的（通过addWorker方法）。必须为所有调用程序做好准备，以使addWorker失败，这可能反映出系统或用户的策略限制了线程数。
 * 即使未将其视为错误，创建线程失败也可能导致新任务被拒绝或现有任务仍停留在队列中。
 */
private volatile ThreadFactory threadFactory;
/**
 * 在执行饱和或关闭时调用处理程序。
 */
private volatile RejectedExecutionHandler handler;
/**
 * 空闲线程等待工作的超时时间（以纳秒为单位）。
 * 当存在多个corePoolSize或allowCoreThreadTimeOut时，线程将使用此超时。 否则，他们将永远等待。
 */
private volatile long keepAliveTime;
/**
 * 如果为false（默认值），则即使处于空闲状态，核心线程也保持活动状态。 
 * 如果为true，则核心线程使用keepAliveTime来超时等待工作。
 */
private volatile boolean allowCoreThreadTimeOut;
/**
 * 除非设置allowCoreThreadTimeOut，否则核心线程池大小是保持alive（不允许超时等）工作的最小数量，在这种情况下，最小值为零。
 */
private volatile int corePoolSize;
/**
 * 最大线程池大小。 请注意，实际最大值在内部受CAPACITY限制。
 */
private volatile int maximumPoolSize;
/**
 * 默认拒绝执行处理程序
 */
private static final RejectedExecutionHandler defaultHandler =
    new AbortPolicy();

/* 执行终结器时使用的上下文，或者为null。 */
private final AccessControlContext acc;
```
现在解释关键的成员：
#### 线程池状态
线程池的生命周期，总共有五种状态：
- RUNNING：接受新任务并且处理阻塞队列里的任务；
- SHUTDOWN：关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务。在线程池处于 RUNNING 状态时，调用 shutdown()方法会使线程池进入到该状态。（finalize() 方法在执行过程中也会调用shutdown()方法进入该状态）；
- STOP：拒绝新任务并且抛弃阻塞队列里的任务同时会中断正在处理的任务。在线程池处于 RUNNING 或 SHUTDOWN 状态时，调用 shutdownNow() 方法会使线程池进入到该状态；
- TIDYING：如果所有的任务都已终止了，workerCount (有效线程数) 为0，线程池进入该状态后会调用 terminated() 方法进入TERMINATED 状态。
- TERMINATED：终止状态，在terminated() 方法执行完后进入该状态，默认terminated()方法中什么也没有做。

线程池状态的转化：
- RUNNING -> SHUTDOWN：
显式调用shutdown()方法，或者隐式调用了finalize(),它里面调用了shutdown（）方法。
- (RUNNING or SHUTDOWN) -> STOP：
显式 shutdownNow()方法
- SHUTDOWN -> TIDYING：
当线程池和任务队列都为空的时候
- STOP -> TIDYING：
当线程池为空的时候
- TIDYING -> TERMINATED：
当 terminated() hook 方法执行完成时候

#### 线程池初始化参数（ThreadPoolExecutor构造函数包含参数）
##### corePoolSize
**corePoolSize：核心线程数量，除非设置了{@code allowCoreThreadTimeOut}，即使它们处于空闲状态也要保留在池中的线程数。**

**当有新任务在execute()方法提交时，会执行以下判断**：
1. 如果运行的线程少于 corePoolSize，则创建新线程来处理任务，即使线程池中的其他线程是空闲的9（因为我们认为corePoolSize数量的线程是最佳的性能体现）；
2. 如果线程池中的线程数量大于等于 corePoolSize 且小于 maximumPoolSize，当workQueue未满的时候任务添加到workQueue中，当workQueue满时才创建新的线程去处理任务；
3. 如果设置的corePoolSize 和 maximumPoolSize相同，则创建的线程池的大小是固定的，这时如果有新任务提交，若workQueue未满，则将请求放入workQueue中，等待有空闲的线程去从workQueue中取任务并处理；
4. 如果运行的线程数量大于等于maximumPoolSize，这时如果workQueue已经满了，则通过handler所指定的策略来处理任务；

**所以，任务提交时，判断任务具体被执行的顺序为 corePoolSize –> workQueue –> maximumPoolSize -> 饱和策略逻辑。**

##### maximumPoolSize
**线程池中允许的最大线程数**
##### keepAliveTime
线程池维护线程所允许的空闲时间。当线程池中的线程数量大于corePoolSize的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了keepAliveTime；
##### TimeUnit
{@code keepAliveTime}参数的时间单位
##### workQueue
用于保存等待执行的任务的阻塞队列。此队列将仅保存由{@code execute}方法提交的{@code Runnable}任务。

当提交一个新的任务到线程池以后,线程池会根据当前线程池中正在运行着的线程的数量来决定对该任务的处理方式，主要有以下几种处理方式:
- 直接切换：这种方式常用的队列是SynchronousQueue，最多只有一个元素的同步队列。
- 使用无界队列：**一般使用基于链表的阻塞队列LinkedBlockingQueue**。如果使用这种方式，那么线程池中能够创建的最大线程数就是corePoolSize，而maximumPoolSize就不会起作用了。当线程池中所有的核心线程都是RUNNING状态时，这时一个新的任务提交就会放入等待队列中。
- 使用有界队列：**一般使用ArrayBlockingQueue**。使用该方式可以将线程池的最大线程数量限制为maximumPoolSize，这样能够降低资源的消耗，但同时这种方式也使得线程池对线程的调度变得更困难，因为线程池和队列的容量都是有限的值，所以要想使线程池处理任务的吞吐率达到一个相对合理的范围，又想使线程调度相对简单，并且还要尽可能的降低线程池对资源的消耗，就需要合理的设置这两个数量。

如果要想降低系统资源的消耗（包括CPU的使用率，操作系统资源的消耗，上下文环境切换的开销等）, 可以设置较大的队列容量和较小的线程池容量, 但这样也会降低线程处理任务的吞吐量。

如果提交的任务经常发生阻塞，那么可以考虑通过调用 setMaximumPoolSize() 方法来重新设定线程池的容量。

如果队列的容量设置的较小，通常需要将线程池的容量设置大一点，这样CPU的使用率会相对的高一些。但如果线程池的容量设置的过大，则在提交的任务数量太多的情况下，并发量会增加，那么线程之间的调度就是一个要考虑的问题，因为这样反而有可能降低处理任务的吞吐量。

##### ThreadFactory
创建线程的工厂。

默认使用Executors.defaultThreadFactory() 来创建线程。使用默认的ThreadFactory来创建线程时，会使新创建的线程具有相同的NORM_PRIORITY优先级并且是非守护线程，同时也设置了线程的名称。
##### RejectedExecutionHandler
饱和策略，当队列满了并且线程个数达到maximunPoolSize后采取的策略，比如
- AbortPolicy：直接抛出异常，默认策略
- CallerRunsPolicy：使用调用者所在线程来运行任务
- DiscardOldestPolicy：调用poll丢弃阻塞队列中最靠前一个任务，执行当前任务
- DiscardPolicy：直接丢弃任务,不抛出异常

#### 线程池execute()任务逻辑
execute(Runnable command)源码：
```
public void execute(Runnable command) {
    if (command ** null)
        throw new NullPointerException();
    int c = ctl.get();
    // 判断当前工作线程是否小于核心线程池数量
    if (workerCountOf(c) < corePoolSize) {
        //创建新的线程执行任务
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 当前工作线程大于核心线程池数量，判断线程池是否可以接收新的任务并且任务阻塞队列是否已满，将任务加入BlockingQueue
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 添加任务至阻塞队列后，如果线程池状态不为RUNNING，则去掉刚添加到workQueue的任务
        if (! isRunning(recheck) && remove(command))
            reject(command);
        //如果线程池没有一个alive的线程，则添加一个空的工作线程，保证核心线程池的核心线程存在，预热线程池
        else if (workerCountOf(recheck) ** 0)
            addWorker(null, false);
    }
    //如果无法将任务加入BlockingQueue（队列已满），则创建新的非核心线程来处理任务
    else if (!addWorker(command, false))
        // 如果创建新线程将使当前运行的线程超出maximumPoolSize，任务将被拒绝，调用饱和策略
        reject(command);
    }
```
流程图如下：

![image](http://note.youdao.com/yws/res/143421/E0B3D3CC78E947CC8174B3509DDC68F5)

> 核心线程和非核心线程没有实质的区别，核心线程的数量是我们对应线程池常驻线程数量基于性能的估算

#### 线程池中的线程如何保证被重复利用
线程在执行完run()方法里面的具体逻辑后就会被GC回收，那么线程池是怎样保持线程的存活，并且重复利用线程的呢？

在线程池中我们使用Worker类来包装向线程池中添加的Runnable线程任务。首先来分析一下addWorker()方法。

##### addWorker源码分析

```
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        // 获取当前线程池状态
        int rs = runStateOf(c);
        // 判断当前线程池状态为STOP，TIDYING，TERMINATED、当前线程池状态为SHUTDOWN并且已经有了第一个任务 或 当前线程池状态为SHUTDOWN并且任务队列为空，返回false
        if (rs >= SHUTDOWN &&
            ! (rs ** SHUTDOWN &&
               firstTask ** null &&
               ! workQueue.isEmpty()))
            return false;
        // 循环cas增加线程个数
        for (;;) {
            //获取当前线程池的线程数量
            int wc = workerCountOf(c);
            //如果线程数量超过线程池最大容量，返回false
            //如果是增加核心线程，且当前线程数 >= corePoolSize，应该将任务添加至workQueue，返回false
            //如果是增加非核心线程,且当前线程数 >= maximumPoolSize，表示线程池已满，返回false
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            //CAS增加当前线程数量，同时只有一个线程成功
            if (compareAndIncrementWorkerCount(c))
                break retry;
            //cas失败了，则看线程池状态是否变化了，变化则跳到外层循环重试重新获取线程池状态，否者内层循环重新cas。
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
        }
    }
    
    //CAS增加线程数量成功
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        //创建work实例
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            //加锁，这里会是性能瓶颈，但是只要预热完毕并且保持核心线程的存活，并且任务都可以添加到安全队列里去，就可以有效的避免性能瓶颈。
            final ReentrantLock mainLock = this.mainLock;
            //使用全局的独占锁，保证works同步，因为可能会有多个线程调用了线程池的execute方法
            mainLock.lock();
            try {
                // 重新检查状态，防止获取锁前被shutdown
                int rs = runStateOf(ctl.get());
                //保证线程池处于RUNNING状态，或者线程池处于关闭状态且添加的工作线程为null
                if (rs < SHUTDOWN ||
                    (rs ** SHUTDOWN && firstTask ** null)) {
                    //保证线程还没启动
                    if (t.isAlive()) 
                        throw new IllegalThreadStateException();
                    //添加工作线程
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            //添加成功则启动任务
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            //如果工作线程没有启动成功，加锁回退工作线程数量，并且调用tryTerminate()方法
            addWorkerFailed(w);
    }
    return workerStarted;
}
```
由上可以看出addWorker主要的逻辑还是新增Worker，那么继续分析Worker的源码。

##### 工作线程Worker源码分析（保证线程池工作线程重复使用）
Worker的类图如下：

![image](http://note.youdao.com/yws/res/143508/31124C14B0874C82B58440743EA79CFB)

Worker类继承Runnable,和AbstractQueuedSynchronizer（这个类奠定了Java并发包的基础）

构造函数是：

```
/**
 * Creates with given first task and thread from ThreadFactory.
 * @param firstTask the first task (null if none)
 */
Worker(Runnable firstTask) {
    // 在调用runWorker前禁止中断
    setState(-1); 
    this.firstTask = firstTask;
    //创建一个新线程
    this.thread = getThreadFactory().newThread(this);
}
```
这里添加一个新状态-1是为了避免当前worker线程被中断，比如调用了线程池的shutdownNow,如果当前worker状态>=0则会设置该线程的中断标志。这里设置了-1所以条件不满足就不会中断该线程了。运行runWorker时候会调用unlock方法，该方法吧status变为了0，所以这时候调用shutdownNow会中断worker线程。

线程run()方法重写是：

```
//将主运行循环委托给外部runWorker
public void run() {
    runWorker(this);
}
```

runWorker的源码是：
```
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // status设置为0，允许中断
    w.unlock(); 
    boolean completedAbruptly = true;
    try {
        //当第一次进入的时候task！=null，继续执行wihle里面的逻辑
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // 如果线程池当前状态至少是stop，则设置中断标志;
            // 如果线程池当前状态是RUNNININ，则重置中断标志，重置后需要重新
            // 检查下线程池状态，因为当重置中断标志时候，可能调用了线程池的shutdown方法改变了线程池状态。
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                //执行任务前的处理
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //任务执行完的处理
                    afterExecute(task, thrown);
                }
            } finally {
                //执行完毕将task设置为null
                task = null;
                //统计当前worker完成了多少个任务
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        //worker退出逻辑
        processWorkerExit(w, completedAbruptly);
    }
}
```
由上可知，如果当前task不为空，则直接执行，否者调用getTask从任务队列获取一个任务执行，如果任务队列为空，则worker退出。

所以，当第一个task执行完毕，如何继续复用当前启动的线程处理下一个任务取决于getTask()实现，具体源码：

```
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?
    //阻塞获取新的任务
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);
        // 判断当前线程池是否关闭或者队列中是否存在可执行的任务，如果都不成立，那么直接返回null，销毁当前的工作线程
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
    
        int wc = workerCountOf(c);
    
        // 由于我们可以在线程池初始化时定义超时时间来控制线程池中空闲worker的生命周期。
        // allowCoreThreadTimeOut表示是否设置了超时时间，wc > corePoolSize表示添加的是workQueue队列中可能存在的缓存的任务
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        // 判断当前线程数超过最大线程容量或线程空闲并且超时，同时任务队列为空，回收当前worker
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }
        try {
            //获取一个任务
            // 当timed ** true时，表示通过poll超时获取队列中的任务，超过时间直接返回null
            // 当timed ** false时，通过take()阻塞获取一个任务，队列中没有任务就一直阻塞
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                //获取任务不为空就直接返回这个任务，然后会在worker的run方法中的while循环中继续执行
                return r;
            //标识当前worker空闲
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
    }
```
综上，我们可以发现线程池是如何保证线程（Worker）的重复利用。

##### 工作线程如何被销毁--processWorkerExit源码

```
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();
    //获取全局锁，追加统计整个线程池完成的任务个数
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        //
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
    //尝试设置线程池状态为TERMINATED，如果当前是shutdonw状态并且工作队列为空
    //或者当前是stop状态当前线程池里面没有活动线程
    tryTerminate();
    //如果当前线程个数小于核心线程corePoolSize个数，则增加空任务线程
    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min ** 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```

#### 线程池的shutdown操作
##### shutdown()
调用shutdown()后，线程池就不会在接受新的工作任务了，但是工作队列里面的任务根据上面的分析还是要继续执行的，但是该方法立刻返回，并不等待队列任务完成再返回。

```
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //权限检查
        checkShutdownAccess();
        //设置当前线程池状态为SHUTDOWN，如果已经是SHUTDOWN则直接返回
        advanceRunState(SHUTDOWN);
        //设置中断标志
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    // 尝试状态变为TERMINATED
    tryTerminate();
}

private void interruptIdleWorkers() {
    interruptIdleWorkers(false);
}

private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}

```
##### shutdownNow()操作
调用shutdownNow后，线程池就不会在接受新的任务了，并且丢弃工作队列里面里面的任务，正在执行的任务会被中断，但是该方法立刻返回的，并不等待激活的任务执行完成在返回。返回队列里面的任务列表。

```
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //权限检查
        checkShutdownAccess();
        //设置当前线程池状态为STOP
        advanceRunState(STOP);
        //中断工作线程
        interruptWorkers();
        //移动队列任务到tasks
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
//调用队列的drainTo一次将当前队列的元素移动到taskList，可能失败，如果调用drainTo后队列还不为空，则循环删除，并添加到taskList
private List<Runnable> drainQueue() {
    BlockingQueue<Runnable> q = workQueue;
    ArrayList<Runnable> taskList = new ArrayList<Runnable>();
    q.drainTo(taskList);
    if (!q.isEmpty()) {
        for (Runnable r : q.toArray(new Runnable[0])) {
            if (q.remove(r))
                taskList.add(r);
        }
    }
    return taskList;
}
```

# 目前JDK默认支持的线程池类型--Executors
Executors类封装了几个通用场景的线程池创建，虽然不适合很复杂的场景，但一般场景可以参考使用（最好是自己理解后根据具体业务实际情况设置不同的参数创建ThreadPoolExecutor）：

![image](http://note.youdao.com/yws/res/143672/1E82A68BEABC4B05AE04E0072DCFE9F3)

根据上面的静态封装方法,我们可以看出，Executors可以创建以下几种类型的线程池：
- newCachedThreadPool：返回ExecutorService线程池
- newFixedThreadPool：返回ExecutorService线程池
- newSingleThreadExecutor：返回ExecutorService线程池
- newWorkStealingPool：返回ExecutorService线程池
- ScheduledThreadPool：返回ScheduledExecutorService线程池
- newSingleScheduledThreadExecutor：返回ScheduledExecutorService线程池

## newFixedThreadPool---固定数量的线程池
**使用固定核心线程数的newFixedThreadPool，适用于为了满足资源管理的需求，而需要限制当前线程数量的应用场景，它适用于负载比较重的服务器。**

实例化代码：
```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
//使用自定义线程ThreadFactory的方式创建线程池
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
```
根据源码可以看出，分析对应的**线程池参数**：
- corePoolSize ** maximumPoolSize ：都为初始化的参数nThreads
- workQueue：使用无界队列LinkedBlockingQueue链表阻塞队列
- keepAliveTime：0L，由于使用无界队列LinkedBlockingQueue作为workQueue（阻塞队列长度为Integer.MAX_VALUE），所以当corePoolSize满后，后面添加的线程任务都会添加到workQueue中去，所以maximumPoolSize就失去了意义，这样也就没有必要设置空闲时间。

**当我们使用无界队列LinkedBlockingQueue作为workQueue，会影响**：
1. 当线程池中的线程数达到corePoolSize后，新任务将在无界队列中等待，并不会创建非核心线程，因此线程池中的线程数不会超过corePoolSize。即maximumPoolSize、keepAliveTime将是无效参数。
2. 由于使用无界队列，运行中的FixedThreadPool（未执行方法shutdown()或shutdownNow()）不会拒绝任务（不会调用RejectedExecutionHandler.rejectedExecution方法）。

举例：

```
@Test
public void testNewFixThreadPool(){
    ExecutorService executorService = Executors.newFixedThreadPool(10, new CustomizableThreadFactory("ThreadTest-"));
    for (int i = 0; i < 50; i++){
        executorService.execute(() -> {
            Thread t = Thread.currentThread();
            System.out.println("当前的线程是：" + t.getName());
        });
    }
    executorService.shutdown();
}
```
输出是如下，**可以发现当核心线程创建到指定数量时，才会复用核心线程**。

```
当前的线程是：ThreadTest-1
当前的线程是：ThreadTest-2
当前的线程是：ThreadTest-3
当前的线程是：ThreadTest-4
当前的线程是：ThreadTest-5
当前的线程是：ThreadTest-6
当前的线程是：ThreadTest-7
当前的线程是：ThreadTest-8
当前的线程是：ThreadTest-9
当前的线程是：ThreadTest-9
当前的线程是：ThreadTest-10
当前的线程是：ThreadTest-2
当前的线程是：ThreadTest-1
当前的线程是：ThreadTest-10
当前的线程是：ThreadTest-6
当前的线程是：ThreadTest-8
当前的线程是：ThreadTest-4
当前的线程是：ThreadTest-3
...
```
## newSingleThreadExecutor---单例线程池
**适用于需要保证顺序地执行各个任务；并且在任意时间点，不会有多个线程是活动的应用场景。**

```
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>(),
                                threadFactory));
}
```
根据源码可以看出，分析对应的线程池参数：
- corePoolSize ** maximumPoolSize ：1，单例线程池，所以线程池只有一个重用的线程
- workQueue：使用无界队列LinkedBlockingQueue链表阻塞队列
- keepAliveTime：0L

举例：

```
@Test
public void testNewSingleThreadPool(){
    ExecutorService executorService = Executors.newSingleThreadExecutor(new CustomizableThreadFactory("ThreadTest-"));
    executorService.execute(() -> {
        System.out.println("提交任务：1");
        Thread t = Thread.currentThread();
        System.out.println("当前的线程是：" + t.getName());
    });
    executorService.execute(() -> {
        System.out.println("提交任务：2");
        Thread t = Thread.currentThread();
        System.out.println("当前的线程是：" + t.getName());
    });
    executorService.execute(() -> {
        System.out.println("提交任务：3");
        Thread t = Thread.currentThread();
        System.out.println("当前的线程是：" + t.getName());
    });
    executorService.shutdown();
}
```
输出如下，单线程按顺序执行任务：

```
提交任务：1
当前的线程是：ThreadTest-1
提交任务：2
当前的线程是：ThreadTest-1
提交任务：3
当前的线程是：ThreadTest-1
```
## newCachedThreadPool---缓存线程池
**创建一个会根据需要创建新线程的，适用于执行很多的短期异步任务的小程序，或者是负载较轻的服务器。**

```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
//还有个自定义ThreadFactory的方法
```
根据源码可以看出，分析对应的线程池参数：
- corePoolSize：0，表示无核心线程，线程池中都是非核心线程
- maximumPoolSize：Integer.MAX_VALUE，表示线程池容量可以非常大
- keepAliveTime：60秒，由于没有核心线程的存在，所以设置空闲时间60秒，当非核心线程60秒后没有被重用，将会被销毁，如果没有线程提交给该线程池，超过空闲时间，该线程池就没有非空闲线程，那么该线程池也就不会消耗过多的资源
- workQueue：SynchronousQueue是一个不存储元素的阻塞队列。每一个put操作必须等待一个take操作，否则不能继续添加元素。会导致优先创建线程处理任务

举例说明：
```
@Test
public void testNewCacheThreadPool(){
    ExecutorService executorService = Executors.newCachedThreadPool(new CustomizableThreadFactory("ThreadTest-"));
    for (int i = 1; i < 50; i++){
        final int index = i;
        executorService.execute(() -> {
            Thread t = Thread.currentThread();
            System.out.println("当前的线程是：" + t.getName()+ " 执行任务：" + index);
        });
    }
    executorService.shutdown();
}
```
输出：
```
当前的线程是：ThreadTest-1 执行任务：1
当前的线程是：ThreadTest-2 执行任务：2
当前的线程是：ThreadTest-3 执行任务：3
当前的线程是：ThreadTest-4 执行任务：4
当前的线程是：ThreadTest-5 执行任务：5
当前的线程是：ThreadTest-6 执行任务：6
当前的线程是：ThreadTest-6 执行任务：8
当前的线程是：ThreadTest-4 执行任务：12
当前的线程是：ThreadTest-3 执行任务：9
当前的线程是：ThreadTest-2 执行任务：11
当前的线程是：ThreadTest-1 执行任务：10
当前的线程是：ThreadTest-4 执行任务：16
当前的线程是：ThreadTest-6 执行任务：17
当前的线程是：ThreadTest-2 执行任务：14
当前的线程是：ThreadTest-3 执行任务：15
...
```
## newScheduledThreadPool---定时线程池
**它主要用来在给定的延迟之后运行任务，或者定期执行任务，例如定时轮询数据库中的表的数据。**
```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
public static ScheduledExecutorService newScheduledThreadPool(
        int corePoolSize, ThreadFactory threadFactory) {
    return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
}
```
ScheduledThreadPoolExecutor继承自ThreadPoolExecutor，构造函数是：
```
public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue(), threadFactory);
}
```
分析线程池参数：
- corePoolSize：corePoolSize，用户自定义
- maximumPoolSize：Integer.MAX_VALUE
- keepAliveTime：0
- workQueue：DelayedWorkQueue，使用延迟队列作为缓存队列

**ScheduledThreadPoolExecutor线程池任务提交方式也有差别**：

![image](http://note.youdao.com/yws/res/143810/07373779411344D985454DB64A406277)

```
/**
 * 该方法表示在给定的delay延迟时间后执行一次
 * @param command 提交的任务，Callable有返回值
 * @param delay 执行延迟时间
 * @param unit 时间单位
 */
public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit);
public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit);
/**
 * 该方法在initialDelay时间后开始周期性的按period时间间隔执行任务
 * @param command 提交的任务
 * @param initialDelay 初始延迟时间
 * @param period 表示两个任务连续执行的时间周期，第一个任务开始到第二个任务的开始，包含了任务的执行时间
 * @param unit 时间单位
 */
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit);

/**
 * 
 * @param command 提交Runnable任务
 * @param initialDelay 第一次执行初始延迟时间
 * @param delay 表示延迟时间 第一个任务结束到第二个任务开始的时间间隔
 * @param unit 时间单位
 */
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                 long initialDelay,
                                                 long delay,
                                                 TimeUnit unit);
```

举例说明：
```
@Test
public void testNewScheduledThreadPool() throws Exception{
    ScheduledExecutorService executorService = Executors.newScheduledThreadPool(10, new CustomizableThreadFactory("ThreadTest-"));
    Runnable runnable = new Runnable() {
        @Override
        public void run() {
            SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd hh:mm:ss");
            System.out.println(format.format(new Date()));
        }
    };
    executorService.scheduleWithFixedDelay(runnable, 10,5, TimeUnit.SECONDS);
    //executorService.shutdown();
    Thread.sleep(200000);
}
```
输出：
```
2019-03-25 12:19:14
2019-03-25 12:19:19
2019-03-25 12:19:24
2019-03-25 12:19:29
2019-03-25 12:19:34
2019-03-25 12:19:39
2019-03-25 12:19:44
2019-03-25 12:19:49
...
```