# BSV上的委托

> Alice 允许 Bob 花费她的BSV

我们为 Alice 引入了一种新颖的方式，允许 Bob 使用她的 UTXO/硬币，而无需向 Bob 提供她的私钥。Alice 签Bob 的公钥，从而将支出的权力委托给 Bob。委托有很多用例。例如，用户可以授权应用代表她花费BSV。或者，首席财务官希望部门负责人发放拨给该部门的资金。

![delegation](./delegation.png)

## 委托

以前，我们开发了一种方法，让 Alice 使用她的比特币私钥在交易中[签署任意消息](https://xiaohuiliu.medium.com/ecdsa-based-oracles-on-bitcoin-e69d15afe6c5)。对于委托工作，消息只是 Bob 的公钥。通过签署 Bob 的密钥，Alice 将使用她的硬币的权力委托给 Bob<sup>1</sup>。

当 Alice 想要委托时，她首先将资金锁定到以下合约中。

```js
import "./oracle.scrypt";

contract Delegate {
    PubKey owner;

    public function delegate(Sig ownerSig, PubKey delegate, Sig delegateSig, 
            PubKey derivedPubKey, PubKey X, int lambda, SigHashPreimage txPreimage) {
        
        // ensure delegate's public key is signed by the owner
        require(Oracle.verifyData(delegate, ownerSig, this.owner, derivedPubKey, X, lambda, txPreimage));

        // delegate signs
        require(checkSig(delegateSig, delegate));
    }
}
```

<center><a href="https://github.com/sCrypt-Inc/boilerplate/tree/master/contracts/delegate.scrypt">委托合约</a></center>

她使用自己的私钥签署 Bob 的公钥，即第 `6` 行的 `ownerSig`。她将部分签名的交易交给 Bob。Bob 将他的签名（第 `6` 行的 `delegateSig`）添加到交易中并广播它以花费锁定的资金。第 `10` 行验证 Bob 的公钥已签名<sup>2</sup>，并因此得到 Alice 的授权。第 `13` 行检查 Bob 的签名。

请注意，委托不会创建任何链上交易，这只发生在 Bob 实际花费 UTXO 时。

如果 Alice 想自己花费 UTXO，她可以简单地通过签署她自己的公钥来委托给自己。因此，她可以在 Bob 移动硬币之前撤销委托。

## 委托与多重签名

与 Alice 和 Bob 之间传统的 2-of-2 多重签名相比，上述合约可以委托给任何人（例如，委托给 Charlie 或 Dave），即使在创建 UTXO 之后也是如此。受托人是事先未确定的，以后可以随意选择，而多重签名必须事先知道所有受托人，以后不能更改。委派更加灵活。

## 多次委托

在 Alice 委托给 Bob 之后，Bob 委托给 Charlie，Charlie 再次委托给 Dave，以此类推。

以下修改后的合约允许多次委托：

```js
contract Delegate {
    PubKey owner;

    static const int N = 3;

    public function delegateN(Sig[N] ownerSig, PubKey[N] delegate, Sig delegateSig, 
            PubKey[N] derivedPubKey, PubKey[N] X, int[N] lambda, SigHashPreimage txPreimage) {
        
        PubKey lastDelegate = this.owner;
        loop (N) : i {
            PubKey owner = i == 0 ? this.owner : delegate[i - 1];
            // check if the next delegate is delegated by the current delegate
            if (Oracle.verifyData(delegate[i], ownerSig[i], owner, derivedPubKey[i], X[i], lambda[i], txPreimage))
                lastDelegate = delegate[i];
        }
        
        // last delegate signs
        require(checkSig(delegateSig, lastDelegate));
    }
}
```

基本上，Alice 签署 Bob 的公钥，Bob 签署 Charlie 的公钥，……第 `13` 行验证签名，从而像以前一样验证委托。只有最后一位代表在 `18` 行签署才能花费硬币。

---------------

[1] 也可以使用替代签名算法，例如 [Rabin 签名](https://blog.csdn.net/freedomhero/article/details/107237537)。

[2] 注意 `Oracle.verifyData()` 在内部使用 `SIGHASH_NONE`，因此 Bob 可以添加他想要的任何输出。