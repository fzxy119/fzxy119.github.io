---
title: Jprofiler
date: 2022-07-18 16:56:20
tags:
- jvm
categories:
- 技术
---

# 1. 核心组件

JProfiler 包含用于采集目标 JVM 分析数据的 JProfiler agent、用于可视化分析数据的 JProfiler UI、提供各种功能的命令行工具.

它们之间的关系如下图所示:

{% asset_img JPROFILER.png JPROFILER架构图 %}

### JProfiler agent

JProfiler agent 是一个本地库，它可以在 JVM 启动时通过参数-agentpath:<path to native library>进行加载或者在程序运行时通过 JVM Attach 机制进行加载。Agent 被成功加载后，会设置 JVMTI 环境，监听虚拟机产生的事件，如类加载、线程创建等。例如，当它监听到类加载事件后，会给这些类注入用于执行度量操作的字节码。

### JProfiler UI

JProfiler UI 是一个可独立部署的组件，它通过 socket 和 agent 建立连接。这意味着不论目标 JVM 运行在本地还是远端，JProfiler UI 和 agent 间的通信机制都是一样的。

JProfiler UI 的主要功能是展示通过 agent 采集上来的分析数据，此外还可以通过它控制 agent 的采集行为，将快照保存至磁盘，展示保存的快照。

### 命令行工具

JProfiler 提供了一系列命令行工具以实现不同的功能。

jpcontroller - 用于控制 agent 的采集行为。它通过 agent 注册的 JProfiler MBean 向 agent 传递命令。

jpenable - 用于将 agent 加载到一个正在运行的 JVM 上。

jpdump - 用于获取正在运行的 JVM 的堆快照。

jpexport & jpcompare - 用于从保存的快照中提取数据并创建 HTML 报告。

要想对一个JVM进行分析，JProfiler的分析代理必须被加载到该JVM中。这可以通过两种不同的方式实现： 通过在启动脚本中指定一个-agentpath VM参数，或者通过Attach API将代理加载到已经运行的JVM中

两种模式JProfiler都支持。添加VM参数是分析的首选方式，集成向导、IDE插件和从JProfiler内启动JVM的会话配置都使用这种方式。 Attach既可在本地使用，也可通过SSH远程使用。


# 2. JProfiler 设置

## 数据采集模式

JProfier 提供两种数据采集模式 Sampling 和 Instrumentation。

1. Sampling（采样） - 适合于不要求数据完全精确的场景。优点是对系统性能的影响较小，缺点是某些特性不支持（如方法级别的统计信息）

2. Instrumentation（仪表） - 完整功能模式，统计信息也是精确的。缺点是如果需要分析的类比较多，对应用性能影响较大。为了降低影响，往往需要和 Filter 一起使用

## 应用启动模式

通过为 JProfiler agent 指定不同的参数可以控制应用的启动模式。

1. 等待模式 - 只有在 Jprofiler GUI 和 agent 建立连接并完成分析配置设置后，应用才会真正启动。在这种模式下，您能够获取应用启动时期的分析数据。对应的命令为-agentpath:<path to native library>=port=8849

2. 立即启动模式 - 应用会立即启动，Jprofiler GUI 会在需要时和 agent 建立连接并设置分析配置。这种模式相对灵活，但会丢失应用启动初期的分析数据。对应的命令为-agentpath:<path to native library>=port=8849,nowait。


# 3. -agentpath VM参数 启动profiler agent

-agentpath是JVM提供的一个通用VM参数，用于加载任何一种使用JVMTI接口的本地库。 由于分析接口JVMTI是一个本地接口，所以分析代理必须是一个本地库。这意味着你只能在 明确支持的平台上进行分析。 32位和64位JVM也需要不同的本地库。另外，Java代理是用-javaagent VM参数加载的，并且只能访问有限的功能。

在-agentpath: 后面附加本地库的完整路径名。有一个等效参数-agentlib:， 你只需要指定特定平台的库名，但必须确保库路径中包含该库。在库的路径之后，你可以添加一个等号，并向代理传递选项，用逗号隔开。例如，在Linux上，整个参数看起来可能像这样：

> -agentpath:/opt/jprofiler10/bin/linux-x64/libjprofilerti.so=port=8849,nowait

```
# jprofiler agent配置
nohup java -agentpath:/usr/local/src/jprofiler12.0.4/bin/linux-x64/libjprofilerti.so=port=8849,nowait -jar -Xms256m -Xmx256m  ./$RESOURCE_NAME id.properties  >/dev/null 2>&1  &
```

第一个等号将路径名与参数分开，第二个等号是参数port=8849 的一部分。 这个常用参数定义了分析代理监听来自JProfiler GUI连接的端口，实际上8849是默认端口，所以也可以省略这个参数。 如果你想在同一台机器上分析多个JVM，必须分配不同端口， 不过IDE插件和本地启动的会话会自动分配这个端口，对于集成向导，你必须显式选择端口。

第二个参数nowait告诉分析代理不要在启动时为了等待JProfiler GUI连接而阻塞JVM。 启动时的阻塞是默认的，因为分析代理不是以命令行参数的形式接收其分析设置，而是从JProfiler GUI或配置文件接收。 命令行参数仅用于引导分析代理，告诉它如何启动和传递调试标记。

默认情况下，JProfiler代理将通信Socket绑定到所有可用的网络接口。如果出于安全原因不希望这样做， 你可以添加选项address=[IP地址]以便选择指定接口，或loopback 只监听来自本地机器的请求。对于通过JProfiler UI启动的或通过IDE集成的JVM，会自动添加loopback



# 4. jpenable 将 agent 加载到一个正在运行的 JVM 上

1. 服务器运行 jpenable
```shell script

[root@localhost bin]# ./jpenable 
选择一个 JVM:
org.apache.catalina.startup.Bootstrap start [6221] [1]
org.apache.catalina.startup.Bootstrap start [9791] [2]
org.elasticsearch.bootstrap.Elasticsearch -d [11417] [3]
agentManageInternetCard-dubbo.jar [11778] [4]
agentManageKafka-dubbo.jar [19613] [5]
org.apache.catalina.startup.Bootstrap start [20276] [6]
org.apache.catalina.startup.Bootstrap start [22974] [7]
./agentManageJob.jar [24583] [8]
./account-Job-1.0-SNAPSHOT.jar /us...project/job/id.properties [26293] [9]
agentManageJobOrder-dubbo.jar [26438] [10]
sun.tools.jstatd.Jstatd -p 10000 [27456] [11]
agentManageAccount-dubbo.jar [27463] [12]

8  # 选择添加agnet的进程

请选择分析模式:
GUI模式(与 JProfiler GUI Attach) [1, 回车(Enter)]
离线模式(使用配置文件去设置分析设置) [2]
1

请输入分析端口
[39923]
1234
你现在可以使用 JProfiler GUI 连接端口 1234

```

2. Jprofiler ui Attach 到远程agnet

{% asset_img jenable.png jenable %}

[参考文档](https://zhuanlan.zhihu.com/p/54274894)

[官方帮助文档](https://www.ej-technologies.com/resources/jprofiler/help_zh_CN/doc/main/architecture.html)