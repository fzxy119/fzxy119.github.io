---
title: JAVA自带工具命令
date: 2022-07-18 09:46:01
tags:
- jvm
categories:
- 技术
---

# JAVA 自带工具 
1. 监控工具
2. 故障排除工具
3. Java故障排除、分析、监控和管理工具
4. 程方法调用(RMI)工具
5. 国际化工具
6. 基础工具
7. 安全
8. 脚本工具

## 一. 监控工具
+ ### jps

> JVM进程状态工具——列出目标系统上已检测的HotSpot Java虚拟机    jps [ options ] [ hostid ] 

> hostid -> ([protocol:][[//]hostname][:port][/servername])  

> protocol 默认为rmi

> servername 此参数的处理取决于实现。对于优化的本地协议，该字段被忽略。对于rmi协议，该参数是一个字符串，表示远程主机上RMI远程对象的名称。请参见jstatd命令的-n选项

```shell script
# 输出传递给main方法的参数。对于嵌入式JVM，输出可能为空
jps -m

# 输出传递给JVM的参数。
jps -v

# 服务器上启动jstatd 链接远程服务器
jps -v rmi://12.3.10.105:10000

# 输出应用程序主类的完整包名或应用程序JAR文件的完整路径名。
jps -l
```

+ ### jstat 

> jstat [ generalOption | outputOptions vmid [interval[s|ms] [count]] ]

> vmid  =>  [protocol:][//]lvmid[@hostname[:port]/servername]

一些常用的工具

> jstat -gcnew -h 3 21891 250 #年轻代统计

> jstat -gcutil 21891 250 7 #gc统计信息

> jstat -gcutil 40496@remote.domain 1000 #监控远程服务
1. generalOption（常规命令行选项 statOption）

 一个常规命令行选项(-help或-options)  jstat -options
 ```
    -class
    -compiler
    -gc
    -gccapacity
    -gccause
    -gcnew
    -gcnewcapacity
    -gcold
    -gcoldcapacity
    -gcpermcapacity
    -gcutil
    -printcompilation
```
 
2. outputOptions （输出格式选项）
一个或多个输出选项，由单个statOption加上-t、-h和-J选项组成。
```

-h n

每n个样本(输出行)显示一个列标题，其中n是正整数。默认值为0，表示在第一行数据上方显示列标题

如：jstat -gcutil -h 2 24583 1000 10 （每两行输出一下标题）

-t

将时间戳列显示为输出的第一列。时间戳是自目标JVM启动以来的时间。

-JjavaOption
将javaOption传递给java应用程序启动器。例如，-J-Xms48m将启动内存设置为48兆字节。有关选项的完整列表，请参见Java-Java应用程序启动器

```
下表总结了jstat为每个statOption输出的列。
```
-class Option  类加载统计
Column	Description
Loaded	Number of classes loaded.
Bytes	Number of Kbytes loaded.
Unloaded	Number of classes unloaded.
Bytes	Number of Kbytes unloaded.
Time	Time spent performing class load and unload operations.

-compiler Option  HotSpot Just-In-Time Compiler Statistics JIT编译统计
Column	Description
Compiled	Number of compilation tasks performed.
Failed	Number of compilation tasks that failed.
Invalid	Number of compilation tasks that were invalidated.
Time	Time spent performing compilation tasks.
FailedType	Compile type of the last failed compilation.
FailedMethod	Class name and method for the last failed compilation.

-gc Option 垃圾收集的堆统计信息
Column	Description
S0C	Current survivor space 0 capacity (KB).
S1C	Current survivor space 1 capacity (KB).
S0U	Survivor space 0 utilization (KB).
S1U	Survivor space 1 utilization (KB).
EC	Current eden space capacity (KB).
EU	Eden space utilization (KB).
OC	Current old space capacity (KB).
OU	Old space utilization (KB).
PC	Current permanent space capacity (KB).
PU	Permanent space utilization (KB).
YGC	Number of young generation GC Events.
YGCT	Young generation garbage collection time.
FGC	Number of full GC events.
FGCT	Full garbage collection time.
GCT	Total garbage collection time.

-gccapacity Option  各代内存池和空间容量
Column	Description
NGCMN	Minimum new generation capacity (KB).
NGCMX	Maximum new generation capacity (KB).
NGC	Current new generation capacity (KB).
S0C	Current survivor space 0 capacity (KB).
S1C	Current survivor space 1 capacity (KB).
EC	Current eden space capacity (KB).
OGCMN	Minimum old generation capacity (KB).
OGCMX	Maximum old generation capacity (KB).
OGC	Current old generation capacity (KB).
OC	Current old space capacity (KB).
PGCMN	Minimum permanent generation capacity (KB).
PGCMX	Maximum Permanent generation capacity (KB).
PGC	Current Permanent generation capacity (KB).
PC	Current Permanent space capacity (KB).
YGC	Number of Young generation GC Events.
FGC	Number of Full GC Events.

-gccause Option  该选项显示与-gcutil选项相同的垃圾收集统计信息摘要，但包括上次垃圾收集事件的原因和(如果适用)当前垃圾收集事件。除了为-gcutil列出的列之外，该选项还添加了以下列:

Garbage Collection Statistics, Including GC Events
Column	Description
LGCC	Cause of last Garbage Collection.
GCC	Cause of current Garbage Collection.

-gcnew Option 年轻代统计信息
Column	Description
S0C	Current survivor space 0 capacity (KB).
S1C	Current survivor space 1 capacity (KB).
S0U	Survivor space 0 utilization (KB).
S1U	Survivor space 1 utilization (KB).
TT	Tenuring threshold.
MTT	Maximum tenuring threshold.
DSS	Desired survivor size (KB).
EC	Current eden space capacity (KB).
EU	Eden space utilization (KB).
YGC	Number of young generation GC events.
YGCT	Young generation garbage collection time.

-gcnewcapacity Option  年轻代空间统计
Column	Description
NGCMN
Minimum new generation capacity (KB).
NGCMX	Maximum new generation capacity (KB).
NGC	Current new generation capacity (KB).
S0CMX	Maximum survivor space 0 capacity (KB).
S0C	Current survivor space 0 capacity (KB).
S1CMX	Maximum survivor space 1 capacity (KB).
S1C	Current survivor space 1 capacity (KB).
ECMX	Maximum eden space capacity (KB).
EC	Current eden space capacity (KB).
YGC	Number of young generation GC events.
FGC	Number of Full GC Events.

-gcold Option 来年代和永久代统计信息
Column	Description
PC	Current permanent space capacity (KB).
PU	Permanent space utilization (KB).
OC	Current old space capacity (KB).
OU	old space utilization (KB).
YGC	Number of young generation GC events.
FGC	Number of full GC events.
FGCT	Full garbage collection time.
GCT	Total garbage collection time.

-gcoldcapacity Option 老年代统计
Column	Description
OGCMN	Minimum old generation capacity (KB).
OGCMX	Maximum old generation capacity (KB).
OGC	Current old generation capacity (KB).
OC	Current old space capacity (KB).
YGC	Number of young generation GC events.
FGC	Number of full GC events.
FGCT	Full garbage collection time.
GCT	Total garbage collection time.

-gcpermcapacity Option 永久代统计
Column	Description
PGCMN	Minimum permanent generation capacity (KB).
PGCMX	Maximum permanent generation capacity (KB).
PGC	Current permanent generation capacity (KB).
PC	Current permanent space capacity (KB).
YGC	Number of young generation GC events.
FGC	Number of full GC events.
FGCT	Full garbage collection time.
GCT	Total garbage collection time.

-gcutil Option 垃圾收集统计信息摘要
Column	Description
S0	Survivor space 0 utilization as a percentage of the space's current capacity.
S1	Survivor space 1 utilization as a percentage of the space's current capacity.
E	Eden space utilization as a percentage of the space's current capacity.
O	Old space utilization as a percentage of the space's current capacity.
P	Permanent space utilization as a percentage of the space's current capacity.
YGC	Number of young generation GC events.
YGCT	Young generation garbage collection time.
FGC	Number of full GC events.
FGCT	Full garbage collection time.
GCT	Total garbage collection time.

-printcompilation Option HotSpot Compiler Method Statistics 方法统计
Column	Description
Compiled	Number of compilation tasks performed by the most recently compiled method.
Size	Number of bytes of bytecode of the most recently compiled method.
Type	Compilation type of the most recently compiled method.
Method	Class name and method name identifying the most recently compiled method. Class name uses "/" instead of "." as namespace separator. Method name is the method within the given class. The format for these two fields is consistent with the HotSpot - XX:+PrintComplation option.
```


+ ### jstatd

JVM jstat Daemon——启动RMI服务器应用程序，该应用程序监视被检测的HotSpot Java虚拟机的创建和终止，并提供一个接口，允许远程监视工具连接到在本地系统上运行的Java虚拟机。

## 二. 故障排除工具

+ ### jinfo

jinfo打印给定Java进程或核心文件或远程调试服务器的Java配置信息。配置信息包括Java系统属性和Java虚拟机命令行标志。

如果给定的进程在64位虚拟机上运行，您可能需要指定-J-d64选项，例如: jinfo -J-d64 -sysprops pid
```
[root@localhost bin]# ./jinfo -flags 27463
Attaching to process ID 27463, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.172-b11
Non-default VM flags: -XX:CICompilerCount=3 -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=536870912 -XX:MaxNewSize=178782208 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=89128960 -XX:OldSize=179306496 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC 
Command line:  -Xms256m -Xmx512m

#查看某个flag
[root@localhost bin]# ./jinfo -flag CICompilerCount 27463
-XX:CICompilerCount=3

#修改某个配置项值
[root@localhost bin]# ./jinfo -flag CICompilerCount=4 27463
#将配置项修改为false 禁用配置
[root@localhost bin]# ./jinfo -flag -UseParallelGC 27463
#将配置项修改为true 启用配置
[root@localhost bin]# ./jinfo -flag +UseParallelGC 27463

#查看jvm配置信息， 环境配置，系统配置 
[root@localhost bin]# ./jinfo 27463
Attaching to process ID 27463, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.172-b11
Java System Properties:

java.runtime.name = Java(TM) SE Runtime Environment
java.vm.version = 25.172-b11
sun.boot.library.path = /usr/local/src/jdk/jre/lib/amd64
java.vendor.url = http://java.oracle.com/
java.vm.vendor = Oracle Corporation
path.separator = :
file.encoding.pkg = sun.io
java.vm.name = Java HotSpot(TM) 64-Bit Server VM
sun.os.patch.level = unknown
sun.java.launcher = SUN_STANDARD
user.country = CN
user.dir = /usr/local/agentManageAccount-dubbo/releases/20220406-170930
java.vm.specification.name = Java Virtual Machine Specification
java.runtime.version = 1.8.0_172-b11
java.awt.graphicsenv = sun.awt.X11GraphicsEnvironment
os.arch = amd64
java.endorsed.dirs = /usr/local/src/jdk/jre/lib/endorsed
java.io.tmpdir = /tmp
line.separator = 

java.vm.specification.vendor = Oracle Corporation
os.name = Linux
sun.jnu.encoding = UTF-8
java.library.path = /usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
java.specification.name = Java Platform API Specification
java.class.version = 52.0
sun.management.compiler = HotSpot 64-Bit Tiered Compilers
os.version = 2.6.32-431.el6.x86_64
user.home = /root
user.timezone = PRC
java.awt.printerjob = sun.print.PSPrinterJob
file.encoding = UTF-8
java.specification.version = 1.8
user.name = root
java.class.path = agentManageAccount-dubbo.jar
java.vm.specification.version = 1.8
sun.arch.data.model = 64
sun.java.command = agentManageAccount-dubbo.jar
java.home = /usr/local/src/jdk/jre
user.language = zh
java.specification.vendor = Oracle Corporation
awt.toolkit = sun.awt.X11.XToolkit
java.vm.info = mixed mode
java.version = 1.8.0_172
java.ext.dirs = /usr/local/src/jdk/jre/lib/ext:/usr/java/packages/lib/ext
sun.boot.class.path = /usr/local/src/jdk/jre/lib/resources.jar:/usr/local/src/jdk/jre/lib/rt.jar:/usr/local/src/jdk/jre/lib/sunrsasign.jar:/usr/local/src/jdk/jre/lib/jsse.jar:/usr/local/src/jdk/jre/lib/jce.jar:/usr/local/src/jdk/jre/lib/charsets.jar:/usr/local/src/jdk/jre/lib/jfr.jar:/usr/local/src/jdk/jre/classes
java.vendor = Oracle Corporation
file.separator = /
java.vendor.url.bug = http://bugreport.sun.com/bugreport/
sun.io.unicode.encoding = UnicodeLittle
sun.cpu.endian = little
sun.cpu.isalist = 

VM Flags:
Non-default VM flags: -XX:CICompilerCount=3 -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=536870912 -XX:MaxNewSize=178782208 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=89128960 -XX:OldSize=179306496 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC 
Command line:  -Xms256m -Xmx512m
```

+ ### jhat
堆转储浏览器-在堆转储文件(例如，由jmap -dump生成的文件)上启动web服务器，允许浏览堆。

> 使用 jhat file 查看。
>
+ ### jmap
打印给定进程或核心文件或远程调试服务器的共享对象内存映射或堆内存细节。

+ ### jsadebugd

```
jsadebugd pid [ server-id ]
jsadebugd executable core [ server-id ]
```
Java可服务性代理调试守护进程——附加到进程或核心文件，并充当调试服务器。

&nbsp;&nbsp;使用jinfo，jmap，jstack命令不仅可监控本地主机上的JVM进程，也可监控远端主机上的JVM进程，前者称为“本地模式”，后者称为“联网模式”。“联网模式”下的整个监控体系实质上是一个RMI应用程序。在“联网模式”下，一般情况下，远端主机上“运行rmiregistry服务，运行RMI Server”，本地主机上“运行RMI Client”，整个体系中“无需运行Web Server服务（因为所有相关类都可在本地获取）”。

&nbsp;&nbsp;如果rmiregistry没有启动，jsadebugd将在标准(1099)端口内部启动rmiregistry。可以通过向调试服务器发送SIGINT(按Ctrl-C)来停止调试服务器。

&nbsp;&nbsp;在远端主机上，通过运行jsadebugd命令开启RMI Server（需要注意的是，运行jsadebugd命令同时会自动运行rmiregistry服务，而无需手动运行），使用“PID”选项值指定远端主机上待被监控的JVM进程ID，使用“ServerId”选项值作为RMI Server内向rmiregistry服务注册所生成Remote Object实例的名称（如果未指定，则使用默认名称）；在本地主机上，运行jinfo，jmap，jstack命令作为RMI Client

> 在远端主机上，执行jsadebugd -J-Djava.rmi.server.hostname=12.3.10.105 pid serviceID 命令开启RMI Server（同时开启rmiregistry服务），并使用“remoteObject1”名称向rmiregistry服务注册所生成的Remote Object实例

>在本地主机上，执行jinfo/jmap/jstack serviceID@remoteserver.host 命令作为RMI Client与远端主机上的rmiregistry服务和RMI Server进行交互，而达到监控远端主机上相应JVM进程的目标。执行jinfo，jmap，jstack命令时可使用的选项参数可具体参考这些命令的“本地模式”用法

+ ### jstack
打印给定进程或核心文件或远程调试服务器的线程堆栈跟踪。

## 三. Java故障排除、分析、监控和管理工具
+ ###jcmd

JVM诊断命令工具——向正在运行的Java虚拟机发送诊断命令请求。

```
jcmd [ options ]

jcmd [ pid | main-class ] PerfCounter.print

jcmd [ pid | main-class ] command [ arguments ]

jcmd [ pid | main-class ] -f file
```

> ./jcmd -l #打印正在运行的Java进程及其进程id、主类和命令行参数的列表。
> ./jcmd 1055 help #查看进程1055可执行的命令 ，获取结果如下:
```
1055:
The following commands are available:
JFR.stop
JFR.start
JFR.dump
JFR.check
VM.native_memory
VM.check_commercial_features
VM.unlock_commercial_features
ManagementAgent.stop
ManagementAgent.start_local
ManagementAgent.start
VM.classloader_stats
GC.rotate_log
Thread.print
GC.class_stats
GC.class_histogram
GC.heap_dump
GC.finalizer_info
GC.heap_info
GC.run_finalization
GC.run
VM.uptime
VM.dynlibs
VM.flags
VM.system_properties
VM.command_line
VM.version
help
```
> ./jcmd 1055 GC.run #对进程1055 pid 执行GC.run 命令



+ ###jconsole

一个符合JMX标准的图形化工具，用于监控Java虚拟机。它可以监控本地和远程JVM。它还可以监控和管理应用程序。 有关更多信息，请参见Java平台的监控和管理。

+ ###jmc

Java Mission Control (JMC)客户端包括监控和管理Java应用程序的工具，而不会引入通常与这些类型的工具相关的性能开销。

+ ###jvisualvm

一个图形工具，当基于Java技术的应用程序(Java应用程序)在Java虚拟机中运行时，它提供有关这些应用程序的详细信息。Java VisualVM提供内存和CPU分析、堆转储分析、内存泄漏检测、MBeans访问和垃圾收集。有关更多信息，请参见Java VisualVM。



## 四. 程方法调用(RMI)工具

## 五. 国际化工具

+ ###native2ascii

> JAVA_HOME\bin\native2ascii -encoding GBK D:\src\resources.properties D:\classes\resources.properties

native2ascii [options] [inputfile [outputfile]]

options：

-reverse

执行相反的操作:将采用Unicode转义的ISO-8859-1编码的文件转换为Java运行时环境支持的任何字符编码的文件

-encoding encoding_name

指定转换过程要使用的字符编码的名称。如果该选项不存在，则使用默认的字符编码(由Java.nio.charset.charset.default charset方法确定)。encoding_name字符串必须是Java运行时环境支持的字符编码的名称，[请参见支持的编码文档。](https://docs.oracle.com/javase/7/docs/technotes/guides/intl/encoding.doc.html)

-Joption

将option传递给java虚拟机，其中option是Java应用程序启动程序参考页上描述的选项之一。例如，-J-Xms48m将启动内存设置为48兆字节

## 六. 基础工具

## 七. 安全

## 八. 脚本工具