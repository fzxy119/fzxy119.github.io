---
title: 镜像构建
date: 2021-10-28 18:04:19
tags:
- Docker
categories:
- 技术
---



[TOC]



{% asset_img 576507.png docker架构图 %}

## Docker镜像的创建方法

基于Dockerfile进行创建，docker build .

基于容器进行创建

## 基于Dockerfile创建

### Dockerfile构建需要的信息：

**Dockerfile、Context 和 .dockerignore**，build指令的执行时在Dorker daemon程序中，第一步就是地递归的将环境Context发送到daemon段，通常情况下会采用一个空的目录作为Context,并将Dockerfile文件至于Context目录中

通常情况下Context下要有一个名称为Dockerfile的文件，可以通过-f 指定任意位置的Dockerfile 文件

我们可以通过.dockerignore文件忽略上下文环境下的文件和目录

我们可以通过-t 来指定新的镜像的repository和标签信息,可以指定多个 -t 来对多个repository仓库的镜像image进行打标签

如：docker build -t shykes/myapp:1.0.2 -t shykes/myapp:latest .

<!-- more -->

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

命令格式为 docker build -t ImageName:TagName    Dir

- `-t` − 给镜像加一个Tag
- `ImageName` − 给镜像起的名称
- `TagName` − 给镜像的Tag名
- `Dir` − Dockerfile所在目录

docker build -t bd/sentineld  .

- `.` 表示当前目录，即Dockerfile所在目录

### .dockerignore

```gitignore
# comment
*/temp*
*/*/temp*
temp?
```



| `# comment` | Ignored.                                                     |
| ----------- | ------------------------------------------------------------ |
| `*/temp*`   | Exclude files and directories whose names start with `temp` in any immediate subdirectory of the root. For example, the plain file `/somedir/temporary.txt` is excluded, as is the directory `/somedir/temp`. |
| `*/*/temp*` | Exclude files and directories starting with `temp` from any subdirectory that is two levels below the root. For example, `/somedir/subdir/temporary.txt` is excluded. |
| `temp?`     | Exclude files and directories in the root directory whose names are a one-character extension of `temp`. For example, `/tempa` and `/tempb`are excluded. |



## 指令解析

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

### FROM

```dockerfile
FROM [--platform=<platform>] <image> [AS <name>]
FROM [--platform=<platform>] <image>[:<tag>] [AS <name>]
FROM [--platform=<platform>] <image>[@<digest>] [AS <name>]
```

 Dockerfile必须是以FROM指令开始

`ARG` 是唯一一个优先于 地位高于 FROM指令的，

```dockerfile
ARG VERSION=latest
FROM busybox:$VERSION
ARG VERSION
RUN echo $VERSION > image_version
```

--platform，指定镜像能被使用的平台信息，默认情况下是执行build request的平台信息被作为platform，如：linux/amd64`, `linux/arm64，windows/amd64

`as name`通查是可选的，他可以指定自定义构建步骤的名称

### RUN

RUN指令执行有两种形式

shell形式  RUN  <command>，此指令执行在shell上下文环境中，默认是 /bin/sh -c  ’code‘

exec 形式 RUN  ["executable", "param1", "param2"]，exec 模式会将后续的参数解析成json 数组，也就是说我们的参数需要用双引号包括，而不是单引号。

run命令的每次执行在当前镜像的最顶层生成新的层，并提交结果，并且将在下一个步骤中使用，也就是说每层之间是独立的，shell环境也是独立的。我们可以使用  `\` 来使用多行单指令 如：

```dockerfile
RUN /bin/bash -c 'source $HOME/.bashrc; \
echo $HOME'
#两者含义相同
RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'
```

### CMD

- `CMD ["executable","param1","param2"]` exec模式
- `CMD ["param1","param2"] 作为`ENTRYPOINT 的默认参数
- `CMD command param1 param2` *shell* 模式

不可将run与cmd指令混淆，run指令会执行命令，并提交结果，cmd并不执行任何东西，在构建的过程当中，他只指定镜像启动预期指令

### LABEL

```dockerfile
LABEL <key>=<value> <key>=<value> <key>=<value> 
```

```dockerfile
LABEL "com.example.vendor"="ACME Incorporated"
LABEL com.example.label-with-value="foo"
LABEL version="1.0"
LABEL description="This text illustrates \
that label-values can span multiple lines."
```

```dockerfile
LABEL multi.label1="value1" multi.label2="value2" other="value3"
```

```dockerfile
LABEL multi.label1="value1" \
      multi.label2="value2" \
      other="value3"
```

### EXPOSE

```dockerfile
EXPOSE <port> [<port>/<protocol>...]
```

默认情况下expose 使用的tcp协议，也可以显示指定

```dockerfile
EXPOSE 80/tcp
EXPOSE 80/udp
```

```dockerfile
docker run -p 80:80/tcp -p 80:80/udp ...
```

### ENV

```dockerfile
ENV <key>=<value> ...
```

env指令设置环境变量信息，那么环境变量信息，将存在于后续构建的各个阶段，

可以通过运行镜像，并指定环境变零的方式对镜像中设置的环境变量进行变更`docker run --env <key>=<value>`.

### ADD

```
ADD [--chown=<user>:<group>] <src>... <dest>
ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

--chown指令只支持linux系统上构建的镜像，在windows环境中并不工作，如果不指定用户或者用户组，默认使用 a UID and GID of 0

src： 可使用通配符，并且路径都是相对于Context的相对路径

​         网络上的远程文件url，如果url不是以斜线结尾，那么将被报错在dest目录下，如果是以斜线结尾，那么问题名称将被推理出来，

​    	  如 ADD http://example.com/foobar/  将创建问文件  /foobar，URL必须有一个非平凡的路径，这样才能找到适当的文件名。http://example.com 不会工作

​          文件必须存在于Context中，不能使用../ 因为build第一步是发送context到docker daemon程 序

​          如果src是一个目录，那么目录的整个内容将被复制，COPY jdk /app/jdk

​		  如果src是一个压缩文件(identity, gzip, bzip2 or xz) ，那么src将被解压到目标目录

dest： 目标路径如果不是以 / 开头的话是以WORKDIR为目录的相对路径

​		如果dest不存在，那么整个路径将被创建

​		如果dest不是以斜线结尾，那么dest将被认为是一个常规文件，src的内容将被写入到文件

​		如果src是多个资源，或者使用了通配符，那么dest必须要以斜线结尾

```
ADD hom* /mydir/
ADD hom?.txt /mydir/
ADD test.txt relativeDir/
ADD test.txt /absoluteDir/

ADD --chown=55:mygroup files* /somedir/
ADD --chown=bin files* /somedir/
ADD --chown=1 files* /somedir/
ADD --chown=10:11 files* /somedir/
```

### COPY

```dockerfile
COPY [--chown=<user>:<group>] <src>... <dest>
COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

src : 文件必须存在于Context中

​        如果src是一个目录，那么他的内容和元数据信息将被拷贝到目标目录 COPY jdk /app/jdk

​		如果src是一个文件，并且dest 是以斜线结尾，那么文件将被拷贝到目录下，如 dest/src 

dest: 如果目录不存在，那么整个文件目录将被创建

​         如果dest没有是以斜杠结尾，那么他将被看做事一个普通文件，并且src将被写入到dest

​		  如果src是多资源文件，那么dest必须是以斜杠结尾





### ENTRYPOINT

```dockerfile
ENTRYPOINT ["executable", "param1", "param2"]   #exec模式
ENTRYPOINT command param1 param2                #shell模式
```

1. Dockerfile 最少需要制定一个CMD或者ENTRYPOINT指令
2. 当容器是可执行的情况下，应该定义ENTRYPOINT
3. CMD应该被用来定义ENTRYPOINT 的默认参数，
4. CMD参数可以在容器运行的时候被替换掉，ENTRYPOINT则不行

|                                | **No ENTRYPOINT**          | **ENTRYPOINT exec_entry p1_entry** | **ENTRYPOINT [“exec_entry”, “p1_entry”]**      |
| :----------------------------- | :------------------------- | :--------------------------------- | ---------------------------------------------- |
| **No CMD**                     | *error, not allowed*       | /bin/sh -c exec_entry p1_entry     | exec_entry p1_entry                            |
| **CMD [“exec_cmd”, “p1_cmd”]** | exec_cmd p1_cmd            | /bin/sh -c exec_entry p1_entry     | exec_entry p1_entry exec_cmd p1_cmd            |
| **CMD [“p1_cmd”, “p2_cmd”]**   | p1_cmd p2_cmd              | /bin/sh -c exec_entry p1_entry     | exec_entry p1_entry p1_cmd p2_cmd              |
| **CMD exec_cmd p1_cmd**        | /bin/sh -c exec_cmd p1_cmd | /bin/sh -c exec_entry p1_entry     | exec_entry p1_entry /bin/sh -c exec_cmd p1_cmd |



### VOLUME

```dockerfile
VOLUME ["/data"]
```

VOLUME指令将从本地主机或者其他的容器创建一个挂载点并使用给定的名称，他的值可以是一个json数组VOLUME ["/var/log/"]，或者是一个字符串 VOLUME /var/log。 或者是多个VOLUME /var/log /var/db。

如果是基于windows的容器，那么这数据卷必须是一个不存在的或者是一个空的目录，并且是C盘以外的数据驱动

volume在声明之后，如果在build的过程中的每一个步骤对volume内容的修改都将被忽略，

声明的时候如果是json数据，必须是以双引号进行包括，而不是单引号

VOLUME指令挂在的目录，并不能通过host_dir来指定映射到主机那个目录，他是在运行容器的时候才确定的，你必须指定一个挂载点，当你创建或者启动一个容器的时候，

### USER

```dockerfile
USER <user>[:<group>]
USER <UID>[:<GID>]
```

### ARG

```
ARG <name>[=<default value>]
```

ARG指令可以在构建的过程中定义一些参数，并设置一些默认值，并且在build的时候可以指定

```
docker build --build-arg user=what_user .
```

**默认值；**如果build的时候没有指定--build-arg user1=what_user 那么someuser将被作为默认值

```
FROM busybox
ARG user1=someuser
ARG buildno=1
```

**作用域：**以下示例，第二行将使用some_user来作为值来使用，第四行将使用what_user，ARG指令定义的变量是从他定义的行开始生效的

```
FROM busybox
USER ${user:-some_user}
ARG user
USER $user
```

```
docker build --build-arg user=what_user .
```

**使用ARG变量：**ENV指令将会覆盖ARG同名参数，如下将使用v1.0.0，

```
FROM ubuntu
ARG CONT_IMG_VER
ENV CONT_IMG_VER=v1.0.0
RUN echo $CONT_IMG_VER
```

```
docker build --build-arg CONT_IMG_VER=v2.0.1 .
```

**ENV和ARG**：ARG变量并不能像EVN环境变量那样保存在镜像之中

```dockerfile
FROM ubuntu
ARG CONT_IMG_VER
ENV CONT_IMG_VER=${CONT_IMG_VER:-v1.0.0}   #可带有默认值
RUN echo $CONT_IMG_VER
```

Docker中包含一些默认的ARG参数，可以不用再Dockerfile中生命

- `HTTP_PROXY`

- `http_proxy`

- `HTTPS_PROXY`

- `https_proxy`

- `FTP_PROXY`

- `ftp_proxy`

- `NO_PROXY`

- `no_proxy`

  ```
  docker build --build-arg HTTPS_PROXY=https://my-proxy.example.com .
  ```

Docker默认的ARG全局范围内的平台参数，这些参数是在全局范围中定义的，因此在构建阶段或运行命令中不自动可用。要在构建阶段中公开这些参数之一，请在没有值的情况下重新定义它

- `TARGETPLATFORM` - platform of the build result. Eg `linux/amd64`, `linux/arm/v7`, `windows/amd64`.

- `TARGETOS` - OS component of TARGETPLATFORM

- `TARGETARCH` - architecture component of TARGETPLATFORM

- `TARGETVARIANT` - variant component of TARGETPLATFORM

- `BUILDPLATFORM` - platform of the node performing the build.

- `BUILDOS` - OS component of BUILDPLATFORM

- `BUILDARCH` - architecture component of BUILDPLATFORM

- `BUILDVARIANT` - variant component of BUILDPLATFORM

  如：

  ```
  FROM alpine
  ARG TARGETPLATFORM
  RUN echo "I'm building for $TARGETPLATFORM"
  ```

### ONBUILD

```
ONBUILD <INSTRUCTION>
```

这是一个触发器指令，镜像构建时候定义，当别的镜像基于当前镜像进行构建的时候 ONBUILD指令将在FROM指令后首先触发基础镜像的ONBUILD指令，

他注册了一个超前指令，并在之后运行，运行环境是构建容器的Context，如果有一个指令失败，那么FROM指令将是失败的

他能在docker inspect指令中被观察到

ONBUILD 只在直接FROM Dockerfile中生效，也就是多级A FROM B FROM C ,那么c的ONBILD 不会再A中生效

```
ONBUILD ADD . /app/src
ONBUILD RUN /usr/local/bin/python-build --dir /app/src
```

ONBUILD不支持连续使用，ONBUILD ONBUILD

ONBUILD不会触发FROM 和 MAINTAINER指令



### STOPSIGNAL

```
STOPSIGNAL signal
```

默认的STOPSIGNAL 可以使用--stop-signal 标记，在docker run 和docker create 的时候修改掉

### HEALTHCHECK

- `HEALTHCHECK [OPTIONS] CMD command` (check container health by running a command inside the container)

- `HEALTHCHECK NONE` (disable any healthcheck inherited from the base image)

  HEALTHCHECK 指令告诉Docker如何测试容器是否在工作。

  当指定了HEALTHCHECK 那么容器除了正常的状态，会添加一些健康状态，初始化是starting，当健康检查通过，状态为healthy，当经理了一系列的故障之后，将变更为unhealthy

  - `--interval=DURATION` (default: `30s`)
  - `--timeout=DURATION` (default: `30s`)
  - `--start-period=DURATION` (default: `0s`)
  - `--retries=N` (default: `3`)

- 0: success - the container is healthy and ready for use
- 1: unhealthy - the container is not working correctly
- 2: reserved - do not use this exit code 预留的状态

```
HEALTHCHECK --interval=5m --timeout=3s CMD curl -f http://localhost/ || exit 1
```

### SHELL

shell指令运行将默认的shell进行调整，默认的linux shell 是["/bin/sh", "-c"] Windows 是`["cmd", "/S", "/C"]`

shell ，在Dockerfile中必须是json模式的
