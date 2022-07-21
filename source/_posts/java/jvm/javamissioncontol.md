---
title: JMC(Java Mission Control)
date: 2022-07-19 13:55:31
tags:
- jvm
categories:
- 技术
---

# JMC 客户端使用JMX 链接到JVM 

一个是JMX端口（需要指定），JMX远程连接端口。
一个是RMI端口（默认随机），实际通信用的端口。
一个是本地服务端口（随机），用于本地jstat、jconsole连接用，本地使用

```
JAVA_OPTS="-Dcom.sun.management.jmxremote=true
-Djava.rmi.server.hostname=10.3.60.150
-Dcom.sun.management.jmxremote.port=10000  
-Dcom.sun.management.jmxremote.rmi.port=10001
-Dcom.sun.management.jmxremote.ssl=false    
-Dcom.sun.management.jmxremote.authenticate=true 
-Dcom.sun.management.jmxremote.password.file=/root/jmx/jmxremote.password
-Dcom.sun.management.jmxremote.access.file=/root/jmx/jmxremote.access
-XX:+UnlockCommercialFeatures
-XX:+FlightRecorder
-XX:StartFlightRecording=duration=60s,filename=150-cms-startFlight.jfr
-XX:FlightRecorderOptions=defaultrecording=true,dumponexit=true,dumponexitpath=150-cms-onexit.jfr"
```


+ 控制密码文件有限传阅
chmode 600 password.file
chmode 600 access.file

# JVM 使用命令行启动飞行器

解锁商业功能:UnlockCommercialFeatures  使JFR可用:FlightRecorder  短暂监控 StartFlightRecording 启动飞行器，持续时间60s 并保持到文件myrecording.jfr

-XX:+|-UnlockCommercialFeatures

-XX:+|-FlightRecorder

-XX:FlightRecorderOptions=defaultrecording=true,dumponexit=true,dumponexitpath=onexit.jfr,loglevel=trace

-XX:StartFlightRecording=duration=60s

```
java -XX:+UnlockCommercialFeatures -XX:+FlightRecorder -XX:StartFlightRecording=duration=60s,filename=startFlight.jfr  -XX:FlightRecorderOptions=defaultrecording=true,dumponexit=true,dumponexitpath=onexit.jfr,loglevel=trace MyApp
```

在 JVM 启动参数中加入-XX:+StartFlightRecording=参数启动记录事件。参数为一个以逗号间隔的键值对数据。下面将介绍这些选项：

name=name
指定记录的名称

defaultrecording=true/false
是否在初始化时启动记录事件，默认为 false，对于反应分析，应设置为 true

settings=path
JFR 配置文件的路径

delay=time
开始记录前的延时

duration=time
记录持续时间

filename=path
保存记录事件的文件路径

compress=true/false
是否使用 gzip 压缩记录数据，默认为 false

maxage=time
环形缓存中保存记录的数据的最长时间

maxsize=size
环形缓存占用的最大空间



# 使用动态命令
[官方参考文档](https://docs.oracle.com/javacomponents/jmc-5-4/jfr-runtime-guide/run.htm#JFRUH164)
### 启动飞行器

```

jcmd process_id JFR.start [option_list]

jcmd pid JFR.start duration=60s filename=myrecording.jfr

```

查看pid jvm 可执行的命令
```shell script
jcmd  pid  help
```

1. JFR.start

启动飞行器

2. JFR.check

检查为指定进程运行的所有录制的状态，包括录制标识号、文件名、持续时间等

3. JFR.stop

停止带有特定标识号的飞行器(默认情况下，停止飞行器1)。

4. JFR.dump

转储到目前为止由具有特定标识号的记录收集的数据(默认情况下，转储来自记录1的数据)。



## Access denied! Invalid access level for requested MBeanServer operation.

配置 JRE_HOME/lib/management/jmxremote.access  ***com.sun.management.****

monitorRole   readonly
controlRole   readwrite \
              create javax.management.monitor.*,javax.management.timer.*,com.oracle.jrockit.*,com.sun.management.* \
              unregister
