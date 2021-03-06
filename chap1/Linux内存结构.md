# 1.2 Linux内存结构

翻译：飞哥 (http://hi.baidu.com/imlidapeng)

版权所有，尊重他人劳动成果，转载时请注明作者和原始出处及本声明。

原文名称：[《Linux Performance and Tuning Guidelines》](http://www.redbooks.ibm.com/abstracts/redp4285.html)

-------------------------------------------------------------------------------------------

1.2.1 物理内存与虚拟内存
1.2.2 虚拟内存管理

--------------------------------------------------------------------------------------------

执行一个进程时，需要Linux 内核分配一段内存区域，用来作为工作区来执行它的工作。这就像你的办工桌，你可以用它来摆放工作需要的纸张、文件和备忘录。不同之处是Linux内核要使用动态的方法来分配内存空间。在内存大小通常是有限的情况下有时进程的数量会达到数万个，Linux内核必须有效地来使用内存。在本章节中，我们会了解到内存的结构、地址分布，以及Linux是怎样有效地管理内存空间的。

## 1.2.1 物理内存与虚拟内存

今天我要面对选择32位系统还是选择64位系统的这样问题。其中对于企业级用户最重要的区别就是虚拟内存地址是否可以超过4GB。从性能角度来看，弄明白32位和64位Linux内核是怎么样将物理内存映射到虚拟内存是非常有益的。

在图1-10中，你可以很明显的看出32位系统和64位系统在内存地址分配上的不同之处。关于物理内存映射到虚拟内存的详细内容已超出本文的范畴，本文将只着重介绍Linux内存结构的部分细节。

在32位架构如IA-32中，Linux内核只能直接访问物理内存的前1GB（当考虑保留部分时只有896MB）。在ZONE_NORMAL之上的内存需要映射到1GB一下，这样的映射对于应用来说是完全透明的，但分配内存页到ZONE_HIGHMEM会导致性能轻微的下降。

在64位架构如x86-64（也称作x64），ZONE_NORMAL可一直延伸到64GB，在IA-64中甚至达到128GB。正如你所见到的，在64位架构中，可以被省去将内存页从ZONE_HIGHMEM映射到ZONE_NORMAL的开销。


图1-10 32位和64位Linux内核内存布局

虚拟内存寻址布局

图1-11展示了32位和64位Linux虚拟寻址布局。

在32位架构中，单个进程能访问最大地址空间位4GB，这就限制了32位虚拟寻址能力。在标准实现中，虚拟地址空间被分为3GB的用户空间和1GB的内核空间，这有些像4 G/4 G寻址布局实现的变种。

在64位架构如X86_64和IA64中，就没有这样的限制。每个单独的进程都能获得巨大的地址空间。


图1-11 32位和64位Linux虚拟寻址布局

## 1.2.2 虚拟内存管理

操作系统的物理内存架构对于应用和用户来说通常是隐藏的，因为操作系统会将所有内存映射到虚拟内存。如果我们想要知道Linux操作系统调优的可能性，我就必须要明白Linux是怎样管理虚拟内存的。正如1.2.1“物理内存和虚拟内存”中讲到的，应用是不能分配到物理内存的，当向Linux内核请求一定大小的内存映射时，得到是一段虚拟内存的映射。如图1-12所示，虚拟内存不一定要映射到物理内存，如果你的应用需要很多内存，其中一部分很有可能是被映射到了硬盘上的交换文件。

如图1-12所示应用通常直接写进缓存区(Cache)或缓冲区【Buffer】而不是硬盘。当内核线程pdflush空闲或文件大小超出缓冲区，pdflush会将缓存区和缓冲区数据清空并写入硬盘。参考“清空脏缓冲”。



图1-12 Linux虚拟内存管理

Closely connected to the way the Linux kernel handles writes to the physical disk subsystem is the way the Linux kernel manages disk cache.其它操作系统只分配部分内存作为硬盘缓存，而Linux在内存资源管理上则更加有效。Linux虚拟内存管理默认将所有空闲内存空间都作为硬盘缓存。因此在拥有数GB内存的生产性Linux系统中，经常可以看到可用的内存只有20MB。

同样，Linux在管理交换空间方面也十分有效。当交换空间被使用是并不表示内存出现瓶颈，这恰恰证明Linux管理系统资源是多么有效。详见“页帧回收”

页帧的分配

页(Page)就是指在物理内存（page frame）或虚拟内存中一组连续的地址。Linux内核就是以页为单位来管理内存的，页的大小通常为4KB。当有进程请求一定数量的页时，如果存在可有的页Linux内核会立即将其分配给进程，否则要从其它的进程或页缓存中取得页。内核清楚知道有多少可以使用的内存页并知道它们都在哪里？

伙伴系统(Buddy System)

Linux内核使用一种名叫伙伴系统的机制来管理自由页(Free Pages)。伙伴系统维护自由页并尝试为有需要的进程分配页，尽可能保持内存区是连续。如果忽视分散的小页，可能会造成内存碎片。这让在连续的区块中分配一大段页变得困难。从而导致内存使用效率降低性能下降。图1-13说明了伙伴系统是如何分配页的。

图1-13 伙伴系统

当尝试分配页失败，页回收被激活。参见“页帧回收”。

你可以在/proc/buddyinfo中获得关于伙伴系统的信息。详细内容参见“Memory used in a zone”。

页帧回收

当进程请求一定数量的页但没有可用的页时，Linux内核会尝试释放某些页（之前被使用过但现在没有被使用，但基于某种原则其仍被标示为活动页）来满足请求并分配内存给新的进程。这个进程被称作页回收。kswapd内核线程和try_to_free_page()内核函数被用来负责页的回收。

kswapd通常在可中断状态中睡眠，当某一区域(Zone)的自由页低于临界值时伙伴系统将呼叫kswapd。它将尝试使用最近最少使用（LRU）原则从活动页中找出候选页。最近最少使用的页将被首先释放。活动列表和非活动列表被用来维护候选页。kswapd扫描部分活动列表，检查页最近的使用情况，将最近没有使用的页放入非活动页。你可以使用vmstat -a查看到哪些内存是活动的哪些内存是非活动的。详细信息参见2.3.2"vmstat"。

kswapd也要遵循其它原则。页的使用主要两个用途是：页缓存(Page Cache)和进程地址空间【Process Address Space】。页缓存是指将页映射到硬盘上的一个文件。一个进程地址空间的页（被称作匿名内存，因为它没有映射到任何文件也没有名字）被用作堆和堆栈。参见1.1.8“进程内存段“。当kswapd回收页时，它会尽量压缩页缓存而不是将进程的页page out（或swap out）。

Page out and swap out：page out和swap out很容易被混淆。”page out“是将某些页（整个地址空间的一部分）置入交换空间，而”swap out“是将整个地址空间置入交换空间。但有时它们可以互换使用。

大部分被回收的页缓存和进程地址空间取决于使用情境从而影响性能。你可以通过/porc/sys/vm/swappiness来控制这样的行为。调优的详细内容参见4.5.1“设置内核交换和pdflush行为”。

swap

正如之前所述，当页回收发生时，在活动列表中属于进程地址空间的候选页会被page out。交换的发生本身并不意味着出了状况。虽然在其它操作系统中交换只不过是当内存过度分配时的一种保障，但Linux在使用交换空间时则更为有效。正如你在图1-12所见，虚拟内存是由物理内存和硬盘子系统或交换空间组成。在Linux中如虚拟内存管理者发现被分配的内存页在一段时间内没有被使用，它将被移至交换空间。

你经常可以看到守护进程如getty自从系统启动后就很少被使用。显然将内存页移至交换空间释放所占用的主内存会更加有效。如果你发现交换分区已经使用了50%。并不需要惊慌，这正体现Linux是怎样管理交换的。事实上交换空间被使用并不意味着内存出现瓶颈，相反验证了Linux使用系统资源的有效。
