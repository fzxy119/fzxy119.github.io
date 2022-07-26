---
title: ES安装与运行
date: 2022-05-19 19:43:46
tags:
- ES
categories:
- DB
---
## ES安装：

安装 Elasticsearch 之前，你需要先安装一个较新的版本的 Java

[Es 官网下载](https://www.elastic.co/cn/downloads/elasticsearch)

* 如果你想把 Elasticsearch 作为一个守护进程在后台运行，那么可以在后面添加参数 -d 

* 如果你是在 Windows 上面运行 Elasticseach，你应该运行 bin\elasticsearch.bat 而不是 bin\elasticsearch

测试是否安装成功

curl 'http://localhost:9200/?pretty'

```json
{
    "name": "node-222",
    "cluster_name": "ryx",
    "cluster_uuid": "AaljyBfbTm2VVmOmyZSgsw",
    "version": {
        "number": "7.15.2",
        "build_flavor": "default",
        "build_type": "tar",
        "build_hash": "93d5a7f6192e8a1a12e154a2b81bf6fa7309da0c",
        "build_date": "2021-11-04T14:04:42.515624022Z",
        "build_snapshot": false,
        "lucene_version": "8.9.0",
        "minimum_wire_compatibility_version": "6.8.0",
        "minimum_index_compatibility_version": "6.0.0-beta1"
    },
    "tagline": "You Know, for Search"
}
```

### 环境变量设置：

 ES_HOME=/usr/local/src/elasticsearch-7.15.2
 ES_JAVA_HOME=/usr/local/src/elasticsearch-7.15.2/jdk

### 用户组准备 es禁止使用root 启动服务，所以需要准备es对应的用户信息 创建用户组 及用户并修改es用户对根目录数据目录日志目录的权限

```
 groupadd es
 useradd es -g es -p es
 chown -R es:es  elasticsearch-7.15.2/
 chown -R es:es  eslog/
 chown -R es:es  esdata/
```

## 部署配置

### 硬件

内存：

&nbsp;&nbsp;&nbsp;&nbsp;64 GB 内存的机器是非常理想的， 但是32 GB 和16 GB 机器也是很常见的。少于8 GB 会适得其反（你最终需要很多很多的小机器），大于64 GB 的机器也会有问题

CPU:

&nbsp;&nbsp;&nbsp;&nbsp;大多数 Elasticsearch 部署往往对 CPU 要求不高。因此，相对其它资源，具体配置多少个（CPU）不是那么关键。你应该选择具有多个内核的现代处理器，常见的集群使用两到八个核的机器。

&nbsp;&nbsp;&nbsp;&nbsp;如果你要在更快的 CPUs 和更多的核心之间选择，选择更多的核心更好。多个内核提供的额外并发远胜过稍微快一点点的时钟频率

硬盘：

&nbsp;&nbsp;&nbsp;&nbsp;硬盘对所有的集群都很重要，对大量写入的集群更是加倍重要（例如那些存储日志数据的）。硬盘是服务器上最慢的子系统，这意味着那些写入量很大的集群很容易让硬盘饱和，使得它成为集群的瓶颈。

&nbsp;&nbsp;&nbsp;&nbsp;如果你负担得起 SSD，它将远远超出任何旋转介质（注：机械硬盘，磁带等）。 基于 SSD 的节点，查询和索引性能都有提升。如果你负担得起，SSD 是一个好的选择。

网络：

&nbsp;&nbsp;&nbsp;&nbsp;快速可靠的网络显然对分布式系统的性能是很重要的。 低延时能帮助确保节点间能容易的通讯，大带宽能帮助分片移动和恢复。现代数据中心网络（1 GbE, 10 GbE）对绝大多数集群都是足够的。

&nbsp;&nbsp;&nbsp;&nbsp;即使数据中心们近在咫尺，也要避免集群跨越多个数据中心。绝对要避免集群跨越大的地理距离。

&nbsp;&nbsp;&nbsp;&nbsp;Elasticsearch 假定所有节点都是平等的—并不会因为有一半的节点在150ms 外的另一数据中心而有所不同。更大的延时会加重分布式系统中的问题而且使得调试和排错更困难

###  文件描述符和MMAP

Es的用户线程打开数量要求最低4096，打开文件描述符开放到65536，进程虚拟内存区域655360

```
#用户级限制配置
vi /etc/security/limits.conf   # 在最后面追加下面内容

es soft nproc 4096 #max number of threads [1024] for user [es] is too low, increase to at least [4096]
es hard nofile 65536 # elasticsearch用户拥有的可创建文件描述的权限太低，至少需要65536
es soft nofile 65536 # elasticsearch用户拥有的可创建文件描述的权限太低，至少需要65536

#切换到root用户修改 max_map_count文件包含限制一个进程可以拥有的VMA(虚拟内存区域)的数量 
vim /etc/sysctl.conf    # 在最后面追加下面内容
vm.max_map_count=655360
#执行  sysctl -p
```

### elasticsearch 相关隐私配置 elasticsearch-keystore

```
bin/elasticsearch-keystore create -p 

#设置xpack.security.transport.ssl.keystore.password
bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password

#设置xpack.security.transport.ssl.truststore.password
bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
```

### 管理账号密码初始

```

.bin/elasticsearch-setup-passwords interactive    

Initiating the setup of passwords for reserved users elastic,kibana,logstash_system,beats_system.
You will be prompted to enter passwords as the process progresses.
Please confirm that you would like to continue [y/N]y
Enter password for [elastic]: 
passwords must be at least [6] characters long
Try again.
Enter password for [elastic]: 
Reenter password for [elastic]: 
Passwords do not match.
Try again.
Enter password for [elastic]: 
Reenter password for [elastic]: 
Enter password for [kibana]: 
Reenter password for [kibana]: 
Enter password for [logstash_system]: 
Reenter password for [logstash_system]: 
Enter password for [beats_system]: 
Reenter password for [beats_system]: 
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [elastic]

```

系统间数据传输添加安全协议层

```shell
#elasticsearch-certutil
bin/elasticsearch-certutil
(
(ca [--ca-dn <name>] [--days <n>] [--pem])
| (cert ([--ca <file_path>] | [--ca-cert <file_path> --ca-key <file_path>])
[--ca-dn <name>] [--ca-pass <password>] [--days <n>]
[--dns <domain_name>] [--in <input_file>] [--ip <ip_addresses>]
[--keep-ca-key] [--multiple] [--name <file_name>] [--pem] [--self-signed])

| (csr [--dns <domain_name>] [--in <input_file>] [--ip <ip_addresses>]
[--name <file_name>])

[-E <KeyValuePair>] [--keysize <bits>] [--out <file_path>]
[--pass <password>]
)

| http

[-h, --help] ([-s, --silent] | [-v, --verbose])


#The following command generates a CA certificate and private key in PKCS#12
#生成 CA证书和私钥 以PKCS#12格式存储
#You are prompted for an output filename and a password. Alternatively, you can specify the --out and --pass parameters
bin/elasticsearch-certutil ca

#You can then generate X.509 certificates and private keys by using the new CA
#You are prompted for the CA password and for an output filename and password. Alternatively, you can specify the --ca-pass, --out, and --pass parameters
#生成X.509 证书和私钥
bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
```



### 系统简单配置

```yml
cluster.name: ryx
node.master: true
node.data: true
node.name: node-222
#discovery.type: single-node
network.host: 0.0.0.0
#和其他程序通信
network.publish_host: 12.3.10.222
#network.publish_host: 12.3.10.205
#network.publish_host: 12.3.10.105
http.port: 9200
transport.tcp.port: 9300
discovery.seed_hosts: ["12.3.10.222","12.3.10.205", "12.3.10.105"]
cluster.initial_master_nodes: ["node-222","node-205","node-105"]
discovery.zen.minimum_master_nodes: 2 #至少连个master节点启动才算集群启动，并开始数据同步
bootstrap.system_call_filter: false #关闭系统版本校验
action.destructive_requires_name: true #索引删除是否名称敏感
action.auto_create_index: false #是否自动创建索引
xpack.security.enabled: true
xpack.license.self_generated.type: basic
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.keystore.path: cer/elastic-certificates.p12
xpack.security.transport.ssl.keystore.type: PKCS12
xpack.security.transport.ssl.keystore.password: 123456
xpack.security.transport.ssl.verification_mode: certificate #指定传输验证模式
xpack.security.transport.ssl.truststore.path: cer/elastic-certificates.p12
xpack.security.transport.ssl.truststore.type: PKCS12
xpack.security.transport.ssl.truststore.password: 123456
```


### 重要的配置项

* 集群名称
&nbsp;&nbsp;&nbsp;&nbsp;Elasticsearch 默认启动的集群名字叫 elasticsearch 。你最好给你的生产环境的集群改个名字，改名字的目的很简单， 就是防止某人的笔记本电脑加入了集群这种意外
```
#elasticsearch.yml
cluster.name: elasticsearch_production
```
* 节点名称
&nbsp;&nbsp;&nbsp;&nbsp;Elasticsearch 会在你的节点启动的时候随机给它指定一个名字。你可能会觉得这很有趣，但是当凌晨 3 点钟的时候， 你还在尝试回忆哪台物理机是 Tagak the Leopard Lord 的时候，你就不觉得有趣了。

&nbsp;&nbsp;&nbsp;&nbsp;更重要的是，这些名字是在启动的时候产生的，每次启动节点， 它都会得到一个新的名字。这会使日志变得很混乱，因为所有节点的名称都是不断变化的

```
#elasticsearch.yml
node.name: elasticsearch_005_data
```


* 路径

&nbsp;&nbsp;&nbsp;&nbsp;默认情况下，Elasticsearch 会把插件、日志以及你最重要的数据放在安装目录下。这会带来不幸的事故， 如果你重新安装 Elasticsearch 的时候不小心把安装目录覆盖了。如果你不小心，你就可能把你的全部数据删掉了

&nbsp;&nbsp;&nbsp;&nbsp;最好的选择就是把你的数据目录配置到安装目录以外的地方， 同样你也可以选择转移你的插件和日志目录。
```
#elasticsearch.yml
#数据目录
path.data: /path/to/data1,/path/to/data2 

#日志目录
path.logs: /path/to/logs

# 插件目录
path.plugins: /path/to/plugins
```
* 最小节点数

&nbsp;&nbsp;&nbsp;&nbsp;minimum_master_nodes  设定对你的集群的稳定 极其 重要。 当你的集群中有两个 masters（注：主节点）的时候，这个配置有助于防止 脑裂 ，一种两个主节点同时存在于一个集群的现象。
                      
&nbsp;&nbsp;&nbsp;&nbsp;如果你的集群发生了脑裂，那么你的集群就会处在丢失数据的危险中，因为主节点被认为是这个集群的最高统治者，它决定了什么时候新的索引可以创建，分片是如何移动的等等。如果你有 两个 masters 节点， 你的数据的完整性将得不到保证，因为你有两个节点认为他们有集群的控制权。

&nbsp;&nbsp;&nbsp;&nbsp;这个配置就是告诉 Elasticsearch 当没有足够 master 候选节点的时候，就不要进行 master 节点选举，等 master 候选节点足够了才进行选举

&nbsp;&nbsp;&nbsp;&nbsp;此设置应该始终被配置为 master 候选节点的法定个数（大多数个）。法定个数就是 ( master 候选节点个数 / 2) + 1

```
#elasticsearch.yml
discovery.zen.minimum_master_nodes: 2


#PUT /_cluster/settings
{
    "persistent" : {
        "discovery.zen.minimum_master_nodes" : 2
    }
}
```

* 集群恢复相关的配置

&nbsp;&nbsp;&nbsp;&nbsp;这三个设置可以在集群重启的时候避免过多的分片交换。这可能会让数据恢复从数个小时缩短为几秒钟

```
#config/elasticsearch.yml
# Elasticsearch 在存在至少 8 个节点（数据节点或者 master 节点）之前进行数据恢复。 这个值的设定取决于个人喜好：整个集群提供服务之前你希望有多少个节点在线？这种情况下，我们设置为 8，这意味着至少要有 8 个节点，该集群才可用
gateway.recover_after_nodes: 8

# 现在我们要告诉 Elasticsearch 集群中 应该 有多少个节点，以及我们愿意为这些节点等待多长时间
gateway.expected_nodes: 10
gateway.recover_after_time: 5m
#这意味着 Elasticsearch 会采取如下操作：
    #等待集群至少存在 8 个节点
    #等待 5 分钟，或者10 个节点上线后，才进行数据恢复，这取决于哪个条件先达到。
```

* 使用单播代替组播

&nbsp;&nbsp;&nbsp;&nbsp;Elasticsearch 默认被配置为使用单播发现，以防止节点无意中加入集群。只有在同一台机器上运行的节点才会自动组成集群。

&nbsp;&nbsp;&nbsp;&nbsp;虽然组播仍然 作为插件提供， 但它应该永远不被使用在生产环境了，否则你得到的结果就是一个节点意外的加入到了你的生产环境，仅仅是因为他们收到了一个错误的组播信号。 对于组播 本身 并没有错，组播会导致一些愚蠢的问题，并且导致集群变的脆弱（比如，一个网络工程师正在捣鼓网络，而没有告诉你，你会发现所有的节点突然发现不了对方了）。

&nbsp;&nbsp;&nbsp;&nbsp;使用单播，你可以为 Elasticsearch 提供一些它应该去尝试连接的节点列表。 当一个节点联系到单播列表中的成员时，它就会得到整个集群所有节点的状态，然后它会联系 master 节点，并加入集群。

&nbsp;&nbsp;&nbsp;&nbsp;这意味着你的单播列表不需要包含你的集群中的所有节点， 它只是需要足够的节点，当一个新节点联系上其中一个并且说上话就可以了。如果你使用 master 候选节点作为单播列表，你只要列出三个就可以了。 这个配置在 elasticsearch.yml 文件中
```
discovery.zen.ping.unicast.hosts: ["host1", "host2:port"]
```