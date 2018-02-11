# ZooKeeper Security-4

ZooKeeper提供了简单的基于用户名和密码的认证机制，即DIGEST-MD5认证机制。本文首先介绍使用该认证机制所涉及的一些配置细节，接下来介绍ZooKeeper内部关于DIGEST-MD5认证机制的一些实现细节。

##如何使用

###Client

系统属性配置：

```java
// "zookeeper.sasl.clientconfig"如果不设置，默认值为"Client"
System.setProperty("zookeeper.sasl.clientconfig",   "Client");
System.setProperty("zookeeper.sasl.client",   "true");
```

自定义一个JaasConf对象，继承自javax.security.auth.login.Configuration，目的是为了便于Configuration所需参数的配置：

```java
public class JaasConf extends Configuration {
  private Map<String, AppConfigurationEntry[]> sections =
          new HashMap<String, AppConfigurationEntry[]>();

  public void addSection(String name, String loginModuleName, String... args) {
    Map<String, String> options = new HashMap<String, String>();
    for (int i = 0; i < args.length; i += 2) {
      options.put(args[i], args[i + 1]);
    }

    AppConfigurationEntry[] entries = new AppConfigurationEntry[]{
            new AppConfigurationEntry(loginModuleName,
                    AppConfigurationEntry.LoginModuleControlFlag.REQUIRED, options)};
    this.sections.put(name, entries);
  }

  @Override
  public AppConfigurationEntry[] getAppConfigurationEntry(String name) {
    return this.sections.get(name);
  }
}
```

实例化JaasConf，设置LoginModuleName以及对应的**username/password**等信息：

```java
JaasConf conf = new JaasConf();
// Section Name: "Client", 这里的名称与系统属性"zookeeper.sasl.clientconfig"保持一致
// LoginModule Name: "org.apache.zookeeper.server.auth.DigestLoginModule"
// Options:  
//	     "username": "nosql"
//       "password": "nosql123"
conf.addSection("Client", "org.apache.zookeeper.server.auth.DigestLoginModule",
            "username", "nosql", "password", "nosql123");
Configuration.setConfiguration(conf);
```

### Server

系统属性配置：

```java
System.setProperty("zookeeper.sasl.serverconfig", "Server");
System.setProperty("zookeeper.authProvider.sasl", 
                   "org.apache.zookeeper.server.auth.SASLAuthenticationProvider");
```

实例化JaasConf，并在Server端配置所有允许访问的**username/password**信息：

```java
JaasConf conf = new JaasConf();
// LoginModuleName: "org.apache.zookeeper.server.auth.DigestLoginModule"
// Options:  
//	     "user_nosql: nosql123"
conf.addSection("Server", "org.apache.zookeeper.server.auth.DigestLoginModule",
            "user_nosql", "nosql123");
Configuration.setConfiguration(conf);
```

可以看到，Client端与Server端配置**username/password**的参数名称是不同的：

* Client   用户名通过静态参数"**username**"指定，密码通过静态参数"**password**"指定
* Server  用户名直接配置在一个以"**user_**"开头的动态参数名中，参数值直接为对应的**password**

> Client通过这种模式只能配置一个**username/password**，而Server端的动态参数则允许配置多个Client的**username/password**。原因在于，Client只需要配置一个**username/password**即可，而Server端则允许配置多个Client的**username/password**。

## 实现原理

###整体思路

* Server端在初始化ServerCnxnFactory时，加载预先配置的允许访问的一个或多个**username/password**列表，并执行Login操作
* Client基于配置的**username/password**以及**DigestLoginModule**，执行Login操作
* Client请求与Server端建立Sasl连接，建立连接过程中，通过com.sun.security.sasl.digest.FactoryImpl提供的认证机制，完成对**username/password**的合法校验

###Client初始化

ZooKeeperSaslClient初始化时：

```java
if (login == null) {
  if (LOG.isDebugEnabled()) {
    LOG.debug("JAAS loginContext is: " + loginContext);
  }
  // 初始化Login对象，Login对象是static类型的，也就说，该对象在进程级别内
  // 是共享的. Login对象利用Java JAAS机制执行login操作，具体的Login机制由
  // 配置的LoginContext来实现.
  login = new Login(loginContext, new ClientCallbackHandler(null));
  login.startThreadIfNeeded();
}
Subject subject = login.getSubject();
SaslClient saslClient;
// ZooKeeper支持的认证主要是GSSAPI(Kerberos)以及DIGEST-MD5. 如果基于GSSAPI,
// 认证成功后会在Subject中添加对应的Principal信息. 如果Subject中的Principal
// 信息为空，则认为要使用DIGEST-MD5认证(注: 这种设计并不太好)
if (subject.getPrincipals().isEmpty()) {
  // no principals: must not be GSSAPI: use DIGEST-MD5 mechanism instead.
  LOG.info("Client will use DIGEST-MD5 as SASL mechanism.");
  String[] mechs = {"DIGEST-MD5"};
  // 从subject中获取username与password信息
  String username = (String)(subject.getPublicCredentials().toArray()[0]);
  String password = (String)(subject.getPrivateCredentials().toArray()[0]);
  // 初始化SaslClient时，将username传入，password在ClientCallbackHandler中.
  // "zk-sasl-md5" is a hard-wired 'domain' parameter shared with
  // zookeeper server code (see ServerCnxnFactory.java)
  saslClient = Sasl.createSaslClient(mechs, username, "zookeeper", 
              "zk-sasl-md5", null, new ClientCallbackHandler(password));
  return saslClient;
}
```

关于如上源码的更多备注信息：

1. Login阶段，已经配置了LoginModule为`org.apache.zookeeper.server.auth.DigestLoginModule`

2. `DigestLoginModule`中在初始化时已经将Client配置的**username**和**password**信息加载到subject中：

   ```java
   public void initialize(Subject subject, CallbackHandler callbackHandler, 
                          Map<String,?> sharedState, Map<String,?> options) {
        if (options.containsKey("username")) {
          // Zookeeper client: get username and password from JAAS conf 
          // (only used if using DIGEST-MD5).
          this.subject = subject;
          String username = (String)options.get("username");
          this.subject.getPublicCredentials().add((Object)username);
          String password = (String)options.get("password");
          this.subject.getPrivateCredentials().add((Object)password);
        }
        return;
   }
   ```

3. Sasl.createSaslClient的流程：


```java
String mechFilter = "SaslClientFactory." + mechName;
Provider[] provs = Security.getProviders(mechFilter);
for (int j = 0; provs != null && j < provs.length; j++) {
  className = provs[j].getProperty(mechFilter);
  if (className == null) {
    // Case is ignored
    continue;
  }

  fac = (SaslClientFactory) loadFactory(provs[j], className);
  if (fac != null) {
    mech = fac.createSaslClient(
      new String[]{mechanisms[i]}, authorizationId,
      protocol, serverName, props, cbh);
    if (mech != null) {
      return mech;
    }
  }
}
```

"SaslClientFactory.DEGIEST-MD5"所关联的SaslClientFactory实现为：

```java
com.sun.security.sasl.digest.FactoryImpl
```

所有的SaslClientFactory的实现信息都被注册在**java.security.Security**中。

> **<u>Security与ProviderRegistry</u>: **
>
> **java.security.Security**:  Java Security框架中的定义，用来注册SaslClientFactory. 每一个SaslClientFactory都关联着一个Name.
>
> **org.apache.zookeeper.server.auth.ProviderRegistry**: ZooKeeper中自定义的用来注册所有的AuthenticationProvider的类，每一个AuthenticationProvider关联一个schema

###Server端初始化

**ServerCnxnFactory#configureSaslLogin**中的一些关键源码：

```java
String serverSection = System.getProperty("zookeeper.sasl.serverconfig", "Server");

// Note that 'Configuration' here refers to javax.security.auth.login.Configuration.
AppConfigurationEntry entries[] = null;
SecurityException securityException = null;
try {
  entries = Configuration.getConfiguration().getAppConfigurationEntry(serverSection);
} catch (SecurityException e) {
  // handle below: might be harmless if the user doesn't intend to use JAAS authentication.
  securityException = e;
}
// ...中间略去一下非关键源码....
try {
  // 初始化SaslServerCallbackHandler
  saslServerCallbackHandler = new SaslServerCallbackHandler(Configuration.getConfiguration());
  // 初始化Login对象，利用配置的LoginModule执行login操作.
  login = new Login(serverSection, saslServerCallbackHandler);
  login.startThreadIfNeeded();
} catch (LoginException e) {
  // ....
}

```

`SaslServerCallbackHandler`初始化过程中，加载配置的一个或多个**username/password**信息：

```java
public SaslServerCallbackHandler(Configuration configuration) throws IOException {
  String serverSection = System.getProperty("zookeeper.sasl.serverconfig",
                                            "Server");            
  AppConfigurationEntry configurationEntries[] = 
    configuration.getAppConfigurationEntry(serverSection);

  if (configurationEntries == null) {
    String errorMessage = "Could not find a 'Server' entry in" +
      " this configuration: Server cannot start.";
    LOG.error(errorMessage);
    throw new IOException(errorMessage);
  }
  credentials.clear();
  for(AppConfigurationEntry entry: configurationEntries) {
    Map<String,?> options = entry.getOptions();
    // 所有的用户名都被配置在以"user_"为前缀的属性名中
    for(Map.Entry<String, ?> pair : options.entrySet()) {
      String key = pair.getKey();
      if (key.startsWith(USER_PREFIX)) {
        String userName = key.substring(USER_PREFIX.length());
        credentials.put(userName,(String)pair.getValue());
      }
    }
  }
}
```

## 总结

该机制虽然实现了基于用户名和密码的简单认证机制，但所有的用户名和密码信息都是静态配置的，无法支持用户的动态增加，这是该方案的最大软肋。