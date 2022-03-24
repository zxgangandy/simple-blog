---
title: "分布式ID"
date: 2022-03-18
tags: [distributed, uid, id, micro-service]
author: zxgangandy
layout: zh-cn/layouts/post.njk
image: /img/posts/zxgangandy/uuid-shellcode/uuid-in-memory.png
---

<!-- summary -->
在我们的日常业务开发中，通常需要对一些数据做唯一标识，例如为大量抓取的文章入库时分配一个唯一的id，为用户下的订单分配订单号等等。并发量小的时候，通常会使用数据库自增的主键id作为唯一id。但是在并发量大、存在分库分表的情况或者是在微服务系统中我们通常会考虑使用分布式ID的生成方案来生成id。如果是小型应用，或者是业务量不大的情况下，单库单表完全可以支撑现有业务，数据再大一点搞个MySQL主从同步读写分离也能解决。此种情况不属于这里讨论的范畴。
<!-- summary -->

## 分布式ID的特点
- 全局唯一：必须保证ID是全局性唯一的，这是基本的要求
- 高性能：高性能就要求低延时，ID生成响应必须要快，否则整个id的生成过程反倒会成为业务瓶颈
- 趋势递增：最好趋势递增，比如趋势递增的订单id能够从时间和范围上反应出某些业务属性，便于运营做数据分析
- 信息安全：很难从生成的ID推导出一些业务特征，否则对某些业务来说存在一定的风险(比如爬虫顺序爬取或者是根据订单号推导下单量)
- 简单易用：在系统设计、实现以及使用上都要尽可能的简单

## 常见的分布式ID算法

- UUID
- 数据库自增ID
- 数据库多主模式
- 号段模式
- Redis
- 雪花算法（SnowFlake）
- 滴滴出品（TinyID）
- 百度 （Uidgenerator）
- 美团（Leaf）

### 1. UUID
通用唯一识别码（Universally Unique Identifier，缩写：UUID）是用于计算机体系中以识别信息数目的一个128位标识符，也就是可以通过16个字节来表示。
UUID 由开放软件基金会（OSF）标准化，作为分布式计算环境（DCE）的一部分。
UUID的标准型式包含32个16进位数字，以连字号分为五段，形式为8-4-4-4-12的32个字元。范例：550e8400-e29b-41d4-a716-446655440000
在其规范的文本表示中，UUID 的 16 个 8 位字节表示为 32 个十六进制（基数16）数字，显示在由连字符分隔 '-' 的五个组中，"8-4-4-4-12" 总共 36 个字符（32 个字母数字字符和 4 个连字符）。例如：

```diff
123e4567-e89b-12d3-a456-426655440000
xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx
```
四位数字 M表示 UUID 版本，数字 N的一至三个最高有效位表示 UUID 变体。在例子中，M 是 1 而且 N 是 a(10xx)，这意味着此 UUID 是 "变体1"、"版本1" UUID；即基于时间的 DCE/RFC 4122 UUID。
对于 "变体(variants)1" 和 "变体2"，标准中定义了五个"版本(versions)"，并且在特定用例中每个版本可能比其他版本更合适。
版本由 M 字符串中指示。
"版本1" UUID 是根据时间和节点 ID（通常是MAC地址）生成;
"版本2" UUID是根据标识符（通常是组或用户ID）、时间和节点ID生成;
"版本3" 和 "版本5" 确定性UUID 通过散列 (hashing) 命名空间 (namespace) 标识符和名称生成;
"版本4" UUID 使用随机性或伪随机性生成。
更详细的信息可以参考[wikipedia](https://en.wikipedia.org/wiki/Universally_unique_identifier)和[RFC](https://tools.ietf.org/html/rfc4122)文档。

#### 优点:
- 容易实现，产生快
- ID唯一(几乎不会产生重复id)
- 无需中心化的服务器，本地生成，没有网络消耗
- 不会泄漏商业机密

#### 缺点:
- 可读性差
- 不容易存储
- 占用空间太多(16个字节), MySQL官方有明确的建议主键要尽量越短越好，36个字符长度的UUID不符合要求。 
- 影响数据库的性能,对MySQL索引不利, 比如[UUID or GUID as Primary Keys? Be Careful!](https://tomharrisonjr.com/uuid-or-guid-as-primary-keys-be-careful-7b2aa3dcb439)

## 2.基于数据库自增ID
基于数据库的auto_increment自增ID完全可以充当分布式ID，具体实现：需要一个单独的MySQL实例用来生成ID，建表结构如下：
```sql
CREATE DATABASE `SEQ_ID`;
CREATE TABLE SEQID.SEQUENCE_ID (
    id bigint(20) unsigned NOT NULL auto_increment, 
    value char(10) NOT NULL default '',
    PRIMARY KEY (id),
) ENGINE=MyISAM;
```
```sql
insert into SEQUENCE_ID(value)  VALUES ('values');
```
当我们需要一个ID的时候，向表中插入一条记录返回主键ID。
#### 优点：
实现简单，ID单调自增，数值类型查询速度快
#### 缺点：
DB单点存在宕机风险，无法扛住高并发场景，这种方式访问量激增时MySQL本身就是系统的瓶颈，用它来实现分布式服务风险比较大。
如果公开（显示在页面或 URL 的一部分上），它可能会泄露商业智能数据。例如，如果我现在订购了一个ID为345的订单，而我在一个月后订购了一个ID为445 的订单，那么我可以推断该商店每月收到大约100个订单。

## 3.基于数据库集群模式
上面说了单点数据库方式不可取，那对上边的方式做一些高可用优化，换成主从模式集群。害怕一个主节点挂掉没法用，那就做双主模式集群，也就是两个Mysql实例都能单独的生产自增ID。
那这样还会有个问题，两个MySQL实例的自增ID都从1开始，会生成重复的ID怎么办？
解决方案：设置起始值和自增步长

MySQL_1 配置：
```sql
set @@auto_increment_offset = 1;     -- 起始值
set @@auto_increment_increment = 2;  -- 步长
```
MySQL_2 配置：
```sql
set @@auto_increment_offset = 2;     -- 起始值
set @@auto_increment_increment = 2;  -- 步长
```
这样两个MySQL实例的自增ID分别就是：
1、3、5、7、9 2、4、6、8、10

如果系统对性能和高并发有更高的要求就需要对MySQL扩容增加节点，这是一个比较麻烦的事。
水平扩展的数据库集群，有利于解决数据库单点压力的问题，同时为了ID生成特性，将自增步长按照机器数量来设置。
增加第三台MySQL实例需要人工修改一、二两台MySQL实例的起始值和步长，把第三台机器的ID起始生成位置设定在比现有最大自增ID的位置远一些，但必须在一、二两台MySQL实例ID还没有增长到第三台MySQL实例的起始ID值的时候，否则自增ID就要出现重复了，必要时可能还需要停机修改。

#### 优点：
解决DB单点问题
#### 缺点：
不利于后续扩容，而且实际上单个数据库自身压力还是大，依旧无法满足高并发场景。


## References
- https://zhuanlan.zhihu.com/p/107939861
- https://tech.meituan.com/2017/04/21/mt-leaf.html
- https://colobu.com/2020/02/21/ID-generator/
- https://zh.wikipedia.org/zh-hans/%E9%80%9A%E7%94%A8%E5%94%AF%E4%B8%80%E8%AF%86%E5%88%AB%E7%A0%81
- https://www.callicoder.com/distributed-unique-id-sequence-number-generator/
