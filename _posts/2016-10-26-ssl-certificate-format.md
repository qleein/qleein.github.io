---
layout: post
title:  "SSL证书格式"
date:   2016-10-26 22:51:00 +0800
---

## PKCS

Public-Key Cryptography Standards (PKCS)是由美国RSA数据安全公司及其合作伙伴制定的一组公钥密码学标准，其中包括证书申请、证书更新、证书作废表发布、扩展证书内容以及数字签名、数字信封的格式等方面的一系列相关协议。

## 公钥文件

RSA公钥包括两部分modules 和publicExponent，是两个正整数，用n和e表示，

在PKCS中是以ANS.1的语法定义的。如下所示，SEQUENCE是一个有序序列，序列包括两个元素，皆为正整数，用INTEGER表示。

```
RSAPublicKey ::= SEQUENCE {
    modulus           INTEGER,  -- n
    publicExponent    INTEGER   -- e
}
```

### DER与PEM

ANS.1只是抽象的语法表示，需要采用特定的编码规则保存下来，这里常用的是DER编码，DER编码采用TVL三元组(tag-length-value)来表达。

30:06:02:01:11:02:01:11(十六进制表示)  这8个字节就可以表示一个简单到极点的RSA公钥的格式。30是一个tag，表示一个SEQUENCE，06是后面值的长度。02是tag代表整数，01指一个字节，value是17,，当然实际的证书不可能这么短。

DER编码成二进制的字节流就可以作为公钥文件保存了，这就是DER格式的文件。但是这种形式很不直观，不太方便在网络上传输，于是人们对这种二进制的文件进行base64编码，再添加一些额外的头部尾部，就成了PEM格式的文件。

```
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDbTzz3OkCxoYe+8tqo1vLnv9I8
2cylAun523hbS9XIDzcheoEX+T5kRb+dkMc2OcGFHK1nD0vwVP+/OThxMtlxE3T5
JwOfKhgkIINdoaQqtINFuW8veC8dwDGOIYNqYXG98PYSA7pviZCshoZpkoU17Pnc
8k/WbUYepf8Y27gsRQIDAQAB
-----END PUBLIC KEY-----
```

这是一个PEM格式的RSA公钥文件，头部和尾部的BEGIN END是很明显的PEM格式，表明这是一个RSA公钥，中间的数据是base64编码的，base64编码的64个字符是A-Za-z0-9+/。换行只是为了展示方便添加的，没有实际意义。

去掉----BEGIN PUBLIC KEY---- 和-----END PUBLIC KEY-----，对中间的内容进行base64解码，如下所示。

```
\x30 \x81 \x9f 

  \x30 \x0d 
     \x06 \x09 
          \x2a \x86 \x48 \x86 \xf7 \x0d \x01 \x01 \x01 
     \x05 \x00 
  \x03 \x81
       \x8d \x00
  \x30 \x81 \x89 
       \x02 \x81 \x81
            \x00 \xdb \x4f \x3c 
            \xf7 \x3a \x40 \xb1 \xa1 \x87 \xbe \xf2 
            \xda \xa8 \xd6 \xf2 \xe7 \xbf \xd2 \x3c 
            \xd9 \xcc \xa5 \x02 \xe9 \xf9 \xdb \x78 
            \x5b \x4b \xd5 \xc8 \x0f \x37 \x21 \x7a 
            \x81 \x17 \xf9 \x3e \x64 \x45 \xbf \x9d 
            \x90 \xc7 \x36 \x39 \xc1 \x85 \x1c \xad 
            \x67 \x0f \x4b \xf0 \x54 \xff \xbf \x39 
            \x38 \x71 \x32 \xd9 \x71 \x13 \x74 \xf9 
            \x27 \x03 \x9f \x2a \x18 \x24 \x20 \x83 
            \x5d \xa1 \xa4 \x2a \xb4 \x83 \x45 \xb9 
            \x6f \x2f \x78 \x2f \x1d \xc0 \x31 \x8e 
            \x21 \x83 \x6a \x61 \x71 \xbd \xf0 \xf6 
            \x12 \x03 \xba \x6f \x89 \x90 \xac \x86 
            \x86 \x69 \x92 \x85 \x35 \xec \xf9 \xdc 
            \xf2 \x4f \xd6 \x6d \x46 \x1e \xa5 \xff 
            \x18 \xdb \xb8 \x2c \x45 
       \x02 \x03 
            \x01 \x00 \x01 
```

这里用十六进制表示，如前面所述，最开始的x30是tag表示后面是一个序列，长度比较特殊，x81表示后面的长度用一个字节表示， 为x9f.

主要关注最后一个序列x30 x81 x89。这个序列有两个元素，一个整数长度x81，即129个字节。后一个也是整数， 长度为3,值为x010001即65537,。这两个就是RSA公钥的两个参数。

RSA密钥为1024位，应该占128个字节，因为整数有正负，最高位为1时为负数。这里在最前方增加一个x00，表示是正整数，所以长度是129字节

