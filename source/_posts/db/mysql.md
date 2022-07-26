---
title: Mysql数据类型
date: 2022-07-26 17:53:21
tags:
- Mysql
categories:
- DB
---

# 数字类型

{% asset_img mysqlint.png jvmmap %}

在整型类型中，有 signed 和 unsigned 属性，其表示的是整型的取值范围，默认为 signed

## 浮点类型 高精度

* 浮点类型  Float 和 Double

MySQL 之前的版本中存在浮点类型 Float 和 Double，但这些类型因为不是高精度，也不是 SQL 标准的类型，所以在真实的生产环境中不推荐使用

* 高精度类型 DECIMAL(8,2)

> 整形类型与自增
>> 用 BIGINT 做主键，而不是 INT
>> 自增值并不持久化，可能会有回溯现象（MySQL 8.0 版本前）,MySQL 8.0 版本前，自增不持久化，自增值可能会存在回溯问题
>> 资金类字段设计 建议采用BIG INT（8字节定长）,以分为单位


# 字符类型

MySQL 数据库的字符串类型有 CHAR、VARCHAR、BINARY、BLOB、TEXT、ENUM、SET。不同的类型在业务设计、数据库性能方面的表现完全不同，其中最常使用的是 CHAR、VARCHAR

CHAR(N) 用来保存固定长度的字符，N 的范围是 0 ~ 255，请牢记，N 表示的是字符，而不是字节。

VARCHAR(N) 用来保存变长字符，N 的范围为 0 ~ 65536， N 表示字符。

在超出 65536 个字符的情况下，可以考虑使用更大的字符类型 TEXT 或 BLOB，两者最大存储长度为 4G，其区别是 BLOB 没有字符集属性，纯属二进制存储

和 Oracle、Microsoft SQL Server 等传统关系型数据库不同的是，MySQL 数据库的 VARCHAR 字符类型，最大能够存储 65536 个字符，所以在 MySQL 数据库下，绝大部分场景使用类型 VARCHAR 就足够了


## 字符集

在表结构设计中，除了将列定义为 CHAR 和 VARCHAR 用以存储字符以外，还需要额外定义字符对应的字符集，因为每种字符在不同字符集编码下，对应着不同的二进制值。常见的字符集有 GBK、UTF8，通常推荐把默认字符集设置为 UTF8

而且随着移动互联网的飞速发展，推荐把 MySQL 的默认字符集设置为 UTF8MB4,否则，某些 emoji 表情字符无法在 UTF8 字符集下存储，比如 emoji 笑脸表情，对应的字符编码为 0xF09F988E

包括 MySQL 8.0 版本在内，字符集默认设置成 UTF8MB4，8.0 版本之前默认的字符集为 Latin1。因为不同版本默认字符集的不同，你要显式地在配置文件中进行相关参数的配置
```
[mysqld]
character-set-server = utf8mb4
```

另外，不同的字符集，CHAR(N)、VARCHAR(N) 对应最长的字节也不相同。比如 GBK 字符集，1 个字符最大存储 2 个字节，UTF8MB4 字符集 1 个字符最大存储 4 个字节。所以从底层存储内核看，在多字节字符集下，CHAR 和 VARCHAR 底层的实现完全相同，都是变长存储

* 正确设置字符集

```
mysql> ALTER TABLE emoji_test CHARSET utf8mb4;
mysql> ALTER TABLE emoji_test CONVERT TO CHARSET utf8mb4;
```

## 排序规则

排序规则（Collation）是比较和排序字符串的一种规则，每个字符集都会有默认的排序规则，你可以用命令 SHOW CHARSET 来查看

```
mysql> SHOW CHARSET LIKE 'utf8%';

+---------+---------------+--------------------+--------+

| Charset | Description   | Default collation  | Maxlen |

+---------+---------------+--------------------+--------+

| utf8    | UTF-8 Unicode | utf8_general_ci    |      3 |

| utf8mb4 | UTF-8 Unicode | utf8mb4_0900_ai_ci |      4 |

+---------+---------------+--------------------+--------+

2 rows in set (0.01 sec)

mysql> SHOW COLLATION LIKE 'utf8mb4%';

+----------------------------+---------+-----+---------+----------+---------+---------------+

| Collation                  | Charset | Id  | Default | Compiled | Sortlen | Pad_attribute |

+----------------------------+---------+-----+---------+----------+---------+---------------+

| utf8mb4_0900_ai_ci         | utf8mb4 | 255 | Yes     | Yes      |       0 | NO PAD        |

| utf8mb4_0900_as_ci         | utf8mb4 | 305 |         | Yes      |       0 | NO PAD        |

| utf8mb4_0900_as_cs         | utf8mb4 | 278 |         | Yes      |       0 | NO PAD        |

| utf8mb4_0900_bin           | utf8mb4 | 309 |         | Yes      |       1 | NO PAD        |

| utf8mb4_bin                | utf8mb4 |  46 |         | Yes      |       1 | PAD SPACE     |
```

排序规则以 _ci 结尾，表示不区分大小写（Case Insentive），_cs 表示大小写敏感，_bin 表示通过存储字符的二进制进行比较。需要注意的是，比较 MySQL 字符串，默认采用不区分大小的排序规则

