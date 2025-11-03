---
title: " Linux - RCU"
categories: 操作系统
---
## 参考博客
- [Linux内核同步机制之（七）：RCU基础](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html)
- **[Linux内核同步机制之（七）：RCU基础](http://www.wowotech.net/kernel_synchronization/rcu_fundamentals.html#:~:text=%EF%BC%881%EF%BC%89rcu)**
- **[深入理解RCU核心原理](https://zhuanlan.zhihu.com/p/384227338)**
- **[Linux RCU 原理剖析（一）](https://www.cnblogs.com/LoyenWang/p/12681494.html)**
- **[Linux RCU 原理剖析（二）](https://www.cnblogs.com/LoyenWang/p/12770878.html)**

## 简介

每种兵器都有自己的适用场合，内核同步机制亦然.

## 并行程序设计演进

如何正确有效的保护共享数据是编写并行程序必须面临的一个难题，通常的手段就是同步。同步可分为阻塞型同步（Blocking Synchronization）和非阻塞型同步（ Non-blocking Synchronization）。

**阻塞型同步**是指当一个线程到达临界区时，因另外一个线程已经持有访问该共享数据的锁，从而不能获取锁资源而阻塞（睡眠），直到另外一个线程释放锁。常见的同步原语有 mutex、semaphore 等。如果同步方案采用不当，就会造成死锁（deadlock），活锁（livelock）和优先级反转（priority inversion），以及效率低下等现象。

为了降低风险程度和提高程序运行效率，业界提出了不采用锁的同步方案，依照这种设计思路设计的算法称为**非阻塞型同步**，其本质就是停止一个线程的执行不会阻碍系统中其他执行实体的运行。

  

## 阻塞型同步

**互斥锁**（英語：Mutual exclusion，缩写Mutex）是一种用于多线程编程中，防止两条线程同时对同一公共资源进行读写的机制。该目的通过将代码切片成一个一个的临界区域（critical section）达成。临界区域指的是一块对公共资源进行存取的代码。

**信号****量**(Semaphore)，是在多线程环境下使用的一种设施，是可以用来保证两个或多个关键代码段不被并发调用，可以认为mutex是0-1信号量；

**读写锁**是计算机程序的并发控制的一种同步机制，它把对共享资源的访问者划分成读者和写者，读者只对共享资源进行读访问，写者则需要对共享资源进行写操作，读操作可并发重入，写操作是互斥的。

## 非阻塞型同步

当今比较流行的非阻塞型同步实现方案有三种：

1. Wait-free（无等待） Wait-free 是指任意线程的任何操作都可以在有限步之内结束，而不用关心其它线程的执行速度。Wait-free 是基于 per-thread 的，可以认为是 starvation-free 的。非常遗憾的是实际情况并非如此，采用 Wait-free 的程序并不能保证 starvation-free，同时内存消耗也随线程数量而线性增长。目前只有极少数的非阻塞算法实现了这一点。 简单理解：任意时刻所有的线程都在干活；
2. Lock-free（无锁） Lock-Free是指能够确保执行它的所有线程中至少有一个能够继续往下执行。由于每个线程不是 starvation-free 的，即有些线程可能会被任意地延迟，然而在每一步都至少有一个线程能够往下执行，因此系统作为一个整体是在持续执行的，可以认为是 system-wide 的。所有 Wait-free 的算法都是 Lock-Free 的。 简单理解：任意时刻至少一个线程在干活；
3. Obstruction-free（无障碍） Obstruction-free 是指在任何时间点，一个孤立运行线程的每一个操作可以在有限步之内结束。只要没有竞争，线程就可以持续运行。一旦共享数据被修改，Obstruction-free 要求中止已经完成的部分操作，并进行回滚。所有 Lock-Free 的算法都是 Obstruction-free 的。 简单理解：只要数据有修改，就会重新获取，并且把已经完成操作回滚重来；

综上所述，不难得出 Obstruction-free 是 Non-blocking synchronization 中性能最差的，而 Wait-free 性能是最好的，但实现难度也是最大的，因此 Lock-free 算法开始被重视，并广泛运用于各种程序设计中，这里主要介绍Lock_free算法。

lock-free（无锁）往往可以提供更好的性能和伸缩性保证，但实际上其优点不止于此。早期这些概念首先是在操作系统上应用的，因为一个不依赖于锁的算法，可以应用于各种场景下，而无需考虑各种错误，故障，失败等情形。比如死锁，中断，甚至CPU失效。

## 主流无锁技术

### Atomic operation（[原子操作](https://zhida.zhihu.com/search?content_id=173514525&content_type=Article&match_order=1&q=%E5%8E%9F%E5%AD%90%E6%93%8D%E4%BD%9C&zhida_source=entity)）

在单一、不间断的步骤中读取和更改数据的操作。需要处理器指令支持原子操作：

● test-and-set (TSR)

● compare-and-swap (CAS)

● load-link/store-conditional (ll/sc)

### Spin Lock（自旋锁）

是一种轻量级的同步方法，一种非阻塞锁。当 lock 操作被阻塞时，并不是把自己挂到一个等待队列，而是死循环 CPU 空转等待其他线程释放锁。

### Seqlock ([顺序锁](https://zhida.zhihu.com/search?content_id=173514525&content_type=Article&match_order=1&q=%E9%A1%BA%E5%BA%8F%E9%94%81&zhida_source=entity))

是Linux 2.6 内核中引入一种新型锁，它与 spin lock 读写锁非常相似，只是它**为写者赋予了较高的优先级**。也就是说，即使读者正在读的时候也允许写者继续运行，读者会检查数据是否有更新，如果数据有更新就会重试，因为 [seqlock](https://zhida.zhihu.com/search?content_id=173514525&content_type=Article&match_order=1&q=seqlock&zhida_source=entity) 对写者更有利，只要没有其他写者，写锁总能获取成功。

### RCU(Read-Copy Update)

顾名思义就是读-拷贝修改，它是基于其原理命名的。对于被RCU保护的[共享数据结构](https://zhida.zhihu.com/search?content_id=173514525&content_type=Article&match_order=1&q=%E5%85%B1%E4%BA%AB%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84&zhida_source=entity)，读者不需要获得任何锁就可以访问它，但写者在访问它时首先拷贝一个副本，然后对副本进行修改，最后使用一个回调（callback）机制在适当的时机把指向原来数据的指针替换为新的被修改的数据。这个时机就是**所有引用该数据的****CPU****都退出**对共享数据的访问。

#### 要使用RCU机制保护自定义数据结构

```Go
struct foo {
        int a;
        int b;
        int c;
        struct rcu_head rcu;
        struct list_head list;
};
```

***自定义数据结构必须包含RCU机制相关的部分数据结构***

由于 RCU 旨在最小化读取端开销，因此仅在以更高速率使用同步逻辑进行读取操作时才使用它。**如果更新操作超过10%，性能反而会变差**，所以应该选择另一种同步方式而不是RCU。

RCU 的全称 Read-Copy-Update，它是 Linux 内核中一种重要的同步机制。以读写信号量为例，除了上述缺点外，读写信号量还有一个致命弱点，它允许多个读者同时存在，但是读者和写者不能同时存在。因此 RCU 机制要实现的目标是，读者线程没有同步开销，或者说同步开销变得很小，甚至可以忽略不计，不需要额外的锁，不需要使用原子操作指令和内存屏障指令，即可畅通无阻地访问；而**把需要同步的任务交给写者线程**，写者线程等待所有读者线程完成后才会把旧数据销毁。

在 RCU 中，如果有多个写者同时存在，那么需要额外的保护机制。RCU 机制的原理可以概括为 RCU 记录了所有指向共享数据的指针的使用者，**当要修改共享数据时，首先创建一个副本，在副本中修改**。所有读者线程离开读者临界区之后，指针指向修改后的副本，并且删除旧数据。

RCU 的一个重要的应用场景是链表，链表可以有效地提高遍历读取数据的效率。读取链表成员数据时通常只需要 rcu_read_lock() 函数，允许多个线程同时读取该链表，并且允许一个线程同时修改链表。那为什么这个过程能保证链表访问的正确性呢？ 在读者遍历链表时，假设另外一个线程删除了一个节点。删除线程会**把这个节点从链表中移出，但不会直接销毁它**。RCU 会等到所有读线程读取完成后，才销毁这个节点。

## vs RWLOCK

当线程在多个cpu上争抢进入临界区的时候，都会操作那个**在多个cpu之间共享的数据lock**（玫瑰色的block）。cpu 0操作了lock，为了数据的一致性，cpu 0的操作会导致其他cpu的L1中的lock变成无效，在随后的来自其他cpu对lock的访问会导致L1 cache miss（更准确的说是communication cache miss），必须从下一个level的cache中获取，同样的，其他cpu的L1 cache中的lock也被设定为invalid，从而引起下一次其他cpu上的communication cache miss。

**RCU**的read side不需要访问这样的“共享数据”**，从而**极大的提升了reader侧的性能。

rwlock允许多个reader并发，因此，在上图中，三个rwlock reader愉快的并行执行。当rwlock writer试图进入的时候（红色虚线），只能spin，直到所有的reader退出临界区。一旦有rwlock writer在临界区，任何的reader都不能进入，直到writer完成数据更新，立刻临界区。绿色的reader thread们又可以进行愉快玩耍了。rwlock的一个特点就是确定性，白色的reader一定是读取的是old data，而绿色的reader一定获取的是writer更新之后的new data。RCU和传统的锁机制不同，当RCU updater进入临界区的时候，即便是有reader在也无所谓，它可以长驱直入，不需要spin。同样的，即便有一个updater正在临界区里面工作，这并不能阻挡RCU reader的步伐。由此可见，RCU的并发性能要好于rwlock，特别如果考虑cpu的数目比较多的情况，那些处于spin状态的cpu在无谓的消耗，多么可惜，随着cpu的数目增加，rwlock性能不断的下降。RCU reader和updater由于可以并发执行，因此这时候的被保护的数据有两份，一份是旧的，一份是新的，**对于白色的RCU reader，其读取的数据可能是旧的，也可能是新的，和数据访问的timing相关**，当然，当RCU update完成更新之后，新启动的RCU reader 读取的一定是新的数据。

目前的趋势是：CPU和存储器件之间的速度差别在逐渐扩大。因此，那些**基于一个multi-processor之间的共享的counter的锁机制已经不能满足性能的需求**，在这种情况下，RCU机制应运而生（当然，更准确的说RCU一种内核同步机制，但不是一种lock，**本质上它是lock-free的**），它克服了其他锁机制的缺点，但是，甘蔗没有两头甜，RCU的使用场景比较受限，主要适用于下面的场景：

（1）RCU**只能保护动态分配的数据结构**，并且**必须是通过指针访问该数据结构**

（2）受RCU保护的临界区内不能sleep（SRCU不是本文的内容）

（3）读写不对称，对writer的性能没有特别要求，但是reader性能要求极高

（4）reader端对新旧数据不敏感

## 旧数据的回收时机

1. 写者调用synchronize_rcu()：同步等待所有现存的读访问【旧数据】完成
2. call_rcu()：注册一个回调函数，当所有现存的读访问完成后，调用这个回调函数销毁旧数据
## 度过宽限期上报

RCU的核心理念是读者访问的同时，写者可以更新访问对象的副本，但写者需要等待所有读者完成访问之后，才能删除老对象。这个过程**实现的关键和难点就在于如何判断所有的读者已经完成访问**。通常把写者开始更新，到所有读者完成访问这段时间叫做宽限期（Grace Period）。内核中实现宽限期等待的函数是synchronize_rcu。

每个CPU在**时钟中断**的处理函数中，都会判断当前CPU是否度过quiescent state。普通的RCU例如内核线程、系统调用等场景，使用rcu_read_lock或者rcu_read_lock_sched，他们的实现是一样的；软中断上下文则可以使用rcu_read_lock_bh，使得宽限期更快度过。

每个CPU度过quiescent state之后，需要向上汇报直至所有CPU完成quiescent state，从而标识宽限期的完成，这个汇报过程在**软中断**RCU_SOFTIRQ中完成。软中断的唤醒则是在上述的时钟中断中进行。

通过`__call_rcu`注册的这些回调函数也是在`RCU_SOFTIRQ`软中断中调用【触发perCPU的Kthread进行调用】

## 经典RCU vs Tree RCU

经典 RCU 的实现在超级大系统中遇到了问题，特别是有些系统的 CPU 内核超过了 1024 个，甚至达到 4096 个。经典 RCU 在判断是否完成一次 GP 时采用全局的 cpumask 图。如果每位表示一个 CPU，那么在 1024 个 CPU 内核的系统中， cpumask 位图就有 1024 位。每个 CPU 在 GP 开始时要设置位图中对应的位，GP 结束时要清除相应的位。全局的 cpumask 位图会导致很多 CPU 竞争使用，因此需要自旋锁来保护位图。这样导致锁争用变得很激烈，激烈程度随着 CPU 的个数线性递增。

Tree RCU 的实现巧妙地解决了 cpumask 位图竞争锁的问题，以上述 4 核 CPU 为例，假设 Tree RCU 以两个 CPU 为一个 rcu_node, 这样 4 个 CPU 被分配到两个 rcu_node，使用另外一个 rcu_node 来管理这两个 rcu_node。节点 1 管理 pu0 和 cpu1，节点 2 管理 pu2 和 cpu3。而节点 0 是根节点，管理节点 1 和节点 2。每个节点只需要两位的位图就可以管理各自的 CPU 或者节点，每个节点都通过各自的自旋锁来保护相应的位图。