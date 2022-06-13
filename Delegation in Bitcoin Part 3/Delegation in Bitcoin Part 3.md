# BSV上的委托: 第三部分

> 委托给脚本

之前，我们介绍了两种委托使用比特币的方法：一种在[脚本级别](https://medium.com/coinmonks/delegation-in-bitcoin-ac7afcab991e)，另一种在[交易级别](https://medium.com/coinmonks/delegation-in-bitcoin-part-2-4cd8a7c29c99)。

我们将前者概括为委托给任何脚本/智能合约，而不仅仅是公钥。它允许一个人授权任意智能合约来花费一个人的比特币。

![委托](./delegation.jpeg)

完整代码如下所示：

```js
contract DelegateToScript {
    PubKey owner;

    public function delegate(Sig ownerSig, bytes delegatedScript, 
            PubKey derivedPubKey, PubKey X, int lambda, SigHashPreimage txPreimage) {
        require(Tx.checkPreimage(txPreimage));
        
        // ensure delegated script is signed by the owner
        require(Oracle.verifyData(delegatedScript, ownerSig, this.owner, derivedPubKey, X, lambda, txPreimage));

        // use delegated script as the new locking script, while maintaining value
        bytes output = Utils.buildOutput(delegatedScript, SigHash.value(txPreimage));
        require(hash256(output) == SigHash.hashOutputs(txPreimage));
    }
}
```

<center><a href="https://github.com/sCrypt-Inc/boilerplate/tree/master/contracts/delegateToScript.scrypt">DelegateToScript 合约源代码</a></center>

第 `9` 行检查委托脚本是否已签名并因此由所有者[授权](https://xiaohuiliu.medium.com/ecdsa-based-oracles-on-bitcoin-e69d15afe6c5)。第 `12` 行和第 `13` 行使用 OP_PUSH_TX 确保委托脚本作为新锁定脚本进入支出交易的输出，类似于 [Pay to Script Hash (P2SH)](https://blog.csdn.net/freedomhero/article/details/112344420) 的模拟。