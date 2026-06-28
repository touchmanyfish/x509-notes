# CA,OCSP服务器,OCSP证书,普通证书(TODO)


## 角色

**CA:**

签发证书的机构，可以签发普通证书和OCSP证书；可以指定OCSP服务器；

**OCSP服务器:**

处理证书吊销状态查询请求。OCSP服务器可以持有多张由同一CA签发的OCSP证书；CA也可以签发多张OCSP证书，将其分配个多个OCSP服务器。

**OCSP证书:**

CA签发的证书，OCSP服务器在为证书状态查询response签名时，可以选择使用OCSP证书的私钥进行签名。
OSCP证书带有id-pkix-ocsp-nocheck 扩展，表示不用验证它的吊销状态。

**普通证书:**

用于身份验证的证书，例如CA证书或者终端证书。

## 谁可以签发OCSP证书


* 客户端本地配置直接信任的机构（Matches a local configuration...）。
* 响应由签发该证书的同一个 CA（the CA that issued the certificate）亲自签名，或者由该 CA 直接委派（MUST be issued directly by the CA...）的 OCSP 证书签名。

RFC 原文：
"This certificate MUST be issued directly by the CA that is identified in the request."

## OCSP请求与返回结果对应的ASN.1结构理解

### 请求结构

观察请求结构可知，可以在查询时同时提交多个证书的信息。

### 返回结构

观察返回结构可知，返回结构包含一个返回数据的签名，和一个或者多个证书的吊销状态

当返回结果包含success(0)时，返回结果才会被签名。

## 查询

### 查询的证书的要求

你查询时提交了一批证书，这批证书的签发者CA必须是同一个

```text
ca_A证书 -> 证书1(AIA:ocsp1.url)
ca_A证书 -> 证书2(AIA:ocsp1.url)

ca_B证书 -> 证书3(AIA:ocsp2.url)
```
如果你查询时提交(证书1，证书2，证书3)，是不行的。

#### 查询方式1:按相同ca分组
```text
ca_A证书 -> ocsp1证书
ca_A证书 -> ocsp2证书

ca_A证书 -> ocsp3证书

ca_A证书 -> 证书1(AIA:ocsp1.url)
ca_A证书 -> 证书2(AIA:ocsp2.url)

ca_B证书 -> 证书3(AIA:ocsp3.url)
ca_B证书 -> 证书4(AIA:ocsp3.url)
```
如果要查询证书1～4，可以按照ca分为两组进行查询:
* (证书1，证书2) -> ocsp1.url或ocsp2.url
* (证书3，证书4) -> ocsp3.url

#### 查询方式2:按ocsp_url分组

```text
ca_A证书 -> ocsp1证书
ca_A证书 -> ocsp2证书
ca_A证书 -> ocsp3证书


ca_A证书 -> 证书1(AIA:ocsp1.url)
ca_A证书 -> 证书2(AIA:ocsp1.url)

ca_A证书 -> 证书3(AIA:ocsp2.url)
ca_A证书 -> 证书4(AIA:ocsp3.url)
```
如果要查询证书1～4，可以按照ocsp_url分为三组进行查询:
* (证书1，证书2) -> ocsp1.url或者ocsp2.url
* (证书2) -> ocsp2.url
* (证书3) -> ocsp3.url

## 签名方式

ocsp服务器收到的待查询证书都是由同一个CA(假设叫做ca_A)签发并且可以返回succes(0)时，需要对返回结果进行签名。

(注：服务器可以同时支持如下2种签名方式)

### 委派签名(Authorized Responder)

如果ocsp服务器持ca_A签发的ocsp证书，则可以从这些由ca_A签发的ocsp证书中选择一张，使用它的的私钥对返回结果签名。

### CA 直接签名(CA Direct Signing)

如果ocsp服务器可以向ca_A发起“实时在线签名”或者CA给ocsp服务器提供“预签名机制”，则查询结果可以被ca_A的私钥签名。

**工业界实践：**
* **实时在线签名（Online Signing）：** OCSP 服务器可以直接调用高安全级别的硬件安全模块（HSM）中的 CA 私钥进行实时签名。但因为 CA 私钥权限极高，这种做法对安全防护要求极严。
* **预签名机制（Pre-signing）：** 这是目前更主流的高并发方案（如大厂或 CDN 常用的 OCSP Stapling 衍生方案）。CA 定期（如每隔 1~3 天）批量生成
好所有证书的 OCSP 响应并用 CA 私钥签名，OCSP 服务器只需要将这些静态的预签名文件直接分发给查询者，性能极高且无私钥泄露风险。

### certs取值情况

responderID标识给response签名的证书，对应的证书有如下情况

1. 情况 A（CA 亲自签名）：该证书的公钥与目标证书的 Issuer的公钥完全一致(CA的原版/交叉签名版证书)。
2. 情况 B（CA 委派签名）：该证书由目标证书的Issuer直接签署，且该证书带有 id-kp-OCSPSigning 扩展。
3. 其他情况：一律表示验证失败，向调用者返回结果。

certs可能的取值为：null或者证书链(一些辅助构建证书链的证书，可能不包含直接签名response的证书)

在做后续验证前，需要先查找并验证responderID对应的证书。查找优先级为：certs -> 本地

当找到对应证书以后如果：
1.如果是目标查询证书的直接上级(这里是ca1)，或者
2.这个证书被目标查询证书的直接上级(这里是ca1)签署，并且包含id-kp-OCSPSigning

则表示初步验证通过，可以对这个reponse包含的证书链进行后续的验。

## 通过ocsp验证指定证书的流程

参考[通过ocsp验证指定证书的流程](通过ocsp验证指定证书的流程.md)

### 为什么只有一个证书，responses还要设计为一个列表


