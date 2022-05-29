
# 使用 sCrypt 构建井字棋游戏 - 第四部分

# 摘要

* 编译合约
    * 使用IDE 编译合约
    * 合约描述文件
* 集成 scryptlib
    * 准备
    * scryptlib 介绍
    * scryptlib 安装
    * web3 工具类 介绍
* 动手环节
    * 编译合约
    * 加载合约描述文件

## 编译合约

### 编译

完成上一节课以后，我们的井字棋 dApp 的 sCrypt 合约部分就完成了。接下来我们需要编译合约。

sCrpt IDE 提供一个右键编译合约的功能。我们使用它来编译刚刚编写的 TicTacToe 合约。

> 操作编译合约

### 合约描述文件

编译合约会输出一个对应的合约描述文件 （Contract Description File) tictactoe_release_desc.json。合约描述文件是一个命名为 xxx_desc.json 的 JSON 文件。可用于在链下构建锁定脚本和解锁脚本并实例化合约。

以下是合约描述文件的结构：

> 打开合约描述文件

## 集成 scryptlib

### 准备

要注意的是这个 APP 界面将使用 JavaScript 来写，并不是 sCrypt。React App 项目 tic-tac-toe 的 webapp 分支包含一个只有前端代码的井字棋游戏。 请克隆此项目并切换到 webapp 分支。我们假设你已经具备前端开发的基础知识，因此我们不会花时间来介绍这部分代码。

```
git clone -b webapp https://github.com/sCrypt-Inc/tic-tac-toe
```

> 操作克隆代码

### scryptlib

dApp 需要在前端页面与合约进行交互。 要做到这一点，我们将使用 sCrypt 官方发布的 JavaScript 库 —— scryptlib.

scryptlib 用于集成以 sCrypt 语言编写的 Bitcoin SV 智能合约的 Javascript/TypeScript SDK。

通过 scryptlib ，你就能方便地编译，测试，部署，调用合约了。

下面是一段使用 scryptlib 实例化合约和执行合约公共方法的代码:

```
const Demo = buildContractClass(loadDesc('demo_desc.json'));
const demo = new Demo(7, 4);

const result = demo.add(11).verify()
assert(result.success);
```

### scryptlib 安装

scryptlib 可以通过 npm 安装。

```
// use NPM
npm install scryptlib

// use Yarn
yarn add scryptlib
```

> 操作安装 scryptlib

### web3 工具类

这里我们为你提供了 web3 工具类。该工具类提供了进行合约与网络交互的工具函数以及对钱包接口的封装。你可以直接使用 web3.loadContract() 从网络中加载合约描述文件。

## 动手环节

接下来同学跟着我一起来完成动手环节。每节课动手环节下面都有个 commit。代表这节课动手环节要完成的内容。本节课的动手环节包含两个部分。

* 编译合约
* 加载合约描述文件

# 总结

这节课我们一起学习了编译合约、合约描述文件和安装集成 scryptlib。

