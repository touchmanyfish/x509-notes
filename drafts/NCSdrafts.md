public TrustManagerImpl(

        KeyStore keyStore, CertPinManager manager, ConscryptCertStore certStore) 



manager会被赋值给pinManager，当调用TrustManagerImpl的checkXXXTrusted()方法时，pinManager.checkChainPinning()会被调用(如果它不为null)



if (pinManager != null) {

    pinManager.checkChainPinning(host, wholeChain);

}



checkChainPinning()方法负责对证书链进行证书固定验证。



public interface CertPinManager {

    /**

     * Given a {@code hostname} and a {@code chain} this verifies that the

     * certificate chain includes pinned certificates if pinning is requested

     * for {@code hostname}.

     */

    void checkChainPinning(String hostname, List<X509Certificate> chain)

            throws CertificateException;

}





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



你注意这一行：

mDelegate = new TrustManagerImpl(store, null, certStore);



TrustManagerImpl第二个CertPinManager参数为null，所以当TrustManagerImpl当checkXXXTrusted()被调用时，他内部不会进行证书固定的检查。



## TrustManagerImpl第二个参数总是null，为了控制证书固定权限的位置
* [android-7.0.0_r1 NetworkSecurityTrustManager.java](https://github.com/aosp-mirror/platform_frameworks_base/blob/android-7.0.0_r1/core/java/android/security/net/config/NetworkSecurityTrustManager.java)

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

        // gemini搜索了所有NetworkSecurityTrustManager的版本
        // TrustManagerImpl的第二个参数总传递的null
        // gemini说android系统希望将证书固定的控制权留在framework层而不是公共的安全组件Conscrypt中
        mDelegate = new TrustManagerImpl(store, null, certStore);
    } catch (GeneralSecurityException | IOException e) {
        throw new RuntimeException(e);
    }
}
```
## checkPins(List<X509Certificate> chain)

```xml
<pin digest="摘要算法">disest_摘要算法(证书公钥)=</pin>

<!--例如-->
<pin digest="SHA-256">disest_SHA_256(证书公钥)=</pin>
```
checkPins()方法用于检查chain中是否至少有一个证书公钥的digest值与对应pin-set中匹配。

### 主pin backpin检查

android中没有有对backup pin检查

### RootTrustManager什么时候被使用？



### okhttp如何使用NCS中的东西


### okhtt的pinner

