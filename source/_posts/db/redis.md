---
title: Redis集群
date: 2022-07-26 13:34:35
tags:
- Redis
categories:
- DB
---
# 安装

cd redis/src
make MALLOC=libc

cd redis
make && make install


```
port 7001 # 客户端连接端口
bind 12.3.10.222 #实例绑定的IP地址
logfile "usr/local/src/redis-5.0.14/logs/6479.log"
dir /usr/local/src/redis-5.0.14/datas_6479 # redis实例数据配置存储位置
daemonize yes # 是否以后台进程的方式启动redis实例
pidfile /var/run/redis_7001.pid # 指定该进程pidfile

cluster-enabled yes # 开启集群模式
cluster-config-file # 集群中该实例的配置文件，该文件会在data目录下生成
appendonly yes # 开启aop日志
protected-mode no # 关闭保护模式
requirepass cx # master开启密码保护
masterauth cx # replica同master交互密码
```


```

 src/redis-server cluster/redis-6474.config  
 src/redis-server cluster/redis-6475.config 
 src/redis-server cluster/redis-6476.config 
 src/redis-server cluster/redis-6477.config 
 src/redis-server cluster/redis-6478.config 
 src/redis-server cluster/redis-6479.config 

#文本替换
:1,$s/6479/6474/gc



src/redis-cli -p cx --cluster create 12.3.10.222:6479 12.3.10.222:6478 12.3.10.222:6477 12.3.10.222:6476 12.3.10.222:6475 12.3.10.222:6474 --cluster-replicas 1 -a cx


 ps aux|grep redis|grep -v 'grep'|awk '{print $2}'|xargs kill -15


```