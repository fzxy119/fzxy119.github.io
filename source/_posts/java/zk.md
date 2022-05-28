---
title: zookeeper
date: 2022-05-28 10:29:19
tags:
- zookeeper
categories:
- 分布式
---
# ZK集群
## ZK相关配置
```properties
tickTime=2000

dataDir=/var/lib/zookeeper/data

dataLogDir=/var/lib/zookeeper/datalog

clientPort=2181

initLimit=5

syncLimit=2

maxClientCnxns=60

peerType=observer  #observer配置

server.1=10.3.60.151:2888:3888

server.2=10.3.60.152:2888:3888

server.3=10.3.60.153:2888:3888

server.x=host:port1:port2:observer #observer配置
```


## ZK中Leader选举
**集群中节点角色**

| 角色       | 读写       | 选举         | 过半写      |
|----------|----------|------------|----------|
| Leader   | 负责客户端读写  | 参与leader选举 | 参与过半写成功  |
| Follower | 负责客户端读   | 参与leader选举 | 参与过半写成功  |
| Observer | 负责客户端读   | 不参与leader选举|不参与过半写成功  |

> Observer 在不影响写性能的条件下，提升集群的读性能

**相关属性：**

SID:服务编号，及dataDir目录下myid编号

ZXID：事务编号，改变zookeeper几圈状态的操作，数据节点创建及删除，数据变更，客户单建立链接，及链接失效等变更都会产生新的事务编号。一般情况先会选ZXID大的节点为Leader ,如果都一样会选择SID大的节点为Leader

QUORUM：过半计数器

EPOCH：每个leader的纪元，生成新的leader都会生成一个新的增长的纪元。


## 脑裂问题

**脑裂问题描述：**
对于一个集群，想要提高这个集群的可用性，通常会采用多机房部署，比如现在有一个由6台zkServer所组成的一个集群，部署在了两个机房：

{% asset_img zk1.png 脑裂问题 %}

&emsp;&emsp;正常情况下，此集群只会有一个Leader，那么如果机房之间的网络断了之后，两个机房内的zkServer还是可以相互通信的，如果不考虑过半机制，那么就会出现每个机房内部都将选出一个Leader。

{% asset_img zk2.png 脑裂问题 %}

&emsp;&emsp;这就相当于原本一个集群，被分成了两个集群，出现了两个"大脑"，这就是所谓的"脑裂"现象。对于这种情况，其实也可以看出来，原本应该是统一的一个集群对外提供服务的，现在变成了两个集群同时对外提供服务，如果过了一会，断了的网络突然联通了，那么此时就会出现问题了，两个集群刚刚都对外提供服务了，数据该怎么合并，数据冲突怎么解决等等问题。刚刚在说明脑裂场景时有一个前提条件就是没有考虑过半机制.所以实际上Zookeeper集群中是不会轻易出现脑裂问题的，原因在于过半机制.

{% asset_img zk3.png 脑裂问题 %}

&emsp;&emsp;zookeeper的过半机制（Quorums (法定人数) 方式）：在领导者选举的过程中，如果某台zkServer获得了超过半数的选票，则此zkServer就可以成为Leader了。举个简单的例子：如果现在集群中有5台zkServer，那么half=5/2=2，那么也就是说，领导者选举的过程中至少要有三台zkServer投了同一个zkServer，才会符合过半机制，才能选出来一个Leader。

> 过半机制很好的杜绝了脑列的问题


## ZK作为master选举脑裂导致数据一致性问题：
&emsp;&emsp;kafka将数据以主题划分，并且每个主题可以有多个分片，每个分片可以有多个副本，在多个副本中需要选择出主副本和副本，有主副本完成数据的操作，副本分片完成数据同步。
那么当出现主副本与zk因为网络问题，导致zk认为主副本掉线，重新选举主副本，但是此时主副本没有死掉，所以导致了主副本的假死现象。
&emsp;&emsp;如果客户端还未完成副本列表切换，那么会导致数据写入假死的副本，而完成了切换的节点数会写入到新的主副本，那么就导致了数据不一致,
&emsp;&emsp;Epoch leader纪元或者版本号或者叫做leader周期，当产生新的Leader后纪元会随之增加，那么老的leader数据不会同步到新的集群范围内，通过Epoch来控制集群的数据不一致性，及通过集群状态的版本号


## 过半机制:
&emsp;&emsp;过半机制很好的杜绝了脑列的问题,过半验证还提高了系统效率，在写操作或者投票过程中不需要所有节点都完成，只要过半就完成投票或者写操作。

**容忍度问题：容忍度为n,那么集群建议的部署节点为2N+1**

&emsp;&emsp;一般情况下我们建议集群部署的节点为奇数个节点。至于为什么最好为奇数个节点？这样是为了以最大容错服务器个数的条件下，能节省资源。比如，最大容错为2的情况下，对应的zookeeper服务数，奇数为5，而偶数为6，也就是6个zookeeper服务的情况下最多能宕掉2个服务，所以从节约资源的角度看，没必要部署6（偶数）个zookeeper服务节点



## ZK权限认证方式
ACL全称为Access Control List（访问控制列表），用于控制资源的访问权限,可以控制节点的读写操作,保证数据的安全性 。

ZooKeeper使用ACL来控制对其znode的防问。

Zookeeper ACL 权限设置分为 3 部分组成，分别是：权限模式（Scheme）、授权对象（ID）、权限信息（Permission）

基于scheme:id:permission的方式进行权限控制： scheme表示授权模式、id模式对应值、permission即具体的增删改权限位。

### 权限模式（Scheme）

| 方案     | 描述                                                                  |     |
|--------|---------------------------------------------------------------------|-----|
| world  | 开放模式，world表示全世界都可以访问（这是默认设置）                                        |     |
| ip     | ip模式，限定客户端IP防问                                                      |     |
| auth   | 用户密码认证模式，只有在会话中添加了认证才可以防问                                           |     |
| digest | 与auth类似，区别在于auth用明文密码，而digest 用sha-1+base64加密后的密码。在实际使用中digest 更常见。 |     |
| super  | 管理员                                                                 |     |



>范围验证 范围验证就是说 ZooKeeper 可以针对一个 IP 或者一段 IP 地址授予某种权限。我们可以让一个 IP 地址为“ip：192.168.11.123”的机器对服务器上的某个数据节点具有写入的权限。或者也可以通过“ip:192.168.0.1/24”给一段 IP 地址的机器赋权。
### 授权对象（ID）
授权对象就是说我们要把权限赋予谁，而对应于 4 种不同的权限模式来说，

1. 如果我们 选择采用 IP 方式，使用的授权对象可以是一个 IP 地址或 IP 地址段

2. 使用 Digest 或 Super 方式，则对应于一个用户名

3. World 模式，是授权系统中所有的用户

### 权限信息（Permission）

权限就是指我们可以在数据节点上执行的操作种类，如下所示：在 ZooKeeper 中已经定义好的权限有 5 种：

* 数据节点（c: create）创建权限，授予权限的对象可以在数据节点下创建子节点；
* 数据节点（w: wirte）更新权限，授予权限的对象可以更新该数据节点；
* 数据节点（r: read）读取权限，授予权限的对象可以读取该节点的内容以及子节点的列表信息；
* 数据节点（d: delete）删除权限，授予权限的对象可以删除该数据节点的子节点；
* 数据节点（a: admin）管理者权限，授予权限的对象可以对该数据节点体进行 ACL 权限设置。

### ACL相关命令
<table><thead><tr><th style="text-align:left"><div><div class="table-header"><p>命令</p></div></div></th><th style="text-align:left"><div><div class="table-header"><p>使用方式</p></div></div></th><th style="text-align:left"><div><div class="table-header"><p>描述</p></div></div></th></tr></thead><tbody><tr><td style="text-align:left"><div><div class="table-cell"><p>getAcl</p></div></div></td><td style="text-align:left"><div><div class="table-cell"><p>getAcl path</p></div></div></td><td style="text-align:left"><div><div class="table-cell"><p>读取ACL权限</p></div></div></td></tr><tr><td style="text-align:left"><div><div class="table-cell"><p>setAcl</p></div></div></td><td style="text-align:left"><div><div class="table-cell"><p>setAcl path acl</p></div></div></td><td style="text-align:left"><div><div class="table-cell"><p>设置ACL权限</p></div></div></td></tr><tr><td style="text-align:left"><div><div class="table-cell"><p>addauth</p></div></div></td><td style="text-align:left"><div><div class="table-cell"><p>addauth scheme auth</p></div></div></td><td style="text-align:left"><div><div class="table-cell"><p>添加认证用户</p></div></div></td></tr></tbody></table>

{% asset_img zk4.png ACL相关命令 %}

### 跳过ACL检测
可以通过系统参数zookeeper.skipACL=yes进行配置，默认是no,可以配置为true, 则配置过的ACL将不再进行权限检测

zkServer.sh

{% asset_img zk.png ACL相关命令 %}

### 权限扩展体系

[](https://zookeeper.apache.org/doc/r3.8.0/zookeeperProgrammers.html#sc_ZooKeeperPluggableAuthentication)

1: zookeeper支持自定义权限扩展，只需要集成通用的标准接口

```java
public interface AuthenticationProvider {
    String getScheme();
    KeeperException.Code handleAuthentication(ServerCnxn cnxn, byte authData[]);
    boolean isValid(String id);
    boolean matches(String id, String aclExpr);
    boolean isAuthenticated();
}
```
2: 插件配置 (两种方式只有一个起作用)
方式1 系统变量

-Dzookeeeper.authProvider.X=com.f.MyAuth

方式2 配置文件

```properties
authProvider.1=com.f.MyAuth
authProvider.2=com.f.MyAuth2
```

## 实操ACL

### 生成授权ID
* 方式一 Code
```java
public void generateSuperDigest() throws NoSuchAlgorithmException {
    String sId = DigestAuthenticationProvider.generateDigest("artisan:xgj");
    System.out.println(sId); 
}
```
* 方式二 shell命令
```shell
echo -n <user>:<password> | openssl dgst -binary -sha1 | openssl base64
```
```shell
[root@localhost bin]# echo -n artisan:xgj | openssl dgst -binary -sha1 | openssl base64
Xe7+HMYId2eNV48821ZrcFwIqIE=
[root@localhost bin]# 
```

### 方式一 digest 密文授权
创建Node的时候 设置acl
```shell
[zk: localhost:2181(CONNECTED) 10] create /artisan_node artisan_value digest:artisan:Xe7+HMYId2eNV48821ZrcFwIqIE=:cdrwa
Created /artisan_node
[zk: localhost:2181(CONNECTED) 11] get /artisan_node  # 直接查看没有访问权限的 
org.apache.zookeeper.KeeperException$NoAuthException: KeeperErrorCode = NoAuth for /artisan_node
[zk: localhost:2181(CONNECTED) 12] 
```
创建node的时候不指定acl ,然后用setAcl 设置
```shell
setAcl /artisan_node digest:artisan:Xe7+HMYId2eNV48821ZrcFwIqIE=:cdrwa
```
如何才能有访问权限呢？ 因为是给artisan这个用户赋权的
【访问前需要添加授权信息】addauth
```shell
[zk: localhost:2181(CONNECTED) 12] addauth digest artisan:xgj
[zk: localhost:2181(CONNECTED) 13] get /artisan_node
artisan_value
[zk: localhost:2181(CONNECTED) 14] 
```
### 方式二 auth 明文授权
```shell
[zk: localhost:2181(CONNECTED) 14] addauth digest aaa:passwddd
[zk: localhost:2181(CONNECTED) 15] create /artisanNNN  nodeValue auth:aaa:passwddd:cdwra   # 这是aaa用户授权信息会被zk保存，可以认为当前的授权用户为aaa
Created /artisanNNN
[zk: localhost:2181(CONNECTED) 16] get /artisanNNN
nodeValue
[zk: localhost:2181(CONNECTED) 17] 
```
### 方式三 IP授权模式
创建时设置ip的权限
```shell
create /node-ip  data  ip:192.168.11.123:cdwra
```
或者创建完成以后 手工调用setAcl
```shell
setAcl /node-ip ip:192.168.11.123:cdwra
```
### 方式四 Super 超级管理员模式
是一种特殊的Digest模式， 在Super模式下超级管理员用户可以对Zookeeper上的节点进行任何的操作.
```shell
-Dzookeeper.DigestAuthenticationProvider.superDigest=super:<base64encoded(SHA1(password))
```

{% asset_img zk5.png %}

## Zookeeper应用场景

