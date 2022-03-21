# 比特币与以太坊智能合约 第二部分

*以太坊从来没有被需要过*

以太坊之所以存在的全部原因是为了克服比特币脚本语言的“局限性”。
以太坊白皮书中列出了[四个限制](https://ethereum.org/en/whitepaper/#scripting)。我们分析了它们中的每一个，并表明所有假设的限制都是错误的，对其存在提出质疑。

![比特币 vs 以太坊](bitcoin-vs-eth.jpg)

## 非图灵完备 (Lack of Turing-completeness)

> 虽然比特币脚本语言支持大量计算子集，但它几乎不支持大部分计算。缺少的主要类别是循环。

与普遍的看法相反，比特币是图灵完备的。不仅有同行评审的理论 <sup>1</sup> 证明 <sup>2</sup>，而且还在比特币区块链上进行了验证<sup>3</sup>。

一个常见的误解在于，比特币[脚本](https://wiki.bitcoinsv.io/index.php/Script)是[虚拟机](https://blog.csdn.net/freedomhero/article/details/106801904)的低级指令集（也称为操作码）。它本身不需要循环指令。循环可以而且应该在更高层次上构建，就像所有现代编程语言一样。高级编程语言 sCrypt 已经在比特币脚本的基础上构建了[循环](https://scryptdoc.readthedocs.io/zh_CN/latest/loop.html)。如果同样的逻辑应用于以太坊虚拟机，它也是图灵不完备的，因为以太坊的操作码中也没有循环。Java 也是如此，因为它的虚拟机 JVM 也缺少循环操作码。这显然是错误的，因为众所周知 Java 是图灵完备的。

还有其他在比特币中循环的方式： 包括[链式交易](https://blog.csdn.net/freedomhero/article/details/107307306)和[支付通道](https://wiki.bitcoinsv.io/index.php/Payment_Channels)。


## 价值盲 (Value-blindness)

> UTXO脚本不能为账户的取款额度提供精细的的控制。

有一种方法可以让 UTXO 脚本使用一种称为 [OP_PUSH_TX](https://blog.csdn.net/freedomhero/article/details/107306604?spm=1001.2014.3001.5501) 的技术来完全控制以任何粒度提取的数量。它允许脚本访问其 UTXO 中的金额，并指定如何将其用于各种输出。已经开发了一些脚本/合约，只允许提取一定数量的金额，例如[这个](https://xiaohuiliu.medium.com/patreon-on-bitcoin-4c3626d4ce5)。


## 缺乏状态 (Lack of state)

> 没有给需要任何其它内部状态的多阶段合约或者脚本留出生存空间

有一些方法可以使用 OP_PUSH_TX 在比特币合约中[维护内部状态](https://blog.csdn.net/freedomhero/article/details/107307306)。已经开发了许多多阶段合同，例如 Tic-Tac-Toe<sup>4</sup> 和[拍卖](https://blog.csdn.net/freedomhero/article/details/114638176?spm=1001.2014.3001.5501)。

这类似于函数式编程如何保持状态。即使所有的函数都是纯的和无状态的，可变状态仍然可以使用抽象来构建，比如 [State monad](https://en.wikibooks.org/wiki/Haskell/Understanding_monads/State)。比特币脚本/UTXO 本身是无状态的，但它可以使用类似的方法实现状态。

## 区块链盲 (Blockchain-blindness)

> UTXO 脚本中无法访问随机数、时间戳和前一个区块哈希等区块链数据。

比特币 UTXO/脚本不允许访问区块链数据的一个主要原因是：**安全性**。如果脚本可以访问外部信息，攻击者就可以操纵这些信息以获得不公平的优势。具有讽刺意味的是，正是这些漏洞导致了对以太坊的许多攻击，例如[ SWC-116](https://swcregistry.io/docs/SWC-116) 和 [SWC-120](https://swcregistry.io/docs/SWC-120)，这在比特币上是不可能的。




##  脚注
--------------------------------------------------

[1] :  Craig S. Wright: [A Proof of Turing Completeness in Bitcoin Script. IntelliSys (1) 2019](https://www.dropbox.com/s/3u92p84l9eswkq7/proof_tc_bitcoin_script.pdf?dl=0): 299–313

[2] :  Craig S. Wright: [Bitcoin: A Total Turing Machine. IntelliSys (1) 2019](https://www.dropbox.com/s/ld8tu3l3sym1lnf/bitcoin_total_turing_machine.pdf?dl=0): 237–252

[3] [比特币是图灵完备的:使用 sCrypt 智能合约在比特币上实现康威生命游戏](https://blog.csdn.net/freedomhero/article/details/111152834) 

[4] [ 基于 sCrypt 合约开发一个完整的 dApp：井字棋游戏](https://blog.csdn.net/freedomhero/article/details/115419901) 