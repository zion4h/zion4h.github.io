---
title: MIT 6.Null 安全和密码学
date: 2023-05-04 22:16:10
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img010.png
thumbnail: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img010.png
categories: [编程, Labs, MIT 6.Null]
tags:
    - shell
    - security
    - cryptography
toc: true
---

标题很唬人，但实际上我们并不需要去学习如何设计安全系统或密码协议，而只需要理解和实际编程相关的东西，比如在 Git 中使用 **Hash 函数**或在 SSH 中使用 **KDF** 和**对称非对称加密**，密码学突然就接地气了。

<!-- more -->

## 熵 Entropy

想要保证计算机安全，就需要高强度的密码，让别人猜不着。但是什么才算高强度的密码呢？答案是**香农熵**。香农熵常用来测量随机性，而_随机性越高的密码强度也更高_。

![密码强度](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/password_strength.png)

正如上面 XKCD 漫画所描述的那样，密码强度这个概念可能有点反直觉，因为人们通常会将 “**高强度密码**” 同 “**人类难记住的密码**” 相混淆。那些用古诗去切换密码并洋洋得意的人，比如 `2Ghymcl-1Hblsqt`，甚至不少是计算机专家。

言归正传，香农熵强度单位是 bit，可以由 `log_2(# of possibilities)` 计算得到，比如投一个骰子会出现 6 种点数，那它的随机性就是 2.58 bits。多少强度的密码可以被认为是安全的呢？一般来说，线上密码应该大于 40，线下则要求更高，需要 80。原因不难想到，线下能够利用硬件加速，而线上有网络等限制。

## Hash 函数

**CHF([Cryptographic hash function](https://en.wikipedia.org/wiki/Cryptographic_hash_function))**  可以将任意长度的数据都映射成一个固定长度。

我们拿 **[SHA1](https://en.wikipedia.org/wiki/SHA-1)** 举例，Git 就有采用 SHA1，它能够将任意输入都转化成一个 160-bit 的输出，合 40 个 HEX 码。我们也可以在命令行中尝试使用，

```sh
[vagrant@localhost ~]$ printf 'hello\n' | sha1sum
f572d396fae9206628714fb2ce00f72e94f2258f  -
[vagrant@localhost ~]$ printf 'hello\n' | sha1sum
f572d396fae9206628714fb2ce00f72e94f2258f  -
[vagrant@localhost ~]$ printf 'Hello\n' | sha1sum
1d229271928d3f9e2bb0375bd6ce5db6c6d348d9  -
```

理想的哈希函数有以下属性：

* **Deterministic 确定性**，相同的输出总是得到相同的结果，函数本身不会变
* **Non-invertible 不可逆**，通过结果你很难倒退出一个合理输入
* **Target collision resistant 目标抗碰撞**，给一个输入，你很难找到一个具有相同输出的另一输入
* **Collision resistant 抗碰撞**，你很难找到具有相同输出的两个不同的输入，和目标抗碰撞概念相仿但明显要求更加严格

虽然 SHA-1 很流行，但它已经 [不再](https://shattered.io/) 被认为是强力 CHF，这里有一张 [CHF 生命周期表](https://valerieaurora.org/hash.html)，

## KDFs

**KDFs**（[key derivation functions 密钥推导函数](https://en.wikipedia.org/wiki/Key_derivation_function)）是一个和 CHF 相关的概念，常用来生成一个固定输出，并将其用作其他加密函数的密钥。KDFs 会故意放慢速度，来减缓离线暴力攻击。

具体到实际运用场景，我们用户并不希望网站明文传输或者存储我们的密码，这样安全风险太高且侵犯隐私。因此，可以将密码和 [盐](https://en.wikipedia.org/wiki/Salt_\(cryptography\))（随机生成）混合得到 KDF 并存储。而当需要验证时，再将用户输入和同一份**盐**混合，与之前存储的 KDF 对照。

## 对称加密

**对称加密**解决的是隐藏信息问题，也就是让通信双方以一种独有方式进行交流，其他人无法破解他们的交流内容。实现的原理也很简单，双方持有一份相同的密钥，可以用这个密钥加密明文，也能用这个密钥解开密文。

```plain-text
keygen() -> key  (this function is randomized)

encrypt(plaintext: array<byte>, key) -> array<byte>  (the ciphertext)
decrypt(ciphertext: array<byte>, key) -> array<byte>  (the plaintext)
```

密钥 key 可以由 KDF 生成，如 `key = KDF(passphrase)`。可以参考一些使用较为广泛的对称加密算法，比如 [AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard)。

## 非对称加密

“非对称” 指的是有两个密钥，具有两个不同的角色。顾名思义，私钥是用来保密的，而公钥可以公开共享并且不会影响安全性（与对称密钥中的共享密钥相区别）。

非对称加密提供加密、解密和签名、验证功能：

```plain-text
keygen() -> (public key, private key)  (this function is randomized)

encrypt(plaintext: array<byte>, public key) -> array<byte>  (the ciphertext)
decrypt(ciphertext: array<byte>, private key) -> array<byte>  (the plaintext)

sign(message: array<byte>, private key) -> array<byte>  (the signature)
verify(message: array<byte>, signature: array<byte>, public key) -> bool  (whether or not the signature is valid)
```

由公钥加密的信息只能由私钥解开，用函数表达就是 `decrypt(encrypt(m, public key), private key) = m`

## 实例

### 密码管理

常用的密码有 [KeePassXC](https://keepassxc.org/)（我最喜欢的💖），[pass](https://www.passwordstore.org/) 和 1Password（收费😅）。密码管理工具可以避免重复使用密码，随机生成的高熵密码，并且所有密码保存在一个地方，使用对称加密，密钥使用 KDF 从密码短语生成。

### 2FA

**2FA**（[Two-factor authentication](https://en.wikipedia.org/wiki/Multi-factor_authentication) 双因子认证）要求使用**知识因子**（“你知道的东西”，其实就是密码😁）和另一个**占有因子**（“你拥有的东西”）。例如，网上银行应用程序要求用户输入密码和通过短信发送到手机上的验证码时，就使用了 **2FA**。

由于破解第二个认证因子需要付出更多，并且其他类型的因子更难以窃取或伪造，因此 **2FA** 可提高帐户安全性，并更好地保护组织及其用户免遭未经授权的访问。

### 全盘加密

**全盘加密**使用对称加密保护整个磁盘，是在电脑被盗时用来保护数据的。在 Linux 上使用 [cryptsetup + LUKS](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_a_non-root_file_system)，在 Windows 上使用自带的 [BitLocker](https://fossbytes.com/enable-full-disk-encryption-windows-10/)，在 macOS 上使用 [FileVault](https://support.apple.com/en-us/HT204837)。

### 私信

使用 Signal 或 Keybase，通过非对称加密保证端到端的安全性，但获取联系人的公钥是这里的关键步骤。

### SSH

在运行 `ssh-keygen` 时，它会随机生成（随机性由操作系统提供信息生成）一个非对称密钥对 **public\_key** 和 **private\_key**。公钥按原样存储（它是公开的，所以保密并不重要），但私钥应该在磁盘上加密。 `ssh-keygen` 程序会提示用户输入密码，并通过 KDF 生成密钥，然后使用这个对称密钥对私钥进行加密。

在使用中，一旦服务器知道客户端的公钥（存储在 **.ssh/authorized\_keys** 文件中），连接的客户端就可以使用**非对称签名**证明其身份，通过**挑战应答认证机制**（[CRAM](https://en.wikipedia.org/wiki/Challenge%E2%80%93response_authentication)）来完成。在宏观层面上看，服务器选择一个随机数并将其发送给客户端，然后客户端签署此消息并将签名发送回服务器，服务器根据记录的公钥检查签名。这有效地证明客户端拥有与服务器的 **.ssh/authorized\_keys** 文件中的公钥对应的私钥，因此服务器可以允许客户端登录。

## 其他

[_Cryptographic Right Answers_](https://latacora.micro.blog/2018/04/03/cryptographic-right-answers.html)
