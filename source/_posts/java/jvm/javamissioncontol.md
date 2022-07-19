---
title: JMC(Java Mission Control)
date: 2022-07-19 13:55:31
tags:
- jvm
categories:
- 技术
---

# JMC 客户端使用JMX 链接到JVM 

-Dcom.sun.management.jmxremote=true
-Djava.rmi.server.hostname=127.0.0.1
-Dcom.sun.management.jmxremote.port=10000       
-Dcom.sun.management.jmxremote.ssl=false    # Disabling SSL
-Dcom.sun.management.jmxremote.authenticate=true # Disabling Security
-Dcom.sun.management.jmxremote.password.file=JRE_HOME/lib/management/jmxremote.password
-Dcom.sun.management.jmxremote.access.file=JRE_HOME/lib/management/jmxremote.access

# JVM 使用命令行启动飞行器

UnlockCommercialFeatures 解锁商业功能 FlightRecorder 使JFR可用 ，短暂监控 StartFlightRecording 启动飞行器，持续时间60s 并保持到文件myrecording.jfr

```
java -XX:+UnlockCommercialFeatures -XX:+FlightRecorder -XX:StartFlightRecording=duration=10min,filename=myrecording.jfr -XX:FlightRecorderOptions=defaultrecording=true MyApp
```



# 使用动态命令
[官方参考文档](https://docs.oracle.com/javacomponents/jmc-5-4/jfr-runtime-guide/run.htm#JFRUH164)
启动飞行器

jcmd pid JFR.start duration=60s filename=myrecording.jfr

查看pid jvm 可执行的命令

jcmd  pid  help


1. JFR.start

启动飞行器

2. JFR.check

检查为指定进程运行的所有录制的状态，包括录制标识号、文件名、持续时间等

3. JFR.stop

停止带有特定标识号的飞行器(默认情况下，停止飞行器1)。

4. JFR.dump

转储到目前为止由具有特定标识号的记录收集的数据(默认情况下，转储来自记录1的数据)。

要配置JFR，您可以使用-XX:FlightRecorderOptions选项


```
-XX:FlightRecorderOptions=defaultrecording=true,dumponexit=true,dumponexitpath=path,maxsize=sizek/K m/M g/G,maxage=age s/m/h/d,delay=delay  s/m/h/d
```



 

## Access denied! Invalid access level for requested MBeanServer operation.

配置 JRE_HOME/lib/management/jmxremote.access  ***com.sun.management.****

monitorRole   readonly
controlRole   readwrite \
              create javax.management.monitor.*,javax.management.timer.*,com.oracle.jrockit.*,com.sun.management.* \
              unregister
