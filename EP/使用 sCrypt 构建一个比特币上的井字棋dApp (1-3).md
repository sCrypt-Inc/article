
# 使用 sCrypt 构建井字棋游戏 - 第一部分 第三小节

# 摘要

* 比特币脚本语言
    * UTXO 模型
    * 比特币脚本语言Script
    * Script与sCrypt
* 获取、检查 Sighash 原像
    * 什么是 Sighash 原像
    * 如何获取 Sighash 原像
    * 如何检查 Sighash 原像
* 维护游戏状态
    * 声明状态属性
    * 更新状态属性
    * 保存状态属性

## 比特币脚本语言

### UTXO 模型

比特币被“锁”在交易的输出中。要想在另一个交易中花费它，其输入中必须含有匹配的“钥匙”。只有钥匙能打开锁时，比特币才能被转移到新的输出中。这就是所谓的 UTXO（Unspent Transaction Outputs） 模型。如下图所示，两个交易各有一个输入和输出。右边的交易输入花费左边交易的输出。

### 比特币脚本语言Script

锁和钥匙都是用比特币脚本语言Script表示。它是比特币虚拟机的指令集，是一种类似于汇编的低级语言。锁和钥匙对应的比特币脚本分别被称为锁定脚本和解锁脚本。输出由两部分组成：锁定脚本和以 satoshis 表示的比特币数量。

### Script与sCrypt

使用汇编进行编程并不是现代软件开发的主流。sCrypt是一种高级语言，编译生成类似汇编的比特币脚本Script。两者的关系类似于Java和Java虚拟机的字节码。这样可以提高开发效率，并使编写的合约变得可测试，提升可靠性。具体来说，sCrypt里公共函数的参数对应解锁脚本，公共函数的函数体对应锁定脚本，如下图所示。


## 获取 Sighash 原像

### 1. 什么是 Sighash 原像
在为比特币中的消息生成签名时，首先对消息生成哈希摘要，然后对摘要进行签名。 一个交易用于生成哈希摘要的消息称为其 sighash 原像。它大致由包含当前交易和它花费的 UTXO 。 在下面的示例中，tx1 中第一个输入的 sighash 原像用红色圈出。 请注意，不同的输入具有不同的 sighash 原像，即使它们在同一笔交易中，因为它们使用不同的 UTXO。


其详细格式 如下:

```
1. nVersion of the transaction (4-byte little endian)
2. hashPrevouts (32-byte hash)
3. hashSequence (32-byte hash)
4. outpoint (32-byte hash + 4-byte little endian) 
5. scriptCode of the input (serialized as scripts inside CTxOuts)
6. value of the output spent by this input (8-byte little endian)
7. nSequence of the input (4-byte little endian)
8. hashOutputs (32-byte hash)
9. nLocktime of the transaction (4-byte little endian)
10. sighash type of the signature (4-byte little endian)
```

### 2. 如何获取 Sighash 原像

Sighash 原像通常是在链下计算的。官方的 [scryptlib](https://github.com/sCrypt-Inc/scryptlib) SDK 提供了一个计算 Sighash 原像的工具函数 `getPreimage`。请观看专门介绍如何使用这个SDK的课程。

```js
const tx = newTx(inputSatoshis);
let newLockingScript = instance.getNewStateScript({
    counter: 1
});

tx.addOutput(new bsv.Transaction.Output({
  script: newLockingScript,
  satoshis: outputAmount
}))

// 计算 Sighash 原像
const preimage = getPreimage(tx, instance.lockingScript, inputSatoshis)
```

### 3. 如何获取 Sighash 原像

在 sCrypt语言中， Sighash 原像对应的数据类型是 SigHashPreimage。 要检查 sighash 原像， 即检查 SigHashPreimage 类型的参数。只需调用标准库 Tx 合约的 checkPreimage() 函数。通常在公共函数的第一个语句执行检查。

```
contract TicTacToe {
    PubKey alice;
    PubKey bob;
    
    static const int N = 9;
    static const int EMPTY = 0;
    static const int ALICE = 1;
    static const int BOB = 2;

    
    public function move(int n, Sig sig, int amount, SigHashPreimage txPreimage) {
        require(Tx.checkPreimage(txPreimage));
        require(n >= 0 && n < N);
    }


    function won(int play) : bool {
        return true;
    }


    function full() : bool {
        return true;
    }
}
```

## 维护游戏状态

合约可以通过将其状态存储在锁定脚本中来实现链式保持状态。 在下面的例子中，合约从 state0 转换到 state1，然后到再转换到 state2。

下面以一个简单的计数器合约，展示如何通过简单的步骤在合约中维护状态。

### 第一步：声明状态属性

使用装饰器 @state 声明状态属性。

```
contract Counter {

    @state
    int counter;

    public function increment(SigHashPreimage txPreimage, int amount) {
        require(Tx.checkPreimage(txPreimage));
    }
}
```

### 第二步：更新状态属性

您可以将状态属性当作普通属性：读取和更新它。调用编译器为每个合约自动生成的内置函数 this.getStateScript()，可以将状态属性序列化到锁定脚本中。

contract Counter {

    @state
    int counter;

    public function increment(SigHashPreimage txPreimage, int amount) {
        require(Tx.checkPreimage(txPreimage));
        // update counter state
        this.counter++;
        // serialize the state of the contract
        bytes outputScript = this.getStateScript();
    }
}

### 第三步： 保存状态属性

链式保持状态的关键是确保当前交易的输出必须包含这个新状态。也就是包含第二步得到的锁定脚本。sighash 原像的第8个字段包含交易中所有输出的哈希值。如果这个哈希值与当前交易中所有输出的哈希值相同，我们可以确定当前交易的输出与我们在合约中构建的输出一致。 因此，包括更新的状态属性。

```js
contract Counter {

    @state
    int counter;

    public function increment(SigHashPreimage txPreimage, int amount) {
        require(Tx.checkPreimage(txPreimage));
        // increment counter
        this.counter++;
        // serialize the state of the contract
        bytes outputScript = this.getStateScript();
        // construct an output from its locking script and satoshi amount
        bytes output = Utils.buildOutput(outputScript, amount);
        // make sure the transaction contains the expected outputs
        require(hash256(output) == SigHash.hashOutputs(txPreimage));
    }
}
```


TicTacToe 合约的包含两个状态属性 board 和 isAliceTurn，合约在 move() 方法执行的时候更新并保存这两个状态。


```
contract TicTacToe {
    PubKey alice;
    PubKey bob;

    // if it is alice's turn to play
    @state
    bool isAliceTurn;

    // state of the board. For example, a chess with Alice in the first row 
    // and first column is expressed as [1,0,0,0,0,0,0,0,0]
    @state
    int[N] board;
    
    static const int N = 9;
    static const int EMPTY = 0;
    static const int ALICE = 1;
    static const int BOB = 2;

    public function move(int n, Sig sig, int amount, SigHashPreimage txPreimage) {
        require(Tx.checkPreimage(txPreimage));
        require(n >= 0 && n < N);

        // square not filled
        require(this.board[n] == EMPTY);

        int play = this.isAliceTurn ? ALICE : BOB;
        PubKey player = this.isAliceTurn ? this.alice : this.bob;

        // update state properties to make the move
        this.board[n] = play;
        this.isAliceTurn = !this.isAliceTurn;

        bytes outputs = b'';
        。。。
        // serialize the state of the contract
        bytes outputScript = this.getStateScript();
        bytes output = Utils.buildOutput(outputScript, amount);
        outputs = output;
        。。。
        // make sure the transaction contains the expected outputs
        require(hash256(outputs) == SigHash.hashOutputs(txPreimage));

    }
}
```

# 总结

这节课我们一起学习了原生比特币脚本语言，获取、检查 Sighash 原像，状态属性。实现了 TicTacToe 合约的 move 方法。

