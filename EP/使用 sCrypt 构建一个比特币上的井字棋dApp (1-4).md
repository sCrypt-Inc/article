
# 使用 sCrypt 构建井字棋游戏 - 第一部分 第四小节

# 摘要

* 维护游戏状态
    * 声明状态属性
    * 更新状态属性
    * 保存状态属性

合约可以通过将其状态存储在锁定脚本中来实现链式保持状态。 在下面的例子中，合约从 state0 转换到 state1，然后到再转换到 state2。

![](https://github.com/sCrypt-Inc/image-hosting/blob/master/learn-scrypt-courses/06.png?raw=true)

下面以一个简单的计数器合约，展示如何通过简单的步骤在合约中维护状态。

## 第一步：声明状态属性

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

## 第二步：更新状态属性

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

## 第三步： 保存状态属性

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

这节课我们学习了状态属性以及如何维护状态属性。

