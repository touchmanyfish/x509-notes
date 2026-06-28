# 证书Authority Key Identifier (AKI)

## 解决问题

在X.509数字证书体系中，AKI 扩展项（Extension）主要为了解决“如何准确、高效地构建和验证证书链”的问题。具体表现在以下三个核心场景：

* **消除证书重名歧义（唯一性识别）：** 一个CA（证书颁发机构）可能会随着时间推移拥有多个不同的密钥对（例如：老密钥到期更换新密钥），或者多个不同的CA使用了相同的 `Subject`（主体名称）。仅靠证书中的 `Issuer`（颁发者名称）无法唯一确定是哪个具体的密钥签发了当前证书。AKI 通过标识**签发该证书的具体公钥**，消除了这种歧义。
* **加速证书链构建（Path Building）：**
  当客户端（如浏览器、服务器）尝试验证一张用户证书时，需要向上寻找其对应的CA证书。AKI 可以作为一个“精准指引”，让客户端直接匹配CA证书的 **Subject Key Identifier (SKI)**，从而快速挑出正确的上级证书，避免盲目遍历信任库。
* **支持交叉证书与滚动更新（Key Rollover）：**
  在CA密钥平滑更替或多个根CA互相交叉签名时，会有多个CA证书拥有相同的 `Subject` 但不同的公钥。AKI 能够引导验证程序在复杂的证书网络中，准确找到用来签名当前证书的那一个特定CA证书。

---

## AKI的ASN.1结构

根据 [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280) 规范，Authority Key Identifier 的 OID 为 `2.5.29.35`。它在 ASN.1（ could-be 抽象语法标记）中的定义如下：

```asn1
-- OID: 2.5.29.35
AuthorityKeyIdentifier ::= SEQUENCE {
    keyIdentifier             [0] KeyIdentifier           OPTIONAL,
    authorityCertIssuer       [1] GeneralNames            OPTIONAL,
    authorityCertSerialNumber [2] CertificateSerialNumber OPTIONAL
}

KeyIdentifier ::= OCTET STRING

```

### 字段详细解析

AKI 是一个 `SEQUENCE` 结构，包含三个可选（OPTIONAL）字段，但规范要求必须包含 `keyIdentifier` 或 `authorityCertIssuer` 与 `authorityCertSerialNumber` 的组合（通常业界标准首选 `keyIdentifier`）。

| 字段名                             | ASN.1 类型                      | 作用与含义                                              |
|---------------------------------|-------------------------------|----------------------------------------------------|
| **`keyIdentifier`**             | `[0] OCTET STRING`            | **CA公钥的哈希值。** 与直接上游证书的SKI中包含的值相等                   |
| **`authorityCertIssuer`**       | `[1] GeneralNames`            | **颁发当前证书的CA证书的颁发者名称** 对应上游证书中的Issuer和上上游证书的Subject |
| **`authorityCertSerialNumber`** | `[2] CertificateSerialNumber` | **颁发当前证书的CA证书的序列号。** 对应上游证书中的Serial Number         |

> 📌 **最佳实践说明：**
> 在实际的互联网应用中（如 WebPKI），为了节省证书体积并提高匹配效率，绝大多数CA只会使用 `keyIdentifier` 字段。而 `authorityCertIssuer` 和 `authorityCertSerialNumber` 组合（合称 Issuer/Serial 方式）较少使用，通常作为备用或在特定的封闭专网环境中使用。


### authorityCertSerialNumber举例解释

**假设有如下证书颁发情况:**
ca1证书 -> ca2证书 -> ca3证书

**此时:**

ca3证书.AKI.authorityCertIssuer 对应 ca2证书.Issuer 对应 ca1证书.Subject