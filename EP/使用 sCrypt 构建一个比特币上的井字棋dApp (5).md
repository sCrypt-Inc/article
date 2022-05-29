
# 使用 sCrypt 构建井字棋游戏 - 第五部分

# 摘要

* 数据类型
* 实例化合约
* 动手环节
    * 实例化合约

## 数据类型

sCrypt 语言的所有基本类型在 scryptlib 中都有对应的 javascript 类。在使用 scryptlib 实例化合约和调用合约的公共方法时，需要使用相应的 javascript 类来传递数据。这样，可以在运行之前检查参数的类型并检测潜在的错误。

1. int 类型 对应 Int 类
2. bool 类型对应 Bool 类
3. PubKey 类型对应 PubKey 类

基础数据类型是 scryptlib 默认就支持的类型。对于用户自己定义的结构体或者别名，需要通过 buildTypeClasses 动态生成。


### 合约描述文件

编译合约会输出一个对应的合约描述文件 （Contract Description File) tictactoe_release_desc.json。合约描述文件是一个命名为 xxx_desc.json 的 JSON 文件。可用于在链下构建锁定脚本和解锁脚本并实例化合约。

以下是合约描述文件的结构：

> 打开合约描述文件

## 实例化合约

我们已经通过加载合约描述文件得到了合约类 TictactoeContractClass， 接下来通过此合约类来实例化合约。

```
const instance = new TictactoeContractClass(
    new PubKey(alicePubKey),
    new PubKey(bobPubKey),
    true,
    [0,0,0,0,0,0,0,0,0]  // empty board
  );
```


## 动手环节

在 fetchContract 中使用 TictactoeContractClass 合约类实例化合约，并返回实例化合约对象

# 总结

这节课我们一起学习了使用合约类实例化合约。

