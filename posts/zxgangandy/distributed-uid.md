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

### 1. UUDI
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
- 无需中心化的服务器
- 不会泄漏商业机密

#### 缺点:
可读性差
占用空间太多(16个字节)
影响数据库的性能, 比如[UUID or GUID as Primary Keys? Be Careful!](https://tomharrisonjr.com/uuid-or-guid-as-primary-keys-be-careful-7b2aa3dcb439)



## References
- https://zhuanlan.zhihu.com/p/107939861
- https://tech.meituan.com/2017/04/21/mt-leaf.html
- https://colobu.com/2020/02/21/ID-generator/
