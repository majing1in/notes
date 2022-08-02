[TOC]

## 一、线程池是什么，有什么优势

1. 在Java中线程都是映射操作系统的原生线程，在进行线程的创建、销毁、阻塞、唤醒等操作时，都会牵扯到用户态与内核态的切换，如果频繁进行这些操作会大量消耗系统资源。而线程池是一个线程集合，线程池中线程执行完任务后，会根据当前线程池状态判断是否进行销毁，或者进行等待执行下一个任务，从而达到线程的重复利用。
2. 在业务量大的情况下，使用线程池进行多线程处理任务，能加快数据的处理，但是在处理的时候会给服务器带来相应的压力，所以要合理配置线程池参数。

## 二、线程池参数

1. corePoolSize：核心线程数，在线程池中所有的线程都是消费者。
2. maximumPoolSize：最大线程数，其中已经包括了核心线程数。
3. keepAliveTime：非核心线程可存活最大时间。
4. unit：时间单位。
5. workQueue：等待队列。
6. threadFactory：线程工厂。
7. handler：拒绝策略。
   1. AbortPolicy：直接抛出异常。
   2. CallerRunsPolicy：在任务被拒绝添加后，会调用当前线程池的所在的线程去执行被拒绝的任务。
   3. DiscardOldestPolicy：它会抛弃任务队列中最旧的任务也就是最先加入队列的，再把这个新任务添加进去。
   4. DiscardPolicy：被拒绝任务的处理程序，它默默地丢弃被拒绝的任务。

## 三、线程池中的位运算

参考文档：

> [JDK ThreadPoolExecutor核心原理与实践 - 掘金 (juejin.cn)](https://juejin.cn/post/7044787917885014052#heading-8)

ThreadPoolExecutor 中有两个非常重要的参数：

> 线程池状态（rs）以及活跃线程数（wc），前者用于标识当前线程池的状态，后者用于标识活跃线程数。

ThreadPoolExecutor 用一个 Integer 变量（ctl）来设置这两个参数。

Java中的 Integer 变量都是32位，ThreadPoolExecutor 使用前3位（31~29）表示线程池状态，用后29位（28~0）表示活跃线程数。

![tpe](D:\notes\Java\资源\tpe.jpg)

ThreadPoolExecutor 关于状态初始化的源码如下：

```java
// ctl 在一个int中包装了活跃线程数和线程池运行时状态两个变量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// 概念上用于表示状态位与线程数位的分界值，实际用于状态变量等移位操作，此处为 Integer.sixze-3=32-3=29
private static final int COUNT_BITS = Integer.SIZE - 3;
// 表示 ThreadPoolExecutor 的最大容量
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
```

CAPACITY转化方式：

![tpe2](D:\notes\Java\资源\tpe2.jpg)

线程池状态初始化：

```java
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```

这5个状态的计算过程如下图所示，经过移位计算后，数值的后29位全为0，前3位分别代表不同的状态。

![tpe3](D:\notes\Java\资源\tpe3.jpg)

### 3.1 runStateOf(c)方法

```java
private static int runStateOf(int c) {
    return c & ~CAPACITY;
}
```

runStateOf() 方法是用于获取线程池状态的方法，其中形参 c 一般是 ctl 变量，包含了状态和线程数，runStateOf()移位计算的过程如下图所示。

![tpe4](D:\notes\Java\资源\tpe4.jpg)

CAPACITY 取反后高三位置1，低29位置0。取反后的值与 ctl 进行 ‘与’ 操作。由于任何值 ‘与’ 1等于原值，‘与’ 0等于0。因此 ‘与’ 操作过后，ctl 的高3位保留原值，低29位置0，这样就将状态值从 ctl 中分离出来。

### 3.2 workerCountOf(c)方法

```java
private static int workerCountOf(int c) {
    return c & CAPACITY;
}
```

workerCountOf(c) 方法的分析思路与上述类似，就是把后29位从ctl中分离出来，获得活跃线程数，如下图所示。

![tpe5](D:\notes\Java\资源\tpe5.jpg)

### 3.3 ctlOf(rs, wc)方法

```java
private static int ctlOf(int rs, int wc) {
    return rs | wc;
}
```

ctlOf(rs, wc)通过状态值和线程数值计算出 ctl 值。rs是runState的缩写，wc是workerCount的缩写。rs的后29位为0，wc的前三位为0，两者通过 ‘或’ 操作计算出来的最终值同时保留了rs的前3位和wc的后29位，即 ctl 值。

![tpe6](D:\notes\Java\资源\tpe6.jpg)



## 四、线程池的五种状态

![image](http://images.cnitblog.com/blog/497634/201401/08000847-0a9caed4d6914485b2f56048c668251a.jpg)

1、RUNNING

状态说明：接受新任务并处理排队的任务。

状态切换：线程池的初始化状态是RUNNING。换句话说，线程池被一旦被创建，就处于RUNNING状态，并且线程池中的任务数为0。

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

2、 SHUTDOWN

状态说明：不接受新任务，但处理排队的任务。 

状态切换：调用线程池的shutdown()接口时，线程池由RUNNING -> SHUTDOWN。

3、STOP

状态说明：不接受新任务，不处理排队的任务，中断正在进行的任务。 

状态切换：调用线程池的shutdownNow()接口时，线程池由(RUNNING or SHUTDOWN ) -> STOP。

4、TIDYING

状态说明：所有任务都已终止，workerCount 为零，转换到状态 TIDYING 的线程将运行 terminate() 钩子方法。

状态切换：当线程池在SHUTDOWN状态下，阻塞队列为空并且线程池中执行的任务也为空时，就会由 SHUTDOWN -> TIDYING。 当线程池在STOP状态下，线程池中执行的任务为空时，就会由STOP -> TIDYING。

5、 TERMINATED

状态说明： terminated()方法执行完毕后。

状态切换：线程池处在TIDYING状态时，执行完terminated()之后，就会由 TIDYING -> TERMINATED。

## 五、线程池执行流程分析

#### 5.1 线程池添加流程

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    // ctl包含当前线程池的线程数和线程池状态
    int c = ctl.get();
    // 判断是否达到核心线程
    if (workerCountOf(c) < corePoolSize) {
        // 添加线程true
        if (addWorker(command, true))
            return;
        // 更新状态ctl
        c = ctl.get();
    }
    // 先判断线程池状态是否存活，并添加到队列中
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 当添加完成任务后，线程池状态变为了非运行状态，移除任务并执行拒绝策略
        if (!isRunning(recheck) && remove(command))
            reject(command);
        // 注释翻译：
        // 检测从 SHUTDOWN 到 TIDYING 的转换并不像您想要的那么简单，因为队列可能在非空后变为空
        // 在 SHUTDOWN 状态下反之亦然，但我们只能在看到它为空之后终止，我们看到 workerCount为 0
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 添加线程false
    else if (!addWorker(command, false))
        // 超过最大线程数或线程池状态为非运行状态
        reject(command);
}
```



















