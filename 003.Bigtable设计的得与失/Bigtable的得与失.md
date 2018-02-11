# Bigtable设计的“得”与“失”

上篇文章[<回顾Bigtable的经典设计>](http://www.nosqlnotes.com/technotes/bigtable-keydesign/)简要总结了Bigtable的一些关键设计，本文将围绕一些关键设计，来探讨这些设计的“得”与“失”。

### Non-SQL接口

提供自定义的数据读写接口而非用户习惯的SQL接口，会带来额外的学习成本。

MegaStore论文中关于Bigtable接口的评价：

> Even though many projects happily use Bigtable, we have also consistently received complaints from users that Bigtable can be difficult to use for some kinds of applications: those that have complex, evolving schemas, or those that want strong consistency in the presence of wide-area replication.

### Schema Less

Bigtable本是为结构化数据存储而设计的，却采用了Schema Less的设计。论文中提及的关于网页数据存储的场景，应该是促成该设计的最大因素。

该设计的优势在于能够支持Schema的灵活变更，不需要预先定义Schema信息。

但该设计的缺点也非常明显：

* Key-Value中携带了充分的自我描述信息，导致数据有大量的膨胀。
* 在数据压缩粒度上比较受限。

### Range分区

优点：

* Range分区能够很好的保证数据在底层存储上与Row Key的顺序是一致的，对Scan类型查询比较友好。

缺点：

* 对用户Row Key的设计提出了非常高的要求。
* 容易导致数据不均匀。

### 事务支持

Bigtable论文中提到仅需要支持单行事务的设计初衷：

> We initially planned to support general-purpose transactions in our API. Because we did not have an immediate use for them, however, we did not implement them. Now that we have many real applications running on Bigtable, we have been able to examine their actual needs, and have discovered that most applications require only single-row transactions.

也正是在General-purpose Transaction的驱使下，产生了后来的MegaStore以及Spanner。Jeff Dean在2016年的采访中表示没有在Bigtable中支持分布式事务是最大的一个设计遗憾：

> **"What is your biggest mistake as an engineer?"** 
>
> Not putting distributed transactions in BigTable. If you wanted to update more than one row you had to roll your own transaction protocol. It wasn’t put in because it would have complicated the system design. In retrospect lots of teams wanted that capability and built their own with different degrees of success. We should have implemented transactions in the core system. It would have been useful internally as well. Spanner fixed this problem by adding transactions.  --  Jeff Dean, March 7th, 2016

从业界已广泛应用的HBase的应用场景来看，单行级别的事务还是可以满足大多数场景的，也正因为放弃了对复杂事务的支持，Bigtable/HBase才能够在整体吞吐量以及并发上的优势。

因此，尽管Jeff Dean将Bigtable不支持分布式事务视作是一大设计遗憾，这也只针对Google内部的应用场景需求而言，HBase的广泛应用恰恰佐证了Bigtable的这一取舍是一条正确的路。就像当年Cassandra因最终一致性的设计惨遭Facebook内部遗弃，但后来在DataStax的扶持下依然得到了大量用户的追捧。Google/Facebook内部的特殊场景，以及在这种特殊场景下所产生的技术与观点，在很多时候也并不具有普适性的。

### 计算与存储分离

Tablet Server中仅仅提供了数据读写服务入口，但并不存储任何数据，数据文件交由底层的GFS/Colossus来存储，可以说，这是一个典型的计算与存储分离的架构。

该架构优点：

* 分层设计，每一层更专注于自身的业务能力。
* 可更充分的利用每一层的资源，降低整体的成本。
* 更适合云上服务架构模型。

缺点：

* 更多的网络开销。

### 可用性

尽管存储于底层GFS中的文件有多个副本，但Tablet本身却是单副本的，当Tablet Server故障或因负载均衡原因对Tablet进行迁移时，就会导致Tablet短暂不能提供读写服务，带来可用性问题。



*References*

1. Bigtable: A Distributed Storage System for Structured Data
2. MegaStore: Providing Scalable, Highly Available Storage for Interactive Services
3. Spanner: Google’s Globally-Distributed Database
4. Cloud Bigtable Document: https://cloud.google.com/bigtable
5. http://highscalability.com/blog/2016/3/16/jeff-dean-on-large-scale-deep-learning-at-google.html


   ​			
   		
   ​	


​			
​		
​	

