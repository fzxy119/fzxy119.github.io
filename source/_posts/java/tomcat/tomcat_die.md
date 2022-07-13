---
title: 一次tomcat假死问题排查
date: 2022-07-13 10:00:34
tags: 
- tomcat
categories:
- 技术
---

#Tomcat 假死问题排查思路

### Jvm内存问题及线程问题排查


1. JVM内存使用情况查看 
> jmap -heap pid

> jmap -dump:live,format=b,file=/home/xxxx.hprof pid

> jmap -histo:live 27151

> jmap -histo 18230 | sort -n -r -k 2 | head -10  //统计实例最多的类 前十位有哪些

> jmap -histo 18230 | sort -n -r -k 3 | head -10  //统计合计容量前十的类有哪些  

> jstat -gcutil pid 1000 3 //监控gc 状态

> jinfo -flags pid // 查看tomcatJVM配置参数


2. 线程栈问题提取 检查是否有锁问题
> ./jstack pid |grep http-nio-8090-exec （请求处理线程，可以检查处理线程卡在了哪里）
> ./jstack -l pid
>>打印关于锁的其他信息，比如拥有的java.util.concurrent ownable同步器的列表。

> jstack -F pid
>>当 jstack [-l] pid 没有响应时，强制打印一个堆栈转储。
>
3. 查看tomcat进程内线程

> ps -Lf --pid 16139 |wc -l
>>  统计该tomcat进程内的线程个数 

> pstree -p 16139 |wc -l
>> 统计该tomcat进程内的线程个数 

4. 查看进程打开的TCP链接的连接数
> netstat -anpt|grep pid  -c

5. 查看端口的TCP连接数
>   netstat -anpt|grep port -c


### Tomcat相关配置
1. catalina.sh 相关配置
```properties
JAVA_OPTS="-server 
-Dfile.encoding=UTF-8 
-Xms2048m 
-Xmx2048m
-Xloggc:/home/tomcat/images/gclogs/cms-gc-150.log
-verbose:gc -XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+UseGCLogFileRotation
-XX:+PrintGCApplicationStoppedTime
-XX:NumberOfGCLogFiles=10
-XX:GCLogFileSize=100M
-Dlog4j2.formatMsgNoLookups=true
-XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFraction=75
-XX:+UseCMSInitiatingOccupancyOnly
"
```


2. 垃圾回收器
```java
-XX:+UseConcMarkSweepGC
-XX:CMSInitiatingOccupancyFraction=75
-XX:+UseCMSInitiatingOccupancyOnly 
```

-XX:+UseConcMarkSweepGC 
表示年轻代使用 ParNew，老年代的用 CMS 垃圾回收器

-XX:CMSInitiatingOccupancyFraction=75 

由于 CMS 在执行过程中，用户线程还需要运行，那就需要保证有充足的内存空间供用户使用。
如果等到老年代空间快满了，再开启这个回收过程，用户线程可能会产生“Concurrent Mode Failure”的错误，这时会临时启用 Serial Old 收集器来重新进行老年代的垃圾收集，
这样停顿时间就很长了（STW）。这部分空间预留，一般在 30% 左右即可，那么能用的大概只有 70%。参数 -XX:CMSInitiatingOccupancyFraction 用来配置这个比例，但它首先必须配置 -XX:+UseCMSInitiatingOccupancyOnly 参数。

-XX:+UseCMSInitiatingOccupancyOnly


3. TomcatConnector

[官网介绍](https://tomcat.apache.org/tomcat-7.0-doc/config/http.html#Introduction)

```xml
<Connector port="8081" 
               protocol="org.apache.coyote.http11.Http11NioProtocol"
               connectionTimeout="20000"
               keepAliveTimeout="20000"
               acceptorThreadCount="1"
               maxThreads="360"
               maxConnections="1000"
               acceptCount="200"       
               redirectPort="8445" 
               URIEncoding="UTF-8" />

```
>  <Connector port="8081" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8445" URIEncoding="UTF-8" />

+  protocol：

org.apache.coyote.http11.Http11Protocol - blocking Java connector <br/>
BIO：<br/>
一个线程处理一个请求。缺点：并发量高时，线程数较多，浪费资源。<br/>
Tomcat7或以下，在Linux系统中默认使用这种方式。

org.apache.coyote.http11.Http11NioProtocol - non blocking Java connector<br/>
NIO：<br/>
利用Java的异步IO处理，可以通过少量的线程处理大量的请求。<br/>
Tomcat8在Linux系统中默认使用这种方式。<br/>
Tomcat7必须修改Connector配置来启动：<br/>
<Connector port="8080" protocol="org.apache.coyote.http11.Http11NioProtocol" connectionTimeout="20000" redirectPort="8443"/> 

org.apache.coyote.http11.Http11AprProtocol - the APR/native connector.<br/>
APR：<br/>
即Apache Portable Runtime，从操作系统层面解决io阻塞问题。<br/>
Tomcat7或Tomcat8在Win7或以上的系统中启动默认使用这种方式。<br/>
Linux如果安装了apr和native，Tomcat直接启动就支持apr。（安装方法：http://www.cnblogs.com/nb-blog/p/5278502.html）

+ URIEncoding：<br/>
This specifies the character encoding used to decode the URI bytes, after %xx decoding the URL. If not specified, ISO-8859-1 will be used.

+ acceptorThreadCount：<br/>
用于接受连接的线程数。在多CPU的机器上增加这个值，虽然你永远不会真的需要超过2。此外，对于许多非保持活动状态的连接，您可能也想增加该值。默认值为1。


+ connectionLinger<br/>
此连接器使用的套接字在关闭时停留的秒数。默认值为-1，表示禁用套接字逗留

+ connectionUploadTimeout<br/>
指定数据上载过程中使用的超时时间(以毫秒为单位)。这仅在disableUploadTimeout设置为false时生效

+ disableUploadTimeout<br/>
该标志允许servlet容器在数据上传期间使用不同的、通常更长的连接超时。如果未指定，此属性将设置为true，从而禁用更长的超时

+ keepAliveTimeout<br/>
此连接器在关闭连接之前等待另一个HTTP请求的毫秒数。默认值是使用为connectionTimeout属性设置的值。使用值-1表示没有(即无限)超时

+ maxConnections<br/>
服务器在任何给定时间接受和处理的最大连接数。当达到这个数目时，服务器将接受，但不处理，一个进一步的连接。这个额外的连接将被阻塞，直到被处理的连接数低于最大连接数，
此时服务器将再次开始接受和处理新的连接。请注意，一旦达到限制，操作系统可能仍然会根据acceptCount设置接受连接。默认值因连接器类型而异。
对于BIO，默认值是maxThreads的值，除非使用了执行器，在这种情况下，默认值将是来自执行器的maxThreads的值。对于NIO，默认值为10000。对于APR/native，默认值为8192。 
仅对于NIO，将该值设置为-1将禁用maxConnections功能，并且不会计算连接数。

+ maxThreads<br/>
此连接器要创建的最大请求处理线程数，因此决定了可以处理的最大并发请求数。如果未指定，该属性将设置为200。如果一个执行器与这个连接器相关联，那么这个属性将被忽略，
因为连接器将使用执行器而不是内部线程池来执行任务。
注意，如果配置了executor，为该属性设置的任何值都将被正确记录，但它将被报告(例如通过JMX)为-1，以表明它未被使用。

+ acceptCount<br/>
当所有可能的请求处理线程都在使用中时，传入连接请求的最大队列长度。队列已满时收到的任何请求都将被拒绝。默认值为100
