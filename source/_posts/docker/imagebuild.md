---
title: 镜像构建
date: 2021-10-28 18:04:19
tags:
- Docker
categories:
- 技术
---

## 

![img](https://www.runoob.com/wp-content/uploads/2016/04/576507-docker1.png)

## Docker镜像的创建方法

基于Dockerfile进行创建

基于容器进行创建

## 基于Dockerfile创建

### Dockerfile命令列表

| **指令**                             | **含义**                                                     |
| ------------------------------------ | ------------------------------------------------------------ |
| FROM 镜像                            | 指定新镜像所基于的镜像，第一条指令必须为FROM指令，每创建一个镜像就需要一条FROM指令 |
| MAINTAINER 名字                      | 说明新镜像的维护人信息                                       |
| RUN命令                              | 在所基于的镜像上执行命令，并提交到新的镜像中                 |
| CMD [“要运行的程序”,“参数1”,“参数2”] | 指令启动容器时要运行的命令或者脚本，Dockerfile只能有一条CMD命令，如果指定多条则只能最后一条被执行， |
| EXPOSE 端口号                        | 指定新镜像加载到Docker时要开启的端口（EXPOSE暴露的是容器内部端口，需要再映射到一个外部端口上） |
| ENV 环境变量 变量值                  | 设置一个环境变量的值，会被后面的RUN使用                      |
| ADD 源文件/目录 目标文件/目录        | 将源文件复制到目标文件（与COPY的区别是将本地tar文件解压到镜像中） |
| COPY 源文件/目录 目标文件/目录       | 将本地主机上的文件/目录复制到目标地点，源文件/目录要与Dockerfile在相同的目录中 |
| VOLUME [“目录”]                      | 在容器中创建一个挂载点（VOLUME是宿主机中的某一个目录挂载到容器中） |
| USER 用户名/UID                      | 指定运行容器时的用户                                         |
| WORKDIR 路径                         | 为后续的RUN、CMD、ENTRYPOINT指定工作目录（WORKDIR类似于cd，但是只切换目录一次，后续的RUN命令就可以写相对路径了） |
| ONBUILD 命令                         | 指定所生成的镜像作为一个基础镜像时所要运行的命令             |
| HEALTHCHECK                          | 健康检查                                                     |
| ENTRYPOINT                           | 指令和CMD类似，它也是用户指定容器启动时要执行的命令，但如果dockerfile中也有CMD指令，CMD中的参数会被附加到ENTRYPOINT指令的后面。 如果这时docker run命令带了参数，这个参数会覆盖掉CMD指令的参数，并也会附加到ENTRYPOINT 指令的后面。优先级：ENTRYPOINT>CMD>docker run |



### 启动执行脚本 ENTRYPOINT CMD

![img](https://img.jbzj.com/file_images/article/201803/2018031211363722.png)



**shell 模式**

使用 shell 模式时，docker 会以 `/bin/sh -c "task command"` 的方式执行任务命令

**exec 模式**

使用 exec 模式时，容器中的任务进程就是容器内的 1 号进程



我们大概可以总结出下面几条规律：

   • 如果 ENTRYPOINT 使用了 shell 模式，CMD 指令会被忽略。

   • 如果 ENTRYPOINT 使用了 exec 模式，CMD 指定的内容被追加为 ENTRYPOINT 指定命令的参数。

   • 如果 ENTRYPOINT 使用了 exec 模式，CMD 也应该使用 exec 模式。

### 一个Dockerfile例子

```dockerfile
FROM centos:7
MAINTAINER chenxiao@ruiyinxin.com
LABEL version="0.2"
LABEL description="测试创建sentinel"
VOLUME ["/home/tomcat/log/"]
RUN mkdir -p /app/jdk
COPY jdk /app/jdk
COPY sentinel-dashboard.jar /app/
COPY run.sh /app
RUN chmod 777 /app/run.sh
ENV JAVA_HOME /app/jdk
ENV PATH $PATH:$JAVA_HOME/bin
ENV CLASSPATH .:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
EXPOSE 8088
WORKDIR /app
#CMD ["sh","run.sh"]
CMD ["./run.sh"]
#ENTRYPOINT ["./run.sh"]
```

```shell
#! /bin/sh
java -jar -Dspring.profiles.active=test sentinel-dashboard.jar
```

### 执行docker命令来创建docker镜像

docker build -t bd/sentineld .
