---
title: 'jvm参数配置'
date: 2022-07-25 10:42:36
tags:
- jvm
categories:
- 技术
---

{% asset_img jvmmap.png jvmmap %}

---
Serial垃圾收集器（新生代） 新生代使用Serial  老年代则使用SerialOld     
> 开启：-XX:+UseSerialGC 关闭：-XX:-UseSerialGC

>> SerialOld收集器(串行老年代) 搭配Parallel Scavenge使用  及  CMS发生Concurrent Mode Failure之后的预案

ParNew垃圾收集器（新生代） 新生代使用功能ParNew 老年代则使用功能CMS
> 开启 -XX:+UseParNewGC 关闭 -XX:-UseParNewGC

Parallel Scavenge收集器（新生代） 新生代使用功能Parallel Scavenge 老年代将会使用Parallel Old收集器
> 开启 -XX:+UseParallelGC 关闭 -XX:-UseParallelGC 

---
ParallelOld垃圾收集器（老年代） 新生代使用功能Parallel Scavenge 老年代将会使用Parallel Old收集器
 > 开启 -XX:+UseParallelOldGC 关闭 -XX:-UseParallelOldGC
 
CMS垃圾收集器（老年代）  新生代使用功能ParNew 老年代则使用功能CMS 老年代担保失败 则采用SerialOld收集器串行收集
 > 开启 -XX:+UseConcMarkSweepGC 关闭 -XX:-UseConcMarkSweepGC

 G1垃圾收集器
 > 开启 -XX:+UseG1GC 关闭 -XX:-UseG1GC
---

### 对象的一些配置
 * -XX:InitialTenuringThreshol=7  年轻代对象转换为老年代对象最小年龄值，默认值7
 
 * -XX:MaxTenuringThreshold=15 进入老年代最大的GC年龄 年轻代对象转换为老年代对象最大年龄值，默认值15
 
 > 动态年龄判断：
 >> 虚拟机不是永远要去对象的年龄必须达到MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无需等到MaxTenuringThreshold中要求的年龄
 
 * -XX:PretenureSizeThreshold=1000000 新生代可容纳的最大对象，超过此值直接进入老年代
 
 ### 内存空间的配置
 
  * -Xms=2g：初始堆空间内存 （默认为物理内存的1/64）
  * -Xmx=2g：最大堆空间内存（默认为物理内存的1/4）
  * -Xmn=512m：设置新生代的大小。(初始值及最大值)
  * -Xss=512k：线程堆栈大小
  * -XX:MetaspaceSize ：初始化的Metaspace大小
  * -XX:MaxMetaspaceSize：Metaspace最大值
  * -XX:NewRatio=2：配置新生代与老年代在堆结构的占比 老年代:新生代 = 2:1 ，新生代占整个堆的1/3
  * -XX:SurvivorRatio=8：设置新生代中Eden和S0/S1空间的比例 eden:S = 8:1
  * -XX:MaxTenuringThreshold=15：设置新生代垃圾的最大年龄
  * -XX:+PrintGCDetails：输出详细的GC处理日志
  * 打印gc简要信息：① -XX:+PrintGC   ② -verbose:gc
  * -XX:HandlePromotionFailure=true：是否设置空间分配担保
  * -XX:MaxDirectMemorySize=2g：堆外直内存配置

 > 空间分配担保：
 >> jdk6之前在发生minorGc之前，虚拟机会先检查老年代最大可用连续空间是否大于新生代所有对象总空间，如果这个条件成立，那么MinorGc可以确保安全，如果不成立，虚拟机会查看-XX:HandlePromotionFailure
 >> jdk7及以后只要老年代的连续空间大于新生代对象的总大小或者历次晋升到老年代的对象的平均大小就进行MinorGC，否则FullGC，HandlePromotionFailure 不再使用

 
 ### 并发标记清除收集器相关配置
  * -XX:+UseConcMarkSweepGC ：打开CMS收集器 新生代默认使用ParNewGc, 担保失败使用SerialOld
  
  * -XX:+UseCMSInitiatingOccupancyOnly ： 指定用设定的回收阈值(-XX:CMSInitiatingOccupancyFraction参数的值),如果不指定,JVM仅在第一次使用设定值,后续则会根据运行时采集的数据做自动调整，如果指定了该参数，那么每次JVM都会在到达规定设定值时才进行GC
 
  * -XX:CMSInitiatingOccupancyFraction=75 ：老年代被使用的内存空间的阈值，达到该阈值则触发Full GC 75%
 
  * -XX:CMSFullGCsBeforeCompaction=0 ：由于并发收集器不对内存空间进行压缩、整理，所以运行一段时间以后会产生“碎片”，使得运行效率降低。此值设置运行多少次GC以后对内存空间进行压缩、整理。
 
  * -XX:+UseCMSCompactAtFullCollection ：在FullGc后进行整理压缩操作 打开压缩功能
  
 ### 垃圾收集线程
  * -XX:ParallelGCThreads=20 ：并发收集线程数

 
 ### Parallel 收集器 关注吞吐量

-XX:+UseParallelGC 使用Parallel Scavenge + Serial Old组合收集器  新生代

-XX:+UseParallelOldGC 使用Parallel Scavenge +Parallel Old组合收集器 老年代

-XX:+UseAdaptiveSizePolicy  自适应调节策略 配合 -XX:MaxGCPauseMillis -XX:GCTimeRatio 使用,主要控制新生代收集

-XX:MaxGCPauseMillis=100  控制最大垃圾收集停顿时间 值是一个大于0的毫秒数

-XX:GCTimeRatio=99 直接设置吞吐量大小 大于0且小于100的整数 如果把此参数设置为19，那允许的最大GC时间就占总时间的5%（既1 /（1 + 19）），默认值为99，就是允许最大1%（既 1 /（1 + 99））的垃圾收集时间

      

 
 --- 
 
