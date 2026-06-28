# JCA 架构中 Engine、Provider、Service、Spi 的动态路由机制

## SPI

SPI(Service Provider Interface)的形式为接口或者抽象类，具体功能在子类中实现。

### 建立关系

需要为具体的SPI建立如下关系:
```text
// 注意，这里和Provider中的关系注册略有区别
type.key -> SPI接口/抽象类
```

这个关系表示传递"type.key"时期望获取一个SPI接口/抽象类的具体实现实例。

### 替换功能

```kotlin
// 伪代码
// 返回一个SPI的实例
fun getSPI(type:String,key:String):SPI{
    //
}
```

当type.key的值确定时，如何返回不同的SPI实例？

我们可以维护一个"type.key -> SPI实现类全名"的关系列表：

```text
type1.key1 -> SPI_A_Impl1实现类全名
type1.key1 -> SPI_A_Impl2实现类全名

type1.key2 -> SPI_A_Impl3实现类全名
..
```
假设getSPI()收到传递的值"type1.key1",在关系列表中 **查找第一个匹配的条目** :
```text
type1.key1 -> SPI_A_Impl1实现类全名
```
此时我们可以通过"SPI_A_Impl1实现类全名"反射出一个SPI的实例，然后返回。

```kotlin
// 伪代码

// 根据type.key获取对应的SPI实现类
fun getSPI(type:String,key:String):SPI{
    // 从关系列表中查找第一个匹配的条目，获取spi实现类全名
    val spiClassName = TypeKeyList.find(type,key)
    // 反射初始化spi实例
    val spiInstance = newInstanceReflect(spiClassName)
    
    return spiInstance
}
```

于是，对于某个SPI，如果我们知道了它对应的"type.key -> SPI"关系以后，就可以 **往关系列表首位插入**：
```text
type.key -> SPI实现类全名
```
来实现替换getSPI()方法返回SPI实例的功能

```kotlin
// 替换type.key对应SPI实现类
fun replaceSPI(type:String,key:String,spiClassName:String){
    // 将新的SPI实现类插入到关系类表首位
    // 注意，1是首位
    TypeKeyList.insertAt(type,key,spiClassName,1)
}
```

## Provider

jca中保存"type.key -> spi实现类全名"关系的最小单元。

### 注册"type.key -> spi实现类全名"关系

在Provider中可以调用putX()方法注册"type.key -> spi实现类全名"关系，这个关系会被保存在对应的service中:

```text
// type对应一个key 
type.key -> SPI实现类全名

// type对应多个key:

type.key -> SPI实现类全名
type.key2 -> SPI2实现类全名
```

例子:
```kotlin
public NetworkSecurityConfigProvider() {
    super("AndroidNSSP", 1.0, "Android Network Security Policy Provider");
    
    // type.key
    put("TrustManagerFactory.PKIX", PREFIX + "RootTrustManagerFactorySpi");
 
    // 别名语法
    // Alg.Alias.type.key
    put("Alg.Alias.TrustManagerFactory.X509", "PKIX");

}
```

可以调用putService()将已保存好关系的service传递给provider；
如果调用的put()方法则对应的service会在ensureLegacyParsed()调用时被实例化

由于历史原因在与Provider.java同层级的代码中key可能叫做其他名字,例如在Service中叫做alogrithm

### 通过Service实例化SPI实现类

可以通过provider.getService(type,key).newInstance(spi实现类构造器参数)获取对应spi类实例

## ProviderList

用于存放Provider的不可变列表。

### configs

ProviderList中实际保存Provider信息的位置；当ProviderList()构造器被调用时，还会加入
```java
// ProviderList.java

while ((entry = Security.getProperty("security.provider." + i)) != null) {
   // 读取provider信息插入configList        
}
configs = configList.toArray(PC0);
```

### 优先级列表preferredPropList

#### 初始化

在ProviderList()被调用时，会从"jdk.security.provider.preferred"构造一个优先级列表preferredPropList。

preferredPropList是一个static类型，只在ProviderList()被首次调用时进行初始化，并且初始化以后不能被修改。所以需要在ProviderList()被第一次
调用之前设置"jdk.security.provider.preferred"的值:
```java
Security.setProperty("jdk.security.provider.preferred", "匹配条件:providerName")
    
Security.setProperty("jdk.security.provider.preferred", "匹配条件1:providerName1,匹配条件1:providerName2")
    
// 例如    
Security.setProperty("jdk.security.provider.preferred", "TrustManagerFactory.PKIX:SunJSSE")
```

#### 优先级机制

当ProviderList中需要遍历保存的Provider信息时:

1. 先用type,key去preferredPropList中匹配出一组provider信息
2. 优先在匹配出的provider信息中遍历
3. 如果匹配的provider不满足要求再去configs中遍历

#### 匹配机制

```text
// 匹配语法

// 单个匹配项
条件:providerName

// 多个匹配项
条件1:providerName,条件2:providerName,条件3:providerName
```

无论是单个还是多个匹配项，只要type+key与匹配项中的条件匹配，则将providerName对应provider信息加入匹配结果中；后续会优先遍历刚刚匹配的结果。

```text
// 匹配语法
targetKey:providerName

// 例如
PKIX:SunJSSE
```
这个写法表示只要key与targetKey一致就匹配。

```text
// 匹配语法
targetType.targetKey:providerName

// 例如
TrustManagerFactory.PKIX:SunJSSE
```
这个语法表示type与targetType，key与targetKey同时一致时匹配。

```text
// 匹配语法
// 注意，这个语法中，"Group"为固定的字符串
Group.targetGroupName:providerName

// 例如
Group.SHA2:SUN
```

ProviderList中定义了Group，它包含组名和其对应的一组算法。

```java
//ProviderList.java

// 定义组名与其对应的一组算法
// Group definitions
if (type != null && type.compareToIgnoreCase("Group") == 0) {
    // Currently intrinsic algorithm groups
        if (algorithm.compareToIgnoreCase("SHA2") == 0) {
            alternateNames = SHA2Group;
        } else if (algorithm.compareToIgnoreCase("HmacSHA2") == 0) {
            alternateNames = HmacSHA2Group;
        } else if (algorithm.compareToIgnoreCase("SHA2RSA") == 0) {
            alternateNames = SHA2RSAGroup;
        } else if (algorithm.compareToIgnoreCase("SHA2DSA") == 0) {
            alternateNames = SHA2DSAGroup;
        } else if (algorithm.compareToIgnoreCase("SHA2ECDSA") == 0) {
            alternateNames = SHA2ECDSAGroup;
        } else if (algorithm.compareToIgnoreCase("SHA3") == 0) {
            alternateNames = SHA3Group;
        } else if (algorithm.compareToIgnoreCase("HmacSHA3") == 0) {
            alternateNames = HmacSHA3Group;
        }
        if (alternateNames != null) {
            group = true;
        }

        // If the algorithm name given is SHA1
}

/* Defined Groups for jdk.security.provider.preferred */
private static final String SHA2Group[] = { "SHA-224", "SHA-256",
        "SHA-384", "SHA-512", "SHA-512/224", "SHA-512/256" };
private static final String HmacSHA2Group[] = { "HmacSHA224",
        "HmacSHA256", "HmacSHA384", "HmacSHA512"};
private static final String SHA2RSAGroup[] = { "SHA224withRSA",
        "SHA256withRSA", "SHA384withRSA", "SHA512withRSA"};
private static final String SHA2DSAGroup[] = { "SHA224withDSA",
        "SHA256withDSA", "SHA384withDSA", "SHA512withDSA"};
private static final String SHA2ECDSAGroup[] = { "SHA224withECDSA",
        "SHA256withECDSA", "SHA384withECDSA", "SHA512withECDSA"};
private static final String SHA3Group[] = { "SHA3-224", "SHA3-256",
        "SHA3-384", "SHA3-512" };
private static final String HmacSHA3Group[] = { "HmacSHA3-224",
        "HmacSHA3-256", "HmacSHA3-384", "HmacSHA3-512"};

```

对于我们的例子"Group.SHA2:SUN",它表示如果key与SHA2组下任意一个算法一致，则条件匹配:
```java
private static final String SHA2Group[] = { "SHA-224", "SHA-256",
        "SHA-384", "SHA-512", "SHA-512/224", "SHA-512/256" };
```

#### 自定义类如何使用这个优先级机制

ProviderList位于sun.*包下，自定义代码无法直接访问，目前只能通过向能使用这个机制的Engine提供Provider实现来间接使用这个机制:
```kotlin
private fun myManagerProviderDemo() {
    // 当type为TrustManagerFactory，key为PKIX时将名为"MyManagerProvider"的provider放入优遍历列表
    Security.setProperty("jdk.security.provider.preferred", "TrustManagerFactory.PKIX:MyManagerProvider")

    // 注册MyManagerProvider
    Security.addProvider(MyManagerProvider())

    val tmmf = TrustManagerFactory.getInstance("PKIX")

    println(tmmf.provider)
    println(tmmf.trustManagers)

    // log:
    // MyManagerProvider version 1.0
    // MyManagerSpi engineGetTrustManagers
    // null
}

class MyManagerProvider : Provider {

    constructor() : super("MyManagerProvider", "1.0", "MyManagerProviderinfo") {
        put("TrustManagerFactory.PKIX", "providerDemo.MyManagerSpi")
    }

}

class MyManagerSpi : TrustManagerFactorySpi() {
    override fun engineInit(ks: KeyStore?) {
        println("MyManagerSpi engineInit")
    }

    override fun engineInit(spec: ManagerFactoryParameters?) {
        println("MyManagerSpi engineInit")
    }

    override fun engineGetTrustManagers(): Array<out TrustManager?>? {
        println("MyManagerSpi engineGetTrustManagers")
        return null
    }

}
```

### getService(type,key)

这个方法返回第一个找到的(type,key)对应的service，它的逻辑如下：

1. 先依次遍历优先级列表，看是否能有service返回
2. 在遍历configs列表，看是否能有service返回

### getServices(type,key)

这个方法返回一个ServiceList，通过这个ServiceList可以访问到所有和(type,key)对应的service。

#### 最小化查询

当ServiceList被返回时,(type,key)对应的所有service实际还未被查询。当调用serviceList.get(index),内部会调用tryGet(index),会进行如下逻辑:

1. 查询时的遍历顺序和getService(type,key)中一致
2. 首次调用tryGet(index)时查询：0，1，2 ... index，并将结果保存起来
3. 后续调用tryGet(x)(0 <= x <= index)时直接返回保存的结果
4. 后续调用tryGet(y)(y > index)时查询：index + 1,index +2 ... y,并将结果保存起来

假设ServiceList中的(type,key)对应10个元素:

1. tryGet(2)，查询：0,1,2
2. tryGet(1)，直接返回保存的结果
3. tryGet(5)，查询：3，4，5

#### 其他优化点

考虑如下2种情况：

1. serviceList.get(0);大多数情况
2. serviceList.get(其他);少数情况

所以当查询0时，将结果保存到单独的变量firstService中减少内存分配；当少数情况出现时才进行ArrayList分配

```java
// ServiceList
private void addService(Service s) {
    if (firstService == null) {
        firstService = s;
    } else {
        if (services == null) {
            services = new ArrayList<Service>(4);
            services.add(firstService);
        }
        services.add(s);
    }
}
```

## Providers

内部维护2个ProviderList列表
```java
// 全局的ProviderList列表
getSystemProviderList()

// 由访问线程控制的ProviderList列表
getThreadProviderList()

// 如果ThreadProviderList启用则返回它，否则返回SystemProviderList
getProviderList()
```

对于getSystemProviderList()列表，应用层只能通过Security访问:
* `Security.getProviders()`
* `Security.getProviders(String filter)`
* `Security.getProviders(Map<String,String> filter)`
* `Security.getProvider(String name)`
* `Security.addProvider(Provider provider)`
* `Security.insertProviderAt(Provider provider, int position)`
* `Security.removeProvider(String name)`

## GetInstance

在jca中常常需要执行如下步骤：

1. 通过(type,key)查询对应spi实现类全名
2. 通过spi实现类全名实例化实例

GetInstance实现了相关过程并加上了一些额外的处理，我们可以将GetInstance理解为胶水代码。

### 举例分析getInstance()方法实现

```java
public static Instance getInstance(String type, Class<?> clazz, String algorithm) 
        throws NoSuchAlgorithmException {
    
    // 大多数情况
    // In almost all cases, the first service will work.
    // Avoid taking a long path if so.
    ProviderList list = Providers.getProviderList();
    Service firstService = list.getService(type, algorithm);
    
    if (firstService == null) {
        throw new NoSuchAlgorithmException(algorithm + " " + type + " not available");
    }
    
    NoSuchAlgorithmException failure;
    try {
        return getInstance(firstService, clazz);
    } catch (NoSuchAlgorithmException e) {
        failure = e;
    }
    
    // 少数情况
    // If we cannot get the service from the preferred provider,
    // fail over to the next.
    for (Service s : list.getServices(type, algorithm)) {
        if (s == firstService) {
            // Do not retry initial failed service
            continue;
        }
        try {
            return getInstance(s, clazz);
        } catch (NoSuchAlgorithmException e) {
            failure = e;
        }
    }
    
    throw failure;
}
```

其逻辑如下：

对于Providers中当前ProviderList列表，(type,algorithm)在其中对应一组service
，依次尝试使用这些service实例化一个 **clazz** 类型的SPI实例。如果失败则尝试下一个service
。

这里还根据实际情况做了一些优化。让我们考虑如下2种情况:

1. 第一个找到的service就直接成功；大多数
2. 第一个失败了，后面的service也许会成功；少数

GetInstance这里先直接拿第一个满足条件的service来避免一来就直接遍历导致的开销。

## Engine

Engine提供具体的功能实现，被业务层访问，其中包含一个或者多个getInstance(String参数)方法，在getInstance(String参数)方法中会根据参数和它
要实现的功能获取一个或者多个SPI实例。

jca中的Engine样子大多如下如下：

```kotlin
class Engine(
    private val provider: Provider,
    private val spi: Engine?,
) {

    companion object {

        fun getInstance(key: String): Engine {
            val instance = getInstance("Engine", key)
            return Engine(
                provider = instance.provider,
                spi = instance.spi as? Engine
            )
        }

    }

    fun doWork() {
        // 使用当前的spi实现功能
    }

}

interface EngineSpi {

    fun api()

}

fun getInstance(type: String, key: String): Instance {
    // 获取type,key -> spi实例
    // 返回type,key对应的provider和spi实例
}

data class Instance(
    val provider: Provider,// 对应provider
    val spi: Any,//type,key对应的spi实例
)


```

### type/key/SPI

Engine会定义如下关系:
```text
(engine对应的type,key_1) -> SPI_A
(engine对应的type,key_2) -> SPI_A

(engine对应的type,key_3) -> SPI_B
..
```
**type:**

每个engine **只会** 对应同一个type！

**key:**

key与ype下的SPI抽象类/接口是多对一的关系。于是当给engine新增key时，可以不用新增SPI接口。

**SPI:**

engine实现自身业务需要的具体功能实现。一个engine可以可以根据自身需要定义一个或者多个SPI。

engine会将上述关系关系公布出去，其他人就可以使用Provider为engine下的某个key提供对应SPI实现的支持了。

### getInstance(key: String)

这个方法用于Engine根据自身需要实现的功能结合参数去获取 **一个或者多个** 需要的spi实例，然后构造一个自己的实例返回给调用者使用；有时这个方法里
还会做一些判断。

**java官方获取spi的方法：**

```kotlin
// java官方获取spi的方法()
// 1. 获取当前系统注册的全部 Provider 列表
val providerList: ProviderList = Providers.getProviderList()

// 2. 调用底层 GetInstance.getInstance 获取 Instance 对象
// 参数依次为: 服务类型, 基类/接口的 Class, 算法名, Provider 列表
val instance: GetInstance.Instance = GetInstance.getInstance(
    type,
    Class.forName("java.security.MessageDigestSpi"),
    algorithm,
    providerList
)

// 3. 从 Instance 容器中提取出真正的 SPI 实例和对应的 Provider
val spiInstance: Any = instance.impl
val provider: Provider = instance.provider


```

**三方实现者获取spi的方法:**

```kotlin
// 三方实现者获取spi的方法

// 1.获取service
val service: Provider.Service? = Security.getProviders().asSequence()
    .mapNotNull { provider -> provider.getService(type, algorithm) }
    .firstOrNull()

// 2.获取SPI实例
val spiInstance: Any = service.newInstance(null)
```

观察jca中的Engine#getInstance(参数)方法，它们一般使用字符串参数，当Engine新增功能时应用层不需要做任何改动，如果要使用Engine新增的功能，只需要
传新的参数即可。

getInstance()可以有一个或者多个，它的参数也是可以不止一个。

### doWork()

engine此时已经获取到需要的SPI了，利用SPI在这里实现业务逻辑。

### Engine#getInstance(param)的参数与查找SPI时使用的key

param与key是2个不同的东西，只是它们有时恰好一致。例如:
```kotlin
// 假设这个方法内部会用2个key分别查找SPI
getInstance("abc")
```

## 三方开发者为Engine提供/替换SPI实例

首先找到engine公布的type-key-spi关系列表：

```text
(engine对应的type,key_1) -> SPI_A
(engine对应的type,key_2) -> SPI_A

(engine对应的type,key_3) -> SPI_B
..
```

### 提供

假设你的目标是为key_1提供支持，你需要：

1. 实现SPI_A接口/抽象类
2. 实现自定义Provider，在其中注册:
```text
(engine对应的type,key_1) -> 你的SPI_A实现类的全名
```
3. 调用Security的方法将自定义的Provider插入到Providers维护的ProviderList中

### 替换官方Engine的SPI

在第3步时使用如下方法将自定义Provider插到列表首位
```kotlin
Security.insertProviderAt()
```

由于官方的Provider会涉及到优先级列表。如果你有优先级配置权限，则可以为利用配置提高Provider优先级：
```kotlin
Security.setProperty("jdk.security.provider.preferred", "TrustManagerFactory.PKIX:MyManagerProvider")
```

接着就开始祈祷没人覆盖掉你的调用。

### 替换自定义Engine的SPI

在第3步时使用如下方法将自定义Provider插到列表首位
```kotlin
Security.insertProviderAt()
```
然后祈祷不会有人也为同一组(type,key)调用这个方法覆盖掉了你的调用。

## 资料

* [How to Implement a Provider in the Java Cryptography Architecture](https://docs.oracle.com/en/java/javase/11/security/howtoimplaprovider.html#GUID-CC161921-EBD2-48C6-B543-A956658B68B6)
