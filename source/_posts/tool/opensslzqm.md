---
title: 自签名证书
date: 2022-05-30 13:36:38
tags:
- openssl
  categories:
- 工具
---

# Centos7 环境下生成自签名SSL证书的具体过程：
1. 修改 openssl 配置文件
vi /etc/pki/tls/openssl.cnf
match 表示后续生成的子证书的对应项必须和创建根证书时填的值一样，否则报错。以下配置只规定子证书的 countryName 必须和根证书一致。
[policy_match] 段配置改成如下：
countryName = match
stateOrProvinceName = optional
organizationName = optional
organizationalUnitName = optional
commonName = supplied
emailAddress = optional

2. 在服务器 pki 的 CA 目录下新建两个文件
cd /etc/pki/CA && touch index.txt serial && echo 01 > serial
3. 生成CA根证书密钥
cd /etc/pki/CA/ && openssl genrsa -out private/cakey.pem 2048 && chmod 400 private/cakey.pem
4. 生成根证书（根据提示输入信息，除了 Country Name 选项需要记住的，后面的随便填）
openssl req -new -x509 -key private/cakey.pem -out cacert.pem
5. 生成密钥文件
openssl genrsa -out nginx.key 2048
6. 生成证书请求文件（CSR）:
A.根据提示输入信息，除了 Country Name 与前面根证书一致外，其他随便填写
B.Common Name 填写要保护的域名，比如：*.qhh.me
openssl req -new -key nginx.key -out nginx.csr
7. 使用 openssl 签署 CSR 请求，生成证书
openssl ca -in nginx.csr -cert /etc/pki/CA/cacert.pem -keyfile /etc/pki/CA/private/cakey.pem -days 365 -out nginx.crt

> 参数项说明：
1. -in: CSR 请求文件
2. -cert: 用于签发的根 CA 证书
3. -keyfile: 根 CA 的私钥文件
4. -days: 生成的证书的有效天数
5. -out: 生成证书的文件名

至此自签名证书生成完成，最终需要：nginx.key 和 nginx.crt
8. 配置Nginx使用自签名证书
```config
server {
  listen  80;
  server_name     domain;
  return  301     https://$host$request_uri;  # 表示强制将http请求重定向到https端口
}
server {
    listen  443 ssl;
    ssl_certificate       ssl/nginx.crt; # 前面生成的 crt 证书文件
    ssl_certificate_key   ssl/nginx.key; # 前面生成的证书私钥
    server_name     domain;
    location / {
    root /var/www-html;
    index  index.html;
}

```
 