# Hadoop Token认证机制

Hadoop的身份认证机制，目前为止还仅支持Kerberos方案，这使得一些第三方认证服务难以接入到Hadoop中。因此，早在13年时，Intel团队就提出了一种基于通用Token的认证方案，但可惜该方案一直未能落地。本文简单介绍一下该方案的一些设计思路。

## Kerberos认证的痛点

如果在项目中实际应用过Hadoop Kerberos认证，则会体验到如下一些痛点问题：

1. 配置复杂且容易出错。
2. Client多重登录容易导致票据无法续约问题。
3. 如果每一个集群都安装了独立的Kerberos，当Client需要访问多个集群时，涉及到多集群间的互信方案。配置互信时需要重启KDC服务。

##方案大概思路

1. 将认证(Authentication)与授权(Authorization)机制**插件化**。
2. **Token Authentication Service(TAS)**，是对认证服务的包装，实际的认证工作可以由**第三方身份认证服务(IdP)**代理。
3. Client端的用户认证成功以后，由 Authentication Service为用户发放一个**Identity Token**。
4. Client凭借Identity Token去**Authorization Server(AS)**获取服务访问授权，授权成功之后，AS为Client发放**Access Token**。每一个集群关联一个**Authorization Server(AS)**。AS中配置了用户安全访问策略。
5. Client端凭借从AS获取到的Access Token去访问对应的服务（如HDFS/HBase）。
6. 每一个用户都有一个所关联的Domain，TAS中可以针对不同的Domain配置不同的IdP服务，这使得一个Hadoop集群中允许存在多种IdP服务。

基于该设计，不仅能够让Hadoop支持多种身份认证服务，而且更容易实现用户单点登录。

## 通用Token与现有Token的区别

在该方案之前，Hadoop中已经支持Token机制了，如Block Access Token, Delegation Token。Hadoop中现有的Token，可以理解成是一个内部的Token，例如，Block Access Token，其实仅限于一个HDFS集群内部使用，对于其它服务（如HBase）是无意义的。再者，现有的Token存在的前提，是必须先经Kerberos认证成功之后才可获取到，本质上，还是无法脱离Kerberos而存在。

而该方案中的通用Token，有更广泛的意义，能够像Kerberos TGT一样跨服务（HDFS/HBase/ZooKeeper）使用。在社区的早期讨论中，该方案的落地会分为两个阶段，第一阶段能够与现有的Token兼容存在，第二阶段，会将现有的Token升级为通用的Token格式。

## 总结

该方案拥有不错的设计思路，但遗憾未能很好的推动落地。随着公有云大数据服务的不断普及，相信该需求会越来越迫切，也许在不久后的Hadoop Minor Release中能看到该特性或者类似特性的落地。



*Reference*

1. https://issues.apache.org/jira/browse/HADOOP-9392
2. https://issues.apache.org/jira/browse/HADOOP-11766
3. https://issues.apache.org/jira/browse/HADOOP-14808
4. https://issues.apache.org/jira/browse/HADOOP-14831





