---
title: 'jvm参数配置'
date: 2022-07-25 10:42:36
tags:
- jvm
categories:
- 技术
---

{% asset_img jvmmap.png jvmmap %}

Serial垃圾收集器（新生代） 新生代使用Serial  老年代则使用SerialOld(SerialOld收集器(串行老年代) 搭配Parallel Scavenge使用及CMS发生Concurrent Mode Failure之后的预案)
 开启：-XX:+UseSerialGC
 关闭：-XX:-UseSerialGC
 //

ParNew垃圾收集器（新生代） 新生代使用功能ParNew 老年代则使用功能CMS
 开启 -XX:+UseParNewGC
 关闭 -XX:-UseParNewGC

 
Parallel Scavenge收集器（新生代） 新生代使用功能Parallel Scavenge 老年代将会使用Parallel Old收集器
 开启 -XX:+UseParallelGC
 关闭 -XX:-UseParallelGC
 

ParallelOld垃圾收集器（老年代） 新生代使用功能Parallel Scavenge 老年代将会使用Parallel Old收集器
 开启 -XX:+UseParallelOldGC
 关闭 -XX:-UseParallelOldGC
 
CMS垃圾收集器（老年代）  新生代使用功能ParNew 老年代则使用功能CMS
 开启 -XX:+UseConcMarkSweepGC
 关闭 -XX:-UseConcMarkSweepGC

 

 G1垃圾收集器
 开启 -XX:+UseG1GC
 关闭 -XX:-UseG1GC

---
 
# 1、Parallel Scavenge 收集器 关注吞吐量

-XX:+UseParallelGC 使用Parallel Scavenge + Serial Old组合收集器  新生代

-XX:+UseParallelOldGC 使用Parallel Scavenge +Parallel Old组合收集器 老年代

-XX:ParallelGCThreads=16

-XX:+UseAdaptiveSizePolicy  自适应调节策略 配合 -XX:MaxGCPauseMillis -XX:GCTimeRatio 使用,主要控制新生代收集

-XX:MaxGCPauseMillis  控制最大垃圾收集停顿时间 值是一个大于0的毫秒数

-XX:GCTimeRatio 直接设置吞吐量大小 大于0且小于100的整数 如果把此参数设置为19，那允许的最大GC时间就占总时间的5%（既1 /（1 + 19）），默认值为99，就是允许最大1%（既 1 /（1 + 99））的垃圾收集时间

      
# 2、Parallel Old 收集器

-XX:+UseParallelOldGC 使用Parallel Scavenge +Parallel Old组合收集器 老年代

       Parallel Old 收集器是Parallel Scavenge 收集器的老年代版本，使用多线程和“标记-整理”算法。在注重吞吐量以及CPU资源敏感的场合，可以优先考虑Parallel Scavenge 加 Parallel Old 收集器。
---       
       
年轻代对象转换为老年代对象最小年龄值，默认值7，对象在坚持过一次Minor GC之后，年龄就加1，每个对象在坚持过一次Minor GC之后，年龄就增加1
 -XX:InitialTenuringThreshol=7 //

进入老年代最大的GC年龄 年轻代对象转换为老年代对象最大年龄值，默认值15
 -XX:MaxTenuringThreshold=15 //
新生代可容纳的最大对象 
 -XX:PretenureSizeThreshold=1000000 
 
 --- 
 
 -XX:+UseConcMarkSweepGC
 -XX:+UseCMSInitiatingOccupancyOnly
 -XX:CMSInitiatingOccupancyFraction=75