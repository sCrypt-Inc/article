
# 使用 sCrypt 构建井字棋游戏 - 第一部分 第二小节

# 摘要

* 编写合约
    * 函数
    * require 语句

## 函数

函数包含普通函数和公共函数。

### 普通函数 （function）

普通函数使用 `function` 关键字声明。函数主要作用是封装合约内部逻辑及代码重用。定义函数时需要使用冒号 `:` 来说明返回值的类型。

下面的 `Test` 合约包含一个 `clone` 普通函数：

```js
contract Test {

    function clone(int x) : int {
        return x;
    }
}
```

### 公共函数 （public function）

公共函数使用 `public` 修饰符修饰函数，是外部调用合约的接口。公共函数的参数代表 UTXO 模型中的解锁参数，(UTXO 模型在下一节课介绍)。公共函数没有明确的返回类型声明和函数结尾的 `return` 语句。隐形返回 bool 类型。返回值为 true。

下面的 `Test` 合约包含一个 `main` 公共函数:

```js
contract Test {

    function clone(int x) : int {
        return x;
    }

    public function main(int x)  {
        require(true);
    }
}
```

## require 语句

require 语句 包含 require 关键字和一个布尔表达式。该语句会检查布尔表达式是否为真。当不满足某些条件时抛出错误，并停止执行。这与 solidity 语言的 `require` 函数类似。sCrypt 公共函数的最后一个语句必须是 `require` 语句。合约的每个公共函数至少有一个 `require` 语句。当且仅当所有 `require` 语句都检查通过时，合约才能被成功解锁。

```js
contract Test {

    function clone(int x) : int {
        return x;
    }

    public function main(int x)  {
        require(this.clone(x) == 1);
    }
}
```


TicTacToe 合约中有 `3` 个函数：

- move() : 公共函数。TicTacToe 合约提供给外部调用的接口。 Alice 和 Bob 各自将 X 个比特币锁定在包含上述合约的一个 UTXO 中。 接下来，他们通过调用公共函数 move() 交替玩游戏。公共函数 move() 有 `4` 个参数，分别表示：

    1. n : 在棋盘上哪个位置下棋
    2. sig : 玩家的签名
    3. amount : 减去交易手续费后的合约余额
    4. txPreimage: 将在后续小节介绍

- won() : 返回类型 bool，作用是检查是否有玩家已经赢得比赛。他将能取走所有合约锁定的BSV

- full() : 返回类型 bool，作用是检查棋盘所有格子是否都有棋子。如果没人赢得比赛，则两个人平分合约锁定的BSV

我们为合约添加这三个函数。同时为公共函数 move 添加 `require` 语句，要求函数参数 `n` 必须大于等于 `0`， 且小于合约的 `static` 属性 `N`。即参数 `n` 是一个在棋盘中的有效值。

```js
contract TicTacToe {
    PubKey alice;
    PubKey bob;
    
    static const int N = 9;
    static const int EMPTY = 0;
    static const int ALICE = 1;
    static const int BOB = 2;

    public function move(int n, Sig sig, int amount, SigHashPreimage txPreimage) {

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

# 总结

这节课我们学习了 sCrypt 语言中的函数是怎么样定义的，包括普通函数和公共函数。并为 TicTacToe 合约的添加了 3 个函数。

