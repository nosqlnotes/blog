# MegaStore有哪些值得借鉴的设计

对比于Spanner，MegaStore略显"过气"，但MegaStore论文刚被发表出来时，的确带给了大家很多启发，很多NewSQL数据库的设计都或多或少的从MegaStore中做了借鉴，包括Spanner。本文主要从大的方面粗谈MegaStore的一些关键设计，暂不展开过多的细节。

## Schema与类SQL接口

Bigtable本是为结构化数据而设计，但为了设计的精简性，舍弃了对SQL接口以及Schema能力的支持。从RDBMS到Bigtable，从性能上有了质的提升，但对于熟悉SQL的传统应用而言，在易用性上的体验可以说是一个巨大的落差。

MegaStore在NoSQL与RDBMS之间做了折中，即支持了NoSQL的横向扩展能力，又借鉴了RDBMS的易用性。从论文所提供的信息来看，MegaStore在Bigtable的接口基础上，支持了简单的类SQL接口。对于SQL或类SQL，Schema其实是一个基础需求，而结构化数据的Schema本来就是相对固定的，支持Schema可以说是一个更佳的选择。同时，支持Schema能够带来另外一个优势：能够结合数据类型来选择更佳的编码/压缩算法。

##同步容灾能力

对于海量用户数据，能够基于Paxos协议来支持跨集群的**同步容灾能力**，可以说是MegaStore设计中最具有革命意义的一次尝试。正如论文中这么描述：

> While many systems use Paxos solely for locking, master election, or replication of metadata and configurations, we believe that Megastore is the largest system deployed that uses Paxos to replicate primary user data across datacenters on every write.

MegaStore可以说是在多个Bigtable集群之上，基于Paxos协议，实现了跨集群间的同步容灾能力，而在一个集群内部，Bigtable的数据文件在GFS/Colossus中也是三副本存储的，从架构上而言，这里涉及到两个跨副本的一致性协议实现：MegaStore基于Paxos实现的一致性协议，以及GFS/Colossus内部的多副本一致性协议。Facebook曾在HBase之上基于Raft设计了类似的HydraBase方案，但该方案最后因运维方面的巨大复杂性而被Facebook内部放弃。

##跨行事务

在事务能力上，MegaStore在Bigtable的单行事务基础上，可以说向前迈进了一大步。

### 简单跨行事务

在一个Entity Group内部，可以支持跨行事务。一个Bigtable Cluster中的一个Entity Group中的数据，本身属于一个Bigtable的Tablet的，因此，一个Entity Group的跨行事务，本质上属于一个进程内部的事务，因此，这一点是比较容易实现的。

### Two-Phase Commit

跨Entity Group之间的多行事务，需要基于两阶段提交机制。但MegaStore并不推荐使用该类事务，因为时延过高，再者会加剧资源/锁竞争。

## 二级索引

对于结构化数据应用而言，二级索引也可以说是一个必备特性，Bigtable基于Key的单维度查询能力对应用而言的确是非常受限的，因此，二级索引提供了基于非Key字段的查询能力。MegaStore中支持如下几种类型的索引：

### Local Index

Entity Group级别内的索引。该索引主要用来加速一个Entity Group内部的查询。

### Global Index

Global Index中的索引数据可横跨所有的Entity Groups。也就是说，给定索引列的值进行查询时，查询到的任意一条数据可能源自任意一个Entity Group。

### Repeated Indexes

如果一个列有多个值，那么基于Repeated Index可能会产生多行索引数据。

### Inline Indexes

由于MegaStore采用了一种级联的Data Model来优化特定的关联查询场景（在后面的文章中会详细讨论该模型），在这种级联场景下，将一个Parent Table以及多个Child Table关联在一个Bigtable表中，它们通过特定的相同字段进行关联。利用Inline Indexes，可以将Child Table中的某些列提取到Parent Table的行中，来加速某些场景的查询。

## 总结

本文主要从Schema、类SQL接口、基于Paxos的同步容灾能力、跨行事务以及二级索引等方面探讨了MegaStore的一些值得借鉴的设计，在后面的文章中将会展开更多的细节。



*References*

1. Megastore: Providing Scalable, Highly Available Storage for Interactive Services