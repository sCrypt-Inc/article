
# 使用 sCrypt 构建井字棋游戏 - 第一部分

# 摘要

* 课程概述
    * 试玩游戏
* 开发环境搭建
    * sCrypt IDE
* 编写合约
    * helloWorld 合约
    * 基本数据类型
    * 属性
    * static 属性
    * const 变量
    * 函数
    * require 语句

## 课程概述

通过此课程，你将学会如何使用 sCrypt 构建一个比特币上的井字棋dApp。
该应用程序非常简单。它所做的就是使用两个玩家(分别是 Alice 和 Bob)的比特币地址初始化合约，各自下注相同的金额锁定到合约中。只有赢得那个人可以取走合约里面的钱。如果最后没有人赢，则两个玩家各自可以取走一半的钱。

我们将逐步完成构建app，包括:

1. 编写合约
2. App前端集成合约

在开始这个课程之前，我们先试玩一下该游戏 [Go to Play](https://scrypt.io/tic-tac-toe)

> 操作玩游戏

## 开发环境搭建

在开始编写合约之前。我们需要先搭建开发环境。 sCrypt 的开发环境非常容易搭建。只需安装一下软件:

1. 安装 VS Code 
2. 安装 sCrypt IDE
3. 安装 nodejs
4. git

sCrypt IDE 是 sCrypt 官方开发工具, 包含用于编写、编译、调试、测试、部署 sCrypt 智能合约的 GUI 构建器和工具， 为 sCrypt 开发者提供强有力的支持。 可用于专业开发。也适用于sCrypt智能合约的学习与教学。IDE 可以在 VS Code 的插件视图中直接搜索 “sCrypt” 进行安装。


> 操作打开插件视图搜索 “sCrypt”

本电脑已经安装好。请同学们自行安装。

## HelloWorld 合约

一份sCrypt合约就是比特币dApp的基本模块。它是你所有dApp的起点。sCrypt 合约由关键字 contract 定义，类似面向对象语言 Java/C++ 里面的 class。下面是 sCrypt 语言的 HelloWorld 的合约:

```js
contract HelloWorld {
	public function unlock(bytes message) {
		require(message == "hello world, sCrypt 😊");
	}
}
```

接下来我们建立一个名为 TicTacToe 的空合约。

```js
contract TicTacToe {

}
```


## 基本数据类型

sCrypt 是强类型语言。TicTacToe 合约将会用到的基本数据类型包括：

1. bool: 布尔值，取值 true 或者 false

2. int: 带符号的整数

3. int[3]: 大小为3的整数数组

4. PubKey：公钥

5. Sig：签名

如果你想了解更多的基本类型，可以查看语言[参考文档](https://scryptdoc.readthedocs.io/zh_CN/latest/) 

## 属性

和一般的面向对象语言一样： 每个合约可以拥有若干个成员变量，在该合约的函数中可以通过 this 关键字访问。

```js
contract Test {
    int x;
    bool y;
    bytes z;

    public function unlock(int y) {
        require(this.x == y);
    }
}
```

井字棋游戏合约支持两个玩家，需要保存两个玩家的公钥地址。在合约运行结束后，如果游戏不是以平局结束，合约自动将锁定的比特币打给赢家。因此我们定义 两个属性 alice 和 bob，数据类型都是 PubKey。

```js
contract TicTacToe {
    PubKey alice;
    PubKey bob;
}
```

## Static 属性

带有 static 关键字修饰的属性是 static 属性，声明 static 属性时必须初始化。在该合约的函数中可以通过合约名加属性名字（中间加点）来访问。如果在定义它的同一合约中使用，则可以省略该前缀。


## Const 变量

使用 const 关键字的修饰局的变量一旦初始化就不能更改。

可以同时使用两者来声明属性。 这通常用于定义常量。

TicTacToe 合约包含以下 常量属性:

1. N: 类型为 int，值为 9。它表示存储棋盘状态的数组长度，井字棋游戏一共公有9个棋盘位置
2. EMPTY: 类型为 int，值为 0。它表示该棋盘位置还未落子
3. ALICE: 类型为 int，值为 1。它表示该棋盘位置被玩家 alice 落子
4. BOB: 类型为 int，值为 2。它表示该棋盘位置被玩家 bob 落子

```js
contract TicTacToe {
    PubKey alice;
    PubKey bob;
    
    static const int N = 9;
    static const int EMPTY = 0;
    static const int ALICE = 1;
    static const int BOB = 2;
}
```

## 函数

### 普通函数 （function）
函数使用 function 关键字声明。函数主要作用是封装合约内部逻辑及代码重用。定义函数时需要使用冒号 : 来说明返回值的类型。

### 公共函数 （public function）

公共函数使用 public 修饰符修饰函数，是外部调用合约的接口。公共函数的参数代表 UTXO 模型(在下一节课介绍)中的解锁参数。公共函数没有明确的返回类型声明和函数结尾的 return 语句，隐形返回 bool 类型，返回值为 true。

TicTacToe 合约中有 3 个函数：

- move() : 公共函数。TicTacToe 合约提供给外部调用的接口。 Alice 和 Bob 各自将 X 个比特币锁定在包含上述合约的一个 UTXO 中。 接下来，他们通过调用公共函数 move() 交替玩游戏。公共函数 move()有 4 个参数，分别表示：

    1. n : 在棋盘上哪个位置下棋
    2. sig : 玩家的签名
    3. amount : 减去交易手续费后的合约余额
    4. txPreimage: 将在第七章介绍

- won() : 回类型 bool，作用是检查是否有玩家已经赢得比赛，他将能取走所有合约锁定的赌注

- full() : 回类型 bool，作用是检查棋盘所有格子是否都有棋子了，如果没人赢得比赛，则两个人平分赌注

我们为合约添加这三个函数。

## require 语句

require 语句 包含 require 关键字和一个布尔表达式。该语句会检查布尔表达式是否为真。当不满足某些条件时抛出错误，并停止执行。这与 solidity 语言的 require 类似。sCrypt 公有函数的最后一个语句必须是 require 语句，合约的每个公有函数至少有一个 require 语句。当且仅当所有 require 语句 都检查通过，合约才能被成功解锁。即参数 n 是一个在棋盘中的有效值。

接下来为公共函数 move 添加 require 语句，要求函数参数 n 必须大于等于 0， 且小于合约的 static 属性 N。即参数 n 是一个在棋盘中的有效值。

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

这节课我们一起来学习 sCrypt 开发环境搭建，基础语言语法知识。编写了 TicTacToe 合约的基本框架。

