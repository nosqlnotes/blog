# 分布式系统中引入Kerberos所涉及到的框架与技术



如果将Kerberos引入到一个分布式系统中提供身份认证与服务授权时，还涉及到一系列与安全有关的框架技术与接口协议，包括：如何对认证的对象或访问的资源进行抽象？应用通过什么样的接口进行Kerberos认证？如何保障跨节点通信时的安全传输？Java Security中均提供了现成的实现。

## JAAS

JAAS是Java Authentication and Authorization Service的缩写，提供了认证与授权相关的服务框架与接口定义:

- 认证：认证主要是验证一个用户的身份。
- 授权：授权用户访问或操作某些敏感的资源。

前一篇文章[<认证与授权-JAAS基础概念>](http://www.nosqlnotes.com/technotes/jaas-concept/)已经做了详细的介绍。

## GSS-API

Java GSS-API是 Generic Security Services Application Program Interface的缩写，主要应用于跨应用程序之间的安全信息交换(secure message exchanges)， [RFC 2853](http://www.ietf.org/rfc/rfc2853.txt)提供了详细的接口协议定义。应用程序可以通过GSS-API访问Kerberos服务。

JAAS的认证请求，通常先于GSS-API的调用。也就是说，先通过JAAS的接口完成登录认证，而后通过GSS-API来确保后面交换信息的安全性。两者经常被联合一起使用，但也可以单独使用，例如，基于JAAS也可以完成一些简单的认证与授权，一些别的场景下，也可以只使用GSS-API而不使用JAAS。

回忆一下之前一篇文章[<图解Kerberos协议原理>](http://www.nosqlnotes.com/technotes/kerberos-protocol/)中讲到的Kerberos认证与授权的4个关键步骤：

1. 用户输入用户名/密码信息进行登录。
2. Client到AS进行认证，获取TGT。
3. Client基于TGT以及Client/TGS Session Key去TGS去获取Service Ticket(Client-To-Server Ticket)。
4. Client基于 Client-To-Server Ticket以及Client/Server SessionKey去Server端请求服务。

第1步，在Client端完成对密码信息的单向加密。

第2步，基于JAAS进行认证。

第3,4步，则基于GSS-API进行交互。

## SASL

SASL是*Simple Authentication and Security Layer*的缩写，主要应用于**跨节点通信**时的认证与数据加密，如Client与Server基于RPC进行通信时，可以基于SASL来提供身份认证以及数据加密能力。

当SASL中使用Kerberos服务时，也需要使用到GSS-API来与Kerberos之间进行安全信息交换。

## 总结

本文主要介绍了JAAS,GSS-API以及SASL，简单总结如下：

* JAAS **认证与授权**服务框架与接口定义
* GSS-API 跨应用程序之间的**安全信息交换**，如Kerberos授权时Client与TGS/SS之间的信息交换。
* SASL 主要应用于**跨节点通信**时的认证与数据加密。



*References*

1. https://docs.oracle.com/javase/8/docs/technotes/guides/security/jgss/tutorials/index.html
2. https://docs.oracle.com/javase/8/docs/technotes/guides/security/jgss/tutorials/ClientServer.html
3. https://docs.oracle.com/javase/8/docs/technotes/guides/security/jgss/tutorials/BasicClientServer.html
4. https://docs.oracle.com/javase/8/docs/technotes/guides/security/sasl/sasl-refguide.html



