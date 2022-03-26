---
title: "分布式ID"
date: 2022-03-18
tags: [distributed, uid, id, micro-service]
author: zxgangandy
layout: zh-cn/layouts/post.njk
image: /img/posts/zxgangandy/uuid-shellcode/uuid-in-memory.png
---

<!-- summary -->
在我们的日常业务开发中，通常需要对一些数据做唯一标识，例如
- 大量抓取的文章ID
- 用户ID
- 微博ID
- 聊天消息ID
- 帖子ID
- 订单ID

通常会使用数据库自增的主键id作为唯一id。但是在并发量大、存在分库分表的情况或者是在微服务系统中我们通常会考虑使用分布式ID的生成方案来生成id。如果是小型应用，或者是业务量不大的情况下，单库单表完全可以支撑现有业务，数据再大一点搞个MySQL主从同步读写分离也能解决。此种情况不属于这里讨论的范畴。
<!-- summary -->

## 分布式ID的特点
- 全局唯一：必须保证ID是全局性唯一的，这是基本的要求
- 高性能：高性能就要求低延时，ID生成响应必须要快，否则整个id的生成过程反倒会成为业务瓶颈
- 趋势递增：由于我们的分布式ID，是用来标识数据唯一性的，所以多数时候会被定义为主键或者唯一索引。并且绝大多数互联网公司使用的数据库是"MySQL"，存储引擎为innoDB。对于B + Tree这个数据结构来讲，数据以自增顺序来写入的话，b+tree的结构不会时常被打乱重塑，存取效率是最高的。
- 信息安全：很难从生成的ID推导出一些业务特征，否则对某些业务来说存在一定的风险(比如爬虫顺序爬取或者是根据订单号推导下单量)
- 尽可能短：位数更短的ID在查询和存储等方面都有优势，为高性能提供基础保障

## 常见的分布式ID算法
- UUID
- 数据库自增ID
- 数据库多主模式
- 号段模式
- Redis
- 雪花算法（SnowFlake）

### 1. UUID
通用唯一识别码（Universally Unique Identifier，缩写：UUID）是用于计算机体系中以识别信息数目的全球唯一的一个128位标识符。
更详细的信息可以参考[wikipedia](https://en.wikipedia.org/wiki/Universally_unique_identifier)和[RFC](https://tools.ietf.org/html/rfc4122)文档。
#### 优点:
- 容易实现，产生快
- ID唯一(几乎不会产生重复id)
- 无需中心化的服务器，本地生成，没有网络消耗
- 不会泄漏商业机密
#### 缺点:
- 可读性差
- 不容易存储
- 占用空间太多(16个字节)，不符合MySQL官方主键要尽量越短越好的建议 
- 影响数据库的性能，不利于MySQL索引
### 2.基于数据库自增ID
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
- 实现简单
- ID单调自增，数值类型查询速度快
#### 缺点：
- DB单点存在宕机风险，无法扛住高并发场景 
- 这种方式访问量激增时MySQL本身就是系统的瓶颈
- ID有安全隐患，可能会泄露商业智能数据。例如，如果我现在订购了一个ID为123的订单，而我在一个月后订购了一个ID为223的订单，那么我可以推断该商店每月收到大约100个订单

### 3.基于数据库集群模式
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
这样两个MySQL实例的自增ID分别就是：1、3、5、7、9 2、4、6、8、10
如果系统对性能和高并发有更高的要求就需要对MySQL扩容增加节点，这是一个比较麻烦的事。水平扩展的数据库集群，有利于解决数据库单点压力的问题，同时为了ID生成特性，将自增步长按照机器数量来设置。增加第三台MySQL实例需要人工修改一、二两台MySQL实例的起始值和步长，把第三台机器的ID起始生成位置设定在比现有最大自增ID的位置远一些，但必须在一、二两台MySQL实例ID还没有增长到第三台MySQL实例的起始ID值的时候，否则自增ID就要出现重复了，必要时可能还需要停机修改。
#### 优点：
- 解决DB单点问题
- 简单
- ID递增
#### 缺点：
- 水平扩展困难，不利于后续扩容 
- 而且实际上单个数据库自身压力还是大，依旧无法满足高并发场景
- 安全系数也低
### 4.号段模式
号段模式是当下分布式ID生成器的主流实现方式之一，号段模式可以理解为从数据库批量的获取自增ID，每次从数据库取出一个号段范围，例如 (1,1000] 代表1000个ID，具体的业务服务将本号段，生成1~1000的自增ID并加载到内存。
表结构如下：
```sql
CREATE TABLE id_generator (
  id int(10) NOT NULL,
  max_id bigint(20) NOT NULL COMMENT '当前最大id',
  step int(20) NOT NULL COMMENT '号段的布长',
  biz_type	int(20) NOT NULL COMMENT '业务类型',
  version int(20) NOT NULL COMMENT '版本号',
  PRIMARY KEY (`id`)
) 
```
biz_type ：代表不同业务类型
max_id ：当前最大的可用id
step ：代表号段的长度
version ：是一个乐观锁，每次都更新version，保证并发时数据的正确性
等这批号段ID用完，再次向数据库申请新号段，对max_id字段做一次update操作，update max_id= max_id + step，update成功则说明新号段获取成功，新的号段范围是(max_id ,max_id +step]。update id_generator set max_id = #{max_id+step}, version = version + 1 where version = # {version} and biz_type = XXX。由于多业务端可能同时操作，所以采用版本号version乐观锁方式更新，这种分布式ID生成方式不强依赖于数据库，不会频繁的访问数据库，对数据库的压力小很多。
#### 优点：
- 避免了每次生成ID都要访问数据库并带来压力，提高了性能
#### 缺点：
- 属于本地生成策略，存在单点故障，服务重启造成ID不连续
### 5.基于Redis模式
Redis通过利用redis的 incr命令实现ID的原子性自增。
```sql  
    127.0.0.1:6379> set seq_id 1     // 初始化自增ID为1
    OK
    127.0.0.1:6379> incr seq_id      // 增加1，并返回递增后的数值
    (integer) 2
```
用redis实现需要注意一点，要考虑到redis持久化的问题。redis有两种持久化方式RDB和AOF。RDB会定时打一个快照进行持久化，假如连续自增但redis没及时持久化，而这会Redis挂掉了，重启Redis后会出现ID重复的情况。AOF会对每条写命令进行持久化，即使Redis挂掉了也不会出现ID重复的情况，但由于incr命令的特殊性，会导致Redis重启恢复的数据时间过长。
#### 优点：
- 性能比数据库好，能满足有序递增。
#### 缺点：
- 由于redis是内存的KV数据库，即使有AOF和RDB，但是依然会存在数据丢失，有可能会造成ID重复。
- 依赖于redis，redis要是不稳定，会影响ID生成。

适用：由于其性能比数据库好，但是有可能会出现ID重复和不稳定，这一块如果可以接受那么就可以使用。也适用于到了某个时间，比如每天都刷新ID，那么这个ID就需要重置，通过(Incr Today)，每天都会从0开始加。
### 6.基于雪花算法（Snowflake）模式
雪花算法（Snowflake）是twitter公司内部分布式项目采用的ID生成算法。Snowflake生成的是Long类型的ID，一个Long类型占8个字节，每个字节占8比特，也就是说一个Long类型占64个比特。
Snowflake ID组成结构：正数位（占1比特）+ 时间戳（占41比特）+ 机器ID（占5比特）+ 数据中心（占5比特）+ 自增值（占12比特），总共64比特组成的一个Long类型。

第一个bit位（1bit）：Java中long的最高位是符号位代表正负，正数是0，负数是1，一般生成ID都为正数，所以默认为0。
时间戳部分（41bit）：毫秒级的时间，不建议存当前时间戳，而是用（当前时间戳 - 固定开始时间戳）的差值，可以使产生的ID从更小的值开始；41位的时间戳可以使用69年，(1L << 41) / (1000L * 60 * 60 * 24 * 365) = 69年
工作机器id（10bit）：也被叫做workId，这个可以灵活配置，机房或者机器号组合都可以
序列号部分（12bit）：自增值支持同一毫秒内同一个节点可以生成4096个ID

根据这个算法的逻辑，封装为一个工具方法，那么各个业务应用可以直接使用该工具方法来获取分布式ID，只需保证每个业务应用有自己的工作机器id即可，而不需要单独去搭建一个获取分布式ID的应用。
Java版本的Snowflake算法简单实现：
```java
public class SnowFlake {

    /**
     * 起始的时间戳
     */
    private final static long START_TIMESTAMP = 1480166465631L;

    /**
     * 每一部分占用的位数
     */
    private final static long SEQUENCE_BIT = 12;   //序列号占用的位数
    private final static long MACHINE_BIT = 5;     //机器标识占用的位数
    private final static long DATA_CENTER_BIT = 5; //数据中心占用的位数

    /**
     * 每一部分的最大值
     */
    private final static long MAX_SEQUENCE = -1L ^ (-1L << SEQUENCE_BIT);
    private final static long MAX_MACHINE_NUM = -1L ^ (-1L << MACHINE_BIT);
    private final static long MAX_DATA_CENTER_NUM = -1L ^ (-1L << DATA_CENTER_BIT);

    /**
     * 每一部分向左的位移
     */
    private final static long MACHINE_LEFT = SEQUENCE_BIT;
    private final static long DATA_CENTER_LEFT = SEQUENCE_BIT + MACHINE_BIT;
    private final static long TIMESTAMP_LEFT = DATA_CENTER_LEFT + DATA_CENTER_BIT;

    private long dataCenterId;  //数据中心
    private long machineId;     //机器标识
    private long sequence = 0L; //序列号
    private long lastTimeStamp = -1L;  //上一次时间戳

    private long getNextMill() {
        long mill = getNewTimeStamp();
        while (mill <= lastTimeStamp) {
            mill = getNewTimeStamp();
        }
        return mill;
    }

    private long getNewTimeStamp() {
        return System.currentTimeMillis();
    }

    public SnowFlakeShortUrl(long dataCenterId, long machineId) {
        if (dataCenterId > MAX_DATA_CENTER_NUM || dataCenterId < 0) {
            throw new IllegalArgumentException("DtaCenterId can't be greater than MAX_DATA_CENTER_NUM or less than 0！");
        }
        if (machineId > MAX_MACHINE_NUM || machineId < 0) {
            throw new IllegalArgumentException("MachineId can't be greater than MAX_MACHINE_NUM or less than 0！");
        }
        this.dataCenterId = dataCenterId;
        this.machineId = machineId;
    }

    public synchronized long nextId() {
        long currTimeStamp = getNewTimeStamp();
        if (currTimeStamp < lastTimeStamp) {
            throw new RuntimeException("Clock moved backwards.  Refusing to generate id");
        }

        if (currTimeStamp == lastTimeStamp) {
            //相同毫秒内，序列号自增
            sequence = (sequence + 1) & MAX_SEQUENCE;
            //同一毫秒的序列数已经达到最大
            if (sequence == 0L) {
                currTimeStamp = getNextMill();
            }
        } else {
            //不同毫秒内，序列号置为0
            sequence = 0L;
        }

        lastTimeStamp = currTimeStamp;

        return (currTimeStamp - START_TIMESTAMP) << TIMESTAMP_LEFT //时间戳部分
                | dataCenterId << DATA_CENTER_LEFT       //数据中心部分
                | machineId << MACHINE_LEFT             //机器标识部分
                | sequence;                             //序列号部分
    }
    
    public static void main(String[] args) {
        SnowFlake snowFlake = new SnowFlake(2, 3);

        for (int i = 0; i < (1 << 4); i++) {
            System.out.println(snowFlake.nextId());
        }
    }
}
```
#### 优点：
- 毫秒数在高位，自增序列在低位，整个ID都是趋势递增的
- 可以根据自身业务特性分配bit位，非常灵活
- 存储少，8个字节
- 可读性高
- 性能好，可以中心化的产生ID，也可以独立节点生成
#### 缺点：
- 强依赖机器时钟，如果机器上时钟回拨，会导致发号重复或者服务不可用
- ID生成有规律性，信息容易泄漏
## References
- https://zhuanlan.zhihu.com/p/107939861
- https://tech.meituan.com/2017/04/21/mt-leaf.html
- https://colobu.com/2020/02/21/ID-generator/
- https://zh.wikipedia.org/zh-hans/%E9%80%9A%E7%94%A8%E5%94%AF%E4%B8%80%E8%AF%86%E5%88%AB%E7%A0%81
- https://www.callicoder.com/distributed-unique-id-sequence-number-generator/
