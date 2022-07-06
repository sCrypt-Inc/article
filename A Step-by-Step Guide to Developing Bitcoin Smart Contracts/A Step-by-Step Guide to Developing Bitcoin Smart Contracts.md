# A Step-by-Step Guide to Developing Bitcoin Smart Contracts

We show the complete workflow of the design, development, testing, deployment, and invocation process of smart contracts in sCrypt.

# Design

The first step in building any smart contract is a high-level design. Here we choose to implement a common transaction type Pay To Public Key Hash (P2PKH) in the Bitcoin network. There are two main reasons to use this type of transaction as an example:

* P2PKH is the most popular type of transactions in the bitcoin network used to sending bitcoins from one to another, which is necessary for beginners to understand

* By implementing this classic transaction type, one can more intuitively understand the capabilities and usage of Script/sCrypt

## What’s P2PKH?

It’s locking script is:

```
OP_DUP OP_HASH160 <Public Key Hash> OP_EQUALVERIFY OP_CHECKSIG
```

It’s unlocking script is:

```
<Signature> <Public Key>
```

## Receiving of P2PKH

If Alice wants to transfer bitcoins to Bob, Bob first needs to tell Alice his public key hash value ( usually known as the bitcoin address, which is similar to a bank account number). Alice uses this value to construct a P2PKH locking script (here denoted as LS¹) and sends the transaction to the miner. After the miner verifies that it is correct, the transaction is recorded on chain.

## Spending of P2PKH

When Bob wants to spend the bitcoins received, he needs to provide two pieces of information to construct a unlocking script:

* The original public key that the above public key hash value is calculated from;

* A Signature calculated by using the private key corresponding to the original public key.

After constructing the unlocking script and placing it in a transaction, Bob broadcasts the transaction.

## Verification of P2PKH

When miners receive the new transaction, they need to validate it, which involves two main steps:

* Concatenate the unlocking script with the locking script to form the following verification Script:

```
<Signature> <Public Key> OP_DUP OP_HASH160 <Public Key Hash> OP_EQUALVERIFY OP_CHECKSIG
```

* Use the Bitcoin Virtual Machine to execute this verification Script to check whether the execution result is valid. There are two critical checks in the verification process:

1. Verify that the hash, which can be calculate by the public key information provided in the unlocking script, is equal to the hash provided in the locking script. If it passes, the public key is indeed the recipient address of the previous transaction. (It is equivalent to verifying that the receiving address of the previous transfer is Bob’s bank account number)

2. Verify that the signature provided in the unlocking script matches the public key information. If it passes, it means that Alice does have control over the private key corresponding to this public key.

If the verification passes, it proves that Alice does own and can control the bitcoins, and the miner will record the new spending transaction on chain. This is the main process and principle of P2PKH type transactions.

In summary, our goal for contract design is also very clear: to implement a sCrypt contract that is equivalent to the function of P2PKH.

## Development

With this design goal in mind, we can get started. Firstly, we install [the sCrypt extension in VS Code](https://marketplace.visualstudio.com/items?itemName=bsv-scrypt.sCrypt).


```
git clone https://github.com/sCrypt-Inc/boilerplate.git
```

Actually, the P2PKH contract we wanted is included in the project. So let us look directly at the code at `contracts/p2pkh.scrypt`.

```js
contract DemoP2PKH {
    PubKeyHash pubKeyHash;

    constructor(PubKeyHash pubKeyHash) {
        this.pubKeyHash = pubKeyHash;
    }

    public function unlock(Sig sig, PubKey pubKey) {
        require(hash160(pubKey) == this.pubKeyHash);
        require(checkSig(sig, pubKey));
    }
}
```

<center><a href="https://github.com/sCrypt-Inc/boilerplate/blob/master/contracts/p2pkh.scrypt">P2PKH Contract</a></center>

The main body of the contract includes:

1. A contract variable pubKeyHash of type `PubKeyHash`, corresponding to the previous P2PKH locking script `<Public Key Hash>` ;


Constructor `constructor`, used to initialize the contract variable;

A custom public function named unlock which has two parameter with type Sig and PubKey, corresponding to the previous P2PKH unlocking script `<Signature> <Public Key>`.The implementation logic also corresponds to the P2PKH validation described earlier.

It is obvious that the sCrypt implementation is much easier to learn and write than previous scripts. And the more complex the contractual logic, the more obvious the advantage of using sCrypt is.

## Unit testing

With the code, the next step is to verify that the implementation is correct, and the normal way to do this is to add some unit tests. The test file for the above contract is `tests/js/p2pkh.scrypttest.js`, the code is as follows:

```js
const { expect } = require('chai');
const { bsv, buildContractClass, PubKeyHash, Sig, PubKey, signTx, toHex } = require('scryptlib');

/**
 * an example test for contract containing signature verification
 */
const { compileContract, inputIndex, inputSatoshis, newTx } = require('../../helper');

const privateKey = new bsv.PrivateKey.fromRandom('testnet')
const publicKey = privateKey.publicKey
const pkh = bsv.crypto.Hash.sha256ripemd160(publicKey.toBuffer())
const privateKey2 = new bsv.PrivateKey.fromRandom('testnet')
const tx = newTx();

describe('Test sCrypt contract DemoP2PKH In Javascript', () => {
  let demo, sig, context

  before(() => {
    const DemoP2PKH = buildContractClass(compileContract('p2pkh.scrypt'))
    demo = new DemoP2PKH(new PubKeyHash(toHex(pkh)))
    // any contract that includes checkSig() must be verified in a given context
    context = { tx, inputIndex, inputSatoshis }
  });

  it('signature check should succeed when right private key signs', () => {
    sig = signTx(tx, privateKey, demo.lockingScript, inputSatoshis)
    result = demo.unlock(sig, new PubKey(toHex(publicKey))).verify(context)
    expect(result.success, result.error).to.be.true

  });

  it('signature check should fail when wrong private key signs', () => {
    sig = signTx(tx, privateKey2, demo.lockingScript, inputSatoshis)
    result = demo.unlock(sig, new PubKey(toHex(publicKey))).verify(context)
    expect(result.success, result.error).to.be.false
  });
});
```

<center><a href="https://github.com/sCrypt-Inc/boilerplate/blob/master/tests/js/p2pkh.scrypttest.js">p2pkh.scrypttest.js</a></center>


Anyone familiar with JavaScript might immediately recognize that this is a pure JS test file based on the **MOCHA + Chai** framework. Let’s take a closer look at this test case.

* First Import sCrypt’s Javascript/Typescript library [scryptlib](https://github.com/sCrypt-Inc/scryptlib):

```js
const { buildContractClass, bsv } = require('scryptlib');
```

* Use the function `buildContractClass` to get the class object reflected into Javascript of the contract DemoP2PKH:

```js
const DemoP2PKH = buildContractClass(compileContract('p2pkh.scrypt'))
```

* Use arguments (that is, the hex format of the public key hash) to instantiate the contract:

```js
demo = new DemoP2PKH(new PubKeyHash(toHex(pkh)))
```

* Test the public method of the contract instance and expect it to succeed:

```js
sig = signTx(tx, privateKey, demo.lockingScript, inputSatoshis)
result = demo.unlock(sig, new PubKey(toHex(publicKey))).verify(context)
expect(result.success, result.error).to.be.true
```

* Expect it to fail if the signature cannot be verified because the wrong private key is used:

```js
sig = signTx(tx, privateKey2, demo.lockingScript, inputSatoshis)
result = demo.unlock(sig, new PubKey(toHex(publicKey))).verify(context)
expect(result.success, result.error).to.be.false
```

Before running the test, we need to run npm install in the root directory of the project to ensure that the dependencies have been successfully installed. Then right click the test file in VS Code editor and select **Run sCrypt Test** menu. The result is shown in the **Output** view.

## Debug

Having only the above unit test is not enough, because when the test fails, we can hardly figure out why and there is no more information to help us pinpoint the problem. Enter the sCrypt Debugger.

A sample debug configuration for the DemoP2PKH contract can be found in the file `.vscode/launch.json` :

```json
{
    "type": "scrypt",
    "request": "launch",
    "internalConsoleOptions": "openOnSessionStart",
    "name": "Debug DemoP2PKH",
    "program": "${workspaceFolder}/contracts/p2pkh.scrypt",
    "constructorArgs": [
        "Ripemd160(b'25f129c0199618bfd69efda98e4fc451ef8fc69f')"
    ],
    "pubFunc": "unlock",
    "pubFuncArgs": [
        "Sig(b'3044022016567cdde8cf4ae00ae5851267243dd40da5f04d92cdb530a06dcac4b545cbe802200e8d34c96e2aeae0d93bd28f89b4f15eab5276d52ac2932b2f75c1bb43a425bb41')",
        "PubKey(b'03c06625af044217934879f8f2eea11e8d0360fc15589f7f89f109829376617e52')"
    ],
    "txContext": {
        "hex": "01000000010aba16ff60c6500cc9c2235fe824f1bb8d5c67bb6be985e99bc7ba27fb618b6a0000000000ffffffff0000000000",
        "inputIndex": 0,
        "inputSatoshis": 100000
    }
}
```

<center><a href="https://github.com/sCrypt-Inc/boilerplate/blob/master/.vscode/launch.json#L162">launch.json</a></center>

The key parameters are:

- `program`: the contract file for this debug configuration.

- `constructorArgs`: the contract’s constructor argument list, separated by comma.

- `pubFunc`: specifies the public function name to be debugged.

- `pubFuncArgs`: specify the actual argument list of the public function to be debugged, also separated by comma.

- `txContext`: specifies context information about the current transaction at debugging time, where:

    1. `txContext.hex`: the hex format of the transaction, can be signed or unsigned.
    2. `txContext.inputIndex`: the input index corresponding to the contract-locked UTXO to be spent.
    3. `txContext.inputSatoshis`: The amount of bitcoins corresponding to the contract-locked UTXO to be spent, in unit of satoshi.

So how does one obtain these arguments in general?  You can call `genLaunchConfig` to get a `launch.json` file: 

```js
const file = demo.unlock(new Sig(toHex(sig)), new PubKey(toHex(publicKey))).genLaunchConfig()
console.log(file)
```

This file is exactly what you need for the debug configuration, and you can get it for other contracts in a similar way.

With the configuration in place, you can use the “F5” shortcut to start code debugging. See the [sCrypt IDE documentation](https://scrypt-ide.readthedocs.io/en/latest/debugger.html#id6) for details of what the debugger does and how to use it.

## Deploy and Call the Contract

Before using a contract in a production environment, a developer should test on testnet to ensure that the contract code meets expectations. For the example in this article, you can run the command `node testnet/p2pkh.js` in the project root directory.

### Preparations

When we first run the file, we see output like this:

```
Missing private key, generating a new one ...
Private key generated: 'cW1pmzsvWiRmL6VFK2ZxNaZdRXPjj8ZNwDjmrRDs7e7q3o6g2Ebq'
You can fund its address 'mrRw3MxwAXQaeeKrNW8hn68soDhjMw2fL8' from sCrypt faucet https://scrypt.io/#faucet
```

Because the code requires two things to work:

A private key on the test network is required;
The private key has sufficient balance in the corresponding address for testing.

If you already have such a private key, you can find and modify the following line of code (using a private key in WIF format instead of a empty character) in [privateKey.js](https://github.com/sCrypt-Inc/boilerplate/blob/master/privateKey.js):

```js
const privKey = ''
```

Of course, you can also use the private key in the output above, but first you need to get the test coin for the address in the output, such as on [our faucet](https://scrypt.io/#faucet).

## Expected Result

With the aforementioned setup in place, you are ready to run the command again. Normally you would see the following output:

```
locking txid:      7a5ece9f371fa64480b1a10a3c3168179cf0a5f297ef6f61bdfeb5f86220aaeb
Contract Method Called Successfully! TxId:  62ee0a42f93562dd9676f6c7f9f9d25aad157f8ca591f9b7b46894c2dae4ae40
```

The above result shows that the contract deployment and call have been successful, and you can go to a [blockchain browser](https://test.whatsonchain.com/) to see the corresponding transaction details (using the txid in the output results) .

The full code is available at `testnet/p2pkh.js`.

```js
const { buildContractClass, toHex, signTx, Ripemd160, Sig, PubKey, bsv } = require('scryptlib');

const {
  deployContract,
  createInputFromPrevTx,
  sendTx,
  showError,
  loadDesc
} = require('../helper')

const { privateKey } = require('../privateKey');

async function main() {
  try {
    const publicKey = privateKey.publicKey

    // Initialize contract
    const P2PKH = buildContractClass(loadDesc('p2pkh_debug_desc.json'))
    const publicKeyHash = bsv.crypto.Hash.sha256ripemd160(publicKey.toBuffer())
    const p2pkh = new P2PKH(new Ripemd160(toHex(publicKeyHash)))

    const amount = 10000
    // deploy contract on testnet
    const lockingTx = await deployContract(p2pkh, amount);
    console.log('locking txid:     ', lockingTx.id)


    // call contract method on testnet
    const unlockingTx = new bsv.Transaction();

    unlockingTx.addInput(createInputFromPrevTx(lockingTx))
      .change(privateKey.toAddress())
      .setInputScript(0, (tx, output) => {
        const sig = signTx(unlockingTx, privateKey, output.script, output.satoshis)
        return p2pkh.unlock(sig, new PubKey(toHex(publicKey))).toScript()
      })
      .seal()


    const unlockingTxid = await sendTx(unlockingTx)
    console.log('Contract Method Called Successfully! TxId: ', unlockingTxid)

  } catch (error) {
    console.log('Failed on testnet')
    showError(error)
  }
}

main()
```

<center><a href="https://github.com/sCrypt-Inc/boilerplate/blob/master/testnet/p2pkh.js">p2pkh.js</a></center>


Let’s take a look at the specific implementation of contract deployment and invocation.

### Contract Deployment

1. Initialize contract:

```js
// Initialize contract
const P2PKH = buildContractClass(loadDesc('p2pkh'))
const publicKeyHash = bsv.crypto.Hash.sha256ripemd160(publicKey.toBuffer())
const p2pkh = new P2PKH(new Ripemd160(toHex(publicKeyHash)))
```

2. Call the `deployContract` function to deploy the contract instance to the testnet and specify the balance locked by the contract

```js
const amount = 10000
// deploy contract on testnet
const lockingTx = await deployContract(p2pkh, amount);
console.log('locking txid:     ', lockingTx.id)
```

### Contract Call

1. Create a new unlocking transaction from the transaction that deploys the contract: 

```js
const unlockingTx = new bsv.Transaction();
unlockingTx.addInput(createInputFromPrevTx(lockingTx))
```

2. Add a change output:

```js
.change(privateKey.toAddress())
```

3. Get the unlocking script corresponding to the contract method call in `setInputScript` and seal the transaction:

```js
.setInputScript(0, (tx, output) => {
    const sig = signTx(unlockingTx, privateKey, output.script, output.satoshis)
    return p2pkh.unlock(sig, new PubKey(toHex(publicKey))).toScript()
})
.seal()
```

4. Send transaction to the network:

```js
const unlockingTxid = await sendTx(unlockingTx)
```

## Conclusion

We walk through an example of developing a smart contract on Bitcoin, which serves a starting point for developing more.