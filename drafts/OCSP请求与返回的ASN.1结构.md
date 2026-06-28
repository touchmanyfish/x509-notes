# OCSP请求与返回的ASN.1结构

OCSP（在线证书状态协议，Online Certificate Status Protocol）基于 **ASN.1（Abstract Syntax Notation One）** 进行结构界定，并通常使用 **DER（Distinguished Encoding Rules）** 进行二进制编码。

以下是根据 **RFC 6960** 标准整理的 OCSP 请求（Request）与返回（Response）的核心 ASN.1 结构解析。

---

## 一、 OCSP 请求结构 (OCSP Request)

一个完整的 OCSP 请求最外层是 `OCSPRequest` 结构，它包含了要查询的证书信息以及可选的签名。

### 1. 核心 ASN.1 定义

```asn
OCSPRequest     ::=     SEQUENCE {
    tbsRequest                  TBSRequest,
    optionalSignature   [0]     EXPLICIT Signature OPTIONAL }

TBSRequest      ::=     SEQUENCE {
    version             [0]     EXPLICIT Version DEFAULT v1,
    requestorName       [1]     EXPLICIT GeneralName OPTIONAL,
    requestList                 SEQUENCE OF Request,
    requestExtensions   [2]     EXPLICIT Extensions OPTIONAL }

Signature       ::=     SEQUENCE {
    signatureAlgorithm          AlgorithmIdentifier,
    signature                   BIT STRING,
    certs               [0]     EXPLICIT SEQUENCE OF Certificate OPTIONAL }

Version         ::=     INTEGER  { v1(0) }

Request         ::=     SEQUENCE {
    reqCert                     CertID,
    singleRequestExtensions [0] EXPLICIT Extensions OPTIONAL }

```

### 2. 关键字段解析：CertID

`Request` 结构中最核心的是 `reqCert`（`CertID`）。为了保护隐私，OCSP 请求**不直接发送证书明文**，而是发送证书签发者（Issuer）的 Hash 值以及目标证书的序列号。

```asn
CertID          ::=     SEQUENCE {
    hashAlgorithm       AlgorithmIdentifier,
    issuerNameHash      OCTET STRING, -- 签发者 DN 的 Hash
    issuerKeyHash       OCTET STRING, -- 签发者公钥 的 Hash
    serialNumber        CertificateSerialNumber }

```

* **issuerNameHash**: 签发者主体名称（Subject Name）的哈希值。
* **issuerKeyHash**: 签发者公钥（Public Key）的哈希值。
* **serialNumber**: 目标证书的序列号（明文）。

---

## 二、 OCSP 返回结构 (OCSP Response)

OCSP 响应的最外层是 `OCSPResponse`。它首先返回一个状态码，只有当状态码为 `successful` 时，才会包含具体的证书状态内容。

### 1. 最外层结构

```asn
OCSPResponse ::= SEQUENCE {
    responseStatus         OCSPResponseStatus,
    responseBytes      [0] EXPLICIT ResponseBytes OPTIONAL }

OCSPResponseStatus ::= ENUMERATED {
    successful            (0),  -- 正常响应
    malformedRequest      (1),  -- 请求格式错误
    internalError         (2),  -- 服务器内部错误
    tryLater              (3),  -- 服务器忙，请稍后再试
    -- (4) 未使用
    sigRequired           (5),  -- 请求需要签名
    unauthorized          (6)   -- 未授权的请求
}

ResponseBytes ::= SEQUENCE {
    responseType   OBJECT IDENTIFIER, -- 对于 OCSP，通常是 id-pkix-ocsp-basic
    response       OCTET STRING }     -- 包含 BasicOCSPResponse 的 DER 编码

```

### 2. 核心内容结构：BasicOCSPResponse

当 `responseStatus` 为 `successful` 时，`response` 字段内解包后得到 `BasicOCSPResponse`。这也是包含实际证书状态和 CA 签名的核心部分。

```asn
BasicOCSPResponse       ::=     SEQUENCE {
    tbsResponseData             ResponseData,
    signatureAlgorithm          AlgorithmIdentifier,
    signature                   BIT STRING,
    certs               [0]     EXPLICIT SEQUENCE OF Certificate OPTIONAL }

ResponseData ::= SEQUENCE {
    version             [0]     EXPLICIT Version DEFAULT v1,
    responderID                 ResponderID,
    producedAt                  GeneralizedTime, -- 响应生成时间
    responses                   SEQUENCE OF SingleResponse,
    responseExtensions  [1]     EXPLICIT Extensions OPTIONAL }

ResponderID ::= CHOICE {
    byName              [1]     Name,    -- 响应者的 DN
    byKey               [2]     KeyHash  -- 响应者公钥的 SHA-1 哈希
}

```

### 3. 核心字段解析：SingleResponse

`ResponseData` 中的 `responses` 包含一个或多个 `SingleResponse` 结构，对应请求中的证书状态。

```asn
SingleResponse ::= SEQUENCE {
    certID                      CertID,          -- 对应的证书 ID
    certStatus                  CertStatus,      -- 证书状态（核心）
    thisUpdate                  GeneralizedTime, -- 本次状态更新时间
    nextUpdate          [0]     EXPLICIT GeneralizedTime OPTIONAL, -- 下次状态更新时间
    singleExtensions    [1]     EXPLICIT Extensions OPTIONAL }

CertStatus ::= CHOICE {
    good        [0]     IMPLICIT NULL,
    revoked     [1]     IMPLICIT RevokedInfo,
    unknown     [2]     IMPLICIT UnknownInfo }

RevokedInfo ::= SEQUENCE {
    revocationTime              GeneralizedTime, -- 吊销时间
    revocationReason    [0]     EXPLICIT CRLReason OPTIONAL } -- 吊销原因

```

* **certStatus**:
* `good` (0): 证书有效。
* `revoked` (1): 证书已被吊销（会附带吊销时间和原因）。
* `unknown` (2): 响应商不知道此证书（通常是因为它不是该 CA 签发的）。



---

## 三、 总结：结构对应关系图

为了让你更直观地理解，可以把 OCSP 的请求与响应看作以下简化层级：

* **OCSPRequest**
  └── **TBSRequest** (待签名数据)
  ├── **CertID** (Hash算法 + IssuerNameHash + IssuerKeyHash + **序列号**)
  └── Extensions (扩展项，如 Nonce 防重放)
* **OCSPResponse**
  ├── **responseStatus** (如: 0 代表 successful)
  └── **ResponseBytes** (BasicOCSPResponse)
  ├── **ResponseData** (待签名数据)
  │   └── **SingleResponse** -> **CertStatus** (**Good / Revoked / Unknown**)
  └── **Signature** (OCSP 服务器对 ResponseData 的签名)