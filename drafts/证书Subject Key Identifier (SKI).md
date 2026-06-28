# 证书 Subject Key Identifier (SKI)

## 解决问题

在没有 SKI 之前，构建证书链（Certificate Chain Validation）主要依赖颁发者名称（Issuer）**与**使用者名称（Subject）的字符串匹配。但在实际应用中，这种方式暴露出了两个核心痛点：

* **同名 CA 证书冲突（CA 密钥更新）**：当一个 CA 的证书到期或算法升级时，管理员通常会使用**完全相同的 Subject 名称**生成一把新的密钥对。此时，如果不看密钥本身，仅凭名字无法分辨由该 CA 签发的下级证书到底属于旧密钥还是新密钥。
* **证书链构建效率低下**：在交叉证书（Cross-Certification）或复杂的网状 PKI 拓扑中，同一个 Subject 可能会有多张不同的证书。客户端在验证时，必须下载并尝试对每一张同名证书进行签名验签，这种高昂的密码学运算开销在网络握手时是无法接受的。

**SKI 的引入，本质上就是为证书的公钥建立了一个唯一的“指纹”或“身份证号”。**
客户端在构建证书链时，SKI中包含的hash值可以加快证书的定位。

## SKI的ASN.1结构

在 X.509 V3 证书中，SKI 作为一个标准的扩展项（Extension）存在，其内部呈现**双层嵌套**的八位位组串（`OCTET STRING`）结构。

### 1. 语法定义

根据 RFC 5280 规范，SKI 在 ASN.1 中的标准定义极为精简：

```asn
-- 外层属于通用的 Extension 结构
Extension ::= SEQUENCE {
    extnID      OBJECT IDENTIFIER,       -- 固定为 2.5.29.14 (id-ce-subjectKeyIdentifier)
    critical    BOOLEAN DEFAULT FALSE,   -- 固定为 FALSE（非关键扩展）
    extnValue   OCTET STRING             -- 外部包装容器，内含真正结构的 DER 编码
}

-- 内层才是 SKI 专属的结构定义
SubjectKeyIdentifier ::= OCTET STRING    -- 纯粹的二进制字节数组，存放公钥哈希

```

### 2. 嵌套拓扑视图

在二进制 DER 编码层面，这种双层设计会导致出现两次类型标签 `04`（OCTET STRING）：

```text
Sequence (Extension 容器)
 ├── OBJECT IDENTIFIER = 2.5.29.14 (id-ce-subjectKeyIdentifier)
 └── OCTET STRING (extnValue，外层盲盒包装)
      └── OCTET STRING (SubjectKeyIdentifier，内层真实的公钥哈希值)

```

---

## 计算SKI的值

虽然 ASN.1 仅在语法上规定 SKI 必须是个 `OCTET STRING`（字节数组），但为了确保不同 CA 厂商生成的标识符具有互操作性，RFC 5280 明确推荐了两种具体的计算方法。

无论是哪种方法，计算的**输入源**都必须是证书中的 **`SubjectPublicKeyInfo` 结构中的 `subjectPublicKey` 字段（即去除了算法标识、纯粹的公钥比特串数据，不包含外层的 ASN.1 Tag 和 Length）**。

### 方法一：160位 SHA-1 哈希（最主流、最常见）

这是目前大多数公共证书颁发机构（如 Let's Encrypt、DigiCert）以及工具（如 OpenSSL）默认采用的方法。

* **计算公式**：

$$\text{SKI} = \text{SHA-1}(\text{subjectPublicKey})$$


* **输出结果**：固定的 20 字节（160 比特）二进制流。

### 方法二：四位固定比特前缀拼接（常用于特定兼容性场景）

这种方法同样基于 SHA-1，但通过缩短哈希长度并加入固定信息，生成一个更短的 64 位标识符。

1. 首先对 `subjectPublicKey` 计算 SHA-1 哈希，得到 160 位数据。
2. 取该哈希值的**最后 60 位（比特）**。
3. 在这 60 位数据的**最前端，拼接 4 位的固定比特串 `0100`（即十六进制的 `4`）**。

* **输出结果**：固定的 8 字节（64 比特）二进制流。

> **现代演进趋势：**
> 随着 SHA-1 在密码学上的逐渐退役，现代一些特定的闭环安全系统或新标准开始允许使用 **SHA-256** 来计算 `subjectPublicKey` 的哈希作为 SKI 的值（输出 32 字节）。但由于老旧客户端的兼容性限制，公网证书目前仍以方法一（20字节 SHA-1）最为普及。