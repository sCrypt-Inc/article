
# 使用 sCrypt 构建井字棋游戏 - 第二部分 第一小节

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

完成上一节课以后，我们的井字棋 dApp 合约就开发完成了。接下来我们需要编译合约。

sCrpt IDE 提供一个右键编译合约的功能。我们使用它来编译刚刚编写的 TicTacToe 合约。

> 操作编译合约

### 合约描述文件

编译合约会输出一个对应的合约描述文件 （Contract Description File) tictactoe_release_desc.json。合约描述文件是一个命名为 xxx_desc.json 的 JSON 文件。可用于在链下构建锁定脚本和解锁脚本，并实例化合约。

以下是合约描述文件的结构：

```json
{
    "version": 8,
    "compilerVersion": "1.14.0+commit.9fdbe60",
    "contract": "TicTacToe",
    "md5": "fb6b0618f95002b289dda96a20be139e",
    "structs": [],
    "library": [],
    "alias": [
        {
            "name": "PubKeyHash",
            "type": "Ripemd160"
        }
    ],
    "abi": [
        {
            "type": "function",
            "name": "move",
            "index": 0,
            "params": [
                {
                    "name": "n",
                    "type": "int"
                },
                {
                    "name": "sig",
                    "type": "Sig"
                },
                {
                    "name": "amount",
                    "type": "int"
                },
                {
                    "name": "txPreimage",
                    "type": "SigHashPreimage"
                }
            ]
        },
        {
            "type": "constructor",
            "params": [
                {
                    "name": "alice",
                    "type": "PubKey"
                },
                {
                    "name": "bob",
                    "type": "PubKey"
                },
                {
                    "name": "isAliceTurn",
                    "type": "bool"
                },
                {
                    "name": "board",
                    "type": "int[9]"
                }
            ]
        }
    ],
    "stateProps": [
        {
            "name": "isAliceTurn",
            "type": "bool"
        },
        {
            "name": "board",
            "type": "int[9]"
        }
    ],
    "buildType": "release",
    "file": "",
    "asm": "OP_1 40 76 88 a9 ac 00 OP_1 OP_2 $__codePart__ $alice $bob $is_alice_turn $board ...",
    "hex": "5101400176018801a901ac01005152<alice><bob>615b79610 ...",
    "sources": [
    ],
    "sourceMap": [ 
    ]
}
```

> 打开合约描述文件

## 集成 scryptlib

### 准备

要注意的是将使用 JavaScript 来编写这个 APP 的界面。React App 项目 [tic-tac-toe](https://github.com/sCrypt-Inc/tic-tac-toe) 的 `webapp` 分支，包含一个只有前端代码的井字棋游戏。 请克隆此项目并切换到 `webapp` 分支。我们假设你已经具备前端开发的基础知识，因此我们不会花时间来介绍这部分代码。

```
git clone -b webapp https://github.com/sCrypt-Inc/tic-tac-toe
```

> 操作克隆代码

### scryptlib

dApp 需要在前端页面与合约进行交互。 要做到这一点，我们将使用 sCrypt 官方发布的 JavaScript 库 —— scryptlib.

scryptlib 是用于集成 sCrypt 智能合约的 Javascript/TypeScript SDK。

通过 scryptlib，你就能方便地编译，测试，部署，调用合约了。

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

这里我们为你提供了 [web3](https://github.com/sCrypt-Inc/tic-tac-toe/blob/7ae1eb8cb46bd8315d9c7d858b6a190ba3c4c306/src/web3/web3.ts) 工具类。该工具类提供了一组进行网络交互的工具函数，以及对钱包接口的封装。你可以直接使用 `web3.loadContract()` 从网络中加载合约描述文件。

## 动手环节

接下来同学跟着我一起来完成动手环节。每个小节的动手环节下面都有个 commit。代表该小节需要修改的内容。本节课的动手环节包含两个部分。

* 编译合约
* 加载合约描述文件

参考这个 [commit](https://github.com/sCrypt-Inc/tic-tac-toe/commit/5cf4afb31925d141c201d28355ac7ab7597eb1d7)

# 总结

这节课我们一起学习了编译合约、合约描述文件和安装集成 scryptlib。

