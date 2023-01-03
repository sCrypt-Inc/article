
# scryptTS new version released


A new version of scryptTS is released, mainly bringing two new features. You need to use the following versions to experience: 

```json
"dependencies": {
    "scrypt-ts": "=0.1.5-beta.2"
},
```

## 1. The transaction preimage OP_PUSH_TX technology is hidden in scryptTS


Using [OP_PUSH_TX](https://medium.com/@xiaohuiliu/op-push-tx-3d3d279174c1) allows the contract code to access the entire transaction data, including all inputs and outputs.
This allows us to set any constraints on the data. This opens up endless possibilities for running various smart contracts on the BSV network.


OP_PUSH_TX requires the [transaction preimage](https://github.com/bitcoin-sv/bitcoin-sv/blob/master/doc/abc/replay-protected-sighash.md#digest-algorithm) to be computed externally, and the transaction preimage is passed into the contract through the public function parameters of the contract.

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

The calculation of the transaction preimage involves the construction of the transaction and the signature format used to calculate the preimage. It is easy for users to make mistakes in the calculation process.

**scryptTS** encapsulates the computation of the transaction preimage. Users do not need to explicitly calculate and pass the transaction preimage. The data of the entire transaction can be accessed through `ScriptContext`.

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

Correspondence between `ScriptContext` structure and transaction preimage `txPreimage`:

| ScriptContext  | transaction preimage `txPreimage`  |
| ------------- | ------------- | 
| nVersion | nVersion of the transaction  |
| utxo.value | value of the output spent by this input (8-byte little endian)  |
| utxo.scriptCode | scriptCode of the input (serialized as scripts inside CTxOuts) |
| utxo.outpoint.txid | prevTx id in 32-byte hash |
| utxo.outpoint.outputIndex | outputIndex in prevTx |
| hashPrevouts | `hashPrevouts` is the double SHA256 of the serialization of all input outpoints; |
| hashSequence | `hashSequence` is the double SHA256 of the serialization of nSequence of all inputs; |
| nSequence | nSequence of the input  |
| hashOutputs | `hashOutputs` is the double SHA256 of the serialization of all output amount (8-byte little endian) with scriptPubKey (serialized as scripts inside CTxOuts); |
| nLocktime| nLocktime of the transaction |
| sigHashType| sighash type of the signature |




You can directly access the relevant data of the transaction preimage through `this.ctx` in the public functions of the contract (access in non-public functions is not supported).


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

There is no need to pass in the transaction preimage `txPreimage` when calling the contract:


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


## 2. Support for displaying transpiler errors in vscode editor

Just put this file [task.json](https://github.com/sCrypt-Inc/scryptTS-examples/blob/master/.vscode/tasks.json) in the `.vscode` directory of your current project. Then run `Shift+Command+B` (Windows: `Ctrl+Shift+B`, Linux: `Ctrl+Shift+B`), you can see the error output:

![](./1.jpg)









