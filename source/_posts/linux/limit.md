---
title: Linux limit
date: 2022-07-14 11:47:51
tags:
- linux
categories:
- 技术
---
# Linux limit 系统资源限制相关配置信息及其含义介绍

## 资源限制范围

资源限制分为系统级别和用户界别

1. /etc/sysctl.conf 文件用于设置系统范围的资源限制

2. /etc/security/limits.conf 为 Oracle、MariaDB 和 Apache 等特定用户设置资源限制


## sysctl 介绍

sysctl 用来设置系统的核心参数

```
-a 显示当前所有设置
-A 以表格的形式显示
-e 模糊模式
-n 忽略关键词
-w 设置变量值
-p<文件> 指定配置文件
-h 显示帮助信息
-V 版本信息  
```

*sysctl -a   #显示系统核心设置*

*sysctl -w net.ipv4.tcp_max_syn_backlog=256 #设置环境变量*

*sysctl -p   #立马生效*

## 1. 系统级别限制配置

+  下次重启之前有效 修改系统级限制配置

```shell script
sysctl -w fs.file-max=100000
```

+ 无需注销和重新启动配置 修改系统级别限制配置

```shell script
vi /etc/sysctl.conf

fs.file-max = 100000                       #最大打开文件数
fs.file-nr = 2080       0       1621092    #文件描述符数量

sysctl -p #使上述更改立即生效

cat /proc/sys/fs/file-max #验证新的更改是否生效
#或者
sysctl -a|grep file
```
+ 查看系统相关配置
```shell script
cat  /proc/sys/fs/file-nr #系统打开文件数描述符限制

cat /proc/sys/fs/file-max #系统打开文件数限制

sysctl -a|grep file
```

## 2. 用户及系统限制

+ ulimit指令 控制shell程序的资源

```
 -a 　显示目前资源限制的设定。 
  -c  　设定core文件的最大值，单位为区块。 
  -d <数据节区大小> 　程序数据节区的最大值，单位为KB。 
  -f <文件大小> 　shell所能建立的最大文件，单位为区块。 
  -H 　设定资源的硬性限制，也就是管理员所设下的限制。 
  -m <内存大小> 　指定可使用内存的上限，单位为KB。 
  -n <文件数目> 　指定同一时间最多可开启的文件数。 
  -p <缓冲区大小> 　指定管道缓冲区的大小，单位512字节。 
  -s <堆叠大小> 　指定线程栈大小的上限，单位为KB。 

  -S 　设定资源的弹性限制。 

  -t  　指定CPU使用时间的上限，单位为秒。 
  -u <程序数目> 　用户最多可开启的程序数目。 
  -v <虚拟内存大小> 　指定可使用的虚拟内存上限，单位为KB。

ulimit -n –> 显示打开文件数限制
ulimit -c –> 显示核心转储文件大小
umilit -u –> 显示登录用户的最大用户进程数限制
ulimit -f –> 显示用户可以拥有的最大文件大小
umilit -m –> 显示登录用户的最大内存大小
ulimit -v –> 显示最大内存大小限制

```

```
ulimit -a                    #显示系统资源的设置

core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 3846
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 3846
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited


ulimit -u 65525				 #设置单一用户程序数目上限

core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 3846
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 65525		   #已改变
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited

```

+ 检查登录用户打开文件数量的硬限制和软限制

```
shashi@Ubuntu ~}$ ulimit -Hn
1048576
shashi@Ubuntu ~}$ ulimit -Sn
1024
```

+ 设置用户级资源限制

```shell script
# hard limit for max opened files for linuxtechi user
linuxtechi       hard    nofile          4096
# soft limit for max opened files for linuxtechi user
linuxtechi       soft    nofile          1024
# hard limit for max number of process for oracle user
oracle           hard    nproc          8096
# soft limit for max number of process for oracle user
oracle           soft    nproc          4096
```

+ 设置用户组级资源限制

```shell script
# hard limit for max opened files for sysadmin group
@sysadmin        hard         nofile            4096 
# soft limit for max opened files for sysadmin group
@sysadmin        soft         nofile            1024
```


+ 切换用户并验证配置

```shell script

su - linuxtechi

ulimit -n -H #打开文件最大限制数硬限制
4096

ulimit -n -S #打开文件最大限制数软限制
1024

su - oracle

ulimit -H -u #最大线程数
8096

ulimit -S -u #最大线程数
4096

```

+ 使配置生效到配置用户



```
# /etc/security/limits.conf
#
#Each line describes a limit for a user in the form:
#
#<domain>        <type>  <item>  <value>
#
#Where:
#<domain> can be:
#        - an user name
#        - a group name, with @group syntax
#        - the wildcard *, for default entry
#        - the wildcard %, can be also used with %group syntax,
#                 for maxlogin limit
#
#<type> can have the two values:
#        - "soft" for enforcing the soft limits  软限制(soft limit) 设定资源的软限制。警告的设定，可以超过这个设定值，但是若超过则有警告信息
#        - "hard" for enforcing hard limits      硬限制(hard limit) 设定资源的硬性限制，也就是管理员所设下的限制。严格的设定，必定不能超过这个设定的数值
#
#<item> can be one of the following:
#        - core - limits the core file size (KB) core文件其实就是内存的映像，当程序崩溃时，存储内存的相应信息，主用用于对程序进行调试。当程序崩溃时便会产生core文件，其实准确的应该说是core dump 文件,默认生成位置与可执行程序位于同一目录下，文件名为core.***,其中***是某一数字。
#        - data - max data size (KB)
#        - fsize - maximum filesize (KB)  所能建立的最大文件，单位为区块。
#        - memlock - max locked-in-memory address space (KB)
#        - nofile - max number of open files  指定同一时间最多可开启的文件数。
#        - rss - max resident set size (KB)
#        - stack - max stack size (KB)
#        - cpu - max CPU time (MIN)
#        - nproc - max number of processes  程序孵出的最大子进程数量 
#        - as - address space limit (KB)  地址空间限制  进程可用的内存的最大数量，包括堆栈、全局变量、动态内存
#        - maxlogins - max number of logins for this user 此用户允许登录的最大数目
#        - maxsyslogins - max number of logins on the system 系统最大同时在线用户数
#        - priority - the priority to run user process with 运行用户进程的优先级
#        - locks - max number of file locks the user can hold 用户可以持有的文件锁的最大数量
#        - sigpending - max number of pending signals
#        - msgqueue - max memory used by POSIX message queues (bytes)
#        - nice - max nice priority allowed to raise to values: [-20, 19]
#        - rtprio - max realtime priority
#
#<domain>      <type>  <item>         <value>
```
