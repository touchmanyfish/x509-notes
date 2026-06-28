# Online Certificate Status Protocol

**世界上所有 `id-ad-ocsp` 中包含的 URL，在接收和处理 OCSP 请求时，其请求和响应的格式都有严格、统一的国际标准规定。**

这个统一的规范是由 IETF（互联网工程任务组）制定的 **RFC 6960**（其前身是 RFC 2560），它定义了 **OCSP（在线证书状态协议，Online Certificate Status Protocol）** 的标准消息格式和传输行为。

无论证书是由 DigiCert、GlobalSign 签发的，还是由你公司内部的私有 CA 签发的，只要宣称支持标准 OCSP，其底层的数据结构都必须严格遵循这个规范。

以下是关于这个“统一规定”的底层细节和传输行为的深入拆解：

---

## 1. 统一的数据结构（ASN.1 定义）

OCSP 请求和响应在传输时，并不是简单的 JSON 或 XML，而是使用 **ASN.1（抽象语法标记一）** 进行结构化定义，并采用 **DER（可分辨编码规则）** 序列化为二进制流。

### 统一的请求格式（OCSPRequest）

一个标准的 OCSP 请求包核心包含以下信息：

* **Version**：协议版本号（目前通常是 v1）。
* **RequestList**：可以同时查询多个证书的状态，每个条目包含：
* **HashAlgorithm**：计算哈希所用的算法（如 SHA-1 或 SHA-256）。
* **IssuerNameHash**：签发者（CA）主体名称（Subject）的哈希值。
* **IssuerKeyHash**：签发者公钥的哈希值。
* **SerialNumber**：**被查询证书的唯一序列号**。


* **RequestExtensions**（可选）：比如 **Nonce（随机数）**，用于防止重放攻击。

> **为什么请求里不直接传证书，而是传这些哈希和序列号？**
> 为了极大地压缩请求包体积，通常一个标准的 OCSP 请求只有不到 100 字节，非常利于高并发的网络传输。

### 统一的响应格式（OCSPResponse）

OCSP 服务器（Responder）返回的响应同样是 DER 编码的二进制，其核心结构为：

* **ResponseStatus**：响应状态码（成功、不合法请求、内部错误、尝试稍后等）。
* **ResponseBytes**（成功时包含）：
* **CertID**：对应请求中的证书标识。
* **CertStatus**：证书状态，严格定义为三种：
* `0`: **Good**（有效/安全）
* `1`: **Revoked**（已被吊销，通常会附带吊销时间和原因）
* `2`: **Unknown**（未知，CA 表示没签发过这个证书，或找不到记录）


* **ThisUpdate / NextUpdate**：本次状态的刷新时间和下次更新时间（缓存控制）。
* **Signature**：**OCSP 服务器对上述内容的数字签名**，客户端会用 CA 的公钥（或专门的 OCSP 签名证书）来验证这个签名的真伪，确保响应没被篡改或伪造。



---

## 2. 统一的传输方式（HTTP Method）

RFC 6960 规定，OCSP 请求主要通过 **HTTP** 协议进行传输（即 AIA 中延伸出的 `http://...` 网址）。标准允许两种 HTTP 方法，客户端通常会优先选择 **GET**。

### 方式 A：HTTP GET（主流，支持缓存）

为了提高性能，标准规定可以将 DER 编码的 OCSP 请求体先进行 **Base64 编码**（并进行 URL 编码），然后直接拼接在 AIA URL 的后面。

* **统一的格式模板**：`{AIA-OCSP-URL}/{Base64编码后的OCSPRequest}`
* **真实网络请求示例**：
```http
GET /MFMwUTBPMHAwajBAMB0WBBT6%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FBBAf8%3D HTTP/1.1
Host: ocsp.digicert.com

```


* **优势**：由于这是一个标准的 GET 请求且 URL 唯一，**它对 CDN 非常友好**。各大 CA 会将这些响应深度缓存在边缘节点（Edge）上，甚至遵循 HTTP 缓存协议，客户端能实现毫秒级响应。

### 方式 B：HTTP POST（备用）

如果请求体太长，或者客户端有特殊限制，则使用 POST 方式。

* **统一的 Content-Type**：请求头的 `Content-Type` 必须严格指定为 **`application/ocsp-request`**。
* **统一的 Accept**：客户端期待的返回类型必须是 **`application/ocsp-response`**。
* **真实网络请求示例**：
```http
POST / HTTP/1.1
Host: ocsp.digicert.com
Content-Type: application/ocsp-request
Content-Length: 115

[115字节的DER二进制数据]

```



---

## 3. 为什么现实中有些差异感？

虽然协议格式在 RFC 层面上高度统一，但我们在实际开发或抓包时，可能会产生“格式不统一”的错觉，这通常是因为以下原因：

1. **哈希算法的差异**：有些客户端在构建 `CertID` 时使用 `SHA-1` 计算 CA 的 Hash，而有些现代客户端使用 `SHA-256`。这导致生成的二进制请求数据不同，但**它们依然属于标准的 ASN.1 语法结构**。
2. **URL 后缀的微小差异**：有些 CA 的 AIA 里面的 URL 后面自带斜杠（如 `http://ocsp.com/`），有些不带（如 `http://ocsp.com`）。客户端在拼接 Base64 串时需要自行做格式化处理。
3. **非标的扩展（Extensions）**：某些私有 CA 可能会在标准请求的 `RequestExtensions` 字段中塞入自定义的 KV 数据。虽然这扩展了内容，但它的容器（ASN.1 框架）依然是标准的。

---

## 总结

不管是哪家 CA 的 `id-ad-ocsp` URL，它们在网络层都是一个**接收标准 `application/ocsp-request` 并返回 `application/ocsp-response` 的 HTTP 接口**。

也正是因为有了这一套完全一致的铁律规范，浏览器、Android、iOS 等操作系统的网络底层，才可以用同一套通用的代码逻辑（例如通过 OpenSSL、BoringSSL 或系统自带的 Security 框架），去校验全世界任何一张标准合规证书的实时状态。