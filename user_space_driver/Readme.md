# 										User Space Driver  Tech Study Notes

- What

> 通常驱动程序和OS内核一切运行在内核态！如Linux这种宏内核的OS更是如此！所以驱动程序一旦实现的有问题，则直接导致OS挂掉！所以Linux对于驱动程序的开发技术、规范、运行等都有严格的限制！从而导致内核态驱动程序难以编写和调试！Linux的哲学一切皆是文件！所以Linux将硬件的读写和管理抽象为统一的`API`, 主要形式就是对于文件的读写和配置！传统的OS如Linux/Windows等一般都会将硬件抽象封装起来，屏蔽其细节，统一其接口！从而使之易用！一般网络应用程序需要和网卡硬件交互则调用OS提供的统一接口，如socket等，同时从用户态切换到内核态，并且需要将应用读写硬件的data在内核态和用户态之间复制！ 还要经过网络协议栈逐层处理，然后到达网卡buffer, 最后网卡执行实际的data收发和中断的触发！
>
> 如果每秒并发connection比较少的情况下，一般几千个时，也就是`C10K`级别以下，性能还可以的！可以一旦上升到一万，十万，百万，千万级并发connection时，操作系统首先就扛不住了！所以国外大牛写了一篇文章主要观点就是：对于`C10M`级别下的并发connection处理，操作系统内核不是解决方案，反而是性能瓶颈！
>
> 所以User Space Driver技术就被提出！思想很单纯和直接， 如果这个硬件就为你一个`应用程序`服务，比如网卡， 那么最好由应用程序自己在User Space 模式下直接管理和读写硬件！最好无需OS参与！当然让OS完全不参与也未必好！比如可以让OS只担任介绍人， 剩下的事你俩自己搞定，OS乐得清闲自在！应用程序与硬件直接交互，硬件中断也由应用程序自己处理，避免了无谓的上下文切换和data复制！对于网卡硬件而言， 网络应用程序可以根据应用需要只运行定制的最小化最单一的网络协议栈！最大限度避免做无用功和过多的层级！而且编写应用程序可不像编写内核态驱动程序有那么多技术限制，可以充分自由发挥利用各种技术，最大化开发利用硬件的速率潜力！好啰嗦呀，我掰吃清楚了吧。
>
> 如果想要详细深入地研究User Space Driver技术的出现动机和背景，请参考：
> `http://www.kegel.com/c10k.html`
> `http://highscalability.com/blog/2013/5/13/the-secret-to-10-million-concurrent-connections-the-kernel-i.html`
>
> `https://www.hackingnote.com/en/distributed-systems/c10k`



- Which

> `DPDK`、`Snabb`、`ixy`等。



- 用户态Driver实现核心方法归类

> 1. Mapping memory to user space（将OS内核态的内存map到user space中供`App`直接读写）
>2. `UIO` drivers
> 3. `VFIO` drivers
>4. User-space network drivers（将硬件的寄存器和buffer等直接map到user space态的内存区域中直接供`App`读写操控，相比较上面123而言最彻底撇开OS）当然它也有难点：硬件中断处理、应用重启或崩溃时硬件如何管理、没有形成统一的硬件访问接口、硬件如何共享、管理各种硬件内存buffers、运行自己的网络协议栈，如：`TCP/IP` 。优点那就是快！撇开了OS这个性能瓶颈后，User Space Driver可以最大化开发利用硬件速率！
> 



- 实现用户态Driver的核心原理

> 我参阅了一些资料， 妄自揣测，如有谬误，希望指教。
>
> 目标：将硬件的各种寄存器和data buffer等直接map(映射)到应用程序所在的User Space模式下的内存区域，从而实现应用程序只需读写相应的内存区域，就可以配置管理读写硬件！同时还要将硬件中断map(映射)给此应用程序，由应程序自己处理硬件中断，事件通知和dispatch等。所以如`DPDK`等User Space Driver 开发工具包就是帮助我们实现此目标！同时降低复杂度！所以明白了关键点后，我们就可以从容地学习各种开发工具包！因为万变不离其宗！



- 实现难点

> 主要难点有二， 一就是硬件中断如何处理， 二就是如何将硬件，如网卡的各种寄存器和buffer映射到用户态应用程序的内存下？我着重谈一下第二点：
>
> 硬件内核态driver和OS内核一起都运行在内核态， 人家活在真实的世界， 内存地址就是真实物理地址， 不是虚拟地址！而每个应用程序活在虚拟的世界里， 好似独占内存，可那个内存是OS给它虚拟出来的！所以是虚拟内存地址！所以需要经过`MMU`和OS配合才能转换成真实的内存地址，才能读写data！问题来了， 以前硬件都归OS内核统一管理，人家是体制内的人， 现在要下海单干脱离体质， 难点在于应用程序如何为这个硬件分配指定真实的物理内存地址！应用程序有这个能力和权利吗？学个操作系统`虚拟内存`的朋友应该了解， OS分配给应用程序的虚拟内存实际对应到那块真实物理内存是动态变化的！完全由OS决定， OS不高兴的话也可以把你直接swap到硬盘上去！反正你手里拿着饭票，而饭在哪里完全由OS说了算！！！哈哈！既然硬件，如网卡需要真实的物理内存地址才能实现映射！那么用户态的应用程序如何满足呢？！ 不同的开发包，如`DPDK`等都有自己的trick(黑科技) 。 
>
> 所以知道了核心原理、目标、难点后，再去学习各种User Space Driver技术开发包就不容易晕乎了！



- Reference Links

> >  `https://github.com/emmericp/ixy`
> >  `https://github.com/ixy-languages/ixy-languages`
> >  `https://github.com/ixy-languages/ixy.rs`
>
> > `http://alvarom.com/2014/12/17/linux-user-space-drivers-with-interrupts/`
> > `https://lwn.net/Articles/232575/`
> >
> > `https://blog.zhangpf.com/2018/08/19/write-userspace-driver-in-rust/`
> >
> > `https://www.embedded.com/device-drivers-in-user-space/`
> >
> > `https://www.kernel.org/doc/html/latest/driver-api/vfio.html`
> > `https://www.kernel.org/doc/html/latest/driver-api/uio-howto.html`
> >
> > `https://www.net.in.tum.de/fileadmin/bibtex/publications/theses/2018-ixy-rust.pdf`
> >
> > `https://elinux.org/images/c/c8/Userspace-drivers-csimmonds-elce-2018_Chris-Simmonds.pdf`
> >
> > `https://www.net.in.tum.de/fileadmin/bibtex/publications/papers/ixy-writing-user-space-network-drivers.pdf`
> >
> > `http://www.kegel.com/c10k.html`
> > `http://highscalability.com/blog/2013/5/13/the-secret-to-10-million-concurrent-connections-the-kernel-i.html`
> >
> > `https://www.hackingnote.com/en/distributed-systems/c10k`



