---
layout: post
title: ShadowSocks启动报错undefined symbol EVP_CIPHER_CTX_cleanup
category: ErrorNote
tags: ShadowSocks ErrorNote Proxy
---


* TOC 
{:toc}

> 解决openssl升级到1.1.0以上版本，导致shadowsocks2.8.2启动报undefined symbol: EVP_CIPHER_CTX_cleanup错误。





## 准备翻墙冲浪的时候Shadowsocks报错如下：

```bash
AttributeError: /usr/local/ssl/lib/libcrypto.so.1.1: undefined symbol: EVP_CIPHER_CTX_cleanup
shadowsocks start failed
```

## 解决方法 ：

### 1. vim打开文件openssl.py

`vim /usr/local/lib/python3.5/dist-packages/shadowsocks/crypto/openssl.py `

> 路径不同根据报错路径而定

### 2. 修改libcrypto.EVP_CIPHER_CTX_cleanup.argtypes 

`:%s/cleanup/reset/`

`:x`

> 以上两条为VIM命令， 替换文中**libcrypto.EVP_CIPHER_CTX_cleanup.argtypes** 为**libcrypto.EVP_CIPHER_CTX_reset.argtypes **共两处，并保存

### 3. 运行Shadowsocks

OK

## 原因： 

这个问题是由于在openssl1.1.0版本中，废弃了EVP_CIPHER_CTX_cleanup函数，如官网中所说：

> EVP_CIPHER_CTX was made opaque in OpenSSL 1.1.0. As a result, EVP_CIPHER_CTX_reset() appeared and EVP_CIPHER_CTX_cleanup() disappeared. EVP_CIPHER_CTX_init() remains as an alias for EVP_CIPHER_CTX_reset().
> 
> **EVP_CIPHER_CTX_reset函数替代了EVP_CIPHER_CTX_cleanup函数**

EVP_CIPHER_CTX_reset函数说明：

> EVP_CIPHER_CTX_reset() clears all information from a cipher context and free up any allocated memory associate with it, except the ctx itself. This function should be called anytime ctx is to be reused for another EVP_CipherInit() / EVP_CipherUpdate() / EVP_CipherFinal() series of calls.

EVP_CIPHER_CTX_cleanup函数说明：

> EVP_CIPHER_CTX_cleanup() clears all information from a cipher context and free up any allocated memory associate with it. It should be called after all operations using a cipher are complete so sensitive information does not remain in memory.

**可以看出，二者功能基本上相同，都是释放内存，只是应该调用的时机稍有不同，所以用reset代替cleanup问题不大。**

## 提示：

openssl1.1.0目前兼容性很不好，大部分的软件都不支持
目前支持的有nginx-1.11.5、curl-7.50.3
不支持的有PHP-7.0.12、openssh-7.3p1
所以如果决定使用openssl1.1.0需要考虑很多兼容问题，必须保留1.0.2或1.0.1(不推荐，存在一些已知漏洞，最重要的是如果服务器要开http2，由于新版chrome必须使用ALPN的限制，只有1.0.2版本支持ALPN，所以必须升级到1.0.2)版本以便编译其他程序。
