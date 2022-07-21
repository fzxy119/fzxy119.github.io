---
title: JVM监控工具
date: 2022-07-14 21:00:46
tags:
- jvm
categories:
- 技术
---
# JVM运行监控

1. java自带工具命令
2. Eclipse Memory Analyzer
3. JMX Jstatd - Jconsole VisualVm
4. Btrace 
5. 火焰图
6. Flight Recorder  JavaMissionControl
7. JProfiler

> 监控整个集群的各项资源的使用情况以及各个服务的存活情况

> 监控代码问题导致的线程死锁，OOM


# JMX Jstatd 服务及Jconsole VisualVm
1. JMX:使用JMX需要远程JVM在启动的时候开启远程访问支持，设定JMX端口等，每一个JMX连接一个远程JVM。

2. JStatD:使用jstatd连接方式时，需要在远程主机上创建安全策略文件然后启动jstatd进程，并且此进程需要一直保持运行状态,**客户端可以看到远程主机上当前用户的所有JVM的信息**，即只要创建一个jstatd连接

## JMX方式
JMX，是Java Management Extensions(Java管理扩展)的缩写，是一个为应用程序植入管理功能的框架。用户可以在任何Java应用程序中使用这些代理和服务实现管理（Mbean）

[管方说明](https://docs.oracle.com/javase/7/docs/technotes/guides/management/agent.html#:~:text=Using%20File-Based%20Password%20Authentication%201%20Copy%20the%20password,and%20rename%20it%20to%20jmxremote.password.%20More%20items...%20)

{% asset_img jmxstracture.jpg JMX架构 %}

### 配置信息
|配置|功能|默认值|
|:---|:---|:---|
|-Dcom.sun.management.jmxremote=true|相关 JMX 代理侦听开关|true / false. Default is true.|
|-Djava.rmi.server.hostname|服务器端的IP|服务ip|
|-Dcom.sun.management.jmxremote.rmi.port=1000|rmi端口|默认随机|
|-Dcom.sun.management.jmxremote.port=29094|相关 JMX 代理侦听请求的端口|Port number. No default.|
|-Dcom.sun.management.jmxremote.authenticate=false|指定是否需要密码验证|
|-Dcom.sun.management.jmxremote.password.file=JRE_HOME/lib/management/jmxremote.password | 密码认证文件，可自行指定，不指定在默认位置 | JRE_HOME/lib/management/ jmxremote.password|
|-Dcom.sun.management.jmxremote.access.file=JRE_HOME/lib/management/jmxremote.access|权限控制文件，可自行指定，不指定在默认位置|JRE_HOME/lib/management/jmxremote.access|
|-Dcom.sun.management.jmxremote.ssl | 指定是否使用 SSL 通讯 | true / false. Default is true.|
|-Dcom.sun.management.jmxremote.registry.ssl | Binds the RMI connector stub to an RMI registry protected by SSL. |true / false. Default is false.|
|-Dcom.sun.management.jmxremote.ssl.enabled.protocols | A comma-delimited list of SSL/TLS protocol versions to enable. Used in conjunction with com.sun.management.jmxremote.ssl. | Default SSL/TLS protocol version. |
|-Dcom.sun.management.jmxremote.ssl.enabled.cipher.suitese | A comma-delimited list of SSL/TLS cipher suites to enable. Used in conjunction with com.sun.management.jmxremote.ssl. | Default SSL/TLS cipher suites. |
|-Dcom.sun.management.jmxremote.ssl.need.client.auth | If this property is true and the property com.sun.management.jmxremote.ssl is also true, then client authentication will be performed. It is recommended that you set this property to true. | true / false. Default is false. |
|-Dcom.sun.management.jmxremote.login.config | Specifies the name of a Java Authentication and Authorization Service (JAAS) login configuration entry to use when the JMX agent authenticates users. When using this property to override the default login configuration, the named configuration entry must be in a file that is loaded by JAAS. In addition, the login modules specified in the configuration should use the name and password callbacks to acquire the user's credentials. For more information, see the API documentation for javax.security.auth.callback.NameCallback and javax.security.auth.callback.PasswordCallback. | Default login configuration is a file-based password authentication. |


```shell script
JAVA_OPTS="
$JAVA_OPTS 
-Djava.rmi.server.hostname=127.0.0.1
-Dcom.sun.management.jmxremote.port=10000       
-Dcom.sun.management.jmxremote.ssl=false    # Disabling SSL
-Dcom.sun.management.jmxremote.authenticate=true # Disabling Security
"
```

远程监控和管理需要安全性，以确保未经授权的人无法控制或监控您的应用程序。默认情况下，启用安全套接字层(SSL)和传输层安全性(TLS)上的密码身份验证。您可以分别禁用密码验证和SSL	
	
###   密码认证

JMX支持多种认证方式 使用LDAP认证，基于文件的密码认证，也可以禁用密码认证

##### 禁用密码及禁用ssl

```shell script
-Dcom.sun.management.jmxremote.ssl=false    # Disabling SSL
-Dcom.sun.management.jmxremote.authenticate=false # Disabling Security
```

##### 基于文件的密码认证

**环境类型**
1. 单用户环境

To Set up a Single-User Environment

You set up the password file in the JRE_HOME/lib/management directory as follows.

Copy the password template file, jmxremote.password.template, to jmxremote.password.

Set file permissions so that only the owner can read and write the password file.

Add passwords for roles such as monitorRole and controlRole.

2. 多用户环境
To Set up a Multiple-User Environment

You set up the password file in the JRE_HOME/lib/management directory as follows.

Copy the password template file, jmxremote.password.template, to your home directory and rename it to jmxremote.password.

Set file permissions so that only you can read and write the password file.

Add passwords for the roles such as monitorRole and controlRole.

Set the following system property when you start the Java VM.

com.sun.management.jmxremote.password.file=pwFilePath

In the above property, pwFilePath is the path to the password file.

**文件配置**
密码文件定义了不同的角色及其密码。访问控制文件(默认为jmxremote.access)定义了每个角色允许的访问权限。要发挥作用，角色必须在密码和访问文件中都有一个条目。

1. 密码文件 

  + 模板文件位置 jdk/jre/lib/management/jmxremote.password.template

  + 拷贝template文件到 JRE_HOME/lib/management/jmxremote.password 或者 你的用户目录, 为文件里的角色添加密码

```
# specify actual password instead of the text password
monitorRole password
controlRole password
```

```
chmod 600 jmxremote.password
```

***您必须确保只有所有者拥有该文件的读写权限，因为它包含明文形式的密码。出于安全原因，系统会检查文件是否只对所有者可读，如果不可读，则返回错误退出。因此，在多用户环境中，您应该将密码文件存储在私有位置，如您的主目录***

2. 访问控制文件

 + 访问文件定义了角色及其访问级别。默认情况下，access文件定义了以下两个主要角色。

    monitorRole, 它授予监视的只读访问权限

    controlRole, 它授予监视和管理的读写权限
 + 访问控制条目由角色名称和关联的访问级别组成。角色名不能包含空格或制表符，并且必须对应于密码文件中的条目。访问级别可以是下列之一。

    readonly,   它授予读取MBean属性的权限。对于监控，这意味着这个角色的远程客户端可以读取测量值，但是不能执行任何改变运行程序环境的操作。远程客户端也可以监听MBean通知    

    readwrite, 它授予读取和写入MBean属性、调用对它们的操作以及创建或删除它们的权限。这种访问应该只授予受信任的客户端，因为它们可能会干扰应用程序的操作。
    
```
# The "monitorRole" role has readonly access.
# The "controlRole" role has readwrite access.
monitorRole readonly
controlRole readwrite
```

***默认情况下，access文件命名为jmxremote.access。属性名是来自与密码文件相同空间的标识。关联的值必须是readonly或readwrite。***


> 注意:

>可以在management.properties文件中指定 com.sun.management.jmxremote.* 属性，而不是在命令行中传递它们。
>在这种情况下，需要系统属性-Dcom.sun.management.config.file=management.properties来指定management . properties文件的位置



## Jstatd 方式

jstatd工具是一个RMI服务器应用程序，它监视被检测的HotSpot Java虚拟机(JVM)的创建和终止，并提供一个接口，允许远程监视工具连接到在本地主机上运行的JVM

jstatd服务器要求本地主机上存在RMI注册表。jstatd服务器将尝试在默认端口或-p port选项指示的端口上连接到RMI注册表。如果没有找到RMI注册中心，将在jstatd应用程序中创建一个，绑定到-p port选项指定的端口，或者如果省略-p port，则绑定到默认的RMI注册中心端口。可以通过指定-nr选项来禁止创建内部RMI注册表。

> 注册表其实不用写任何代码，在你的JAVA_HOME下bin目录下有一个rmiregistry程序，需要在你的程序的classpath下运行该程序

### 配置项

+ -nr

如果没有找到现有的RMI注册中心，不要试图在jstatd进程中创建内部RMI注册中心

+ -p  port

期望在其中找到RMI注册表的端口号，或者，如果没有找到，则在没有指定-nr的情况下创建。

+ -n  rminame

远程RMI对象在RMI注册表中绑定到的名称。默认名称为JStatRemoteHost。如果在同一台主机上启动了多个jstatd服务器，那么通过指定这个选项，可以使每个服务器的导出RMI对象的名称是唯一的。但是，这样做需要在监控客户端的hostid和vmid字符串中包含唯一的服务器名称

+ -Joption

将选项传递给javac调用的java启动器。例如，-J-Xms48m将启动内存设置为48兆字节。对于-J来说，将选项传递给执行用Java编写的应用程序的底层VM是一个常见的约定。

### 安全配置

jstatd服务器只能监视它拥有适当的本地访问权限的JVM。因此，jstatd进程必须使用与目标JVM相同的用户凭证运行。一些用户凭证，例如基于UNIX的系统中的root用户，有权访问系统上任何JVM导出的工具。使用这种凭证运行的jstatd进程可以监控系统上的任何JVM，但是会引入额外的安全问题。

jstatd服务器不提供任何远程客户端的身份验证。因此，运行jstatd服务器进程会向网络上的任何用户公开jstatd进程对其拥有访问权限的所有JVM的检测导出。这种暴露在您的环境中可能是不希望的，在启动jstatd进程之前应该考虑本地安全策略，尤其是在生产环境或不安全的网络中

如果没有安装其他安全管理器，jstatd服务器将安装RMISecurityPolicy的实例，因此需要指定安全策略文件。策略文件必须符合默认策略实现的策略文件语法

以下策略文件将允许jstatd服务器在没有任何安全异常的情况下运行。与授予所有代码库所有权限相比，此策略不够自由，但比授予运行jstatd服务器的最低权限的策略更自由。

+ jstatd -J-Djava.security.policy=jstatd.all.policy

+ 文件位置  jdk/bin/jstatd.all.policy

```
grant codebase "file:${java.home}/../lib/tools.jar" {
   permission java.security.AllPermission;
};
```

### 服务启动

+ 使用内部RMI注册表

```
jstatd -J-Djava.security.policy=all.policy
```

+ 使用外部注册表

```shell script
rmiregistry&
jstatd -J-Djava.security.policy=all.policy

# 这个例子演示了在端口2020上用外部RMI注册服务器启动jstatd。
rmiregistry 2020&
jstatd -J-Djava.security.policy=all.policy -p 2020

# 这个例子演示了在端口2020上用一个外部RMI注册表启动jstatd，绑定到名为AlternateJstatdServerName。
rmiregistry 2020&
jstatd -J-Djava.security.policy=all.policy -p 2020 -n AlternateJstatdServerName

# 这个例子演示了如何启动jstatd，如果没有找到RMI注册表，它就不会创建RMI注册表。这个例子假设RMI注册表已经在运行。如果不是，则会发出适当的错误消息。
jstatd -J-Djava.security.policy=all.policy -nr

#这个例子演示了在启用RMI日志功能的情况下启动jstatd。这项技术有助于故障排除或监控服务器活动
jstatd -J-Djava.security.policy=all.policy -J-Djava.rmi.server.logCalls=true

```

	
	


