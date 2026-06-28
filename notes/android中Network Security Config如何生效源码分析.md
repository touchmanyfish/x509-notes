# android中Network Security Config如何生效源码分析

## 原理

在android系统中，一般是调用如下代码，获取TrustManager来进行证书验证:
```kotlin
// 获取TrustManager
val tmmf = TrustManagerFactory.getInstance("PKIX")
// 或者这个
val tmmf2 = TrustManagerFactory.getInstance("X509")

// 初始化TrustManager
tmmf.init(null as KeyStore?)

// 返回TrustManager
val trustManagers = tmmf.trustManagers
```

android通过注册NetworkSecurityConfigProvider来将上述最终获取到的TrustManager修改为RootTrustManager;而当使用RootTrustManager进行证书
验证相关操作时，network_security_config.xml中的配置会参与其中。


## 修改返回的TrustManager

根据jca架构，我们观察TrustManagerFactory源码可知：

1. TrustManagerFactory是一个engine class，它公布了关系:
```text
("TrustManagerFactory","PKIX") -> TrustManagerFactorySpi
("TrustManagerFactory","X509") -> TrustManagerFactorySpi
```

2. TrustManagerFactory使用GetInstance.getInstance()获取TrustManagerFactorySpi的子类实例，这导致涉及到的Provier遍历会：先遍历优先级列表再遍历普通列表。
3. TrustManagerFactorySpi返回TrustManager的数组 

假设我们要替换最终获取到的TrustManager，可以做如下操作:

1. 实现TrustManagerFactorySpi
2. 实现自定义Provider，在其中注册：
```java
put("TrustManagerFactory.PKIX", PREFIX + "RootTrustManagerFactorySpi");
put("Alg.Alias.TrustManagerFactory.X509", "PKIX");
```
3. 在ProviderList被第一次实例化之前设置自定义Provider的优先级(**三方代码难以实现**)
```kotlin
Security.setProperty("jdk.security.provider.preferred", "TrustManagerFactory.PKIX:MyManagerProvider")
```

4. 将自定义Provider插到普通列表的最前面

此时，我们自定义的Provider就会被优先访问，然后调用如下代码初始化我们自定义的TrustManagerFactorySpi子类实例
```kotlin
provider.getService("TrustManagerFactory","PKIX").newInstance(null)
```

### android中的做法

1. 实现RootTrustManagerFactorySpi(继承TrustManagerFactorySpi)，返回RootTrustManager
2. 实现NetworkSecurityConfigProvider，在其中进行注册
```java
put("TrustManagerFactory.PKIX", PREFIX + "RootTrustManagerFactorySpi");
put("Alg.Alias.TrustManagerFactory.X509", "PKIX");
```
3. 将实现NetworkSecurityConfigProvider插到普通列表的最前面
```java
int pos = Security.insertProviderAt(new NetworkSecurityConfigProvider(), 1);
```


可以通过如下代码查看android中的Provider优先级配置，结果为null：
```kotlin
// 返回null
Security.getProperty("jdk.security.provider.preferred")
```

所以对于NetworkSecurityConfigProvider，不用为它配置优先级。

不过你需要祈祷没人为了同样的目的把Provider插到了NetworkSecurityConfigProvider的前面导致其实效。

## 配置的读取

实用XmlConfigSource类将network_security_config.xml解析出如下2个字段

**1.默认配置mDefaultConfig(NetworkSecurityConfig):**

```text
result_defaultConfig = (base-config覆盖platformconfig) + debug-overrides
```

**2.针对domain的配置mDomainMap:**

在其中按domain,config(NetworkSecurityConfig)保存配置，其中:

```text
// 父级includeSubdomains="false
config = domain-config  -> 覆盖result_defaultConfig

// 父级includeSubdomains="true"
config = domain-config  -> 覆盖parent_domain-config() -> 覆盖result_defaultConfig
```
## 配置生效:RootTrustManager(证书固定，trust anchor)

可以调用它的如下2种方法进行证书验证：
```kotlin
checkServerTrusted()
checkClientTrusted()
```

验证流程：

1. 调用rootTrustManager.checkXTrusted()
2. 根据hostname获取对应的配置NetworkSecurityConfig(准确的说应该是xml的配置结合平台默认配置的结果)
3. 使用配置初始化mDelegate(TrustManagerImpl)
4. 调用mDelegate.checkXTrusted()进行证书验证；配置中的trust anchor会参与这个过程；这一步不进行证书固定的验证。
5. 如果调用的是checkServerTrusted()，则还需要调用checkPins()根据配置进行证书固定的验证

### 根据hostname获取对应配置

```java
ApplicationConfig#getConfigForHostname(String hostname)
```

这个方法用于拿取hostname对应的配置，规则如下：

1. 优先域名完全匹配的那个配置
2. 在includeSubdomains==true的且部分匹配的域名中，优先拿最长域名对应的那个配置
3. 如果没有匹配的则拿默认配置

```text
假设

c.com(在includeSubdomains = true)
b.c.com(在includeSubdomains = false)
a.b.c.com(在includeSubdomains = true)

getConfigForHostname("vip.a.b.c.com")返回"a.b.c.com"对应的配置
```


### mDelegate(TrustManagerImpl)
```java
public NetworkSecurityTrustManager(NetworkSecurityConfig config) {
    if (config == null) {
        throw new NullPointerException("config must not be null");
    }

    mNetworkSecurityConfig = config;

    try {
        TrustedCertificateStoreAdapter certStore = new TrustedCertificateStoreAdapter(config);

        // Provide an empty KeyStore since TrustManagerImpl doesn't support null KeyStores.
        // TrustManagerImpl will use certStore to lookup certificates.
        KeyStore store = KeyStore.getInstance(KeyStore.getDefaultType());
        store.load(null);

        mDelegate = new TrustManagerImpl(store, null, certStore);
    } catch (GeneralSecurityException | IOException e) {
        throw new RuntimeException(e);
    }
}   
```

**config:**

通过hostname获取到的配置

**certStore:**

通过certStore可以访问到配置中指定的trust anchor集合

**store：**

根据注释，这是一个空的东西，可以认为没啥用。

**TrustManagerImpl：**

证书验证的实现

**TrustManagerImpl构造器的第二个参数:**

证书固定验证逻辑，这里为null表明mDelegate中不进行证书固定验证

**组装mDelegate语义：**

mDelegate实现了证书验证逻辑，但是不执行证书固定验证逻辑；hostname对应配置中指定的trust anchor会参与mDelegate
的证书验证逻辑。


### checkPins()

使用对应配置进行证书固定验证

## 资料

[网络安全配置](https://developer.android.com/privacy-and-security/security-config?hl=zh-cn#debug-overrides)

