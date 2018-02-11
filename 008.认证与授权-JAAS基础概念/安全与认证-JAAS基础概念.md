*Java Security*

[TOC]

Java Security是一个与安全机制有关的框架，很多Java项目的安全认证使用到了该框架，如Hadoop Security的实现。本文主要探讨Java Security的一些基础概念及应用场景。



# 基础概念

## Subject

一个Subject代表一个实体对象与安全有关的信息，主要涉及：

* 身份信息
* 密码信息
* 加密秘钥信息

一个Subject可能涉及一个或多个身份，一个身份被称之为Principal。也就是说，一个Subject可能关联一个或多个Principals。

一个Subject可能涉及与安全有关的凭据信息，称之为Credentials。敏感的Credentials需要特别的保护措施，例如，私有秘钥信息，需要保存在<u>私钥集合</u>中。而关于公有秘钥信息(无论是私钥还是公钥，都称之为Credentials)，则被保存在另外一个<u>公钥集合</u>中。

 常用方法如下所示：

```java
Subject subject;
Principal principal;
Object credential;

// 获取所有的Principals列表，并且加入新的Principal。
subject.getPrincipals().add(principal);
// 获取公钥列表信息，并加入新的公钥列表。
subject.getPublicCredentials().add(credential);
```

## Credential





## LoginContext

LoginContext是针对一个Subject对象的认证上下文信息，认证过程也主要是针对一个Subject对象的认证。每一个LoginContext都关联一个<u>Context Name</u>，LoginContext中提供了基础的认证方法的定义。LoginContext的关键方法如下：

* login  登录/认证
* logout  登出
* getSubject  获取认证之后所创建的Subject对象信息

注意：LoginContext仅仅提供了基础的认证方法的定义，具体的认证方法的实现，是由下一章节讲到的LoginModule的实现者提供的。



初始化一个LoginContext对象时，可以提供如下几个信息：

* Context Name[必选]

* Subject[可选]

* CallbackHandler[可选]  

* Configuration[可选]


# LoginModule

## LoginModule关键方法

* LoginModule#login

`boolean login() throws LoginException`

为Subject进行认证。提示用户名和密码信息，校验用户密码，这个方法将会在LoginModule层面暂时保存认证结果。

* LoginModule#commit

`boolean commit() throws LoginException;`

认证成功之后，提交认证的结果。通过LoginModule中暂时保存的认证结果信息来确认认证结果。

Commit方法将Subject内对应的Principals以及Credentials关联起来。

如果Login方法认证失败的话，则该方法将会清理在LoginModule中保存的认证结果信息。 

## Krb5LoginModule

* 只有当#commit方法被调用之后，KerberosTicket信息将会被添加到Subject的Private Credentials列表中。
* 如果设置了storeKey选项为true，则KerberosKey或Keytab信息将会被添加到Subject的Private Credentials列表中：
  * KerberosKey  是由用户密码派生出来的信息。
  * Keytab  每一个Keytab文件都被限制只被一个Principal使用，除非Principal Value的值被设置为*。


* doNotPromp用来限制是否给户密码提示信息。

* Principal名称，有如下几种情形：

  * 一个简单的User Name.
  * 一个Service Name， 例如host/mission.eng.sun.com
  * \*    任何名称可用。

  Principal的名称可以通过系统属性sun.security.krb5.principal来配置，也可以通过Configuration指定。

* useTicketCache 如果为true，则可以从Ticket Cache中获取对应的TGT。这个Cache存放于如下目录下：

  * Linux/Solaris  /tmp/krb5cc_uid  uid是一个数字类型的identifier。
  * Windows  {user.home}{file.separator}krb5cc_{user.name}

  可以通过ticketCache重新指定Ticket Cache的路径。

一些高级配置项用来实现跨不同的Authentication Module之间共享Username以及Password。

* Username存储在Shared State中的"javax.security.auth.login.name"
* Password存储在Shared State中的"javax.security.auth.login.password"

如下是各个配置参数为**true**时的相关说明：

* useFirstPass 

  1. 利用从Shared State中获取的Username以及Password信息，尝试认证。
  2. 如果认证失败，不重试，直接将认证失败的信息反馈给调用者。

* tryFirstPass

  1. 利用从Shared State中获取的Username以及Password信息，尝试第一次认证。
  2. 如果认证失败，通过CallbackHandler去获取一个新的Username以及Password，尝试第二次认证。
  3. 如果第二次认证也失败，则将认证失败的信息反馈给调用者。

* storePass

  * 认证成功之后，将Username以及Password信息保存在该LoginModule中。
  * 如果Username/Password信息已经存在，则不需要再保存，认证失败也不保存。

* clearPass

  当Authentication的Login以及Commit阶段都完成之后，从该LoginModule中清理Username以及Password信息。

如果存在多个获取Ticket/Key的机制，则优先级顺序为：

1. Ticket Cache
2. Keytab
3. Shared State
4. User Prompt

>疑问：Initiator与Acceptor的区别是什么？



> - `Krb5LoginModule` for authentication using Kerberos protocols
> - `JndiLoginModule` for user name/password authentication using LDAP or NIS databases
> - `KeyStoreLoginModule` for logging into any type of key store, including a PKCS#11 token key store



### Keytab





# Kerberos概念





## Keytab

Keytab中所包含的信息：

* Principal Name
* EncryptionKey   



KerberosKey由Principal以及EncryptionKey信息一起来构造。

```java
public KerberosKey[] getKeys(KerberosPrincipal principal) {
  try {
    if (princ != null && !principal.equals(princ)) {
      return new KerberosKey[0];
    }
    PrincipalName pn = new PrincipalName(principal.getName());
    EncryptionKey[] keys = takeSnapshot().readServiceKeys(pn);
    KerberosKey[] kks = new KerberosKey[keys.length];
    for (int i=0; i<kks.length; i++) {
      Integer tmp = keys[i].getKeyVersionNumber();
      // 基于Principal以及EncryptionKey信息构建KerberosKey.
      kks[i] = new KerberosKey(
        principal,
        keys[i].getBytes(),
        keys[i].getEType(),
        tmp == null ? 0 : tmp.intValue());
      keys[i].destroy();
    }
    return kks;
  } catch (RealmException re) {
    return new KerberosKey[0];
  }
}
```







## Domain与Realm

EXAMPLE.COM可哦难过以称之为Domain Name，也可以将所有位于EXAMPLE.COM下的Hostnames名称设置为同一个Realm Name为EXAMPLE.COM下面。

如果需要多个Kerberos Realms，则建议使用如下类似的名称：

* BOSTON.EXAMPLE.COM
* HOUSTON.EXAMPLE.COM



将一个Hostname映射成Realm有三种方式：

* 通过在krb5.conf的[domain_realm]部分设置映射关系。
* 使用KDC Host-based Service referrals。
* 如果dns_lookup_realm在krb5.conf中已被启用，则基于DNS TXT记录信息

## KerberosTicket

包含的一些关键信息：

* asn1Encoding ASN.1 encoding of the ticket as defined by the kerberos protocol specification.
* sessionKey The raw bytes for the session key that must be used to encrypt
* flags
* authTime Time of initial authentication.
* startTime Time after which the ticket is valid.
* endTime Time after which the ticket will not be honored.
* renewTill   For renewable tickets it indicates the maximum endtime that may be included in a renewal. It can be thought of as the absolute expiration time for the ticket, including all renewals. This field may be null for tickets that was not renewable.
* client(KerberosPrincipal)  Client that owns the service ticket.
* server(KerberosPrincipal)  The service for which the ticket was issued.











*Reference:*

1. MIT Kerberos Document: Realm configuration decisions





# 深入Login流程





# 如何应用Java Security?









# Sasl

> Simple Authentication and Security Layer (SASL) is an Internet standard that specifies a protocol for authentication and optional establishment of a security layer between client and server applications. SASL defines how authentication data is to be exchanged, but does not itself specify the contents of that data. It is a framework into which specific authentication mechanisms that specify the contents and semantics of the authentication data can fit. There are a number of standard SASL mechanisms defined by the Internet community for various security levels and deployment scenarios.
>
> The Java SASL API defines classes and interfaces for applications that use SASL mechanisms. It is defined to be mechanism-neutral; an application that uses the API need not be hardwired into using any particular SASL mechanism. Applications can select the mechanism to use based on desired security features. The API supports both client and server applications. The `javax.security.sasl.Sasl` class is used to create `SaslClient` and `SaslServer`objects.
>
> SASL mechanism implementations are supplied in provider packages. Each provider may support one or more SASL mechanisms and is registered and invoked via the standard provider architecture.
>
> The Java platform includes a built-in provider that implements the following SASL mechanisms:
>
> - CRAM-MD5, DIGEST-MD5, EXTERNAL, GSSAPI, NTLM, and PLAIN client mechanisms
> - CRAM-MD5, DIGEST-MD5, GSSAPI, and NTLM server mechanisms



# GSS-API

> The Java platform contains an API with the Java language bindings for the Generic Security Service Application Programming Interface (GSS-API). GSS-API offers application programmers uniform access to security services atop a variety of underlying security mechanisms. The Java GSS-API currently requires use of a Kerberos v5 mechanism, and the Java platform includes a built-in implementation of this mechanism. At this time, it is not possible to plug in additional mechanisms. Note: The `Krb5LoginModule` mentioned in Section 6 can be used in conjunction with the GSS Kerberos mechanism.
>
> The Java platform also includes a built-in implementation of the Simple and Protected GSSAPI Negotiation Mechanism (SPNEGO) GSS-API mechanism.
>
> Before two applications can use the Java GSS-API to securely exchange messages between them, they must establish a joint security context. The context encapsulates shared state information that might include, for example, cryptographic keys. Both applications create and use an`org.ietf.jgss.GSSContext` object to establish and maintain the shared information that makes up the security context. Once a security context has been established, it can be used to prepare secure messages for exchange.
>
> The Java GSS APIs are in the `org.ietf.jgss` package. The Java platform also defines basic Kerberos classes, like `KerberosPrincipal`, `KerberosTicket`, `KerberosKey`, and `KeyTab`, which are located in the `javax.security.auth.kerberos` package.

# GSS-API与JAAS

> The reason both JAAS and Java GSS-API tutorials are presented together is because JAAS authentication is typically performed prior to secure communication using Java GSS-API. Thus JAAS and Java GSS-API are related and often used together. However, it is possible for applications to use JAAS without Java GSS-API, and it is also possible to use Java GSS-API without JAAS. Furthermore, JAAS itself can be used simply for authentication or for both authentication and authorization.



> Since the underlying authentication and secure communication technology used by this tutorial is Kerberos V5, we use Kerberos-style [principal](https://docs.oracle.com/javase/7/docs/technotes/guides/security/jgss/tutorials/glossary.html) names wherever a user or service is called for.
>
> For example, when you run SampleClient you are asked to provide your **user name**. Your Kerberos-style user name is simply the user name you were assigned for Kerberos authentication. It consists of a base user name (like "mjones") followed by an "@" and your realm (like "mjones@KRBNT-OPERATIONS.EXAMPLE.COM").
>
> A server program like SampleServer is typically considered to offer a "service" and to be run on behalf of a particular "**service principal**." A service principal name for SampleServer is needed in several places:
>
> - When you run SampleServer, and SampleClient attempts a connection to it, the underlying Kerberos mechanism will attempt to authenticate to the Kerberos KDC. It prompts you to log in. You should log in as the appropriate service principal.
> - When you run SampleClient, one of the arguments is the service principal name. This is needed so SampleClient can initiate establishment of a security context with the appropriate service.
> - If the SampleClient and SampleServer programs were run with a security manager (they're not for this tutorial), the client and server policy files would each require a ServicePermission with name equal to the service principal name and action equal to "initiate" or "accept" (for initiating or accepting establishment of a security context).
>
> Throughout this document, and in the accompanying login configuration file,
>
> ```
> service_principal@your_realm
> ```
>
> is used as a placeholder to be replaced by the actual name to be used in your environment. 
>
> Any
>
>  Kerberos principal can actually be used for the service principal name. So 
>
> for the purposes of trying out this tutorial, you could use your user name as both the client user name and the service principal name.
>
> In a production environment, system administrators typically like servers to be run as specific principals only and may assign a particular name to be used. Often the Kerberos-style service principal name assigned is of the form
>
> ```
> service_name/machine_name@realm; 
> ```
>
> For example, an nfs service run on a machine named "raven" in the realm named "KRBNT-OPERATIONS.EXAMPLE.COM" could have the service principal name
>
> ```
> nfs/raven@KRBNT-OPERATIONS.EXAMPLE.COM
> ```
>
> Such multi-component names are not required, however. Single-component names, just like those of user principals, can be used. For example, an installation might use the same ftp service principal `ftp@realm` for all ftp servers in that realm, while another installation might have different ftp principals for different ftp servers, such as `ftp/host1@realm` and `ftp/host2@realm` on machines `host1` and `host2`, respectively.



> Before two applications can use Java GSS-API to securely exchange messages between them, they must establish a joint security context using their credentials. (Note: In the case of SampleClient, the credentials were established when the Login utility authenticated the user on whose behalf the SampleClient was run, and similarly for SampleServer.) The security context encapsulates shared state information that might include, for example, cryptographic keys. One use of such keys might be to encrypt messages to be exchanged, if encryption is requested.
>
> As part of the security context establishment, the context initiator (in our case, SampleClient) is authenticated to the acceptor (SampleServer), and may require that the acceptor also be authenticated back to the initiator, in which case we say that "mutual authentication" took place.
>
> Both applications create and use a **GSSContext** object to establish and maintain the shared information that makes up the security context.
>
> The instantiation of the context object is done differently by the context initiator and the context acceptor. After the initiator instantiates a GSSContext, it may choose to set various context options that will determine the characteristics of the desired security context, for example, specifying whether or not mutual authentication should take place. After all the desired characteristics have been set, the initiator calls the `initSecContext` method, which produces a token required by the acceptor's `acceptSecContext` method.
>
> While Java GSS-API methods exist for preparing tokens to be exchanged between applications, *it is the responsibility of the applications to actually transfer the tokens between them.* So after the initiator has received a token from its call to `initSecContext`, it sends that token to the acceptor. The acceptor calls `acceptSecContext`, passing it the token. The `acceptSecContext` method may in turn return a token. If it does, the acceptor should send that token to the initiator, which should then call `initSecContext` again and pass it this token. Each time `initSecContext` or `acceptSecContext` returns a token, the application that called the method should send the token to its peer and that peer should pass the token to its appropriate method (`acceptSecContext`or `initSecContext`). This continues until the context is fully established (which is the case when the context's `isEstablished` method returns `true`).



> This page links to a series of tutorials demonstrating various aspects of the use of JAAS (Java Authentication and Authorization Service) and Java GSS-API.
>
> **JAAS** can be used for two purposes:
>
> - for *authentication* of users, to reliably and securely determine who is currently executing Java code, and
> - for *authorization* of users to ensure they have the access control rights (permissions) required to do security-sensitive operations.
>
> **Java GSS-API** is used for *securely exchanging messages* between communicating applications. The Java GSS-API contains the Java bindings for the Generic Security Services Application Program Interface (GSS-API) defined in [RFC 2853](http://www.ietf.org/rfc/rfc2853.txt). GSS-API offers application programmers uniform access to security services atop a variety of underlying security mechanisms, including Kerberos.
>
> Note: JSSE is another API that can be used for secure communication. For the differences between the two, see [When to use Java GSS-API vs. JSSE](https://docs.oracle.com/javase/8/docs/technotes/guides/security/jgss/tutorials/JGSSvsJSSE.html).
>
> The reason both JAAS and Java GSS-API tutorials are presented together is because JAAS authentication is typically performed prior to secure communication using Java GSS-API. Thus JAAS and Java GSS-API are related and often used together. However, it is possible for applications to use JAAS without Java GSS-API, and it is also possible to use Java GSS-API without JAAS. Furthermore, JAAS itself can be used simply for authentication or for both authentication and authorization.



# 术语缩写

* MIC(Message Integrity Code)  信息完整性校验码

* JAAS(Java Authentication and Authorization Service) Java认证与授权服务

* ​

* GSS-API(The Generic Security Services Application Program Interface) 通用安全服务接口

  GSSManager

  GSSName

  GSSContext

  Initiator Credentials

  Acceptor Names

  ​

* SASL(*Simple Authentication and Security Layer* )  简单认证与安全层网络访问协议


  SaslClient

  SaslServer

* SPNEGO: The Simple and Protected GSSAPI Negotiation Mechanism (SPNEGO) GSS-API mechanism







*References*

1. Java Authentication and Authorization Service(JAAS) Reference Guide.

2. Java TechNotes Guides.

3. https://docs.oracle.com/javase/8/docs/technotes/guides/security/jgss/tutorials/BasicClientServer.html#Progs

4. https://docs.oracle.com/javase/8/docs/technotes/guides/security/jgss/tutorials/ClientServer.html

5. https://docs.oracle.com/javase/8/docs/technotes/guides/security/jgss/tutorials/BasicClientServer.html#Overview

6. https://docs.oracle.com/javase/7/docs/technotes/guides/security/jgss/tutorials/BasicClientServer.html

7. https://docs.oracle.com/javase/7/docs/technotes/guides/security/jgss/tutorials/BasicClientServer.html#KerbNames

8. https://docs.oracle.com/javase/1.5.0/docs/guide/security/jgss/tutorials/BasicClientServer.html

9. https://docs.oracle.com/javase/1.5.0/docs/guide/security/jgss/tutorials/ClientServer.html

10. https://docs.oracle.com/javase/8/docs/technotes/guides/security/jgss/tutorials/MoreToDo.html

11. http://www.dia.uniroma3.it/~afscon09/docs/angius.pdf Cross-realm authentication

12. http://web.mit.edu/kerberos/krb5-latest/doc/appdev/gssapi.html

13. https://tools.ietf.org/html/rfc2744.html#section-5.19

14. https://docs.oracle.com/javase/8/docs/technotes/guides/security/index.html

15. https://docs.oracle.com/javase/8/docs/technotes/guides/security/overview/jsoverview.html

16. http://www.oracle.com/technetwork/java/js-white-paper-149932.pdf

17. https://docs.oracle.com/javase/8/docs/technotes/guides/security/

18. http://www.ietf.org/rfc/rfc2853.txt

    ​