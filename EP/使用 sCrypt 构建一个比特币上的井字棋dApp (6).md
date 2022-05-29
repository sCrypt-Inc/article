
# 使用 sCrypt 构建井字棋游戏 - 第六部分

# 摘要

* 钱包介绍
  * [sensilet](https://sensilet.com) 
  * [dotwallet](https://www.dotwallet.com/)
  * [volt](https://volt.id)

* 集成钱包
  * 钱包接口
  * 钱包初始化
  * 钱包登入
  * 钱包余额

##  钱包介绍

BSV 区块链有很多支持进行合约开发。包括sensilet、dotwallet、volt。
这节课我们以sensilet为例，演示如何集成钱包。

> 操作打开钱包首页

## 钱包接口

我们在 wallet.ts 中定义了一些通用的钱包接口。并使用 sensilet 来实现这些接口。具体实现见: sensiletwallet.ts

## 钱包初始化

在 useEffect 中初始化钱包。首先，为 web3 设置一个 SensiletWallet 对象。然后调用 web3.wallet.isConnected() 将钱包是否连接的状态保存起来。

在 App 的渲染代码中，通过判断 states.isConnected 状态来决定渲染钱包登入组件 Auth 还是钱包余额组件 Balance。

```
return (
  <div className="App">
    <header className="App-header">
      <h2>Play Tic-Tac-Toe on Bitcoin</h2>
      ...
      {states.isConnected ? <Balance></Balance> : <Auth></Auth>}
    </header>
  </div>
);
```

## 钱包登入

下面是实现钱包登入的组件 Auth。用户点击 Sensilet 按钮则调用钱包的 requestAccount 接口来登入钱包。钱包插件会出现授权提示框。


```
import { web3 } from "./web3";

const Auth = (props) => {

  const sensiletLogin = async (e) => {
    try {
      const res = await web3.wallet.requestAccount("tic-tac-toe");
      if (res) {
        window.location.reload();
      }
    } catch (error) {
      console.error("requestAccount error", error);
    }
  };

  return (
    <div className="auth">
      <div>
        <button
          className="pure-button button-large sensilet"
          onClick={sensiletLogin}
        >
          Sensilet
        </button>
      </div>
    </div>
  );
};
```

## 钱包余额

Balance 组件调用了钱包的 getbalance 接口，实现了展示钱包余额的功能。


```
import { useState, useEffect } from "react";
import { web3 } from "./web3";
const Balance = (props) => {
  const [balance, setBalance] = useState(0);

  useEffect(async () => {
    if (web3.wallet) {
      web3.wallet.getbalance().then((balance) => {
        setBalance(balance);
      });
    }
  }, []);

    return (
      <div className="wallet">
        <div className="walletInfo">
          <div className="balance">
            <label>Balance: {balance} <span> (satoshis)</span></label>
          </div>
        </div>
      </div>
    );
};

export default Balance;
```

## 动手环节

1. 在 App 组件中初始化钱包。
2. 在 Auth 组件使用 requestAccount 接口来登入钱包
3. 在 Balance 组件使用 getbalance 接口获取钱包余额并展示出来。


# 总结

这节课我们一起学习了如何集成钱包。

