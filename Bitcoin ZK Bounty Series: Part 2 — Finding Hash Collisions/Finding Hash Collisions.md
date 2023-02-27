# 比特币 ZK 赏金系列：第 2 部分——查找哈希冲突

在我们的[零知识赏金 (ZKB)](https://github.com/sCrypt-Inc/article/blob/c9fd0019ec05566afacd10f4ab2d7453ced7cd2c/Private%20Non-interactive%20Bounties%20for%20General%20Computation%20on%20Bitcoin/Private%20Non-interactive%20Bounties%20for%20General%20Computation%20on%20Bitcoin.md) 系列的第二部分中，我们将其应用于解决哈希冲突难题。在这样的谜题中，两个不同的输入散列到相同的输出。此类赏金可用于：

1. 充当煤矿中的金丝雀，给我们一个有价值的提醒。存在冲突是散列函数较弱的标志，因此我们可以尽早升级以减轻损失。

2. 资助研究以发现哈希函数中的漏洞，特别是对于 [MiMC](https://eprint.iacr.org/2016/492.pdf) 等新函数。

![](./1.webp)

<center><a href="https://shattered.io/">碰撞攻击</a></center>

## 历史

比特币开发者彼得托德于 2013 年[最初发布](https://bitcointalk.org/index.php?topic=293382.0)了用于发现各种哈希函数中的冲突的比特币赏金。SHA1 赏金是在 2017 年[收集](https://www.coindesk.com/markets/2017/02/25/who-broke-the-sha1-algorithm-and-what-does-it-mean-for-bitcoin/)的，在[谷歌破解它](https://security.googleblog.com/2017/02/announcing-first-sha1-collision.html)后不久。

<center>最初的哈希碰撞赏金</center>

这种原始赏金有两个缺点：

1. 一旦有人广播包含解决方案的收集交易，矿工就可以拦截它，提取解决方案，并将奖励重定向到他们自己。

2. 该解决方案是公开的，可以被恶意行为者利用。

ZKB 解决了这两个问题，因此只有发现碰撞的赏金收集者才能赎回它，并且只有赏金制定者才能了解解决方案。

## 实现

与[第 1 部分](https://github.com/sCrypt-Inc/article/blob/4fb520b5ccd56952db5d43fa90c2bfe9e2e4879d/Bitcoin%20ZK%20Bounty%20Series:%20Part%201%20%E2%80%94%20Pay%20for%20Decryption%20Key/Pay%20for%20Decryption%20Key.md#L1)一样，我们只需替换特定于应用程序的电路 C 即可验证两个原像（即散列函数的输入）不同但它们产生相同的散列。我们以 [Poseidon 哈希函数](https://eprint.iacr.org/2019/458.pdf)为例，一种新的 ZK 友好哈希。其他哈希函数可以使用类似方式。这两个原像作为私有输入传递进来，永远不会公开透露。

```js
template Main() {

    // Private inputs:
    signal input preimage0[16]; 
    signal input preimage1[16];
    signal input db[4];                      // Seller (Bob) private key.
    signal input Qs[2][4];                   // Shared (symmetric) key. Used to encrypt w.
    
    // "Public" inputs that are still passed as private to reduce verifier size on chain:
    signal input Qa[2][4];                   // Buyer (Alice) public key.
    signal input Qb[2][4];                   // Seller (Bob) public key.
    signal input nonce;                      // Needed to encrypt/decrypt xy.
    signal input ew[34];                     // Encrypted solution to puzzle.

    // Public inputs:
    signal input Hpub[2];            // Hash of inputs that are supposed to be public.
                                     // As we use SHA256 in this example, we need two field elements
                                     // to acommodate all possible hash values.

    
    //// Assert that public inputs hash to Hpub. ///////////////////////////////////

    ...

    //// Assert that preimages are a valid solution. //////////////////////////////////////////////
    // Check preimage0 and preimage1 are differend and that they produce the same hash.
    var diff = 0;
    for (var i = 0; i < 16; i++) {
        diff += preimage0[i] ^ preimage1[i];
    }
    assert(diff != 0);

    component h0 = Poseidon(16);
    component h1 = Poseidon(16);
    for (var i = 0; i < 16; i++) {
        h0.inputs[i] <== preimage0[i];
        h1.inputs[i] <== preimage1[i];
    }
    h0.out === h1.out;

    //// Assert that (db * Qa) = Qs ////////////////////////////////////////////////

    ...

    //// Assert that (db * G) = Qb /////////////////////////////////////////////////

    ...

    //// Assert that encrypting w with Qs produces ew. /////////////////////////////

    ...
}
```

[GitHub](https://github.com/sCrypt-Inc/zk-bounties) 上提供了完整的代码和测试，包括验证证明并支付赏金收集者的[智能合约](https://github.com/sCrypt-Inc/scrypt-hash-collision-bounty/blob/master/contracts/bounty.scrypt)。