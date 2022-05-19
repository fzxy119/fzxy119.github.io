---
title: ES安装
date: 2022-05-19 19:43:46
tags:
- ES
categories:
- DB
---
## ES安装：

### 环境变量设置：

 ES_HOME=/usr/local/src/elasticsearch-7.15.2
		ES_JAVA_HOME=/usr/local/src/elasticsearch-7.15.2/jdk

### 用户组准备

```
 groupadd es
 useradd es -g es -p es
 chown -R es:es  elasticsearch-7.15.2/
 chown -R es:es  eslog/
 chown -R es:es  esdata/
```

## 系统相关限制

max number of threads [1024] for user [es] is too low, increase to at least [4096]

elasticsearch用户拥有的可创建文件描述的权限太低，至少需要65536

vi /etc/security/limits.conf   # 在最后面追加下面内容
		es soft nproc 4096
        es hard nofile 65536
		es soft nofile 65536

max_map_count文件包含限制一个进程可以拥有的VMA(虚拟内存区域)的数量 

切换到root用户修改
        vim /etc/sysctl.conf    # 在最后面追加下面内容
	    vm.max_map_count=655360
        执行  sysctl -p
    

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
network.publish_host: 12.3.10.205
network.publish_host: 12.3.10.105
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


