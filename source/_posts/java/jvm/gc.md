---
title: JVM G1
date: 2022-07-21 16:25:53
tags:
- jvm
categories:
- 技术
---
# 内存模型

```

#堆
-Xms/-Xmx

#分区 Region
-XX:G1HeapRegionSize=1048576(1m) #可指定分区大小(1MB~32MB，且必须是2的幂)，默认将整堆划分为2048个分区

#卡片(Card) 和全局卡片 (Global Card Table) 和 记录卡片(Rset)

-XX:+UseG1GC 


-XX:InitialHeapSize=1073741824 
-XX:InitiatingHeapOccupancyPercent=35 
-XX:+ManagementServer 
-XX:MarkStackSize=4194304 
#GC与应用的耗费时间比 G1默认为9(10)，而CMS默认为99(100) CMS的设计原则是耗费在GC上的时间尽可能的少
-XX:GCTimeRatio 
#目标暂停时间 默认200ms
-XX:MaxGCPauseMillis=20 
#最大的堆内存容量
-XX:MaxHeapSize=1073741824

####################################年轻代
#年轻代配置固定大小 (固定值)
-Xmn100m
#年轻代比例 (固定值比例)
-XX:NewRatio=1   
#最大的年轻代容量 (固定值大小 可伸缩)
-XX:NewSize=2m 
#最大的年轻代容量 (固定值大小 可伸缩)
-XX:MaxNewSize=2m  
#整个年轻代内存初始空间 (按比例配置 可伸缩)
-XX:G1NewSizePercent=5
#整个年轻代内存最大空间 (按比例配置 可伸缩)
-XX:G1MaxNewSizePercent=60

#####################垃圾回收
#晋升到老年代的对象任期阈值 
-XX:MaxTenuringThreshold=15 

#老年代
#每轮MixedGC回收的Region的最大比例 CSet对堆的总大小占比 10%
-XX:G1OldCSetRegionThresholdPercent=10

#候选老年代分区的CSet准入条件，活跃度阈值 85% 区域内存活的对象低于此值，那么该区将被纳入Cset,当GC时间过长时候，可适当调整低此值，过了此值就会被GC带走
-XX:G1MixedGCLiveThresholdPercent=85

#混合垃圾收集阈值   当老年代占用空间超过整堆比IHOP阈值(默认45%)时，Occupancy(占用) G1就会启动一次混合垃圾收集周期。
#为了满足暂停目标，G1可能不能一口气将所有的候选分区收集掉，
#因此G1可能会产生连续多次的混合收集与应用线程交替执行，每次STW的混合收集与年轻代收集过程相类似
-XX:InitiatingHeapOccupancyPercent=45

#通过候选老年代分区总数与混合周期最大总次数，确定每次包含到CSet的最小分区数量(确定加入混合Cset的分区数量)；根据堆废物百分比，当收集达到参数低于时5%，不再启动新的混合收集（确定是否MixGc）
#混合周期的最大总次数(默认8)
-XX:G1MixedGCCountTarget=8
#堆废物百分比(默认5%)  垃圾占整个堆的百分比，如果超过5%，将触发多轮的MIXgc
-XX:G1HeapWastePercent=5


-XX:GCLogFileSize=104857600 
-XX:MinHeapDeltaBytes=1048576 
-XX:NumberOfGCLogFiles=10 
-XX:+PrintGC 
-XX:+PrintGCDateStamps 
-XX:+PrintGCDetails 
-XX:+PrintGCTimeStamps 
-XX:+UnlockCommercialFeatures 
-XX:+UseCompressedClassPointers 
-XX:+UseCompressedOops 

-XX:+UseGCLogFileRotation 
```

# 分代模型
分代模型的是将关注点放在最近被分配的对像上，因为大部分的对象是可以很快被回收的，只有少量的对象的生命周期是长的。分代可可以避免整堆扫描，避免长生命周期的对象的拷贝，同事独立收集可以降低响应时间。

## 分代 Generation
G1将内存在逻辑上划分为年轻代和老年代Old，其中年轻代又划分为Eden空间和Survivor空间。但年轻代空间并不是固定不变的，当现有年轻代分区占满时，JVM会分配新的空闲分区加入到年轻代空间

## 本地分配缓冲 Local allocation buffer (TLAB)
解决对象在堆内存分配操作时候的并发情况。

由于分区的思想，每个线程均可以"认领"某个分区用于线程本地的内存分配，而不需要顾及分区是否连续。因此，每个应用线程和GC线程都会独立的使用分区，进而减少同步时间，提升GC效率，这个分区称为本地分配缓冲区(Lab)。

TLAB 应用线程本地缓冲区 应用线程可以独占一个本地缓冲区(TLAB)来创建的对象，而大部分都会落入Eden区域(巨型对象或分配失败除外)，因此TLAB的分区属于Eden空间

GCLAB 每个GC线程同样可以独占一个本地缓冲区(GCLAB)用来转移对象，每次回收会将对象复制到Suvivor空间或老年代空间；对于从Eden/Survivor空间晋升(Promotion)到Survivor/老年代空间的对象，同样有GC独占的本地缓冲区进行操作，该部分称为晋升本地缓冲区(PLAB)。

## 分区模型 

+ ### Region  
G1对内存的使用以分区(Region)为单位，而对对象的分配则以卡片(Card)为单位。

+ ### Card/全局卡片Global Card Table
每个分区内部又被分成了若干个大小为512 Byte卡片(Card)，标识堆内存最小可用粒度所有分区的卡片将会记录在全局卡片表(Global Card Table)中，分配的对象会占用物理上连续的若干个卡片，当查找对分区内对象的引用时便可通过记录卡片来查找该引用对象(见RSet)。每次对内存的回收，都是对指定分区的卡片进行处理

+ ###  记录卡片Rset
> Rset 记录了引用当前分区内对象的引用所在的卡片的索引，帮助确定分区内对象是否存活。

在串行和并行收集器中，GC通过整堆扫描，来确定对象是否处于可达路径中。然而G1为了避免STW式的整堆扫描，在每个分区记录了一个已记忆集合(RSet)，内部类似一个反向指针，记录引用分区内对象的卡片索引。当要回收该分区时，通过扫描分区的RSet，来确定引用本分区内的对象是否存活，进而确定本分区内的对象存活情况。

事实上，并非所有的引用都需要记录在RSet中，如果一个分区确定需要扫描，那么无需RSet也可以无遗漏的得到引用关系。那么引用源自本分区的对象，当然不用落入RSet中；同时，G1 GC每次都会对年轻代进行整体收集，因此引用源自年轻代的对象，也不需要在RSet中记录。最后只有老年代的分区可能会有RSet记录，这些分区称为拥有RSet分区(an RSet’s owning region)

+ ### Per Region Table(PRT)
RSet在内部使用Per Region Table(PRT)记录分区的引用情况。由于RSet的记录要占用分区的空间，如果一个分区非常"受欢迎"(该分区内的对象很多都被别的对象引用)，那么RSet占用的空间会上升，从而降低分区的可用空间。G1应对这个问题采用了改变RSet的密度的方式，在PRT中将会以三种模式记录引用：

1. 稀少：直接记录引用对象的卡片索引
2. 细粒度：记录引用对象的分区索引
3. 粗粒度：只记录引用情况，每个分区对应一个比特位
由上可知，粗粒度的PRT只是记录了引用数量，需要通过整堆扫描才能找出所有引用，因此扫描速度也是最慢的

+ ### 收集集合 CSet

收集集合(CSet)代表每次GC暂停时回收的一系列目标分区（待垃圾回收的分区记录集合表），任意一次收集暂停中，CSet所有分区都会被释放，内部存活的对象都会被转移到分配的空闲分区中。因此无论是年轻代收集，还是混合收集，工作的机制都是一致的。

年轻代收集CSet只容纳年轻代分区，而混合收集会通过启发式算法，在老年代候选回收分区中，筛选出回收收益最高的分区添加到CSet中。候选老年代分区的CSet准入条件，可以通过活跃度阈值-XX:G1MixedGCLiveThresholdPercent(默认85%)进行设置（活跃度在85的分区不进入Cset），从而拦截那些回收开销巨大的对象；

同时，每次混合收集时候可以包含候选老年代分区一同收集，可根据CSet对堆的总大小占比-XX:G1OldCSetRegionThresholdPercent(默认10%)设置数量上限,每轮MixedGC回收的Region的最大比例，最多10% ，控制回收时间，可限制候选区一同收集。

由上述可知，G1的收集都是根据CSet进行操作的，年轻代收集与混合收集没有明显的不同，最大的区别在于两种收集的触发条件。

## 巨型对象 Humongous Region

一个大小达到甚至超过分区大小一半的对象称为巨型对象(Humongous Object)。当线程为巨型分配空间时，不能简单在TLAB进行分配，因为巨型对象的移动成本很高，而且有可能一个分区不能容纳巨型对象。因此，巨型对象会直接在老年代分配，所占用的连续空间称为巨型分区(Humongous Region)。G1内部做了一个优化，一旦发现没有引用指向巨型对象，则可直接在年轻代收集周期中被回收。

巨型对象会独占一个、或多个连续分区，其中第一个分区被标记为开始巨型(StartsHumongous)，相邻连续分区被标记为连续巨型(ContinuesHumongous)。由于无法享受Lab带来的优化，并且确定一片连续的内存空间需要扫描整堆，因此确定巨型对象开始位置的成本非常高，如果可以，应用程序应避免生成巨型对象

## 收集

### 年轻代收集  CSet of Young Collection

应用线程不断活动后，年轻代空间会被逐渐填满。当JVM分配对象到Eden区域失败(Eden区已满)时，便会触发一次STW式的年轻代收集。在年轻代收集中，Eden分区存活的对象将被拷贝到Survivor分区；原有Survivor分区存活的对象，将根据任期阈值(tenuring threshold)分别晋升到PLAB中，新的survivor分区和老年代分区。而原有的年轻代分区将被整体回收掉。

同时，年轻代收集还负责维护对象的年龄(存活次数)，辅助判断老化(tenuring)对象晋升的时候是到Survivor分区还是到老年代分区。年轻代收集首先先将晋升对象尺寸总和、对象年龄信息维护到年龄表中，再根据年龄表、Survivor尺寸、Survivor填充容量-XX:TargetSurvivorRatio(默认50%)、最大任期阈值-XX:MaxTenuringThreshold(默认15)，计算出一个恰当的任期阈值，凡是超过任期阈值的对象都会被晋升到老年代。

### 混合收集集合  CSet of Mixed Collection

年轻代收集不断活动后，老年代的空间也会被逐渐填充。当老年代占用空间超过整堆比IHOP阈值-XX:InitiatingHeapOccupancyPercent(默认45%)时，G1就会启动一次混合垃圾收集周期。为了满足暂停目标，G1可能不能一口气将所有的候选分区收集掉，因此G1可能会产生连续多次的混合收集与应用线程交替执行，每次STW的混合收集与年轻代收集过程相类似。

为了确定包含到年轻代收集集合CSet的老年代分区，JVM通过参数混合周期的最大总次数-XX:G1MixedGCCountTarget(默认8)、堆废物百分比-XX:G1HeapWastePercent(默认5%)。通过候选老年代分区总数与混合周期最大总次数，确定每次包含到CSet的最小分区数量；根据堆废物百分比，当收集达到参数时，不再启动新的混合收集。而每次添加到CSet的分区，则通过计算得到的GC效率进行安排