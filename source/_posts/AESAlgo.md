---
title: AES 算法
date: 2018-04-24 22:05:37
tags: 算法
---

## 名称
**Advanced Encryption Standard** 即 高级加密标准，又叫做 **Rijndael算法**，是一种对称加密算法，它已取代了原有的 **DES** ，在全世界被广泛使用。

## 算法流程
算法的输入是明文字符串，它会被转换成多个 4 * 4 的字节矩阵，每个矩阵被称为 **体(state)**。每一个体都会进行一遍 AES 算法流程，最终输出加密后的字符串。
AES 算法流程：

    1. AddRoundKey(轮秘钥加): 距震中的每一个字节都与该次会和密钥做XOR(异或)
    2. SubBytes(字节替换): 通过一个非线性的替换函数，用查找表的方式把每个字节替换成对应的字节
    3. ShiftRows(行移位): 将距震中的每个横列进行循环式移位
    4. MixColumns(列混淆): 为了充分混合矩阵中各个直行的操作。这个步骤使用线性转换来混合每内联的四个字节。最后一个加密循环中省略 MixColumns 步骤，用另外一个 AddRoundKey 代替

### AddRoundKey(轮秘钥加)
在每一次的加密循环中，都会由主密钥生成一个回合秘钥(Rijndael 秘钥生成方案)。该秘钥的大小和一个体的大小相同，都是一个 4 * 4 的矩阵。将回合秘钥与体的对应字节做异或操作。
{% asset_img AES-AddRoundKey.png AddRoundKey %}

### SubBytes(字节替换)
经过了 AddRoundKey 步骤的矩阵，在这一步中，每个字节都会通过 **S-box(S盒)** 转换成另一个字节。S盒具有良好的非线性特性，用于提供加密算法的混淆性。
{% asset_img S盒.png S-box %}

{% asset_img S盒逆.png Inverse S-box %}


S盒 和 S盒逆 都是16 * 16 的矩阵，它们记录了一个字节到另一个自己的转换和逆转换。

### ShiftRows(行移位)
在这个步骤中，4 * 4 矩阵中的每一行都向左循环位移某个偏移量。在 AES 中， 加密时，第一行维持不变，第二行向左循环移动一个字节，第三行两个字节，第四行三个字节。经过 ShiftRows 之后， 矩阵中的每一个列，都由输入的不同列组成。这个步骤提供了 AES 的扩散性。
{% asset_img ShiftRows.svg Shfit Rows %}


### MixColumns(列混淆)
列混淆是利用了 GF(2^8) 域上算术特性的一个代替，同样用于提供 AES 的扩散性。
{% asset_img MixColumns.png Mix Columns %}


## 安全性
针对 AES 的攻击，对于 128 位的秘钥来说，穷举法需要 2的128次方 的复杂度， 目前尚未出现有效的攻击方式。