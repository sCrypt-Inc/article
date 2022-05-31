
# 使用 sCrypt 构建井字棋游戏 - 第一部分 第五小节

# 摘要

* 签名验证
    * 支付到公钥哈希
    * 检查玩家签名
* 数组
* 循环

## 签名验证

### 支付到公钥哈希
比特币网络中最常见的合约是支付到公钥哈希。下面是支付到公钥哈希合约的sCrypt实现。
其中用到了内置函数:

1. hash160: 返回公钥的 PublicKeyHash
2. checkSig: 检查签名是否与公钥匹配，成功则返回 true。失败则整个合约立即失败。

```
contract P2PKH {
    Ripemd160 pubKeyHash;

    public function unlock(Sig sig, PubKey pubKey) {
        require(hash160(pubKey) == this.pubKeyHash);
        require(checkSig(sig, pubKey));
    }
}
```

### 检查玩家签名
move() 函数的 sig 参数是玩家的签名。TicTacToe 合约只有签名是从解锁参数传入，玩家的公钥已经存储在合约的PubKey alice 和 PubKey bob 两个属性中。假如没有对签名进行验证，任何人都可以调用合约的 move() 方法来移动棋子。

```
contract TicTacToe {
    PubKey alice;
    PubKey bob;

    ...
    
    public function move(int n, Sig sig, int amount, SigHashPreimage txPreimage) {
        require(Tx.checkPreimage(txPreimage));
        require(n >= 0 && n < N);

        // square not filled
        require(this.board[n] == EMPTY);

        int play = this.isAliceTurn ? ALICE : BOB;
        PubKey player = this.isAliceTurn ? this.alice : this.bob;

        require(checkSig(sig, player));
        ...
    }
    ...
}
```

# 数组

数组是一组类型相同的值列表。数组元素使用逗号分割。数组大小必须是正整数。数组的声明时必须指定数组大小：

```js
int[3] a = [0, 1, 2];
bool[3] b = [false, false && true, (1 > 2)];
int[2][3] arr2D = [[11, 12, 13], [21, 22, 23]];
int d = a[2];
int idx = 2;
// read
d = a[idx];
d = arr2D[idx][1];
// write
a[idx] = 2;
// assign to an array variable
a = arr2D[1];
```

对于井字棋游戏，是否有玩家赢得比赛的规则是有三个棋子连成一条直线，我们把所有可能的连成一条线的情况列举出来：

```
0, 1, 2
3, 4, 5
6, 7, 8
0, 3, 6
1, 4, 7
2, 5, 8
0, 4, 8
2, 4, 6
```

在 won() 函数中，用一个二维数组 int[8][3] lines 保存以上所有赢得比赛的状态。

```js
function won(int play) : bool {
    // three in a row, a column, or a diagonal
    int[8][3] lines = [[0, 1, 2], [3, 4, 5], [6, 7, 8], [0, 3, 6], [1, 4, 7], [2, 5, 8], [0, 4, 8], [2, 4, 6]];
    ...
}
```

## 循环

sCrypt 使用 loop 关键字定义循环。语法如下：

```js
loop (maxLoopCount) [: i] {
    loopBody
}
```

maxLoopCount 必须是编译时常量。i 是一个归纳变量，表示第几次循环。例如，下面的循环：

```js
contract Loop {
    
    static const int N = 10;
    
    public function unlock(int x) {
    
        loop (N) : i {
            x = x * 2 + i;
        }
        require(x > 100);
    }
}
```
等价于以下展开形式：

```
x = x * 2 + 0;
x = x * 2 + 1;
x = x * 2 + 2;
x = x * 2 + 3;
x = x * 2 + 4;
x = x * 2 + 5;
x = x * 2 + 6;
x = x * 2 + 7;
x = x * 2 + 8;
x = x * 2 + 9;
```

我们在 full 函数中使用 loop 语法来遍历棋盘所有格子，检查是否所有格子都有棋子。

```js
function full() : bool {
    bool full = true;

    loop (N) : i {
        full = full && this.board[i] != TicTacToe.EMPTY;
    }
    
    return full;
}
```

loop 支持嵌套， 我们在won 函数中使用 嵌套 loop 来检查是否有玩家赢得比赛。

```js
function won(int play) : bool {
    // three in a row, a column, or a diagonal
    int[8][3] lines = [[0, 1, 2], [3, 4, 5], [6, 7, 8], [0, 3, 6], [1, 4, 7], [2, 5, 8], [0, 4, 8], [2, 4, 6]];

    bool anyLine = false;
    loop (8) : i {
        bool line = true;
        loop (3) : j {
            line = line && this.board[lines[i][j]] == play;
        }

        anyLine = anyLine || line;
    }

    return anyLine;
}
```


# 总结

这节课我们一起学习了签名验证、数组和loop 语法。实现了 TicTacToe 合约的 won 方法 和 full 方法。恭喜你，你已经对 sCrypt 语言的有了初步的了解，并编写了一个 TicTacToe 合约。如果你想进一步了解 sCrypt 语言，请前往我们的 sCrypt语言参考文档。如果你想完成井字棋 dApp 的开发，请继续学习下一节课《了解如何与前端集成》。

