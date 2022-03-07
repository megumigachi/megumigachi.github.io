---
layout: post
title: "从接口视角看密码学原语"
categories:
  - CS
tags:
  - crypto
toc: true
toc_sticky: true
toc_label: 目录
---

现代密码学主要是建立在问题的困难性上的：解密相当于求解一个困难的数学问题。这些精妙的构造是人类智慧的精华，要想完全理解他们，需要不少的数学基础。但其实密码学也是模块化的，它提供了各种基础的原语，作为对密码学没那么感兴趣的普通人，尤其是开发者，我们可以先不用完全搞懂里面的数学问题，而是看它提供了什么样的接口，保证了什么性质，这种常识性的理解其实更实用也更重要。

*Disclaimer: 因此本文在具体数学构造上会比较简化和不严谨，请勿当真。如有概念理解性错误，请多指正！*

## 对称加密
**假设**：双方提前通过可靠方式共享了一个 secret key。

Secret key 同时用来加密和解密。

```
┌─────┐ ciphertext=encrypt(secret_key,message)┌─────┐
│Alice├──────────────────────────────────────>│ Bob │
└─────┘                                       └─────┘
                                message=decrypt(secret_key,message)
```

这个相信大家都比较熟了，它最简单，性能也好。但是问题在于如何（在两个人之间）安全地共享 secret key，不能泄露。


## 非对称加密/公钥加密
假设：发送方提前通过可靠方式获得接收方的 public key。

用 public key 加密，用 private key 解密。

```
┌─────┐ ciphertext=encrypt(public_key,message)┌─────┐
│Alice├──────────────────────────────────────>│ Bob │
└─────┘                                       └─────┘
                                message=decrypt(private_key,message)
```

这个大家应该也比较熟了，它的性能比对称加密差很多。它的问题同样是如何安全共享密钥：如何知道 public key 真的属于接收方？攻击者可以拿自己的 public key 给 Alice，谎称自己是 Bob，同时向 Bob 发送伪造的信息，这就是中间人攻击（man-in-the-middle attack）。

但现在我们并不是要保密，我们就是要在整个网上分发自己的 public key，我们需要的是认证（authentication），因此需要可信的第三方来背书，需要建立起一个 Public Key Infrastructure（PKI）。数字证书干的就是这个。

如果我们单纯依赖公钥加密来通信，风险是一旦泄露 private key，过去的所有消息都可以被解密，称为缺乏前向安全（forward security）。

## DH Key Exchange

假设：无！

通信双方各自生成自己的秘密 a 和 b。各自用公开参数 g 和秘密进行计算，得到公开参数 A 和 B 发送给对方，然后双方都能算出同一个共享的秘密。

```
┌─────┐               A=g^a, g                ┌─────┐
│Alice│<─────────────────────────────────────>│ Bob │
└─────┘               B=g^b                   └─────┘
shared_secret=B^a                        shared_secret=A^b
             =g^(a+b)                                 =g^(a+b)
```
（这里的运算实际上全部在 mod p 下进行）

它的妙处是无需可信第三方，就能共同**商讨**出 shared secret，用来进行对称加密。商讨的过程是明文在网络中传输，但是窃听者无法从中算出 shared secret。我们可以每次会话临时协商一个 key，从而获得前向安全。

具体怎么用 shared secret 加密就是另外的话题，最简单的是 ElGamal，直接把 message 和 secret 乘起来。另外 Bob 在发送 B 的时候可以顺便同时发送使用 shared secret 加密后的消息。

TLS 也有这个密钥交换的环节，但它其实并不直接用协商的 shared secret 加密，而是把它当作一个 pre-master key，用来再生成一个 master key。理由是根据不同的算法和参数交换得到的 shared secret 的长度是变化的，它希望有一个固定长的密钥用来加密后续的通信消息，它能支持不同的加密算法。

另外也可以理解成 a, A 分别是 Alice 的 private key 和 public key，任何人拿到这个 public key 就可以和 Alice 建立加密通信，Alice 收到的消息却又必须要拥有它的 private key 才能解密。虽然它并不同于公钥加密的方案，加密通信并非直接用 public key 进行，而是另算出了一个 shared key 进行对称加密。

## (Threshold) Secret Sharing

这里的场景是我想要把一个消息发给一群人，要求他们聚在一起才能知道这个消息是什么。最简单的方法当然是直接拆分消息。但是我们想要安全地 share，要有这样的**性质**：部分消息的**信息量**和没有消息一样。也就是无法窥一斑而知全豹。

要求所有人在场过于严格，一般主要考虑有**足够多**的人在场的情形。其中一种特殊情况就是假设一共 n 个人，**任意** t 个人在场就能解密，这个 t 可被称为 threshold，这种情况叫做 (t,n)-threshold secret sharing。
```
                  ┌──────┐
              ┌──>│share1├──┐
              │   └──────┘  │reconstruct┌──────┐
              │             ├──────────>│secret│
┌──────┐split │   ┌──────┐  │           └──────┘
│secret├──────┼──>│share2├──┘
└──────┘      │   └──────┘
              │
              │   ┌──────┐
              └──>│share3│
                  └──────┘
```

最经典的方案就是 Shamir's secret sharing：从一个 t-1 次多项式上取 n 个点！后面的事情大家可以自行脑补（

### Homomorphic Secret Sharing

同态（Homomorphism）是从 A 空间到 B 空间的映射 f，使得在 A 空间的操作 $$*_A$$ 可以对应到 B 空间的操作 $$*_B$$，即 $$f(x *_A y) = f(x)*_B f(y)$$。加密可以看作明文和密文之间的映射，同态加密就是指在密文上操作使得在原文上生效。

Shamir's secret sharing 就有 homomorphic 的性质：对 share 做加法相当于对原文做加法。根据多项式的性质不难想象。

```
                   ┌────────┐
              ┌────┤share1-1│ + ┌──────────┐
              │    ├────────┼──>│sum-share1│
              │ ┌─>│share2-1│   └────┬─────┘     ┌───────┐
┌───────┐split│ │  └────────┘        │reconstruct│secret1│
│secret1├─────┤ │                    ├──────────>│   +   │
└───────┘     │ │  ┌────────┐        │           │secret2│
              ├─┼─>│share1-2│ + ┌────┴─────┐     └───────┘
              │ │  ├────────┼──>│sum-share2│
              │ ├─>│share2-2│   └──────────┘
┌───────┐split│ │  └────────┘
│secret2├─────┼─┤
└───────┘     │ │  ┌────────┐
              └─┼─>│share1-3│ + ┌──────────┐
                │  ├────────┼──>│sum-share3│
                └─>│share2-3│   └──────────┘
                   └────────┘
```

同态加密可以用来做 Secure multi-party computation：多方联合计算出一个结果，但是每个人对输入数据仍然一无所知。

例如上面这个方法可以用来做匿名投票，secret 是每个人的投票（1或0表示同意或否），最终只能恢复出总的票数，不能恢复每个人投的票。当然这个方法比较粗糙，因为根本没有校验输入的合法性。可以投 10 票或者 -10 票影响结果。


## Distributed Key Generation & Threshold Encryption

如果我们有一对公私钥，然后通过 secret sharing 来分发 private key（同时销毁掉完整的 private key），那么岂不是就可以达成这样的效果：用这个公钥加密的消息必须要集体中的人合力才能解密。

这个方案要求有一个可信的分发者。更好的做法当然是每个人自选 private key，然后通过一个协商的过程确定下来（确定出了 public key，但是 private key 仍然是秘密）。

说到用每个人自定的秘密生成一个公共秘密，不禁想到上面的 DH key exchange。假设每个人有自己的 $a_i$，每个人可算出 $A_i=g^{a_i}$ 进行共享，得出集体的 public key 就是 $A=\prod_{i=1}^nA_i$，集体的 private key 是 $a=\sum_{i=1}^na_i$ 仍然未知，需要集体合作才能获得。

当某人需要发送消息给集体时，随机生成一个 private key b，用 $A^b$ 对消息进行加密，同时发送密文和 $A^b$ 即可。

现在我们可以再结合一下 threshold secret sharing 了，每个人把 $a_i$ 再进行 (t,n)-secret sharing，这样任意 t 个人合力就可以恢复出 group 的 private key。这就叫 threshold encryption。


```
     x
    xx
   xxx
  xx xxxxxxxxxxxxxxxxxxxx
 xx                     x
xx   To Be Continued... x
 xx                     x
  xx xxxxxxxxxxxxxxxxxxxx
   xxx
    xx
     x
```