---
title: RSA乘法同态性
date: 2019-6-7 12:10:34
categories: 技术储备
tags: [RSA, 乘法同态性]
---

----

<!-- more -->

# 1. 什么是乘法同态性

已知条件:
a * b = c(基础等式)
加密函数e() 解密函数d() (enctrypt:加密; decrypt:解密)

推导结论:
e(a) * e(b) = e(c) (乘法同态特性)
c = d(e(c)) = d(e(a) * e(b))

限制条件:
上述e(a), e(b) 必须是同一公钥加密.

# 2. RSA乘法同态性Java验证

```Java
import java.math.BigInteger;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.interfaces.RSAPrivateKey;
import java.security.interfaces.RSAPublicKey;
import java.util.Random;

public class Main {
    public static void main(String[] args) throws Exception{
        // 1. 生成RSA密钥对
        // 获取指定算法的密钥对生成器
        KeyPairGenerator gen = KeyPairGenerator.getInstance("RSA");
        // 初始化密钥对生成器（指定密钥长度, 使用默认的安全随机数源）
        gen.initialize(2048);
        // 随机生成一对密钥（包含公钥和私钥）
        KeyPair keyPair = gen.generateKeyPair();
        // 获取 公钥 和 私钥
        RSAPublicKey pubKey = (RSAPublicKey)keyPair.getPublic();
        RSAPrivateKey priKey = (RSAPrivateKey)keyPair.getPrivate();
        // 获得公私钥 和 公共模数
        BigInteger E = pubKey.getPublicExponent();
        BigInteger D = priKey.getPrivateExponent();
        BigInteger N = pubKey.getModulus();

        BigInteger a = BigInteger.valueOf(new Random().nextInt(Integer.MAX_VALUE));
        BigInteger b = BigInteger.valueOf(new Random().nextInt(Integer.MAX_VALUE));

        // 加密
        BigInteger Ca = a.modPow(E, N);
        BigInteger Cb = b.modPow(E, N);
        // 密文相乘
        BigInteger Cc = Ca.multiply(Cb).mod(N);
        // 解密
        BigInteger Mc = Cc.modPow(D, N);
        // 验证
        BigInteger val = a.multiply(b);
        System.out.println(Mc.compareTo(val));
    }
}
```

## 2.1 证明

```text
若: A * B = C
则: (A^d ∗ B^d) mod n = C^d mod n

左边 = ( ( A^d mod n ) * ( B^d mod n ) ) ( mod n ) = ( A的密文 * B的密文 ) ( mod n )
右边 = C的密文

因此,将密文A与密文B的乘积 对 模数N 取余 后的结果就是 C的密文.
对其解密后得到C, C就是明文A和明文B的乘积.

特殊说明:
如果A * B 大于等于 公共模数N, 即 C 大于等于 公共模数N, 则该方法存在问题.
```

## 2.2 注意事项

RSA乘法同态性只对正整数有效.

#  3. 参考链接

[RSA乘法同态加密的python实现](https://blog.csdn.net/u014418725/article/details/82883220)
[深入理解RSA算法](https://www.jianshu.com/p/2af68dabd093)