
# 使用 sCrypt 构建井字棋游戏 - 第七部分

# 摘要 

* 部署合约
  * web3.deploy() 函数
  * 调用 web3.deploy()函数部署合约
* 部署井字棋合约

##  web3.deploy() 函数

钱包初始化并登入后，我们就可以使用钱包来发送交易部署合约了。

web3 工具类同样提供了一个用来部署合约的工具函数 web3.deploy()。合约实例和合约锁定的比特币数量是该函数的参数。该函数将完成以下工作：

1. 通过钱包的 listUnspent 接口查询可用的未花费输出。用来支付部署合约的交易手续费和锁定的余额。
2. 构建包含合约实例的锁定脚本，作为交易的第一个输出。
3. 调用钱包的 getSignature 接口，请求钱包签名交易。
4. 调用钱包的 sendRawTransaction 接口，广播交易。

> 操作打开web3 工具类

## 调用 web3.deploy()函数部署合约

用户在游戏界面填写好合约锁定的比特币数量，并点击开始按钮之后，会回调 App 组件中的 startGame 方法。该函数实现了将合约实例部署到比特币网络上的功能。部署成功后，将包含合约的UTXO到和游戏初始状态保存到 localStorage ，并更新 React 状态。同时，我们可以看到部署合约过程中产生一个交易。该交易由 deploy函数 构建并广播。

```
const startGame = async (amount) => {
    if (web3.wallet && states.instance) {
      web3.deploy(states.instance, amount).then(rawTx => {
            
          ...  
      })
    }
};
```

> 操作开始游戏


## 动手环节

1. 在 startGame 函数中调用 web3.deploy() 函数来部署合约。 并在部署成功后保存交易到 ContractUtxos 和更新游戏状态。

# 总结

这节课我们一起学习了如何部署合约。

