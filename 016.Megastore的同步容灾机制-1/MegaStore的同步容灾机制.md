# MegaStore的同步容灾机制（1）

本文介绍MegaStore基于Paxos的同步容灾机制，内容涉及到对Paxos算法的简单理解，以及MegaStore为了优化读写操作而对Paxos算法的实现所做的一些优化。

## Paxos简介

在一个涉及**多数据副本**的分布式系统中，针对某一个Value，如何在多个副本间达成**共识**，是 Paxos算法要解决的核心问题。在分布式系统中，各种故障场景几乎是"家常便饭"，如网络延迟、网络故障、硬件故障等等，这往往会导致某些副本的数据出现异常，而利用Paxos算法，可以容忍F个副本出现异常，如果数据副本数共有2F+1的话。

如果详细读过Paxos的论文，就会发现，如果多个副本对某一个新写入的Value达成共识的话，共需要至少两轮交互过程：

* Prepare阶段 

  Proposer接收到超过半数的Acceptor的Promise消息，意味着该Proposer拥有为大多数Acceptor发送所建议的Value的权力了。

* Accept阶段

  Proposer正式往Acceptor发送数据，大多数Acceptor响应数据已接收，则该Value已被确认，即该Value在多数据副本间已达成了共识。

而对于读取操作，也需要首先经历一轮Prepare阶段来确定已在多副本间达成共识的Value。如果要在一个分布式数据库系统中实现Paxos算法，应该要尽量减少网络交互的次数来降低请求时延。

## 一个简单的Paxos实现思路

很多系统中在实现Paxos时，为了降低交互时延，采用了一种由**单Master节点代理**所有读或写的操作，该Master节点需要参与所有的读写操作，这样它的状态数据将始终是最新的。这样，Paxos的两轮交互可以被简化到一轮交互，因为它作为唯一的Proposer，Prepare阶段仅请求一次即可，后面的每一次写入均需要经历Accept阶段。同时，该Master节点还可以利用批量写入操作来提升整体的吞吐量。

明显，这种方案并不适用于海量数据背景下的分布式系统，因为Master是一个单点，另外，还需要考虑该Master节点的HA机制，如果采用另外的一个节点作为冷备的Master，原来的Master节点故障时会涉及到复杂的状态机数据同步与恢复机制，同时切换期间还会带来服务可用性问题。

## MegaStore的Paxos实现思路

MegaStore在实现Paxos时，优先考虑了如何能够支持高性能的读写操作，即论文中提及的Fast Reads与Fast Writes。我们可以看一下MegaStore如何达成的这种目标：

### Fast Reads

通常，一次写入操作在所有的副本中都是成功的，也就是说，在大多数场景下，每一个副本中都保持了完整的数据，这使得大多数读取操作可以直接从本地副本中读取数据，这样不仅可以很好的提升资源使用率，还可以降低读取的时延。

但不同副本之间的数据不一致也是时常发生的，因此，MegaStore设计了一个Coordinator的服务，来跟踪每一个Entity Group的多个副本间的数据一致性，如果多副本间数据是一致的，在Coordinator的一个集合中会记录该Entity Group，这样，用户的读写请求所涉及到的Entity Group，如果在Coordinator的集合中，则该Entity Group的副本可以提供本地读取能力。

## Fast Write

为了减少数据写入阶段的交互次数，MegaStore采用了Leader的机制，类似于前一章节提到的采用Master的机制，只不过在MegaStore中同时运行着多个Leaders，这样可以有效提升Paxos Write的吞吐量。

一个Writer在写入时，必须先与Leader进行交互。MegaStore在选择Leader的时候，会参考同一个可用区内的多个应用所选择的Leader信息，这样子，可以使得Leader通常为物理距离最近的一个副本。

### 副本类型

MegaStore支持了三种副本类型：

* Full Replica	可参与正常读写服务的Replica
* Witness Replica   仅仅参与选举以及WAL日志存储，但不存储Entity Data或Index Data。当可用副本数不足时，Witness Replica可协助写入操作达成共识。
* Read-Only Replica   仅用于读取的副本，可以用来读取近期的数据，而且是强一致性的视图，但无法读到实时的数据。

(未完待续..)

