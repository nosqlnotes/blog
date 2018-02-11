# Hadoop认证与授权机制-设计概述

写这篇文章的时候，Hadoop 3.0的GA版本刚刚发布。Hadoop最初的认证与授权方案，是雅虎团队在2009年设计的，尽管已经过去了八年多的时间，但从最新的Hadoop 3.0版本的安全机制来看，基本还是维持了最初的设计思路。本文重点介绍该设计的一些关键点，不展开过多细节。

## 设计初衷

对于企业级的应用而言，**认证与鉴权**可以说是一个**基础需求**：

* 尽管Hadoop集群大多部署在企业内部私有网络环境中，有防火墙隔离，但并不能完全避免网络攻击。
* 内部运维人员在操作集群的时候，因为一些手误导致集群数据被删除的案例时有耳闻。
* 不同业务类型的数据存放在同一个集群中，这导致每一个集群用户都能看到其它业务相关的数据。

##需求描述

1. 用户只允许访问有权限的文件。
2. 用户只能访问或者修改该用户所拥有的MapReduce任务。
3. 服务与服务之间可以相互认证。
4. 用户认证成功之后能够获得Credentials，在服务请求时会用到Credentials，但应用无感知。

## 设计思路

### 认证

Hadoop的**身份认证服务**基于Kerberos实现。

为一个用户进行认证时，提供如下信息：

* Principal名称  用户名。 
* Keytab文件 可简单理解为用户密钥。

认证成功之后，可以获得对应的TGT(Ticket-Granting-Ticket)。

### Service Principal

Kerberos的Principal分为两类：

* User Principal
* Service Principal

User Principal就是普通应用用户。

Service Principal代表一个服务，如HDFS服务，HBase服务。

一个User Principal去请求一个Service Ticket时，本质上是请求这个服务所关联的Service Principal的Service Ticket。

###Client访问HDFS文件

1. Client访问NameNode时，需要进行Kerberos认证(基于前面的**TGT**，去Kerberos获取对应的Service Ticket)。
2. NameNode为Client响应的消息中，包含访问DataNode Block时的**Block Access Token**。
3. Client访问的DataNode时，基于**Block Access Token**即可访问，无需再次进行Kerberos认证。

> 关于Block Access Token：
>
> 1. Block Access Token相比于Kerberos TGT/Service Ticket更加轻量级，从而可减轻对Kerberos KDC节点的访问压力。
> 2. Block Access Token只在HDFS范围内有效，不可用于访问其它服务。
> 3. Block Access Token是有时效性的，过期后可续约。
> 4. 未来可以更好的与其它身份认证服务兼容。

### RPC

如前一篇文章[<分布式系统中引入Kerberos认证所涉及到的框架与技术>](http://www.nosqlnotes.com/technotes/distributed-system-with-kerberos/)中提到的，Client与NameNode/DataNode之间的RPC传输，基于SASL机制进行认证与数据加密，这里不再赘述。

### Delegation Token

 Client凭借Kerberos Ticket去NameNode成功认证之后，可以获取到一个Delegation Token。该Token可以理解为Client与NameNode之间的一个共享密钥。后续该用户所发起的Jobs可以凭借获取的Delegation Token，直接去NameNode进行认证，从而可绕过Kerberos认证流程。





















*Reference*

https://issues.apache.org/jira/browse/HADOOP-4487

https://issues.apache.org/jira/browse/HBASE-2016

https://issues.apache.org/jira/browse/HBASE-2742



