---
title: HTTPS认证过程
categories: 
    - https
tags: 
    - https
---
HTTPS是在HTTP跟TCP之间加了一层对数据进行加解密的SSL/TLS

- SSL: Secure Sockets Layer
- TLS: Transport Layer Security


<br />HTTPS认证过程, 主要有三部分

- Server证书的认证阶段, 即得到CA机构的签名
- Server跟Client之间通过非对称的方式验证彼此身份
- Server跟Client之间通过双方认同的随机数做对称加密数据传输


<br />![证书的认证，以及HTTPS的工作流程.svg](https-1.svg)
