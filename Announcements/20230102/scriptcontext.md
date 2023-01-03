# scryptTS 中隐藏了交易原像 OP_PUSH_TX 技术

使用 [OP_PUSH_TX](https://blog.csdn.net/freedomhero/article/details/107306604) 可以让合约代码访问整个 transaction 数据，包括所有的 inputs 和 outputs。
从而使得我们能够针对这些数据设置任何约束条件。这为在BSV 网络上运行各种智能合约开辟了无限可能。

OP_PUSH_TX 要求在外部计算[交易原像](https://github.com/bitcoin-sv/bitcoin-sv/blob/master/doc/abc/replay-protected-sighash.md#digest-algorithm)，并通过合约的公共函数参数传入。

```ts
export class Counter extends SmartContract {
  @prop(true)
  count: bigint;

  constructor(count: bigint) {
    super(count);
    this.count = count;
  }

  @method()
  public increment(txPreimage: SigHashPreimage) {
    this.count++;
    assert(this.updateState(txPreimage, SigHash.value(txPreimage)));
  }
}
```

交易原像的计算涉及交易的构建，计算原像使用的签名格式。用户在计算的过程中容易出错。


**scryptTS** 封装了交易原像的计算。用户无需显式的计算和传入交易原像。可以通过 `ScriptContext` 来访问整个交易的数据。 


```ts
export type ScriptContext = {
    nVersion: ByteString;
    utxo: UTXO;
    hashPrevouts: ByteString;
    hashSequence: ByteString;
    nSequence: bigint;
    hashOutputs: ByteString;
    nLocktime: bigint;
    sigHashType: SigHashType;
};
```

`ScriptContext` 结构体与交易原像 `txPreimage` 的对应关系：

| ScriptContext  | 交易原像 `txPreimage`  |
| ------------- | ------------- | 
| nVersion | 交易版本号  |
| utxo.value | 此输入花费的输出值 |
| utxo.scriptCode | 输入的 scriptCode（在 CTxOuts 中序列化为脚本） |
| utxo.outpoint.txid | 输入所在交易的交易 Id |
| utxo.outpoint.outputIndex | 输入所在交易的输出索引 |
| hashPrevouts | `hashPrevouts` 是所有输入outpoints序列化的双SHA256 |
| hashSequence | `hashSequence` 是所有输入的nSequence序列化的双SHA256 |
| nSequence | 输入的 nSequence |
| hashOutputs | `hashOutputs` 是用scriptPubKey序列化所有输出量（8字节小端）的双SHA256（在CTxOuts中序列化为脚本） |
| nLocktime| 交易的nLocktime |
| sigHashType| 签名的 sighash 类型 |




你可以直接在 合约的公共函数（不支持在非公共函数中访问）中通过 `this.ctx` 来访问交易原像的相关数据。

```ts
export class Counter extends SmartContract {
  @prop(true)
  count: bigint;

  constructor(count: bigint) {
    super(count);
    this.count = count;
  }

  @method()
  public increment() {
    this.count++;
    assert(this.ctx.hashOutputs == hash256(this.buildStateOutput(this.ctx.utxo.value)));
  }
}
```

调用合约时也无需传入 交易原像 `txPreimage`:

```ts
getCallTx(utxos: UTXO[], prevTx: bsv.Transaction, nextInst: Counter): bsv.Transaction {
const inputIndex = 1;
return new bsv.Transaction().from(utxos)
    .addInputFromPrevTx(prevTx)
    .setOutput(0, (tx: bsv.Transaction) => {
    nextInst.lockTo = { tx, outputIndex: 0 };
    return new bsv.Transaction.Output({
        script: nextInst.lockingScript,
        satoshis: this.balance,
    })
    })
    .setInputScript(inputIndex, (tx: bsv.Transaction) => {
    this.unlockFrom = { tx, inputIndex };
    return this.getUnlockingScript(self => {
        self.increment();
    })
    });
}
```


欢迎尝试使用没有交易原像的 OP_PUSH_TX 技术来编写合约。需要使用最新的 **scryptTS** 版本：

```json
"dependencies": {
    "scrypt-ts": "=0.1.5-beta.2"
},
```





