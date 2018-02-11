关于Bigtable的讨论已是一个略显过时的话题，再次回顾Bigtable的论文，是想审视一下在这十余年的时间里，有哪些设计依然是经典的，以及哪些设计已是过时的。Bigtable的最新发展，对于外界而言一直都是神秘的，除了Google新发表的一些论文略有提及以及一些道听途说的消息之外，就只能从今年开放公测的Cloud Bigtable的官方资料中洞察秋毫。从MegaStore再到现在的Spanner，Bigtable的江湖地位已被大幅削弱，但仍有充足的退守之地。本文主要回顾总结Bigtable的核心设计。

**Outline**

[TOC]

## 设计目标

Bigtable的四个设计目标：

* Wide Applicability： 广泛的适用场景
* Scalability：横向扩展能力
* High Performance： 高性能
* High Availability： 高可用性

##数据模型

### Table

> A Bigtable is a **sparse**, **distributed**, **persistent multi-dimensional sorted map**. 

简单理解一下这句定义所阐述的几个关键点：

* 一个表是一个包含海量Key-Value对的Map，数据是持久化存储的。

* 这个大的Map需要支持多个分区来实现分布式。

* 这个Map按照Key进行排序，这个Key是一个由{Row Key, Column Key, Timestamp}组成的多维结构。

* 每一行列的组成并不是严格的结构，而是稀疏的，也就是说，行与行可以由不同的列组成：

  | Row  | Columns                    |
  | ---- | -------------------------- |
  | Row1 | {ID, Name, Phone}          |
  | Row2 | {ID, Name, Address, Title} |
  | Row3 | {ID, Address, Email}       |

### Row

* 每一行数据都拥有一个唯一的Row Key，可以将Row key理解为主键。
* Row Key是一个Byte String，通常长度在10～100Bytes左右，建议不超过4KB。一行中包含一个或多个列。
* Bigtable支持Row级别操作的原子性。
* 所有的数据按照Row Key的字典顺序进行排序。

### Tablet

* Table的横向数据分区称之为Tablet。
* Tablet是一个连续的Row Key区间。
* Tablet是数据分布与负载均衡的基本单元。
* 一个Tablet增长到一定大小之后可以自动分裂成两个Tablets。
* 多个连续的Tablets可以合并成一个大的Tablet。

### Column Family

* 权限控制的**最小单元**。
* 一个Column Family通常是一个或多个**相同类型**的列的集合，这样在数据压缩率上可以获取更好的效果。
* Column Families的数量不建议超过百级别。
* Column Key的组成结构为<u>Family:Qualifier</u>。Qualifier可以理解成一个Column Family中的列标识。

### Timestamp

* 每一个Column Key可能关联多次更新，因此，Bigtable使用Timestamp来标识不同的版本。
* 同一个Column Key的多个版本按Timestamp倒序存放，这样查询时总是先读取到最新的版本。
* 每一个Column Family允许用户配置最多保留的版本数量，超出的版本将会被清理掉。

## 逻辑架构

### 关键模块

* Client Library

* Master Server

  1）Tablet Server管理。

  2）Tablet到Tablet Server的分配。

  3）负载均衡。

  4）垃圾文件回收。

  5）建表以及Schema变更管理。

* Tablet Server

  管理Tablet的数据服务节点。

### 依赖服务

* GFS（Colossus）分布式文件系统。用来持久化存放Bigtable的关联文件（日志文件以及数据文件）。

* Chubby 

  1）Active Master选举。

  2）Tablet根路由信息。

  3）Tablet Server故障节点通知以及新节点发现。

  4）Table Schema信息。

  5）存储Access Control List信息。

###数据路由

* 在METADATA表中记录了每一个用户表Tablet所关联的Key Range信息以及路由信息。
* METADATA本质上也是一个Bigtable表，因此，自身也由一个或多个Tablet组成。
* Root表中记录了METADATA Tablet的路由信息，Root表有且只有一个Tablet组成。

## 关键设计

### LSM Tree

Bigtable的LSM实现：

* 数据在写入到Tablet之前先**顺序写入**到一个日志文件中。

* 每一个Tablet Server上的多个Tablets**共享**同一个日志文件。

* 数据成功被写入到日志文件之后，再写入到Tablet的Memtable(内存排序Map)中。

* 当Memtable中的数据达到一定的大小之后Flush到GFS中成为SSTable文件。SSTable解释如下：

  > SSTable is a simple abstraction to efficiently store large numbers of key-value pairs while optimizing for high throughput, sequential read/write workloads.

Bigtable的关键优化：

* 写日志时，通过Group Commit减少IOPS，提升写入性能。
* 利用两个写入线程避免GFS的写入时延毛刺。

### Locality Groups

* 一个Locality Group是多个Column Families（经常被一起访问）的组合。
* 同一个Locality Group中的数据会生成到同一个SSTable中。
* 可以将一个Locality Group配置成是否是常驻内存的。

### Compaction

Compaction的三种类型以及对应的作用：

* Minor Compaction: 将Memtable中的数据Flush成GFS中的SSTable文件，主要目的有两个：

  1) 减少Memtable的内存占用。

  2) 加速Tablet迁移或Tablet Server故障之后的因日志重放所消耗的时间。

* Merge Compaction: 将**<u>多个SSTable文件</u>**合并成一个大的SSTable文件，该过程**不回收**被标记删除的数据。

* Major Compaction: 将同一个路径下**<u>所有的SSTables</u>**合并成一个大的SSTable文件，该过程**回收**需要被清理的数据。

### BloomFilter

支持使用BloomFilter来加速关于不存在的Row Key或Column的查询，减少随机磁盘IO。

### Compression

Bigtable支持两层压缩机制：

* 利用Bentley-McIlroy Encoding算法预先对公共的Strings进行压缩编码。
* 在SSTable 16KB的Block级别采用一个快速的通用压缩算法。

### Cache

Bigtable支持两类Cache:

* Scan Cache 缓存Key-Value数据，主要针对频繁发起的相同查询。
* Block Cache 缓存STable的Block，主要是优化临近查询场景。




**References**

1. Bigtable: A Distributed Storage System for Structured Data

2. Cloud Bigtable Document: https://cloud.google.com/bigtable



