[TOC]

## 一、ObjectMonitor类

### 1.1、基本成员

在HotSpot虚拟机中，Monitor是基于C++的ObjectMonitor类实现的，其主要成员包括：

- _owner：指向持有ObjectMonitor对象的线程；
- _waitSet：存放处于wait状态的线程队列，即调用wait方法的线程；
- _entryList：处于存放等待锁block状态的线程队列；
- _count：约为waitSet和entryList的节点数之和；
- _cxq：多个线程争抢锁，会先存入这个单向链表；
- _recursions：记录重入次数。

### 1.2、基本工作机制

![1](D:\notes\Java\资源\1.jpg)

1. 当多个线程同时访问一段同步代码块时，首先会进入到_entryList队列中；
2. 当某个线程获取到对象的Monitor后进入临界区域，并把Monitor中的_owner变量设置为当前线程，同时Monitor中的计数器\_count加1，即获得对象锁；
3. 若持有Monitor的线程调用wait方法，将释放当前持有的Monitor，_owner变量恢复为null，计数器\_count自减1，同时该线程进入\_waitSet集合中等待被唤醒；
4. 在_waitSet集合中的线程会被再次放到\_entryList队列中，重新竞争获取锁；
5. 若当前线程执行完毕后也将释放Monitor并复位变量的值，以便其它线程进入获取锁。

![2](D:\notes\Java\资源\2.jpg)

ObjectMonitor::enter() 和 ObjectMonitor::exit() 分别是ObjectMonitor获取锁和释放锁的方法。

线程解锁后还会唤醒之前等待的线程，根据策略选择直接唤醒\_cxq队列中的头部线程去竞争，或者将_cxq队列中的线程加入\_EntryList，然后再唤醒\_EntryList队列中的线程去竞争。

#### 1.2.1、ObjectMonitor::enter()方法流程

![3](D:\notes\Java\资源\3.jpg)

1. 首先尝试通过 CAS 把 ObjectMonitor 中的 _owner 设置为当前线程，设置成功就表示获取锁成功，通过 _recursions 的自增来表示重入；
2. 如果没有CAS成功，那么就开始启动自适应自旋，自旋还不行的话，就包装成 ObjectWaiter 对象加入到 _cxq 单向链表之中；
3. 加入_cxq链表后，再次尝试是否可以CAS拿到锁，再次失败就要阻塞(block)，底层调用了pthread_mutex_lock。

#### 1.2.2、ObjectMonitor::exit()方法流程

![4](D:\notes\Java\资源\4.jpg)

1. 线程执行Object.wait()方法时，会将当前线程加入到 _waitSet 这个双向链表中，然后再运行ObjectMonitor::exit() 方法来释放锁；
2. 可重入锁就是根据 _recursions 来判断的，重入一次就执行 _recursions++，解锁一次就执行 _recursions--，如果 _recursions 减到 0 ，就说明需要释放锁了；
3. 线程解锁后还会唤醒之前等待的线程，当线程执行 Object.notify() 方法时，从 \_waitSet 头部拿线程节点，然后根据策略（QMode指定）决定将线程节点放在哪里，包括 _cxq 或 _EntryList 的头部或者尾部，然后唤醒队列中的线程。

## 二、Synchronized关键字

从 JVM 规范中可以看到 Synchonized 在 JVM 里的实现原理，JVM 基于进入和退出 Monitor 对象来实现方法同步和代码块同步，但两者的实现细节不一样。代码块同步是使用 monitorenter 和 monitorexit 指令实现的，而方法同步是使用另外一种方式实现的，细节在 JVM 规范里并没有详细说明。但是，方法的同步同样可以使用这两个指令来实现。

monitorenter 指令是在编译后插入到同步代码块的开始位置，而 monitorexit 是插入到方法结束处和异常处，JVM 要保证每个 monitorenter 必须有对应的 monitorexit 与之配对。任何对象都有一个 monitor 与之关联，当且一个 monitor 被持有后，它将处于锁定状态。线程执行到 monitorenter 指令时，将会尝试获取对象所对应的 monitor 的所有权，即尝试获得对象的锁。

### 2.1、Java对象头

Synchronized 用的锁是存在 Java 对象头里的。

如果对象是数组类型，则虚拟机用 3 个字宽（Word）存储对象头，如果对象是非数组类型，则用 2 字宽存储对象头。

在 32 位虚拟机中，1 字宽等于 4 字节，即 32bit，如图所示：

![10](D:\notes\Java\资源\10.png)

Java 对象头里的 Mark Word 里默认存储对象的 HashCode、分代年龄和锁标记位。

32 位 JVM 的 Mark Word 的默认存储结构如图所示：

![11](D:\notes\Java\资源\11.png)

Mark Word 可能变化为存储以下 4 种数据，如表图所示：

![12](D:\notes\Java\资源\12.png)

在 64 位虚拟机下，Mark Word 是 64bit 大小的，其存储结构图所示：

![13](D:\notes\Java\资源\13.png)

### 2.2、锁的升级

Java SE 1.6 为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级 锁”，在 Java SE 1.6 中，锁一共有 4 种状态，级别从低到高依次是：无锁状态、偏向锁状 态、轻量级锁状态和重量级锁状态，这几个状态会随着竞争情况逐渐升级。

锁可以升级但不能降级，意味着偏向锁升级成轻量级锁后不能降级成偏向锁，这种锁升级却不能降级的策略，目的是为了提高获得锁和释放锁的效率。

#### 2.2.1、偏向锁

大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。

当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程 ID，以后该线程在进入和退出同步块时不需要进行 CAS 操作来加锁和解锁，只需简单地测试一下对象头的 Mark Word 里是否存储着指向当前线程的偏向锁。如果测试成功，表示线程已经获 得了锁；如果测试失败，则需要再测试一下 Mark Word 中偏向锁的标识是否设置成 1 （表示当前是偏向锁）：如果没有设置，则使用 CAS 竞争锁；如果设置了，则尝试使用 CAS 将对象头的偏向锁指向当前线程。

**偏向锁的撤销**

偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁 时，持有偏向锁的线程才会释放锁。

偏向锁的撤销，需要等待全局安全点（在这个时间 点上没有正在执行的字节码）。它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果线程不处于活动状态，则将对象头设置成无锁状态；如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的 Mark Word 要么重新偏向于其他线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。

**关闭偏向锁**

偏向锁在 Java 6 和 Java 7 里是默认启用的，但是它在应用程序启动几秒钟之后才激活。

如有必要可以使用 JVM 参数来关闭延迟：-XX:BiasedLockingStartupDelay=0。

如果你确定应用程序里所有的锁通常情况下处于竞争状态，可以通过 JVM 参数关闭偏向锁： -XX:- UseBiasedLocking=false，那么程序默认会进入轻量级锁状态。

#### 2.2.2、轻量级锁

![14](D:\notes\Java\资源\14.png)

**轻量级锁加锁**

线程在执行同步块之前，JVM 会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的 Mark Word 复制到锁记录中，官方称为 Displaced Mark Word。然后线程尝试使用 CAS 将对象头中的 Mark Word 替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。

**轻量级锁解锁**

轻量级解锁时，会使用原子的 CAS 操作将 Displaced Mark Word 替换回到对象头， 如果成功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。

因为自旋会消耗 CPU，为了避免无用的自旋（比如获得锁的线程被阻塞住了），一 旦锁升级成重量级锁，就不会再恢复到轻量级锁状态。当锁处于这个状态下，其他线程试图获取锁时，都会被阻塞住，当持有锁的线程释放锁之后会唤醒这些线程，被唤醒的线程就会进行新一轮的夺锁之争。

**锁的优缺点对比**

![15](D:\notes\Java\资源\15.png)

### 2.3、实现原理

```java
public class SynchronizedDemo {
	//同步方法
    public synchronized void doSth(){
        System.out.println("Hello World");
    }

	//同步代码块
    public void doSth1(){
        synchronized (SynchronizedDemo.class){
            System.out.println("Hello World");
        }
    }
}
```

```java
public synchronized void doSth();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String Hello World
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
  public void doSth1();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: ldc           #5                  // class com/hollis/SynchronizedTest
         2: dup
         3: astore_1
         4: monitorenter
         5: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         8: ldc           #3                  // String Hello World
        10: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        13: aload_1
        14: monitorexit
        15: goto          23
        18: astore_2
        19: aload_1
        20: monitorexit
        21: aload_2
        22: athrow
        23: return
```

对于同步方法，JVM采用ACC_SYNCHRONIZED标记符来实现同步。 对于同步代码块，JVM采用monitorenter、monitorexit两个指令来实现同步。

方法级的同步是隐式的，同步方法的常量池中会有一个ACC_SYNCHRONIZED标志。当某个线程要访问某个方法的时候，会检查是否有ACC_SYNCHRONIZED，如果有设置，则需要先获得监视器锁，然后开始执行方法，方法执行之后再释放监视器锁，这时如果其他线程来请求执行方法，会因为无法获得监视器锁而被阻断住。值得注意的是，如果在方法执行过程中，发生了异常，并且方法内部并没有处理该异常，那么在异常被抛到方法外面之前监视器锁会被自动释放。

同步代码块使用monitorenter和monitorexit两个指令实现。可以把执行monitorenter指令理解为加锁，执行monitorexit理解为释放锁。 每个对象维护着一个记录着被锁次数的计数器。未被锁定的对象的该计数器为0，当一个线程获得锁（执行monitorenter）后，该计数器自增变为 1 ，当同一个线程再次获得该对象的锁的时候，计数器再次自增。当同一个线程释放锁（执行monitorexit指令）的时候，计数器再自减。当计数器为0的时候，锁将被释放，其他线程便可以获得锁。

无论是ACC_SYNCHRONIZED还是monitorenter、monitorexit都是基于Monitor实现的，在Java虚拟机(HotSpot)中，Monitor是基于C++实现的，由ObjectMonitor实现。

ObjectMonitor类中提供了几个方法，如enter、exit、wait、notify、notifyAll等。sychronized加锁的时候，会调用objectMonitor的enter方法，解锁的时候会调用exit方法。sychronized加锁的时候，会调用objectMonitor的enter方法，解锁的时候会调用exit方法。

## 三、原子操作的实现原理

原子（atomic）本意是“不能被进一步分割的最小粒子”，而原子操作（atomic  operation）意为“不可被中断的一个或一系列操作”。在多处理器上实现原子操作就变得有点复杂。

### 3.1、术语定义

![16](D:\notes\Java\资源\16.png)

### 3.2、处理器如何实现原子操作

32 位 IA-32 处理器使用基于对缓存加锁或总线加锁的方式来实现多处理器之间的原子操作。

首先处理器会自动保证基本的内存操作的原子性，处理器保证从系统内存中读取或者写入一个字节是原子的，意思是当一个处理器读取一个字节时，其他处理器不能访问这个字节的内存地址。

> D:\notes\操作系统\文档\缓存一致性协议.md

#### 3.2.1、使用总线锁保证原子性

第一个机制是通过总线锁保证原子性。

如果多个处理器同时对共享变量进行读改写操作（i++就是经典的读改写操作），那么共享变量就会被多个处理器同时进行操作，这样读改写操作就不是原子的，操作完之后共享变量的值会和期望的不一致。

原因可能是多个处理器同时从各自的缓存中读取变量 i，分别进行加 1 操作，然后分别写入系统内存中。那么，想要保证读改写共享变量的操作是原子的，就必须保证 CPU1 读改写共享变量的时候，CPU2 不能操作缓存了该共享变量内存地址的缓存。处理器使用总线锁就是来解决这个问题的，所谓总线锁就是使用处理器提供的一个 LOCK＃ 信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住，那么该处理器可以独占共享内存。

#### 3.2.2、使用缓存锁保证原子性

第二个机制是通过缓存锁定来保证原子性。

在同一时刻，我们只需保证对某个内存地址的操作是原子性即可，但总线锁定把 CPU 和内存之间的通信锁住了，这使得锁定期间，其他处理器不能操作其他内存地址的数据，所以总线锁定的开销比较大，目前处理器在某些场合下使用缓存锁定代替总线锁定来进行优化。

频繁使用的内存会缓存在处理器的 L1、L2 和 L3 高速缓存里，那么原子操作就可以直接在处理器内部缓存中进行，并不需要声明总线锁，在 Pentium 6 和目前的处理器中可以使用“缓存锁定”的方式来实现复杂的原子性。

所谓“缓存锁定”是指内存区域如果被缓存在处理器的缓存行中，并且在 Lock 操作期间被锁定，那么当它执行锁操作回写到内存时，处理器不在总线上声言 LOCK＃信号，而是修改内部的内存地址，并允许它的缓存一致性机制来保证操作的原子性，因为缓存一致性机制会阻止同时修改由两个以上处理器缓存的内存区域数据，当其他处理器回写已被锁定的缓存行的数据时，会使缓存行无效，当 CPU1 修改缓存行中的 i 时使用了缓存锁定，那么 CPU2 就不能同时缓存 i 的缓存行。

**但是有两种情况下处理器不会使用缓存锁定**

第一种情况是：当操作的数据不能被缓存在处理器内部，或操作的数据跨多个缓存 行（cache line）时，则处理器会调用总线锁定。 

第二种情况是：有些处理器不支持缓存锁定。对于 Intel 486 和 Pentium 处理器，就算锁定的内存区域在处理器的缓存行中也会调用总线锁定。 

针对以上两个机制，我们通过 Intel 处理器提供了很多 Lock 前缀的指令来实现。例如，位测试和修改指令：BTS、BTR、BTC；交换指令 XADD、CMPXCHG，以及其他 一些操作数和逻辑指令（如 ADD、OR）等，被这些指令操作的内存区域就会加锁，导 致其他处理器不能同时访问它。

## 四、Java 实现原子操作

在 Java 中可以通过锁和循环 CAS 的方式来实现原子操作。

### 4.1、使用循环 CAS 实现原子操作

JVM 中的 CAS 操作正是利用了处理器提供的 CMPXCHG 指令实现的。自旋 CAS 实 现的基本思路就是循环进行 CAS 操作直到成功为止，以下代码实现了一个基于 CAS 线 程安全的计数器方法 safeCount 和一个非线程安全的计数器 count。

从 Java 1.5 开始，JDK 的并发包里提供了一些类来支持原子操作，如 AtomicBoolean （用原子方式更新的 boolean 值）、AtomicInteger（用原子方式更新的 int 值）和 AtomicLong（用原子方式更新的 long 值）。这些原子包装类还提供了有用的工具方法， 比如以原子的方式将当前值自增 1 和自减 1。

```java
private AtomicInteger atomicI = new AtomicInteger(0);
private int i = 0;
public static void main(String[] args) {
    final Counter cas = new Counter();
    List<Thread> ts = new ArrayList<Thread>(600);
    long start = System.currentTimeMillis();
    for (int j = 0; j < 100; j++) {
        Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 10000; i++) {
                    cas.count();
                    cas.safeCount();
                }
            }
        });
        ts.add(t);
    }
    for (Thread t : ts) {
        t.start();
    }
    // 等待所有线程执行完成
    for (Thread t : ts) {
        try {
            t.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    System.out.println(cas.i);
    System.out.println(cas.atomicI.get());
    System.out.println(System.currentTimeMillis() - start);
}
/**
 * 使用 CAS 实现线程安全计数器
 */
private void safeCount() {
    for (; ; ) {
        int i = atomicI.get();
        boolean suc = atomicI.compareAndSet(i, ++i);
        if (suc) {
            break;
        }
    }
}
/**
 * 非线程安全计数器
 */
private void count() {
    i++;
}
```

**CAS 实现原子操作的三大问题**

1. ABA 问题:

   因为 CAS 需要在操作值的时候，检查值有没有发生变化，如果没有发 生变化则更新，但是如果一个值原来是 A，变成了 B，又变成了 A，那么使用 CAS 进行 检查时会发现它的值没有发生变化，但是实际上却变化了。

   ABA 问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加 1，那么 A→B→A 就会变成 1A→2B→3A。从 Java 1.5 开始，JDK 的 Atomic 包里提供了一个类 AtomicStampedReference 来解决 ABA 问题。这个类的 compareAndSet 方法的作用是首先 检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果全部相 等，则以原子方式将该引用和该标志的值设置为给定的更新值。

2. 循环时间长开销大

   自旋 CAS 如果长时间不成功，会给 CPU 带来非常大的执行开销。如果 JVM 能支持处理器提供的 pause 指令，那么效率会有一定的提升。

   pause 指令有两个作用：第一它可以延迟流水线执行指令（de-pipeline），使 CPU 不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零；第二它可以避免在退出循环的时候因内存顺序冲突（Memory Order Violation）而引起 CPU 流 水线被清空（CPU Pipeline Flush），从而提高 CPU 的执行效率。

3. 只能保证一个共享变量的原子操作

   当对一个共享变量执行操作时，我们可以使用 循环 CAS 的方式来保证原子操作，但是对多个共享变量操作时，循环 CAS 就无法保证 操作的原子性，这个时候就可以用锁。

   还有一个取巧的办法，就是把多个共享变量合并 成一个共享变量来操作。比如，有两个共享变量 i＝2，j=a，合并一下 ij=2a，然后用 CAS 来操作 ij。从 Java 1.5 开始， JDK 提供了 AtomicReference 类来保证引用对象之间的 原子性，就可以把多个变量放在一个对象里来进行 CAS 操作。

### 4.2、使用锁机制实现原子操作

锁机制保证了只有获得锁的线程才能够操作锁定的内存区域。JVM 内部实现了很多种锁机制，有偏向锁、轻量级锁和互斥锁。有意思的是除了偏向锁，JVM 实现锁的方式都用了循环 CAS，即当一个线程想进入同步块的时候使用循环 CAS 的方式来获取锁， 当它退出同步块的时候使用循环 CAS 释放锁。





