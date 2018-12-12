---
title: 可验证随机函数VRF
date: 2018/11/21 22:22:22
tags:
  - blockchain
  - crypto
categories: Blockchain
---

# 哈希函数：

  Result = HASH(Info)

# 带密钥的哈希函数：

  Result = HASH(PrivateKey, Info)

带密钥的哈希函数，通过私钥和源信息产生散列值。所以在仅仅已知源信息的情况下无法得到与带密钥哈希函数所产生的相同的散列值。

但是有些情况下，需要验证某个未知*私钥* 的加密哈希函数的*输出*（Result）是否匹配于*源信息*（Info），此时就需要可验证的加密函数（VRF）来解决。

在VRF情形中，产生散列值的函数记为VRF_HASH，产生证明的函数记为Proof_HASH。验证过程中，由Proof计算出Hash的函数记为VRF_P2H，用来验证Info，PublicKey和Proof的函数记为VRF_Verify。

生成过程如下：

1. Result = VRF_HASH(PrivateKey, Info)
2. Proof = VRF_Proof(PrivateKey, Info)
3. 将Result，Proof，Info，PublicKey向外公布

验证过程如下

1. 计算VRF_P2H（Proof）并与Result比较是否一致
2. 计算VRF_Verify(PublicKey, Info, Proof)是否为True

通过计算Proof是否是通过Info生成的，Proof计算出Result，Info和Result是否匹配，Proof作为中间桥梁。