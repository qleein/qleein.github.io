---
layout: post
title:  "openssl AES 加密/解密"
date:   2017-11-01 22:19:00 +0800
---

## AES算法

AES进行加/解密需要考虑下面三个设置。

### 密钥

使用的密钥长度为128/192/256位，这里以128位为例

### 初始向量

初始向量位128位

### 填充

AES以128位，即16字节为单位进行操作，如果明文长度不是16的整数倍就需要进行填充，openssl默认以PKCS#7方式进行填充。PKCS#7填充时将明文长度扩充为16的整数倍，每一个填充的字节值为填充的长度。

例如：

- 如明文长度为8，填充8个字节，每个字节均为0x8。DD表示明文，08为填充。

```| DD DD DD DD DD DD DD DD 08 08 08 08 08 08 08 08 |```
- 如明文长度为16，额外填充16个字节。DD表示明文，10为填充。

```| DD DD DD DD DD DD DD DD DD DD DD DD DD DD DD DD | 10 10 10 10 10 10 10 10 10 10 10 10 10 10 10 10 |```

## openssl命令进行加/解密

1 指定密钥和初始向量

```
$ openssl enc -aes-128-cbc -in in.txt -out out.txt -K 12345678901234567890 -iv 12345678
```

将in.txt文件的内容进行加密后输出到out.txt中。这里通过-K指定密钥，-iv指定初始向量。注意AES算法的密钥和初始向量都是128位的，这里-K和-iv后的参数都是16进制表示的，最大长度为32。
即-iv 1234567812345678指定的初始向量在内存中为 | 12 34 56 78 12 34 56 78 00 00 00 00 00 00 00 00 |。

通过-d参数表示进行解密 如下

```
$ openssl enc -aes-128-cbc -in in.txt -out out.txt -K 12345678901234567890 -iv 12345678 -d
```

表示将加密的in.txt解密后输出到out.txt中

2 通过字符串密码加/解密

```
$ openssl enc -aes-128-cbc -in in.txt -out out.txt -pass pass:helloworld
```

这时程序会根据字符串"helloworld"和随机生成的salt生成密钥和初始向量，也可以用-nosalt不加盐。

## C中调用openssl库

下面这个是C语言调用openssl的例子，来自[openssl官方文档](https://www.openssl.org/docs/manmaster/man3/EVP_CipherUpdate.html])。

```c
 int do_crypt(FILE *in, FILE *out, int do_encrypt)
 {
     /* Allow enough space in output buffer for additional block */
     unsigned char inbuf[1024], outbuf[1024 + EVP_MAX_BLOCK_LENGTH];
     int inlen, outlen;
     EVP_CIPHER_CTX *ctx;
     /*
      * Bogus key and IV: we'd normally set these from
      * another source.
      */
     unsigned char key[] = "0123456789abcdeF";
     unsigned char iv[] = "1234567887654321";

     /* Don't set key or IV right away; we want to check lengths */
     ctx = EVP_CIPHER_CTX_new();
     EVP_CipherInit_ex(&ctx, EVP_aes_128_cbc(), NULL, NULL, NULL,
                       do_encrypt);
     OPENSSL_assert(EVP_CIPHER_CTX_key_length(ctx) == 16);
     OPENSSL_assert(EVP_CIPHER_CTX_iv_length(ctx) == 16);

     /* Now we can set key and IV */
     EVP_CipherInit_ex(ctx, NULL, NULL, key, iv, do_encrypt);

     for (;;) {
         inlen = fread(inbuf, 1, 1024, in);
         if (inlen <= 0)
             break;
         if (!EVP_CipherUpdate(ctx, outbuf, &outlen, inbuf, inlen)) {
             /* Error */
             EVP_CIPHER_CTX_free(ctx);
             return 0;
         }
         fwrite(outbuf, 1, outlen, out);
     }
     if (!EVP_CipherFinal_ex(ctx, outbuf, &outlen)) {
         /* Error */
         EVP_CIPHER_CTX_free(ctx);
         return 0;
     }
     fwrite(outbuf, 1, outlen, out);

     EVP_CIPHER_CTX_free(ctx);
     return 1;
 }
```

do_encrypt为1时加密，为0时解密。

需要注意的在调用EVP_CipherUpdate时，`EVP_CipherUpdate(ctx, outbuf, &outlen, inbuf, inlen)`这里outbuf的长度必须要超过`inlen + EVP_MAX_BLOCK_LENGTH`。
这是填充的影响，AES-128-CBC加密时末尾会有1~16字节的填充。解密时，有时候在调用EVP_CipherFinal_ex之前无法确定是否密文已经结束。如这里每次从文件中读入1024个字节，调用EVP_CipherUpdate解密时，对最后的16个字节，可能有部分是填充，解密时需要去掉，为此这里只返回1008个字节，outlen为1008，剩余16个字节的数据保存下来。再读入1024个字节进行解密时，便可能返回1024+16个字节，如果outbuf的长度不够，便会发生内存越界。
所以函数声明变量时，`unsigned char inbuf[1024];`和` unsigned char outbuf[1024+EVP_MAX_BLOCK_LENGTH];`的原因（这这里数据块长度为16个字节）。

另外，这里使用密钥`01234567890abcdeF`，对应的openssl命令行操作时`-K 30313233343536373839616263646546`。因为字符0的ASCII码为0x30，以此类推，同样
初始向量`1234567887654321`，对应openssl命令的参数`-iv 31323334353637383837363534333231`。下面的命令与上面的函数效果相同。

```
$ #加密
$ openssl enc aes-128-cbc -in in.txt -out out.txt -K 30313233343536373839616263646546 -iv 31323334353637383837363534333 -e
$ #解密
$ openssl enc aes-128-cbc -in in.txt -out out.txt -K 30313233343536373839616263646546 -iv 31323334353637383837363534333 -d
```

如果是用openssl命令加密文件，在其他程序中读取解密的话，这里就需要额外注意。

也可以通过`int EVP_CIPHER_CTX_set_padding(EVP_CIPHER_CTX *x, int padding);`取消填充，这时明文长度必须要16的整数倍。