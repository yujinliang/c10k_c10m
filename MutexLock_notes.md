# 										锁的核心原理杂记



- 简论

> `锁`大家应该了解，并发编程必备！而且种类众多！锁、自旋锁、递归锁、读写锁等等， 内核态锁、用户态锁等等，线程锁、协程锁等等，阻塞锁、非阻塞锁等等。包括信号量、条件量本质上也是基于锁机制，锁的使命归根到底在于保护临界区，竞态保护，排他性读写！



- 底层核心技术

> 不论那种`锁`类型， 底层技术必须得到硬件CPU的支持， 除非CPU硬件提供一些原子操作（原子操作指令），否则无法实现互斥锁！
>
> 而且不同的CPU不同的指令集提供的`原子操作指令`也不尽相同！，比如：
>
> 1. `Test-and-Set `
> 2. `Compare-and-Swap :CAS`
> 3. `Risc   load-link store-conditional`
> 4. `xchg and lock cmpxchg on x86 `
>
> 所谓一生二，二生三，三生万物！CPU提供原子操作指令帮我们检测寄存器或内存位置是否可以排他性访问，返回true/false结果， 根据这个测试结果，不同的锁种类执行不同的策略逻辑代码！通常将线程或协程打入队列，然后阻塞等待条件满足（得到锁）再唤醒继续执行； 也可以进一步细分为`读者对列`和`写者队列`， 从而实现读写锁；未获得锁时还可以立即返回而不是阻塞等待，还可以不阻塞而是持续检测持有者是否已经释放了锁；包括递归锁可以反复对同一个未释放的锁再次反复加锁！不会死锁，`windows`下的临界区默认是支持递归锁的，而`linux`下的互斥量则需要设置参数`PTHREAD_MUTEX_RECURSIVE_NP`，默认则是不支持； 如果`临界区`被多次并发重入，此时采用递归锁可以避免死锁！因为通常对一个已经加锁的锁再次加锁，则会导致死锁。



- 避免死锁

> 我是C++程序员，从C++入行， 持续做了多年，后来又用过`python , php, golang `等, 知道现在专攻Rust，网络分布式高并发的项目也做过一些， 虽是菜鸟，但是多年来也研读了许多技术资料，特别是高并发这块， 对于如何`避免死锁`这块，我始终记忆犹新，反复研读的资料就是`C++编程思想卷2-11章并发-11.8节：死锁-哲学家进餐`， 其中有一段关于避免死锁的精辟论断，我始终奉为金科玉律！
>
> 同时满足4个条件则死锁必然发生：
>
> 1. 相互排斥。线程使用的资源至少有一个必须是不可共享的，即排他性使用。在这种情况下，一根筷子一次就只能被一个哲学家使用。
> 2. 至少有一个进程必须持有某一种资源，并且同时等待获得正在被另外的进程所持有的资源。也就是说，要发生死锁，则一个哲学家必须持有一根筷子并且等待另一根筷子。
> 3. 不能以抢占式的方式剥夺一个进程的资源。我们的哲学家是有礼貌的，他们不会从别的哲学家手中抢夺筷子。
> 4. 出现一个循环等待，一个进程等待另外的进程所持有的资源，而这个被等待的进程有等待另一个进程所持有的资源，以此类推直到某个进程去等待被第一个进程所持有的资源。因此头尾相接环环相扣，因此大家都被锁住了。
>
> 注意：上面条件中所说的`进程`和`线程` ，可以理解为泛指，具体可以是：进程、线程、协程或其他执行体等。大家明白原理就好，不能死扣。
>
> 【所以上面的4个条件，我们在实际并发编程时只要破坏其中之一，使其不满足，则可避免程序发生死锁！！！】， 经典之所以称为经典就是因为它不会轻易过时，并且对未来仍具有指导性和洞察力。



- 高效并发

> 锁的不当使用可直接导致并发度下降，甚至死锁！所以它不是银弹！下面收集一些经验和建议，开拓思路。



- Reference

> `https://stackoverflow.com/questions/1485924/how-are-mutexes-implemented`
> `http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dht0008a/ch01s03s02.html`
> `https://stackoverflow.com/questions/2679816/implementing-a-mutex-lock-in-c`
> `http://forum.codecall.net/topic/64301-writing-a-simple-lock-mutex-in-cassembly/`
> `https://doc.rust-lang.org/std/sync/struct.Mutex.html`
> `https://docs.rs/async-std/1.6.1/async_std/sync/struct.Mutex.html`
> `https://www.zhihu.com/question/332113890/answer/762392859`
> `https://blog.csdn.net/u011244446/article/details/52574369`
> `https://doc.rust-lang.org/std/sync/struct.RwLock.html`
> `https://stackoverflow.com/questions/37241553/locks-around-memory-manipulation-via-inline-assembly/37246263#37246263`
> `https://stackoverflow.com/questions/52922495/mutex-implementation-without-hardware-support-of-test-and-set-and-cas-operations`
> `https://blog.csdn.net/qq100440110/article/details/51076609`
> `https://www.cnblogs.com/evenleee/p/11309156.html`
>
> 