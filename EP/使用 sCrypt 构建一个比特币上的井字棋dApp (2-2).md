
# 使用 sCrypt 构建井字棋游戏 - 第二部分 第二小节

# 摘要

* 数据类型
* 加载合约描述文件
* 实例化合约
* 动手环节


## 数据类型

sCrypt 语言的所有基本类型在 scryptlib 中都有对应的 javascript 类。在使用 scryptlib 实例化合约和调用合约的公共方法时，需要使用相应的 javascript 类来传递数据。这样，可以在运行之前检查参数的类型并检测潜在的错误。

1. `int` 类型 对应 `Int` 类
2. `bool` 类型对应 `Bool` 类
3. `PubKey` 类型对应 `PubKey` 类

所有sCrypt语言的基础数据类型都可以在 scryptlib 中找到对应的类型。

## 加载合约描述文件

 [web3](https://github.com/sCrypt-Inc/tic-tac-toe/blob/7ae1eb8cb46bd8315d9c7d858b6a190ba3c4c306/src/web3/web3.ts) 工具类提供了一组进行网络交互的工具函数，以及对钱包接口的封装。你可以直接使用 `web3.loadContract()` 从网络中加载合约描述文件。

 ```js
async function fetchContract(alicePubKey, bobPubKey) {
  let { contractClass: TictactoeContractClass } = await web3.loadContract(
    "/tic-tac-toe/tictactoe_release_desc.json"
  );
}
 ```

## 实例化合约

我们已经通过加载合约描述文件得到了合约类 `TictactoeContractClass`， 接下来通过此合约类来实例化合约。

```js
async function fetchContract(alicePubKey, bobPubKey) {
  let { contractClass: TictactoeContractClass } = await web3.loadContract(
    "/tic-tac-toe/tictactoe_release_desc.json"
  );

  return new TictactoeContractClass(
    new PubKey(alicePubKey),
    new PubKey(bobPubKey),
    true,
    [0,0,0,0,0,0,0,0,0]  // empty board
  );

}
```


## 动手环节

添加 `fetchContract` 函数，并使用 `TictactoeContractClass` 合约类实例化合约，并返回实例化合约对象

参考这个 [commit](https://github.com/sCrypt-Inc/tic-tac-toe/commit/4c91169b41332a26ed7d504846b78e66e423546d)


# 总结

这节课我们学习了在前端页面加载合约描述文件，使用合约类实例化合约。

