#从HBase中移除WAL？3D XPoint带来的变革

最近，Intel在HBase社区提交了一个标题为"WALLess HBase on Persistent Memory"的问题单，将3D XPoint技术引入到HBase中，并且移除了WAL。虽然方案还没有公布详细的设计细节，本文借机讨论HBase现有架构的一些痛点，以及利用3D XPoint技术可能为HBase带来的一些变革。

##回顾LSM-Tree

LSM-Tree设计源自Patrick O‘Neil的论文"The Log-Structured Merge-Tree"，自从Bigtable论文发布以后，该设计又被广泛应用于更多的NoSQL系统中。LSM-Tree利用了传统机械硬盘的“**顺序读写速度远高于随机读写速度**”的特点，思路如下：

* 硬盘上存储的数据根据业务需求按顺序组织，实时写入的数据如果要即时修改存放于硬盘上的文件，会导致性能低下，因为会带来大量的随机IO。因此，实时写入的数据暂时缓存在内存中的一个排序集合中，而不是直接去修改底层存储的文件。
* 数据缓存在内存中是不可靠的，因此，新写入的数据也会被写入到一个称之为WAL(Write-Ahead-Log)的日志文件中，WAL的日志按**时间顺序**组织。顾名思义，数据应该先写WAL再写内存，早期版本的HBase的确也是这么设计的，但后来出于性能的考虑，HBase将这两者的写入顺序给颠倒了。
* 缓存在内存中的数据达到一定的阈值之后，被Flush到硬盘中成为一个新的文件。
* 随着Flush次数的不断增多，文件数量越来越多，而读取时，每一个文件都可能需要被查找（例如，基于Key查询时，每一个文件中都可能包含该Key，因此，每一个文件都需要查看一下），这导致读取性能底下。为了改进读取的性能，当文件数量达到一定大小以后，多个文件会被合并成一个大文件，这里的合并带来了一些IO资源的浪费。当然，每一次合并时并不需要合并所有的文件，这里讲究一定的策略，主要是筛选文件的策略，而一个NoSQL系统通常支持多种可配置的策略。

WAL源自LSM-Tree，但因为WAL的数据基于“时间顺序”组织，它还可以被用于其它用途：

* 用于跨集群的容灾备份
* 增量备份
* 跨系统数据同步


但LSM Tree存储模型存在如下一些缺点：

1. 基于传统机械硬盘的IO模型设计，对于新型的存储介质，性能优势无法很好的发挥出来

2. Compaction阶段的层层合并带来大量的IO资源浪费

3. 对读取并不太友好，尤其是存在大量的待合并文件时

4. 故障恢复时，Replay-WAL通常需要耗费数十秒甚至数分钟的时间

5. LSM Tree的架构仅适合存储单行记录较小的数据（最好小于KB级别），对于偏大的数据，IO冗余太大


除了LSM-Tree，bitcask也是一种常见的存储模型，两者最明显的区别为：LSM-Tree是一种有序的数据组织，而bitcask却是无序的，对于两种模型更详细的对比，本文暂不展开。

##HBase现有架构的痛点

HBase的现有运行架构，从上到下可以理解为三层：

* <u>数据库逻辑实现层</u>： HBase的核心数据存储逻辑实现


* <u>数据复制层</u>：通过HDFS实现数据的多副本复制


* <u>本地存储层</u>：硬盘，本地文件系统以及DataNode所在层

分层的架构，有利于一个系统的快速构建，但往往带来了一些性能上的劣势。HBase的现有架构，存在如下几个痛点问题：

* **无法发挥新型存储介质的IO优势**

基于Sata/Sas盘存储时，容易发现部分磁盘的 IOPS会是瓶颈，一个简单的思路是将Sata/Sas盘直接替换为SSD/PCIe SSD的，但实际对比测试后会发现性能提升并不是特别明显，尤其是PCIe-SSD的IO优势很难被发挥出来，这里有Java GC的原因（因此从HBase 2.0开始，尽可能多的使用Offheap区的内存来降低Heap区的GC压力），也有架构设计上的原因（如现有的RPC/WAL/MVCC等模块以及Lock机制均存在改进空间）。值得借鉴的是，与新型硬件的结合上，ScyllaDB提供了更先进的思路：分区与CPU Core绑定，用户空间的IO调度使得底层的IO能够得到更充分的利用，对NUMA架构友好，利用DPDK提升包处理效率等，正是基于这些优秀的设计，使得ScyllaDB的性能能够达到Cassandra的10倍以上。但可惜，这些思路对于HBase而言，都是要动摇架构根基的。

Facebook在Fast14上提交的论文《Analysis of HDFS Under HBase: A Facebook Messages Case Study》中，针对Facebook Messages(缩写为FM)系统的业务模型，通过模拟的方法做了系统的测试，最终给出了一些有价值的结论：

> * FM的业务读写比为**99:1**，而最终反映到磁盘中，读写比却变为了**36:64**。WAL，HDFS Replication，Compaction以及Caching，共同导致了**磁盘写IO的显著放大**。
> * 建议**使用Flash作为Caching**，对于读多写少的应用而言通常能带来明显的性能改善。
> * **Local Compaction**可以降低至少一半的网络IO：将Compaction下移到本地存储层，这样，Compaction执行时读取数据不再通过网络IO而是直接从本地磁盘读取。
> * **Combined Logging**能使得写WAL的时延有6倍的提升： 现在的方案中，一个RegionServer一个实时写入的WAL(暂不考虑Multi-WAL特性的前提下)，多个RegionServer则有多个WAL，对应于HDFS中的多个不同的文件，这会导致一些多余的磁盘seeks操作，因此，Combined Logging考虑将多个RegionServer的WAL合并成一个WAL，测试表明，写WAL的时延能有6倍的提升，但该能力对于读取时延的提升却几乎无帮助。当然，这会带来额外的系统复杂度。
>

* **低CPU使用率**


该问题对于HBase而言，可以说是诟病已久，现有的架构是导致该问题的关键原因。基于虚拟机或容器的公有云HBase服务倒是一个不错的解决方案，可以根据用户实际需求分配合理的计算资源，避免计算资源浪费。


* **读写可用性偏弱**


HBase依赖于HDFS的多副本机制来保障数据的可靠性，Region层其实是单副本的，而每当RegionServer故障时，平均故障恢复时间(MTTR)通常在几十秒甚至几分钟之间，社区花费了大量的精力来优化MTTR过程的每一步骤，但难有质的提升，这与HBase的LSM-Tree架构有关，也与HBase的“强一致性”语义承诺有关。

Region Replica特性使得Region层能够支持多副本机制，但主副本与从副本之间的数据却不是实时同步的，在允许弱一致性读取时，对于读的可用性是可以提升的，但应用场景受限。HBase现有的跨集群容灾能力是异步的（同步容灾特性目前正在开发中，由小米主导），但集群切换对应用端是感知的，在实际应用中常被用来提升数据的可靠性而不是提升服务的可用性。Facebook曾经提过HydraBase的方案，在HBase层实现Raft协议，从而可以支持多HBase集群间的数据同步，在读写可用性上均可带来提升，但因该方案在运维上的巨大复杂度，导致方案最终被放弃。

* **读取性能与数据本地化率强相关**

既然数据本地化率是影响读取性能的关键要素，这就要求HBase与HDFS采用计算与存储"逻辑分离，物理融合”的部署，而这的确也是常见的思路，但在公有云服务中，计算与存储常常物理分离来降低成本，这使得数据本地化率不再有任何意义。在物理融合的部署思路中，RegionServer与DataNode被部署在同一个物理机/虚拟机中，但偶尔的服务故障，或者是Region迁移，都可能导致本地化率下降。


## 什么是3D XPoint

如下信息摘自《从Intel Optane系列Coldstream和AEP谈全闪存性能优化》一文：

> 3D XPoint作为一种SCM(存储级内存)，相比做内存**DRAM**，具备数据掉电也不丢失特性；相比传统**SSD**的**NAND Flash**，不但读写速度更快，而且还支持字节级访问(因为**NAND Flash**要求必须按照**Page**读写，按照几百个**Page**的**Block**擦除)。
>
> Intel称**Optane Memory**(Optane为Intel 3D XPoint技术对应的产品名称)为**Apache Pass**(**AEP**)，为高性能和灵活性而设计的革命性的SCM，称**Optane NVMe SSD**为**Clodstream**，为世界上最快、可用性和可服务性最好的SSD。
>
> 3D XPoint(**包括Apache Pass DIMM和Clodstream SSD**)**位于DRAM和NAND之间**，填补DRAM和NAND之间的性能和时延GAP。理论上，3D XPoint可以比NAND快**100-1000**倍，Apache Pass DIMM比DRAM高出**8-10**倍，其成本优势明显；与DRAM相比，其具有**非易失性**的另一大优势。与Flash相比，由于写入方式不同，Apache Pass DIMM也比Flash NAND更耐用。

##从HBase中移除WAL？

既然3D XPoint的随机IOPS性能足够彪悍，那么，设计干脆来的“任性”一些，直接将WAL移除好了，这就是由Intel Anoop提交的HBASE-20003中的思路。虽然该方案的设计细节还没有被公布出来，但从现有的信息可以推断：

* 数据写入到Memstore，而Memstore中的数据也被同步持久化到本地AEP中，写入过程中不再有WAL。也就是说，写入过程中降低了对HDFS的依赖。

* Memstore是否直接被Flush成HDFS中的HFile？还是先在AEP中缓存一定数量的HFile经Compaction过程写入到HDFS中？

  如果是后者，则完全可以借用In-memory Flush/Compaction特性，来降低HDFS层的IOPS。

* 数据既然被持久化到本地AEP中，那么，如何保障数据的可靠性？

  这里可以在现有Region Replica(HBASE-10070)特性基础上，实现不同的Region Replica的MemStore之间的同步复制。假设有3个副本，这3个副本构成一个Pipeline，这Pipeline中有严格预定义的顺序，第一个副本称之为Leader，随后的两个副本暂且称之为Follower 1, Follower 2。当Commit的时候，按照从后往前的顺序Commit，即，Commit的顺序为Follower 2, Follower 1, Leader。关于这里的数据共识机制，以及Leader Replica所在的RegionServer异常之后如何让原来的Follower Replica升为Leader，是整个方案中的难点之一。

* 既然数据可以在Region的多个副本中进行同步复制，读写可用性均会带来至少1个9的提升。

* Java如何直接访问AEP?  Github中有一个关于Java访问pmem/AEP的项目[**"pcj"**](https://github.com/pmem/pcj)，该项目似乎还在开发中。


如上是关于该方案的一些推断信息。前文提到了WAL的一些其它用途，如Replication，增量备份，跨系统数据同步，如果没有了WAL，这些能力如何实现？另外，写入尽可能的降低对HDFS的依赖以后，CPU是否成为了新的瓶颈？从问题单的信息来看，该方案原型开发以及测试正在进行中，期待能在即将公布的设计文档中获取更多有价值的信息。虽然该特性难以在短期内落地，但的确称得上是一个足以动摇HBase现有架构的大特性。




*Reference*

1. [WALLess HBase on Persistent Memory](https://issues.apache.org/jira/browse/HBASE-20003)
2. Analysis of HDFS Under HBase: A Facebook Messages Case Study
3. Patrick O‘Neil: The Log-Structured Merge-Tree
4. [HBase read high-availability using timeline-consistent region replicas](https://issues.apache.org/jira/browse/HBASE-10070)
5. [从Intel Optane系列Coldstream和AEP谈全闪存性能优化](http://www.10tiao.com/html/683/201708/2650716314/1.html)
6. [Intel Optane  Memory产品规范说明文档](https://ark.intel.com/zh-cn/products/99742/Intel-Optane-Memory-Series-32GB-M_2-80mm-PCIe-3_0-20nm-3D-Xpoint)