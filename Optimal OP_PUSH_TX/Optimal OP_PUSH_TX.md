# 优化 OP_PUSH_TX

自从我们实现 [OP_PUSH_TX](https://blog.csdn.net/freedomhero/article/details/107306604) 以来，已经使用这个强大的功能构建了大量的智能合约。随着这些合约开始部署在比特币网络上，OP_PUSH_TX 必须是轻量级的，以最小化交易成本。我们已经介绍了一种使用[OP_CODESEPARATOR](https://blog.csdn.net/freedomhero/article/details/122497817) 优化OP_PUSH_TX的方法。今天，我们将介绍另外一种优化 OP_PUSH_TX的方法，最高可以优化 `700%`。


## 使用优化版本的 `Tx.checkPreimage`
只需将 `Tx.checkPreimage(SigHashPreimage txPreimage)` 替换为其优化版本 `Tx.checkPreimageOpt(SigHashPreimage txPreimage)`。

```javascript
/**
 * test contract to compare OP_PUSH_TX with and without optimization
 */
 contract OptimalPushTx {
     public function validate(SigHashPreimage txPreimage) {
        // compare the output script size of the following two to see effect of optimization

        // 633 bytes
        // require(Tx.checkPreimage(txPreimage));

        // 92 bytes
        require(Tx.checkPreimageOpt(txPreimage));
    }
}
```

在优化之前，上面的[合约](https://github.com/sCrypt-Inc/boilerplate/blob/master/contracts/optimalPushtx.scrypt)编译成一个 `633` 字节的脚本。之后，它只编译为 `92` 字节，减少了 7 倍。

## SigHash 约束

由于 [Low-S 约束](https://bitcoin.stackexchange.com/questions/85946/low-s-value-in-bitcoin-signature)，在使用 `Tx.checkPreimageOpt()` 时，`txPreimage`的哈希值（又称为 `sighash`）的最高有效字节（MSB）必须小于阈值 `0x7E`。如果交易不满足此约束，则可以通过修改锻造交易轻松解决此问题。每次锻造的成功几率约为 `50%`。只需要花一些时间就可以找到有效的交易。

有很多方法可以在不使交易无效的情况下使交易生效，例如：

- 在锁定脚本中添加一些虚拟操作，例如 `OP_0 OP_DROP`
- 改变它的 *nLockTime* 如下所示

```javascript
it('should return true', () => {
    for (i = 0; ; i++) {
        // malleate tx and thus sighash to satisfy constraint
        tx_.nLockTime = i
        const preimage_ = getPreimage(tx_, test.lockingScript, inputSatoshis)
        preimage = toHex(preimage_)
        const h = Hash.sha256sha256(Buffer.from(preimage, 'hex'))
        const msb = h.readUInt8()
        if (msb < MSB_THRESHOLD) {
            // the resulting MSB of sighash must be less than the threshold
            break
        }
    }
    result = test.validate(new SigHashPreimage(preimage)).verify()
    expect(result.success, result.error).to.be.true
});
```
<center> <a href="https://github.com/sCrypt-Inc/boilerplate/blob/master/tests/js/optimalPushtx.scrypttest.js">修改 nLockTime 来锻造有效交易</a></center>

