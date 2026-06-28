# 证书SignatureValue

在 CA 证书的数字签名体系中，`Signature Value`（签名值）的本质是 CA 私钥对证书主体（`tbsCertificate`
）经过处理后的数据进行密码学运算的结果。其通用的逻辑模型可以表达为：
$$input = 签名算法指定输入$$
$$\text{Signature Value} = \text{签名算法}_{\text{CA_Private_Key}}(\text{input})$$


---

### 举例:签名算法选择 RSA (PKCS #1 v1.5) 时

* **Signature Value**：
  $$input = Padding( \text{DER_Encode}(DigestInfo(RawHash(TBS))) )$$
  $$\text{Signature Value} = \text{RSA}_{\text{CA_Private_Key}}(\text{input})$$

* **具体构造流程**：

1. 使用证书签发者选定的hash算法计算TBSCertificate部分的rawHash
2. 将hash算法对应的OID与rawHash打包进ASN.1编码的DigestInfo结构中
3. 将DigestInfo按照DER规则序列化为二进制
4. 使用PKCS#1 v1.5对二进制进行填充
5. 最后使用RSA对填充后的二进制进行加密得到Signature Value的值

### 举例:签名算法选择 ECDSA 时

* **Signature Value**：
  $$input = RawHash(TBS)$$
  $$\text{Signature Value} = \text{ECDSA}_{\text{CA_Private_Key}}(\text{input})$$

* **具体构造流程**：

1. 使用证书签发者选定的hash算法计算TBSCertificate部分的rawHash
2. 使用ECDSA算法对rawHash签名运算得到Signature Value的值

### 举例:签名算法选择 RSA (PSS 模式) 时

* **Signature Value**：

$$input = EMSA\_PSS\_Encode(RawHash(TBS), salt)$$


$$\text{Signature Value} = \text{RSA}_{\text{CA_Private_Key}}(\text{input})$$


* **具体构造流程**：

1. 使用证书签发者选定的 hash 算法计算 TBSCertificate 部分的 rawHash。
2. 生成一个强随机的字节流作为盐值（salt）。
3. 将 rawHash、salt 以及固定填充通过掩码函数（MGF1）和异或（XOR）运算，在二进制层面“打散编织”成符合 EMSA-PSS 规范的数据块。
4. 将该数据块对齐并撑满至与 RSA 密钥等长，作为最终的输入矩阵 input。
5. 使用 RSA 私钥对 input 进行签名运算得到 Signature Value 的值。

