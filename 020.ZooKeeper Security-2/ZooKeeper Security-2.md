# ZooKeeper Security-2

本文探讨ZooKeeper的SSL安全机制。默认情形下，ZooKeeper的网络通信是没有加密的，但ZooKeeper提供了SSL特性，这样，Client与ZooKeeper进行交互时就可运行在安全模式下，SSL特性仅在使用Netty作为RPC通信层时才能生效，对于ZooKeeper内置的RPC通信层`NIOServerCnxnFactory`并不支持SSL特性。目前在ZooKeeper Server之间的通信中也还不支持SSL。

##SSL简介

SSL全称为`Secure Socket Layer`，它是一种介于传输层和应用层的协议，它通过"握手协议"和“传输协议”来解决信息传输的安全问题，它可以被建立在任何可靠的传输层协议之上(例如TCP，但不能是UDP)。SSL协议主要提供如下三方面的能力：

* 信息的加密传播
* 校验机制，数据一旦被篡改，通信双方均会立刻发现
* 身份证书，防止身份被冒充

SSL的基本设计思想是：

1. Client向Server端索要"**公钥**"
2. Client对获取的"**公钥**"进行校验
3. 双方协商生成“**会话密钥**”
4. 双方基于"**会话密钥**"进行信息交换

前3步称之为"握手阶段"，"握手阶段"采用"**非对称加密**"算法。

第4步称之为"传输阶段"，基于"**对称加密**"算法，"对称加密"算法的性能是远高于"非对称加密"算法的，因此，更适用于大数据量的传输加密。

## 如何使用

### Client端配置 

ZooKeeper Client通过配置如下系统属性来启用基于Netty的RPC通信层:

```java
zookeeper.clientCnxnSocket="org.apache.zookeeper.ClientCnxnSocketNetty"
```

Client需要设置如下参数来启用安全通信：

```java
zookeeper.client.secure=true
```

设置了`zookeeper.client.secure`属性为`true`以后，意味着Client与Server之间只能通过`"secureClientPort"`所指定的端口进行交互。

最后，需要配置KeyStore与TrustStore的相关系统属性：

```java
zookeeper.ssl.keyStore.location="/path/to/your/keystore"
zookeeper.ssl.keyStore.password="keystore_password"
zookeeper.ssl.trustStore.location="/path/to/your/truststore"
zookeeper.ssl.trustStore.password="truststore_password"
```

### Server端配置

ZooKeeper Server通过配置如下系统属性来启用Netty:

```java
zookeeper.serverCnxnFactory="org.apache.zookeeper.server.NettyServerCnxnFactory"
```

在"zoo.cfg"中配置"secureClientPort"端口值，该端口值与原来的"clientPort"端口值应该区别开：

```java
secureClientPort=2281
```

最后也需要设置KeyStore与TrustStore的配置，与Client端配置类似。

### 配置示例

 “bin/zkServer.sh”的配置示例如下:

```bash
export SERVER_JVMFLAGS="
-Dzookeeper.serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory
-Dzookeeper.ssl.keyStore.location=/root/zookeeper/ssl/testKeyStore.jks 
-Dzookeeper.ssl.keyStore.password=testpass 
-Dzookeeper.ssl.trustStore.location=/root/zookeeper/ssl/testTrustStore.jks 
-Dzookeeper.ssl.trustStore.password=testpass" 
```

在 “zoo.cfg”中增加：

```java
secureClientPort=2281
```

"bin/zkCli.sh"的配置为: 

```bash
export CLIENT_JVMFLAGS="
-Dzookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty 
-Dzookeeper.client.secure=true 
-Dzookeeper.ssl.keyStore.location=/root/zookeeper/ssl/testKeyStore.jks 
-Dzookeeper.ssl.keyStore.password=testpass 
-Dzookeeper.ssl.trustStore.location=/root/zookeeper/ssl/testTrustStore.jks 
-Dzookeeper.ssl.trustStore.password=testpass"
```

### X509AuthenticationProvider

默认情况下，SSL认证是由`X509AuthenticationProvider`提供的，对应的schema为`x509`。`X509AuthenticationProvider`基于`javax.net.ssl.X509KeyManager`与`javax.net.ssl.X509TrustManager`提供`Host`的**证书认证**机制。X509AuthenticationProvider仅仅当`zookeeper.serverCnxnFactory`配置为`NettyServerCnxnFactory`时才可使用，ZooKeeper内置的NIO实现类`NIOServerCnxnFactory`并不支持SSL。

关键的配置项如下所示：

| 配置项                               | 配置解释            |
| --------------------------------- | --------------- |
| zookeeper.ssl.keyStore.location   | KeyStore的路径     |
| zookeeper.ssl.trustStore.location | TrustStore的路径   |
| zookeeper.ssl.keyStore.password   | KeyStore的访问密码   |
| zookeeper.ssl.trustStore.password | TrustStore的访问密码 |

在KeyStore JKS文件中保存了Server的证书以及私钥信息，该证书需要由Client端信任，因此，该证书或CA(证书认证机构信息)也会被存储在Client端的TrustStore JKS文件中。同时，Server端的TrustStore JFS文件中存储了所信任的Client的证书/CA信息。

Client认证成功之后，会创建一个ZooKeeper Session，Client可以设置ACLs的schema为"x509". "x509"使用Client认证成功后的X500 Principal作为ACL ID。 ACL信息中包含Client认证后的确切的X500 Principal名称。

> 关于X509与 X500:
>
> X509: 一套数字证书体系标准
>
> X500: 定义了一种区别命名规则，以命名树来确保用户名称的唯一性
>

与`digest`认证类似，Server端可以配置一个X509的`superUser`，对应的Property Key为：

​	`"zookeeper.X509AuthenticationProvider.superUser"`

`superUser`可以绕过ACL配置从而拥有所有znodes的所有权限。

## 定制X509AuthenticationProvider

除了默认的`X509AuthenticationProvider`以外，ZooKeeper允许自定义扩展实现X509的安全信任机制，尤其是Certificate Key Infrastructures不使用JKS时。

自定义实现`X509AuthenticationProvider`应该遵循：

* 继承自`X509AuthenticationProvider`
* KeyManager需要继承自`javax.net.ssl.X509ExtendedKeyManager`
* TrustManager需要继承自`javax.net.ssl.X509ExtendedTrustManager`
* 覆写`X509AuthenticationProvider`的`getKeyManager`与`getTrustManager`方法

这样，自定义的实现才会在SSLEngine中发挥作用。

自定义的AuthenticationProvider需要配置一个对应的`schema`名称，并且通过系统属性`"zookeeper.authProvider.[schema_name]"`来配置新定义的AuthenticationProvider实现类，这样在ProviderRegistry初始化时会自动加载。接下来，还需要设置系统属性`"zookeeper.ssl.authProvider=[schema_name]"`，这样，新定义的AuthenticationProvider才可以被应用在安全认证中。

## 实现细节

`NettyServerCnxnFactory`构造函数中初始化ChannelPipeline时调用初始化SSL的方法：

```java
CnxnChannelHandler channelHandler = new CnxnChannelHandler();

NettyServerCnxnFactory() {
  bootstrap = new ServerBootstrap(
    new NioServerSocketChannelFactory(
      Executors.newCachedThreadPool(),
      Executors.newCachedThreadPool()));
  // parent channel
  bootstrap.setOption("reuseAddress", true);
  // child channels
  bootstrap.setOption("child.tcpNoDelay", true);
  /* set socket linger to off, so that socket close does not block */
  bootstrap.setOption("child.soLinger", -1);
  bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
    @Override
    public ChannelPipeline getPipeline() throws Exception {
      ChannelPipeline p = Channels.pipeline();
      if (secure) {
        // 先初始化SSLContext以及SSLEngine，然后构造SslHandler对象
        // 将构造的SslHandler设置到ChannelPipeline中
        initSSL(p);
      }
      // 将CnxnChannelHandler添加到ChannelPipeline中
      p.addLast("servercnxnfactory", channelHandler);

      return p;
    }
  });
}
```

`NettyServerCnxnFactory#initSSL`方法的实现如下：

```java
private synchronized void initSSL(ChannelPipeline p)
            throws X509Exception, KeyManagementException, NoSuchAlgorithmException {
  // 从配置的系统属性中获取AuthenticationProvider实现类
  String authProviderProp = System.getProperty(ZKConfig.SSL_AUTHPROVIDER);
  SSLContext sslContext;
  if (authProviderProp == null) {
    sslContext = X509Util.createSSLContext();
  } else {
    sslContext = SSLContext.getInstance("TLSv1");
    // 从ProviderRegistry中获取对应的AuthenticationProvider实例
    X509AuthenticationProvider authProvider =
      (X509AuthenticationProvider)ProviderRegistry.getProvider(
      System.getProperty(ZKConfig.SSL_AUTHPROVIDER,
                         "x509"));

    if (authProvider == null)
    {
      LOG.error("Auth provider not found: {}", authProviderProp);
      throw new SSLContextException(
        "Could not create SSLContext with specified auth provider: " +
        authProviderProp);
    }
    // 将X509AuthenticationProvider中已经初始化的
    // KeyManager和TrustManager设置到SSLContext中
    sslContext.init(new X509KeyManager[] { authProvider.getKeyManager() },
                    new X509TrustManager[] { authProvider.getTrustManager() },
                    null);
  }

  SSLEngine sslEngine = sslContext.createSSLEngine();
  sslEngine.setUseClientMode(false);
  // 设置需要Client认证
  sslEngine.setNeedClientAuth(true);
  // 将创建好的SSLEngine设置到ChannelPipeline中
  p.addLast("ssl", new SslHandler(sslEngine));
  LOG.info("SSL handler added for channel: {}", p.getChannel());
}
```

`CnxnChannelHandler#channelConnected`方法的定义如下：

```java
 @Override
public void channelConnected(ChannelHandlerContext ctx,
                             ChannelStateEvent e) throws Exception
{
     NettyServerCnxn cnxn = new NettyServerCnxn(ctx.getChannel(),
                    zkServer, NettyServerCnxnFactory.this);
     ctx.setAttachment(cnxn);

     if (secure) {
        SslHandler sslHandler = ctx.getPipeline().get(SslHandler.class);
        ChannelFuture handshakeFuture = sslHandler.handshake();
        // 在SslHandler的handshake Future中增加监听者来对证书进行校验.
        handshakeFuture.addListener(new CertificateVerifier(sslHandler, cnxn));
     } else {
        allChannels.add(ctx.getChannel());
        addCnxn(cnxn);
     }
}
```

当SslHandler中的handshake Future中的监听者被触发以后，由`CertificateVerifier`来对证书的合法性进行校验，而`CertificateVerifier`对证书进行校验的操作是由`X509AuthenticationProvider`或者自定义的扩展实现类来完成：

```java
public void operationComplete(ChannelFuture future)
                    throws SSLPeerUnverifiedException {
  if (future.isSuccess()) {
    SSLEngine eng = sslHandler.getEngine();
    SSLSession session = eng.getSession();
    cnxn.setClientCertificateChain(session.getPeerCertificates());

    // 从系统属性“zookeeper.ssl.authProvider”中获取
    // 定义的AuthenticationProvider的schema信息
    String authProviderProp
      = System.getProperty(ZKConfig.SSL_AUTHPROVIDER, "x509");

    // 从ProviderRegistry中获取对应的X509AuthenticationProvider实例
    X509AuthenticationProvider authProvider =
      (X509AuthenticationProvider)
      ProviderRegistry.getProvider(authProviderProp);
    
    if (authProvider == null) {
      LOG.error("Auth provider not found: {}", authProviderProp);
      cnxn.close();
      return;
    }

    // 在这里调用AuthenticationProvider#handlerAuthentication方法
    if (KeeperException.Code.OK !=
        authProvider.handleAuthentication(cnxn, null)) {
      LOG.error("Authentication failed for session 0x{}",
                Long.toHexString(cnxn.sessionId));
      // 如果校验失败，则将已经初始化的连接关闭
      cnxn.close();
      return;
    }

    allChannels.add(future.getChannel());
    addCnxn(cnxn);
  } else {
    LOG.error("Unsuccessful handshake with session 0x{}",
              Long.toHexString(cnxn.sessionId));
    cnxn.close();
  }
}

```



*Reference*

1. [Client-Server Mutual Authentication](https://cwiki.apache.org/confluence/display/ZOOKEEPER/Client-Server+mutual+authentication)
2. [ZOOKEEPER-938](https://issues.apache.org/jira/browse/ZOOKEEPER-938)
3. [ZOOKEEPER-2125](https://issues.apache.org/jira/browse/ZOOKEEPER-2125)
4. [ZooKeeper SSL User Guide](https://cwiki.apache.org/confluence/display/zookeeper/zookeeper+ssl+user+guide)
5. [SSL/TLS协议运行机制的概述](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)