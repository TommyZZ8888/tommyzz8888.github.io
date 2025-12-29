---
title: threadpool
date: '2025-12-28T09:38:23+08:00'

categories:
- juc
---




线程池属于开发中常见的一种**池化技术**，这类的池化技术的目的都是为了提高资源的利用率和提高效率，类似的HttpClient连接池，数据库连接池等。

在没有线程池的时候，我们要创建多线程的并发，一般都是通过继承 Thread 类或实现 Runnable 接口或者实现 Callable 接口，我们知道线程资源是很宝贵的，而且线程之间切换执行时需要记住上下文信息，所以过多的创建线程去执行任务会造成资源的浪费而且对CPU影响较大。

为了方便， JDK 1.5 之后为我们提供了几种创建线程池的方法：

- **Executors.newFixedThreadPool(nThreads)**：创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。
- **Executors.newCachedThreadPool()**：创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。
- **Executors.newSingleThreadExecutor()**：创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务， 保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。
- **Executors.newScheduledThreadPool(nThreads)**：创建一个定长线程池，支持定时及周期性任务执行。

虽然这些都是 JDK 默认提供的，但是还是要说它们的定制性太差了而且有点鸡肋，很多时候不能满足我们的需求。例如通过 newFixedThreadPool 方式创建的固定线程池，它内部使用的队列是 LinkedBlockingQueue，但是它的队列大小默认是 Integer.MAX_VALUE，这会有什么问题？

当核心线程满了的时候，任务会进入队列中等待，直到队列满了为止。但是也许任务还未达到 `Integer.MAX_VALUE` 这个值的时候，内存就已经 OOM 了，因为内存放不下这么多的任务，毕竟内存大小有限。

所以更多的时候我们都是自定义线程池，也就是使用 new ThreadPoolExecutor 的方式，其实你看源码你可以发现以上的4个线程池技术底层都是通过 ThreadPoolExecutor 来创建的，只不过它们自己为我们填充了这些参数的固定值而已。

ThreadPoolExecutor 的构造函数如下所示：

```java
ThreadPoolExecutor(int corePoolSize,
                   int maximumPoolSize,
                   long keepAliveTime,
                   TimeUnit unit,
                   BlockingQueue<Runnable> workQueue,
                   ThreadFactory threadFactory,
                   RejectedExecutionHandler handler);
```

我们来看下这几个核心参数的涵义和作用：

- **corePoolSize**： 为线程池的核心线程基本大小。
- **maximumPoolSize**： 为线程池最大线程大小。
- **keepAliveTime** 和 **unit** 则是线程空闲后的存活时间。
- **workQueue**： 用于存放任务的阻塞队列。
- **handler**： 当队列和最大线程池都满了之后的饱和策略。

通过这些参数的配置使得整个线程池的工作流程如下：

[![img](..\img\multiplethread\threadpool\pool01.png)](https://img2018.cnblogs.com/blog/1162587/201909/1162587-20190902171916863-1777758429.png)

前几年一般普通的技术面试了解了以上的知识内容也差不多就够了，但是目前的大环境的影响或者面试更高级的开发上面的知识点是经不起深度考问的。例如以下几个问题你是否了解：线程池的内部有哪些状态？是如何判断核心线程数是否已满的？最大线程数是否包含核心线程数？当线程池中的线程数刚好达到 maximumPoolSize 这个值的时候，这个任务能否正常被执行？......，想要了解这些问题的答案我们只能在线程池的源码中寻找了。

# 实战模拟测试[#](https://www.cnblogs.com/jajian/p/11442929.html#实战模拟测试)

我们自定义一个线程池，然后通过 for 循环连续创建10个任务并打印线程执行信息，整体代码如下所示：

```java
public static void main(String[] args) {

    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(3, 6, 5L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>(4));
    
    for (int i = 0; i < 10; i++) {
        threadPoolExecutor.execute(() -> {
             System.out.println("测试线程池：" + Thread.currentThread().getName() + "," + threadPoolExecutor.toString());
        });
    }
}
```

当 corePoolSize = 3，maximumPoolSize = 6，workQueue 大小为4的时候，我们的打印信息为：

[![img](..\img\multiplethread\threadpool\pool02.png)](https://img2018.cnblogs.com/blog/1162587/201909/1162587-20190907173300360-1101768066.png)

可以发现总的创建了6个线程来执行完成了10个任务，其实很好理解，c=3个核心线程执行了3个任务，然后4个任务在队列中等待核心线程执行，最后额外创建了e=3个线程执行了剩下的3个任务，总创建的线程数就是 c + e = 6 <= 6（最大线程数）。

如果我们调整对象创建的时候的构造函数参数，例如

```java
ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(3, 5, 5L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>(2));
```

我们再次执行上述的代码，则会报错，抛出如下 RejectedExecutionException 异常信息，可以看到是因为拒绝策略拦截的异常信息。

[![img](..\img\multiplethread\threadpool\pool03.png)](https://img2018.cnblogs.com/blog/1162587/201909/1162587-20190907173315516-925878331.png)

还是按照上面的逻辑分析，这时核心线程数是 c = 3，而阻塞队列的大小是 2，因此核心线程会处理掉其中5个任务，而剩下的5个任务会额外创建 e=5个线程去执行，那么总线程数就是 c + e = 8，但是这时的最大线程数 maximumPoolSize = 5，因此超过了最大线程数的限制，这时就执行了默认的拒绝策略抛出异常。其实它在准备创建第6个线程的时候就已经报错了，从这里也可以得知**只要创建的总线程数 >= maximumPoolSize 的时候，线程池就不会继续执行任务了而会去执行拒绝策略的逻辑**。

# 技术来源于生活[#](https://www.cnblogs.com/jajian/p/11442929.html#技术来源于生活)

人们常常在生活中遇到一些困难的时候会进行头脑风暴从而产生一些意想不到的解决方案，这些都是思想和智慧的结晶。我们很多技术的解决方案也都来源于生活。

我经常想如果以后不做程序员应该做什么？餐饮似乎是最大众的了，毕竟民以食为天。

开餐馆前期肯定不能做太大，一是本金的问题，还有就是需要市场试水。在市场需求不明确的情况下租个小店面还是靠谱的，就算亏也不会太多。

店面租个几十平的，就做香辣烤鱼，餐桌大概15桌的样子。然后就是员工了，除了厨师主要是服务员了，但是我不能招15个服务员啊，每桌分配一个太浪费了，需要提高资源利用率控制成本，所以员工不能招太多，我只需要招5个固定服务员负责在大厅招呼顾客和传菜就可以了，每个人负责3个餐桌。

[![img](..\img\multiplethread\threadpool\pool04.png)](https://img2018.cnblogs.com/blog/1162587/201911/1162587-20191122143152129-543326221.png)

但是我没想到我们餐馆做的烤鱼很合大众口味，很受欢迎又加上营销效果好，成了一家网红餐馆。生意更是蒸蒸日上，每天座无虚席。但是空间有限啊，所以我们只能让后来无座的顾客稍微等候了，于是我们安排了一个取号排队等候区，顾客等待叫号有序就餐。

[![img](..\img\multiplethread\threadpool\pool05.png)](https://img2018.cnblogs.com/blog/1162587/201911/1162587-20191122143332767-1314439611.png)

这时候餐馆的人员不变，仍然是5个服务员负责处理大厅的主要服务工作，同时排队等候区面积也不能过大，有个范围限制，不能影响我们的正常人员活动，同时也不能超过餐馆的范围排到餐馆外，如果顾客排队站到门外马路上了，这是就很危险的。随着口碑的发酵，一传十，十传百，我们的顾客络绎不绝，同时我们为了提高消费率又做起了外卖的服务，可以打包外带。

为了避免发生上述这种危险的情况和提高订单处理率，我们只能额外请一些临时工了，让他们来帮忙处理我们的外卖订单从而提高业务处理能力。

但是也不是请的越多越好，我们有成本控制，因为请的临时工我们也需要付工资。那怎么办呢？最终只能忍痛了啊，对于超出我们处理能力的订单，我们就采取一定的拒绝策略，例如告知顾客当天的份额已经售罄，请改天再来。

以上就是我们线程池运行的一个现实生活中的例子，核心线程就是我们的5个固定服务员，而排队等候区就是我们的等待队列，队列不能设为无限大，因为会造成OOM，如果队列满了线程池会另起额外线程去处理任务，也就是上述例子中的临时工，餐馆有经营成本控制所以有员工上限，不能请过多的临时工，这就是最大线程数。如果临时工达到最大数且队列也满了，那么我们只能通过拒绝策略暂时不接受额外的服务要求了。

# 一起看源码[#](https://www.cnblogs.com/jajian/p/11442929.html#一起看源码)

口说无凭，理论都是这样说的，那实际上源码是不是真是这样写的呢？我们一起来看下线程池的源码。通过 `threadPoolExecutor.execute(...)`的入口进入源码，删除了注释信息之后的源码内容如下，由于封装的好，所以只有短短几行。

```java
public void execute(Runnable command) {
    // #1 任务非空校验
    if (command == null)
        throw new NullPointerException();

    // #2 添加核心线程执行任务
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    // #3 任务入队列
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        //二次校验
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    
    // #4 添加普通线程执行任务，如果失败则执行拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}
```

如果不关注细节只关注整体，从以上源码中我们可以发现其中主要分为了四个步骤来处理逻辑。排除第一步的非空校验代码，我们可以看出剩下的三步其实就是我们线程池的运行逻辑，也就是上面的运行流程图的逻辑内容。

- (1) 任务的非空校验。
- (2) 获取当前RUNNING的线程数，如果小于核心线程数，则创建核心线程去执行任务，否则走#3。
- (3) 如果当前线程池处于RUNNING状态，那么就将任务放入队列中。这时还会再做个双重校验，因为可能存在有些线程在我们上次检查后死了，或者从我们进入这个方法后pool被关闭了，所以我们需要再次检查state。如果线程池停止了就需要回滚刚才的添加任务到队列中的操作并通过拒绝策略拒绝该任务，或者如果池中没有线程了，则新开启一个线程执行任务。
- (4) 如果队列满了之后无法在将任务加入队列，则创建新的线程去执行任务，如果也失败了，那么就可能是线程池关闭了或者线程池饱和了，这时执行拒绝策略不再接受任务。

> 双重校验中有以下两个点需要注意：
>
> **1. 为什么需要 double check 线程池的状态？**
> 在多线程环境下，线程池的状态时刻在变化，而 ctl.get() 是非原子操作，很有可能刚获取了线程池状态后线程池状态就改变了。判断是否将 command 加入 workque 是线程池之前的状态。倘若没有 double check，万一线程池处于非 running 状态（在多线程环境下很有可能发生），那么 command 永远不会执行。
>
> **2、为什么 addWorker(null, false) 的任务为null？**
> addWorker(null, false)，这个方法执行时只是创建了一个新的线程，但是没有传入任务，这是因为前面已经将任务添加到队列中了，这样可以防止线程池处于 running 状态，但是没有线程去处理这个任务。

而根据以上代码的具体步骤我们可以画出详细的执行流程，如下图所示

[![img](..\img\multiplethread\threadpool\pool06.png)](https://img2018.cnblogs.com/blog/1162587/201909/1162587-20190909195655324-553758508.png)

以上的源码其实只有10几行，看起来很简单，主要是它的封装性比较好，其中主要有两个点需要重点解释，分别是：**线程池的状态**和 `addWorker()`添加工作的方法，这两个点弄明白了这段线程池的源码差不多也就理解了。

## 线程池运行状态-runState[#](https://www.cnblogs.com/jajian/p/11442929.html#线程池运行状态-runstate)

线程有状态，线程池也有它的运行状态，这些状态提供了主生命周期控制，伴随着线程池的运行，由内部来维护，从源码中我们可以发现线程池共有5个状态：`RUNNING`，`SHUTDOWN`，`STOP`，`TIDYING`，`TERMINATED`。

各状态值所代表的的含义和该状态值下可执行的操作，具体信息如下：

|    运行状态    |                           状态描述                           |
| :------------: | :----------------------------------------------------------: |
|  **RUNNING**   |          接收新任务，并且也能处理阻塞队列中的任务。          |
|  **SHUTDOWN**  |      不接收新任务，但是却可以继续处理阻塞队列中的任务。      |
|    **STOP**    | 不接收新任务，同时也不处理队列任务，并且中断正在进行的任务。 |
|  **TIDYING**   | 所有任务都已终止，workercount(有效线程数)为0，线程转向 TIDYING 状态将会运行 terminated() 钩子方法。 |
| **TERMINATED** |           terminated() 方法调用完成后变成此状态。            |

生命周期状态流转如下图所示：

[![img](..\img\multiplethread\threadpool\pool07.png)](https://img2020.cnblogs.com/blog/1162587/202009/1162587-20200904162641889-2044248271.png)

很多时候我们表示状态都是通过简单的 int 值来表示，例如数据库数据的删除标志 delete_flag 其中0表示有效，1表示删除。而在线程池的源码里我们可以看到它是通过如下方式来进行表示的，

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```

线程池内部使用一个变量维护两个值：运行状态（runState）和线程数量 （workerCount）何做到的呢？将十进制 int 值转换为二进制的值，共32位，其中高3位代表运行状态（runState ），而低29位代表工作线程数（workerCount）。

[![img](..\img\multiplethread\threadpool\pool08.png)](https://img2020.cnblogs.com/blog/1162587/202009/1162587-20200904162656157-429188458.png)

关于内部封装的获取生命周期状态、获取线程池线程数量的计算方法如以下代码所示：

```java
//获取线程池状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
//获取线程数量
private static int workerCountOf(int c)  { return c & CAPACITY; }
// Packing and unpacking ctl
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

通过巧妙的位运算可以分别获取**高3位的运行状态值**和**低29位的线程数量值**，如果感兴趣的可以去看下具体的实现代码，这里就不再赘述了。

## 添加工作线程-addWorker[#](https://www.cnblogs.com/jajian/p/11442929.html#添加工作线程-addworker)

添加线程是通过 addWorker() 方法来实现的，这个方法有两个入参，`Runnable firstTask` 和 `boolean core`。

```java
private boolean addWorker(Runnable firstTask, boolean core){...}
```

- Runnable firstTask 即是当前添加的线程需要执行的首个任务.
- boolean core 用来标记当前执行的线程是否是核心线程还是普通线程.

返回前面的线程池的 execute() 方法的代码中，可以发现这个addWorker() 有三个地方在调用，分别在 #2，#3和#4。

- \#2：当工作线程数 < 核心线程数的时候，通过`addWorker(command, true)`添加核心线程执行command任务。
- \#3：double check的时候，如果发现线程池处于正常运行状态但是里面没有工作线程，则添加个空任务和一个普通线程，这样一个 task 为空的 worker 在线程执行的时候会去阻塞任务队列里拿任务，这样就相当于创建了一个新的线程，只是没有马上分配任务。
- \#4：队列已满的情况下，通过添加普通线程（非核心线程）去执行当前任务，如果失败了则执行拒绝策略。

addWorker() 方法调用的地方我们看完了，接下来我们一起来看下它里面究竟做了些什么，源码如下：

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

这个方法稍微有点长，我们分段来看下，将上面的代码我们拆分成两个部分来看，首先看第一部分：

```java
retry:
for (;;) {
    int c = ctl.get();
    int rs = runStateOf(c);//获取线程池的状态

    // Check if queue empty only if necessary.
    if (rs >= SHUTDOWN &&
        ! (rs == SHUTDOWN &&
           firstTask == null &&
           ! workQueue.isEmpty()))
        return false;

    for (;;) {
        int wc = workerCountOf(c);
        if (wc >= CAPACITY ||
            wc >= (core ? corePoolSize : maximumPoolSize))
            return false;
	    // 尝试通过CAS方式增加workerCount
        if (compareAndIncrementWorkerCount(c))
            break retry;
        c = ctl.get();  // Re-read ctl
        // 如果线程池状态发生变化，重新从最外层循环
        if (runStateOf(c) != rs)
            continue retry;
        // else CAS failed due to workerCount change; retry inner loop
    }
}
```

这部分代码有两层嵌套的 for 死循环，在第一行有个`retry:`代码，这个也许有些同学没怎么见过，这个是相当于是一个位置标记，retry后面跟循环，标记这个循环的位置。

我们平时写 for 循环的时候，是通过`continue;`或`break;`来跳出当前循环，但是如果我们有多重嵌套的 for 循环，如果我们想在里层的某个循环体中当达到某个条件的时候直接跳出所有循环或跳出到某个指定的位置，则使用`retry:`来标记这个位置就可以了。

代码中共有4个位置有改变循环体继续执行下去，分别是两个`return false;`，一个`break retry;`和一个`continue retry;`。

**首先我们来看下第一个`return false;`**，这个 return 在最外层的一个 for 循环，

```java
if (rs >= SHUTDOWN && !(rs == SHUTDOWN && firstTask == null && ! workQueue.isEmpty()))
   return false;
```

这是一个判断线程池状态和线程队列情况的代码，这个逻辑判断有点绕可以改成

```lisp
rs >= shutdown && (rs != shutdown || firstTask != null || workQueue.isEmpty())
```

这样就好理解了，逻辑判断成立可以分为以下几种情况直接返回 false，表示添加工作线程失败。

- rs > shutdown：线程池状态处于 `STOP`，`TIDYING`，`TERMINATED`时，添加工作线程失败，不接受新任务。
- rs >= shutdown && firstTask != null：线程池状态处于 `SHUTDOWN`，`STOP`，`TIDYING`，`TERMINATED`状态且worker的首个任务不为空时，添加工作线程失败，不接受新任务。
- rs >= shutdown && workQueue.isEmppty：线程池状态处于 `SHUTDOWN`，`STOP`，`TIDYING`，`TERMINATED`状态且阻塞队列为空时，添加工作线程失败，不接受新任务。

这样看来，最外层的 for 循环是不断的校验当前的线程池状态是否能接受新任务，如果校验通过了之后才能继续往下运行。

**然后接下来看第二个`return false;`**，这个 return 是在内层的第二个 for 循环中，是判断线程池中当前的工作线程数量的，不满足条件的话直接返回 false，表示添加工作线程失败。

- 工作线程数量是否超过可表示的最大容量（CAPACITY）.
- 如果添加核心工作线程，是否超过最大核心线程容量（corePoolSize）.
- 如果添加普通工作线程，是否超过线程池最大线程容量（maximumPoolSize）.

**后面的`break retry;`** ，表示如果尝试通过CAS方式增加工作线程数workerCount成功，则跳出这个双循环，往下执行后面第二部分的代码，而`continue retry;`是再次校验下线程池状态是否发生变化，如果发生了变化则重新从最外层 for 开始继续循环执行。

通过第一部分代码的解析，我们发现只有`break retry;`的时候才能执行到后面第二部分的代码，而后面第二部分代码做了些什么呢？

```java
boolean workerStarted = false;
boolean workerAdded = false;
Worker w = null;
try {
    //创建Worker对象实例
    w = new Worker(firstTask);
    //获取Worker对象里的线程
    final Thread t = w.thread;
    if (t != null) {
        //开启可重入锁，独占
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // Recheck while holding lock.
            // Back out on ThreadFactory failure or if
            // shut down before lock acquired.
            //获取线程池运行状态
            int rs = runStateOf(ctl.get());

            //满足 rs < SHUTDOWN 判断线程池是否是RUNNING，或者
            //rs == SHUTDOWN && firstTask == null 线程池如果是SHUTDOWN，
            //且首个任务firstTask为空，
            if (rs < SHUTDOWN ||
                (rs == SHUTDOWN && firstTask == null)) {
                if (t.isAlive()) // precheck that t is startable
                    throw new IllegalThreadStateException();
                //将Worker实例加入线程池workers
                workers.add(w);
                int s = workers.size();
                if (s > largestPoolSize)
                    largestPoolSize = s;
                //线程添加成功标志位 -> true
                workerAdded = true;
            }
        } finally {
            //释放锁
            mainLock.unlock();
        }
        //如果worker实例加入线程池成功，则启动线程，同时修改线程启动成功标志位 -> true
        if (workerAdded) {
            t.start();
            workerStarted = true;
        }
    }
} finally {
    if (! workerStarted)
        //添加线程失败
        addWorkerFailed(w);
}
return workerStarted;
```

这部分代码主要的目的其实就是启动一个线程，前面是一堆的条件判断，看是否能够启动一个工作线程。它由两个`try...catch...finally`内容组成，可以将他们拆开来看，这样就很容易看懂。

我们先看里面一层的`try...catch...finally`，当Worker实例中的 Thread 线程不为空的时候，开启一个独占锁`ReentrantLock mainLock`，防止其他线程也来修改操作。

```java
try {
   //获取线程池运行状态
   int rs = runStateOf(ctl.get());

   if (rs < SHUTDOWN ||
       (rs == SHUTDOWN && firstTask == null)) {
       if (t.isAlive()) // precheck that t is startable
           throw new IllegalThreadStateException();
       workers.add(w);
       int s = workers.size();
       if (s > largestPoolSize)
           largestPoolSize = s;
       workerAdded = true;
   }
} finally {
   mainLock.unlock();
}
```

- 首先检查线程池的状态，当线程池处于 RUNNING 状态或者线程池处于 SHUTDOWN 状态但是当前线程的 firstTask 为空，满足以上条件时才能将 worker 实例添加进线程池，即`workers.add(w);`。
- 同时修改 largestPoolSize，largestPoolSize变量用于记录出现过的最大线程数。
- 将标志位 workerAdded 设置为 true，表示添加工作线程成功。
- 无论成功与否，在 finally 中都必须执行 `mainLock.unlock()`来释放锁。

外面一层的`try...catch...finally`主要是为了判断工作线程是否启动成功，如果内层`try...catch...finally`代码执行成功，即 worker 添加进线程池成功，workerAdded 标志位置为true，则启动 worker 中的线程 `t.start()`，同时将标志位 workerStarted 置为 true，表示线程启动成功。

```java
if (workerAdded) {
    t.start();
    workerStarted = true;
}
```

如果失败了，即 workerStarted == false，则在 finally 里面必须执行`addWorkerFailed(w)`方法，这个方法相当于是用来回滚操作的，前面增的这里移除，前面加的这里减去。

```java
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            //从线程池中移除worker实例
            workers.remove(w);
        //通过CAS，将工作线程数量workerCount减1
        decrementWorkerCount();
        //
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```

## Worker类[#](https://www.cnblogs.com/jajian/p/11442929.html#worker类)

上面我们分析了addWorker 方法的源码，并且看到了 `Thread t = w.thread`，`workers.add(w)`和`t.start()`等代码，知道了线程池的运行状态和添加工作线程的流程，那么我们还有一些疑问：

- 这里的 Worker 是什么？和 Thread 有什么区别？
- 线程启动后是如何拿任务？在哪拿任务去执行的？
- 阻塞队列满后，额外新创建的线程是去队列里拿任务的吗？如果不是那它是去哪拿的？
- 核心线程会一直存在于线程池中吗？额外创建的普通线程执行完任务后会销毁吗？

Worker 是 `ThreadPoolExecutor`的一个内部类，主要是用来维护线程执行任务的中断控制状态，它实现了Runnable 接口同时继承了AQS，实现 Runnable 接口意味着 Worker 就是一个线程，继承 AQS 是为了实现独占锁这个功能。

```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;
        
        //构造函数，初始化AQS的state值为-1
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
}
```

至于为什么没有使用可重入锁 ReentrantLock，而是使用AQS，为的就是实现不可重入的特性去反应线程现在的执行状态。

1. lock方法一旦获取了独占锁，表示当前线程正在执行任务中。
2. 如果正在执行任务，则不应该中断线程。
3. 如果该线程现在不是独占锁的状态，也就是空闲的状态，说明它没有在处理任务，这时可以对该线程进行中断。
4. 线程池在执行 shutdown 方法或 tryTerminate 方法时会调用 interruptIdleWorkers 方法来中断空闲的线程，interruptIdleWorkers 方法会使用 tryLock 方法来判断线程池中的线程是否是空闲状态；如果线程是空闲状态则可以安全回收。

Worker 类有一个构造方法，构造参数为给定的首个任务 firstTask，并持有一个线程thread。thread是在调用构造方法时通过 ThreadFactory 来创建的线程，可以用来执行任务；

**firstTask用它来初始化时传入的第一个任务，这个任务可以有也可以为null。如果这个值是非空的，那么线程就会在启动初期立即执行这个任务；如果这个值是null，那么就需要创建一个线程去执行任务列表（workQueue）中的任务，也就是非核心线程的创建。**

# 任务运行-runWorker[#](https://www.cnblogs.com/jajian/p/11442929.html#任务运行-runworker)

上面我们一起看过线程的启动`t.start()`，具体运行是在 Worker 的 run() 方法中

```java
public void run() {
    runWorker(this);
}
```

run() 方法中又调用了runWorker() 方法，所有的实现都在这里

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

很多人看到这样的代码就感觉头痛，其实你细看，这里面我们可以看关键点，里面有三块`try...catch...finally`代码，我们将这三块分别单独拎出来看并且将抛异常的地方暂时删掉或注释掉，这样它看起来就清爽了很多

```java
Thread wt = Thread.currentThread();
Runnable task = w.firstTask;
w.firstTask = null;
//由于Worker初始化时AQS中state设置为-1，这里要先做一次解锁把state更新为0，允许线程中断
w.unlock(); // allow interrupts
boolean completedAbruptly = true;
try {
    // 循环的判断任务（firstTask或从队列中获取的task）是否为空
    while (task != null || (task = getTask()) != null) {
        // Worker加锁，本质是AQS获取资源并且尝试CAS更新state由0更变为1
        w.lock();
        // 如果线程池运行状态是stopping, 确保线程是中断状态;
        // 如果不是stopping, 确保线程是非中断状态. 
        if ((runStateAtLeast(ctl.get(), STOP) ||
             (Thread.interrupted() &&
              runStateAtLeast(ctl.get(), STOP))) &&
            !wt.isInterrupted())
            wt.interrupt();
            
            //此处省略了第二个try...catch...finally
    }
    // 走到这里说明某一次getTask()返回为null，线程正常退出
    completedAbruptly = false;
} finally {
    //处理线程退出
    processWorkerExit(w, completedAbruptly);
}
```

第二个`try...catch...finally`

```java
try {
   beforeExecute(wt, task);
   Throwable thrown = null;
    
    //此处省略了第三个try...catch...finally
    
} finally {
    task = null;
    w.completedTasks++;
    w.unlock();
}
```

第三个`try...catch...finally`

```java
try {
    // 运行任务
    task.run();
} catch (RuntimeException x) {
    thrown = x; throw x;
} catch (Error x) {
    thrown = x; throw x;
} catch (Throwable x) {
    thrown = x; throw new Error(x);
} finally {
    afterExecute(task, thrown);
}
```

上面的代码中可以看到有`beforeExecute`、`afterExecute`和`terminaerd`三个函数，它们都是钩子函数，可以分别在子类中重写它们用来扩展ThreadPoolExecutor，例如添加日志、计时、监视或者统计信息收集的功能。

- beforeExecute()：线程执行之前调用
- afterExecute()：线程执行之后调用
- terminaerd()：线程池退出时候调用

这样拆分完之后发现，其实主要注意两个点就行了，分别是`getTask()`和`task.run()`，`task.run()`就是运行任务，那我们继续来看下`getTask()`是如何获取任务的。

# 获取任务-getTask[#](https://www.cnblogs.com/jajian/p/11442929.html#获取任务-gettask)

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        //1.线程池状态是STOP，TIDYING，TERMINATED
        //2.线程池shutdown并且队列是空的.
        //满足以上两个条件之一则工作线程数wc减去1，然后直接返回null
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        //允许核心工作线程对象销毁淘汰或者工作线程数 > 最大核心线程数corePoolSize
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        //1.工作线程数 > 最大线程数maximumPoolSize 或者timed == true && timedOut == true
        //2.工作线程数 > 1 或者队列为空 
        //同时满足以上两个条件则通过CAS把线程数减去1，同时返回null。CAS把线程数减去1失败会进入下一轮循环做重试
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            /// 如果timed为true，通过poll()方法做超时拉取，keepAliveTime时间内没有等待到有效的任务，则返回null
            // 如果timed为false，通过take()做阻塞拉取，会阻塞到有下一个有效的任务时候再返回（一般不会是null）
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

里面有个关键字`allowCoreThreadTimeOut`，它的默认值为false，在Java1.6开始你可以通过`threadPoolExecutor.allowCoreThreadTimeOut(true)`方式来设置为true，通过字面意思就可以明白这个字段的作用是什么了，即是否允许核心线程超时销毁。

**默认的情况下核心线程数量会一直保持，即使这些线程是空闲的它也是会一直存在的，而当设置为 true 时，线程池中 corePoolSize 线程空闲时间达到 keepAliveTime 也将销毁关闭。**

