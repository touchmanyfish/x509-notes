# 证书中的Must-Staple扩展

* RFC 7633

## 理解

Must-Staple扩展扩展表示证书在被提供的时候，
证书所有者(例如网站)在提供带Must-Staple扩展的证书时需要同时提供对应的ocsp response。证书请求者(例如客户端)如果发现证书中带Must-Staple扩展但是
却没有收到对应的ocsp response，就会强制报错。

在tls握手过程，只有当客户端发送了status_request(tls 1.2/1.3)/status_request_v2(tls 1.2)扩展，服务器才可能会发送stapling信息。所以
为了让证书扩展Must-Staple的语义得到满足，客户端需要尽量发送status_x扩展来支持Must-Staple。

为了满足Must-Staple语义而发送的status_x还会导致服务器为不带Must-Staple扩展的证书发送ocps respone

```text
// 客户端发送扩展
status_request

// 服务器发送证书
root证书 -> ca1证书 -> 服务器证书(Must-Staple)

// 服务器选择发送的stapling
ca1 ocsp respon|服务器证书 ocsp response
```

如果客户端忘记发送status_x扩展，服务器就一定不会发送stapling信息。客户端收到证书时Must-Staple的语义得不到满足，于是强制报错。

