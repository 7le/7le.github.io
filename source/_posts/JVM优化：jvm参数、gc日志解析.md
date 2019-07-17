---
title: JVM优化：jvm参数、gc日志经验分享
date: 2019-07-16 21:32:12
type: "tags"
tags: [jvm]
---

> 从这篇开始写一些jvm调优相关的内容，一起来揭开jvm神秘的面纱吧~  🥕

<!--more-->

之前的文章[深入理解Java虚拟机--读书笔记](https://7le.top/2018/01/29/深入理解Java虚拟机--读书笔记)已经介绍过jvm，包括一些基础知识、GC的算法和垃圾收集器之类。这里就不赘述了，直接进入正题。

``文章中主要描述的是ParNew和CMS，且jdk版本<=1.8。``

### gc日志

> 俗话说工欲善其事，必先利其器。要做好Jvm的优化，那理解GC日志就是基础。

在jvm参数上增加输出日志的参数``-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC -XX:+PrintGCDateStamps -XX:+PrintTenuringDistribution -Xloggc:/logs/gc.log``，jvm就能为我们将日志输出到指定的路径下。

下面开始分析ParNew和CMS，该搭配主要的目标是低延时，比较常用，另外我们线上也是采用这个：
1. 新生代：采用 stop-the-world mark-copy 算法
2. 老年代：采用 Mostly Concurrent mark-sweep 算法

能够达成该目标主要因为以下两个原因：
1. 它不会花时间整理压缩老年代，而是维护了一个叫做 free-lists 的数据结构，该数据结构用来管理那些回收再利用的内存空间；
2. CMS分为四个阶段，只有initial mark和remark会触发STW，其他都是与应用线程一同运行的

#### ParNew/Young GC

```java
{Heap before GC invocations=74 (full 0):
 par new generation   total 341376K, used 294492K [0x0000000083000000, 0x000000009c000000, 0x000000009c000000)
  eden space 273152K, 100% used [0x0000000083000000, 0x0000000093ac0000, 0x0000000093ac0000)
  from space 68224K,  31% used [0x0000000093ac0000, 0x0000000094f970f8, 0x0000000097d60000)
  to   space 68224K,   0% used [0x0000000097d60000, 0x0000000097d60000, 0x000000009c000000)
 concurrent mark-sweep generation total 1638400K, used 96700K [0x000000009c000000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 113286K, capacity 119312K, committed 119424K, reserved 1153024K
  class space    used 14248K, capacity 15218K, committed 15232K, reserved 1048576K
2019-07-10T18:26:55.933+0800: 5756.871: [GC (Allocation Failure) 2019-07-10T18:26:55.933+0800: 5756.871: [ParNew
Desired survivor size 34930688 bytes, new threshold 15 (max 15)
- age   1:    1688360 bytes,    1688360 total
- age   2:      51552 bytes,    1739912 total
- age   3:      59880 bytes,    1799792 total
- age   4:      51472 bytes,    1851264 total
- age   5:     100528 bytes,    1951792 total
- age   6:     136576 bytes,    2088368 total
- age   7:      45280 bytes,    2133648 total
- age   8:     251688 bytes,    2385336 total
- age   9:    4738624 bytes,    7123960 total
- age  10:     173024 bytes,    7296984 total
- age  11:      43856 bytes,    7340840 total
- age  12:     297760 bytes,    7638600 total
- age  13:    1440080 bytes,    9078680 total
- age  14:     809944 bytes,    9888624 total
- age  15:    1862752 bytes,   11751376 total
: 294492K->17247K(341376K), 0.0139261 secs] 391192K->114369K(1979776K), 0.0141007 secs] [Times: user=0.05 sys=0.00, real=0.02 secs] 
Heap after GC invocations=75 (full 0):
 par new generation   total 341376K, used 17247K [0x0000000083000000, 0x000000009c000000, 0x000000009c000000)
  eden space 273152K,   0% used [0x0000000083000000, 0x0000000083000000, 0x0000000093ac0000)
  from space 68224K,  25% used [0x0000000097d60000, 0x0000000098e37ed8, 0x000000009c000000)
  to   space 68224K,   0% used [0x0000000093ac0000, 0x0000000093ac0000, 0x0000000097d60000)
 concurrent mark-sweep generation total 1638400K, used 97121K [0x000000009c000000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 113286K, capacity 119312K, committed 119424K, reserved 1153024K
  class space    used 14248K, capacity 15218K, committed 15232K, reserved 1048576K
}
```

* Heap before GC  - GC前 堆的信息
   * par new generation total 341376K (eden+s0), used 294492K（已被使用）
   * eden space 273152K, 100% used - eden区满了，所以触发 Young GC
   * from space 68224K,  31% used - 存放年龄未到（默认是15岁，``-XX:MaxTenuringThreshold``），仍存活的对象
   * to   space 68224K,   0% used - 准备在这次YGC存放from+eden存活且未到年龄的对象
   * concurrent mark-sweep generation total 1638400K, used 96700K - 老年代的总量和已使用的量
   * Metaspace used 113286K, capacity 119312K, committed 119424K, reserved 1153024K - Metaspace的总量，已使用，提交和保留的量
   * class space used 14248K, capacity 15218K, committed 15232K, reserved 1048576K - 类空间的总量，已使用，提交和保留的量
* Young GC
   * 2019-07-10T18:26:55.933+0800 -GC发生的时间；
   * 5756.871 – 相对于JVM启动时的间隔时间,单位是秒。
   * GC (Allocation Failure) – YGC的原因，在这个case里边，由于新生代不满足申请的空间，因此触发了YGC;
   * ParNew – 收集器的名称，它预示了新生代使用一个并行的 mark-copy stop-the-world 垃圾收集器；
   * new threshold 15 (max 15) - 代表年龄的阀值为15，下面的age是各年龄的数据量
   * 294492K->17247K - 收集前后新生代的使用情况，
   * (341376K), 0.0139261 secs - 整个新生代的容量，以及垃圾收集器在 w/o final cleanup 阶段消耗的时间
   * 391192K->114369K –  在垃圾收集之前和之后堆内存的使用情况。
   * (1979776K), 0.0141007 secs –可用堆的总大小。以及垃圾收集器在标记和复制新生代存活对象时所消耗的时间。包括和ConcurrentMarkSweep收集器的通信开销, 提升存活时间达标的对象到老年代,以及垃圾收集后期的一些最终清理。
   * [Times: user=0.78 sys=0.01, real=0.11 secs] -与Linux命令所输出的时间含义一致，分别代表用户态消耗的CPU时间、内核态消耗的CPU时间和操作从开始到结束所经过的墙钟时间（Wall Clock Time）。墙钟时间与 CPU时间的区别是，墙钟时间包括各种非运算的等待耗时，例如等待磁盘I/O、等待线程紫塞。而CPU时间不包括这些，但是多CPU或者多核，多线程操作会叠加这些CPU时间。
* Heap after GC  - GC后 堆的信息
   * eden space 273152K,   0% used - Young GC后 eden区清空了
   * from space 68224K,  25% used - ``对应的是Heap before中的to``
   * to   space 68224K,   0% used - ``对应的是Heap before中的from``

从以上的信息我们可以计算出在垃圾收集期间, JVM中的内存使用情况。在GC之前总的堆内存使用量为 **391192K** 新生代的使用量为**294492K**。这意味着老年代使用量等于**95M(382-287)** 。GC之后,新生代的使用量减少了**277245K(294492K-17247K)**, 而总的堆内存使用下降了**276823K(391192K-114369K)**。可以算出有 **422K(277245K-276823K)** 的对象从新生代提升到老年代。因为这个取的是服务刚启动不就的gc日志，所以到老年代的对象并不多。


#### CMS/Old GC

```java
2019-07-11T16:30:06.802+0800: 10.105: [GC (CMS Initial Mark) [1 CMS-initial-mark: 1536000K(1638400K)] 1557852K(1979776K), 0.0164961 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
2019-07-11T16:30:06.819+0800: 10.122: [CMS-concurrent-mark-start]
2019-07-11T16:30:06.845+0800: 10.148: [CMS-concurrent-mark: 0.026/0.026 secs] [Times: user=0.02 sys=0.01, real=0.03 secs] 
2019-07-11T16:30:06.846+0800: 10.148: [CMS-concurrent-preclean-start]
2019-07-11T16:30:06.852+0800: 10.155: [CMS-concurrent-preclean: 0.007/0.007 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
2019-07-11T16:30:06.852+0800: 10.155: [CMS-concurrent-abortable-preclean-start]
 CMS: abort preclean due to time 2019-07-11T16:30:11.947+0800: 15.249: [CMS-concurrent-abortable-preclean: 0.374/5.094 secs] [Times: user=0.36 sys=0.02, real=5.10 secs] 
2019-07-11T16:30:11.948+0800: 15.250: [GC (CMS Final Remark) [YG occupancy: 21852 K (341376 K)]2019-07-11T16:30:11.948+0800: 15.250: [Rescan (parallel) , 0.0179172 secs]2019-07-11T16:30:11.966+0800: 15.268: [weak refs processing, 0.0000590 secs]2019-07-11T16:30:11.966+0800: 15.268: [class unloading, 0.0016677 secs]2019-07-11T16:30:11.968+0800: 15.270: [scrub symbol table, 0.0009642 secs]2019-07-11T16:30:11.969+0800: 15.271: [scrub string table, 0.0004421 secs][1 CMS-remark: 1536000K(1638400K)] 1557852K(1979776K), 0.0215229 secs] [Times: user=0.06 sys=0.00, real=0.02 secs] 
2019-07-11T16:30:11.970+0800: 15.273: [CMS-concurrent-sweep-start]
2019-07-11T16:30:11.971+0800: 15.273: [CMS-concurrent-sweep: 0.007/0.007 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
2019-07-11T16:30:11.971+0800: 15.273: [CMS-concurrent-reset-start]
2019-07-11T16:30:11.977+0800: 15.279: [CMS-concurrent-reset: 0.006/0.006 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
```

* CMS Initial Mark - 初始标记（会出现STW），主要标记所有的GC Root，在jdk<1.8的时候可以设置``-XX:+CMSParallelInitialMarkEnabled``来进行并行。
   * 1536000K – 老年代的当前使用量。
   * (1638400K) – 老年代中可用内存总量。
   * 1557852K –  当前堆内存的使用量。
   * (1979776K) – 可用堆的总大小。
   * 0.0164961 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] – 此次暂停的持续时间, 以 user, system 和 real time 3个部分进行衡量
* CMS Concurrent Mark - 并发预清理（与应用线程一起运行），在此阶段, 垃圾收集器遍历老年代，标记所有的存活对象。
   * 0.026/0.026 secs - 此阶段的持续时间, 分别是运行时间和相应的实际时间。
   * [Times: user=0.02 sys=0.01, real=0.03 secs] - 这部分对并发阶段来说没多少意义, 因为是从并发标记开始时计算的,而这段时间内不仅并发标记在运行,程序也在运行。
* CMS Concurrent Preclean - 并发预清理 （与应用线程一起运行），因为前一阶段是与程序并发进行的,可能有一些引用已经改变。如果在并发标记过程中发生了引用关系变化,JVM会（通过Card）将发生了改变的区域标记为**脏**区。
   * 0.007/0.007 secs 此阶段的持续时间, 分别是运行时间和对应的实际时间
   * [Times: user=0.02 sys=0.00, real=0.02 secs] - 同样是与应用线程一起运行，这个时间没什么意义
* CMS Concurrent Abortable Preclean - 并发可取消的预清理（与应用线程一起运行），本阶段尝试在 STW 的 Final Remark 之前尽可能地多做一些工作。
   * 0.374/5.094 secs - 同样是运行时间和相应的实际时间，不同的是这里差距比之前大很多。因为这里只进行了少量的工作 — 0.374秒的CPU时间，GC线程经历了很多等待时间。
   * [Times: user=0.36 sys=0.02, real=5.10 secs] - 同样是与应用线程一起运行，这个时间没什么意义。
* Final Remark - 最终标记，这是此次GC事件中第二次(也是最后一次)STW阶段。本阶段的目标是完成老年代中所有存活对象的标记. 因为之前的 preclean 阶段是并发的, 有可能无法跟上应用程序的变化速度。所以需要 STW暂停来处理复杂情况。
   * YG occupancy: 21852 K (341376 K) - 当前新生代的使用量和总容量。
   * [Rescan (parallel) , 0.0179172 secs] - 在程序暂停时重新进行扫描(Rescan),以完成存活对象的标记。此时 rescan 是并行执行的,消耗的时间为 0179172秒。
   * weak refs processing, 0.0000590 secs  -  处理弱引用的第一个子阶段(sub-phases)的是持续时间。
   * class unloading, 0.0016677 secs - 第二个子阶段, 卸载不使用的类的是持续时间。
   * scrub string table, 0.0004421 secs - 最后一个子阶段, 清理持有class级别 metadata 的符号表(symbol tables),以及内部化字符串对应的 string tables。当然也显示了暂停的时钟时间。
   * 1536000K(1638400K) - 此阶段完成后老年代的使用量和总容量。
   * 1557852K(1979776K) - 此阶段完成后整个堆内存的使用量和总容量。
   * 0.0215229 secs - 此阶段的持续时间。
   * [Times: user=0.06 sys=0.00, real=0.02 secs]  - GC事件的持续时间, 通过不同的类别来衡量: user, system and real time。
* Concurrent Sweep - 并发清除（与应用线程一起运行），目的是删除未使用的对象，并收回他们占用的空间。
   * 0.007/0.007 secs - 此阶段的持续时间, 分别是运行时间和实际时间。
   * [Times: user=0.02 sys=0.00, real=0.02 secs] - 同样是与应用线程一起运行，这个时间没什么意义。
* Concurrent Reset - 并发重置（与应用线程一起运行），此阶段与应用程序并发执行,重置CMS算法相关的内部数据, 为下一次GC循环做准备。
   * 0.006/0.006 secs - 此阶段的持续时间, 分别是运行时间和实际时间。
   * [Times: user=0.00 sys=0.00, real=0.01 secs] - 同样是与应用线程一起运行，这个时间没什么意义。

#### Full GC

```java
2019-07-15T20:39:19.780+0800: 21227.233: [Full GC (Heap Inspection Initiated GC) 2019-07-15T20:39:19.780+0800: 21227.233: [CMS: 50374K->50344K(1536000K), 0.2175503 secs] 59556K->50344K(1962688K), [Metaspace: 82392K->82392K(1126400K)], 0.2181439 secs] [Times: user=0.22 sys=0.00, real=0.22 secs] 
```

* Full GC - 收集整个堆，包括young gen、old gen、perm gen（如果存在的话）等所有部分。
   * Full GC (Heap Inspection Initiated GC) - GC触发的原因，这里我是手动触发的``jmap -histo:live pid``，除了这个和之前的Allocation Failure，一般还会有Ergonomics（表示JVM内部环境认为此时可以进行一次垃圾收集）。
   * [CMS: 50374K->50344K(1536000K), 0.2175503 secs] - 老年代当前占用的内存->回收后占用的内存（老年代总内存）该操作持续的时间。
   * 59556K->50344K(1962688K) - 堆当前占用的内存->堆回收后占用的内存（堆的总内存） 该操作持续的时间。
   * [Metaspace: 82392K->82392K(1126400K)], 0.2181439 secs] - Metaspace当前占用的内存->Metaspace回收后占用的内存（Metaspace总内存？MaxMetaspaceSize我明明设置的是256M，这接近有1100M了，有点奇怪，我查看了``jmap -heap pid``参数设置是有效的） 该操作持续的时间。
   * [Times: user=0.22 sys=0.00, real=0.22 secs] - 事件的持续时间, 通过三个部分来衡量。

### jvm参数

> 有了上面查看日志的基础之后，再结合**jstat、jmap**，我们就可以根据这些数据在jvm参数上进行调优了（这2个工具就不具体介绍了，不然篇幅实在太长了 - -！）。

#### 经验分享

下面结合具体的参数，进行一些经验分享。这是我们线上其中一个服务的jvm参数：
```java
-Xms2000M -Xmx2000M -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M -Xmn500M -XX:SurvivorRatio=4 -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:ParallelGCThreads=8 -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=0 -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSScavengeBeforeRemark -XX:+CMSParallelInitialMarkEnabled -XX:+CMSParallelRemarkEnabled -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC -XX:+PrintGCDateStamps -XX:+PrintTenuringDistribution -Xloggc:/data/logs/login/gc/gc.log -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/logs/login/gc
```

* ``-Xms2000M -Xmx2000M`` - 初始堆大小和最大堆大小。将Xms和Xmx设为一样的值，可以避免jvm在内存不够时，扩容带来的性能消耗。
* ``-XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M`` - jdk>=8移除了Perm，引入了Metapsace，这个参数跟-Xms -Xmx一样建议设置成一样，可以避免Metapsace在内存不够时，扩容带来的性能消耗。具体设置多大，建议稳定运行一段时间后通过``jstat -gc pid``确认且这个值大一些，对于大部分项目256M即可。
* ``-Xmn500M `` - 新生代大小
   * 响应时间优先的应用:老年代使用并发收集器,所以其大小需要小心设置,一般要考虑并发会话率和会话持续时间等一些参数.如果堆设置小了,可能会造成内存碎片,高回收频率以及应用暂停而使用传统的标记清除方式;如果设置大了,则需要较长的收集时间。最优化的方案,一般需要参考以下数据获得:并发垃圾收集信息、持久代并发收集次数、传统GC信息、花在新生代和老年代回收上的时间比例来进行设置。
   * 吞吐量优先的应用:一般吞吐量优先的应用都有一个很大的新生代和一个较小的老年代.原因是,这样可以尽可能回收掉大部分短期对象,减少中期的对象,而老年代尽存放长期存活对象。
   * 使用CMS的时候，可以将新生代设置的稍微较小，小于默认的3/8（新生代/堆），然后老年代利用CMS并行收集， 这样能保证系统低延迟的吞吐效率
* ``-XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=0`` - 因为老年代使用的算法是mark-copy，不会对碎片进行处理，通过这两个参数，可以对碎片进行压缩，解决碎片的问题。
   * ``-XX:+UseCMSCompactAtFullCollection`` - 使用并发收集器时,开启对老年代的压缩。  
   * ``-XX:CMSFullGCsBeforeCompaction=0`` - 上面配置开启的情况下,这里设置多少次Full GC后,对老年代进行压缩。默认是0，就是每次Full GC(注意是Full GC而不是Old GC)都会进行内存压缩。
* ``-XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=70`` - 两者配合使用，在老年代使用率达到70%就会触发Old GC。主要是为了避免**[promotion failed](#promotion failed/空间分配担保)（附录中会有详解）**。
   * ``-XX:CMSInitiatingOccupancyFraction=70`` - 这个阀值需要进行计算，采用的公式为``(Xmx-Xmn)*(1-CMSInitiatingOccupancyFraction/100)>=(Xmn-Xmn/(SurvivorRatior+2))``，按着这个公式，我们的参数算出来会是72.2%，所以设置为70。
* ``-XX:+CMSScavengeBeforeRemark`` - 开启或关闭在CMS重新标记阶段之前的清除ygc尝试，目的在于减少old gen对ygc gen的引用。
   * 为什么remark会耗时较大，主要因为有[跨代引用](#跨代引用)，造成耗时较大。
   * 使用这个参数会是一个取舍，主要看性能瓶颈是否在remark上，如果是的话，该参数能在一定程度上缓解这个问题（因为这个参数并不是一定会执行），如果性能瓶颈不是remark，那就不需要开启了，毕竟会多一次ygc。
   * 一般低延时的服务，如果remark时间>1s是无法忍受的，或者要求更高。因为该阶段会造成STW的，暂停所有用户线程，重新扫描堆中的对象，进行可达性分析，标记活着的对象，所以低延时的服务可以考虑加上。
* gc日志相关的参数建议配置上，不然排查起来就是无头苍蝇了。

#### 善用jmap

> 一个好的服务状态，我们会要求FullGC频率尽可能完全杜绝，但是完全杜绝是一个理想状态，所以我们退而求其次，尽量避免它在白天，甚至业务高峰期出现。

我们之前在获取FGC的日志的时候就是使用了``jmap -histo:live pid``，该命令会强制进行FGC。那我们只需要有一个定时，每天在凌晨几点，业务最低峰期的时候运行改命令，结合``-XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=0``，从而优化内存碎片并压缩堆，降低在业务高峰期发生FGC的概率。


### 附录

#### 内存分配与回收策略

##### 对象优先分配Eden区

> 大多数情况下，对象在新生代Eden区中分配。当Eden空间不足，则会发生一次``Young GC``。

##### 大对象直接进入老年代

> 大对象是指需要大量连续内存空间的java对象（典型的如长字符串和数组），当新生代不足以存放大对象，就会跳过新生代直接老年代。

##### 长期存活的对象将进入老年代

> Jvm给每个对象定义了一个对象年龄(Age)计数器。如果对象在Eden出生并经过第一次``Young GC``后仍然存活，并且能被``Survivor``容纳的话，将被移动到Survivor空间中，并且对象年龄设为1。对象在Survivor区中每熬过一次``Young GC``，年龄就增加1岁。当它的年龄增加到一定程度（默认是15岁），将会被晋升到老年代。可以通过``-XX:MaxTenuringThreshold``参数设置晋升老年代的阈值

##### 动态对象年龄判定

> 为了能更好地适应不同程序的内存情况，Jvm并不总是要求对象的年龄必须达到``MaxTenuringThreshold``才能晋升老年代。如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代。

#### promotion failed/空间分配担保

![空间分配担保](https://github.com/7le/7le.github.io/raw/master/image/jvm/space-allocation-guarantee.png)

图中提到了是否允许担保失败，当新生代使用复制收集算法，但是为了内存利用率，只使用其中一个``Survivor``空间来作为轮换备份，因此当出现大量对象在``Young GC``后依然存活（最极端就是内存回收后新生代中所有对象都存活），就需要老年代进行分配担保，把``Survivor``无法容纳的对象直接进入老年代。但是多少会晋升到老年代在实际内存回收前是无法明确知道的，所以只好取之前每一次回收晋升到老年代对象容量的平均值作为经验值，与老年代剩余空间比较，决定是否进行``Full GC``来让老年代腾出更多空间。

#### 跨代引用

> 新生代对象持有老年代中对象的引用，这种情况称为**跨代引用**

![跨代引用](https://github.com/7le/7le.github.io/raw/master/image/jvm/cross-reference.png)

上图中，对象A因为引用存在新生代中，Remark阶段必须扫描整个堆来判断对象是否存活，包括图中灰色的不可达对象。

灰色对象已经不可达，但仍然需要扫描的原因：新生代GC和老年代的GC是各自分开独立进行的，只有YGC时才会使用根搜索算法，标记新生代对象是否可达，也就是说虽然一些对象已经不可达，但在YGC发生前不会被标记为不可达，CMS也无法辨认哪些对象存活，只能全堆扫描（新生代+老年代）。由此可见堆中对象的数目影响了Remark阶段耗时。 

另外考虑到这个扫描（就是上述日志中的CMS-concurrent-preclean）可能会比较耗时，jvm提供了``CMSMaxAbortablePrecleanTime ，默认为5s``，相当于该可中断的预清理执行超过5s，不管是否发生YGC，都会中止此阶段，进入Remark。 

而``-XX:+CMSScavengeBeforeRemark``可以保证Remark前强制进行一次YGC来回收不可达对象，减少remark的暂停时间。

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)
