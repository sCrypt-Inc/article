# 没有 SegWit 和 Taproot 的比特币序数


序号 （Ordinals）已成为 BTC 圈子中创建不可替代令牌（NFT）的热门话题。 它的主要特点是将 NFT 本身完全存储在链上。


我们表明原始的比特币协议已经可以支持它。 Ordinals 不需要任何重大更改，包括 SegWit 和 Taproot。 实际上，它在原始协议上运行得更好：便宜 100,000 倍，大 12.5 倍。

## 序数是如何工作的

序号是[染色币](https://en.bitcoin.it/wiki/Colored_Coins)的最新表现形式，它标记单个聪（因此“着色”它们），以便它们可以携带额外的信息。 更具体地说，每个聪都根据其开采时间顺序排序，这是 2012 年在[比特币论坛](https://bitcointalk.org/index.php?topic=117224.0)上首次提出的。每个聪都将分配一个介于 0 和 2,100,000,000,000,000 之间的[序号](https://en.wikipedia.org/wiki/Ordinal_number)（因此称为“序数”）。 然后可以在每个独特的 satoshi 上刻上一些其他信息，以创建 NFT（又名铭文）。

考虑一个具有三个输入和两个输出的示例交易。



输入中有来自 3 个地址的 6 个聪。 输出包含 5 个聪到 2 个不同的地址。 剩余的 1 satoshi 作为费用支付给矿工。


Ordinals 使用先进先出算法将输入的每个 satoshi 分配给输出。 任何丢失的序号都相当于交给矿工。 在示例中，序数（及其代表的 NFT，如果有的话）A、B、C 和 D 进入第一个输出，E 进入第二个输出，F 进入矿工。


## 与其他基于比特币的 NFT 比较

以前的染色币通常使用 OP_RETURN，这是一种最多只有 80 字节元数据的不可花费的输出，这意味着存储了对 NFT 的链接/引用。

Ordinals 使用这些技术将整个 NFT 图像/内容存储在链上。

2021 年的 Taproot 升级完全取消了数据限制，只要它适合一个块。 2017 年的隔离见证（Segwit）软件升级允许存储多达 3MB 的见证数据，“超出”1MB 的块限制。 它们共同为铭文内容提供了高达 4MB 的存储空间。
铭文存储在操作码 OP_IF 和 OP_ENDIF 之间的“信封”中。 OP_FALSE 在 OP_IF 之前以确保此数据永远不会在脚本执行中实际使用并且不会占用堆栈空间。 没有使用 OP_RETURN，包含铭文的输出是可花费的，因此不可修剪。 以下示例写入文本“Hello, world!”：

## BSV 上的序数

今天的原始比特币协议 BSV 早在 [2021 年](https://coingeek.com/blockpress-the-easiest-way-to-create-nfts-on-bitcoin/)就率先将 NFT 完全上链存储。由于 BSV 和 BTC 发行 satothis 的算法相同，因此它可以以相同的方式分配 satoshis 序号。 BSV 上的序数比 BTC 上的效果要好得多。

1. 成本：BTC 费用约为 [10 sat/byte](https://privacypros.io/tools/bitcoin-fee-estimator/)，而 BSV 仅为 0.05 sat/byte¹。 考虑到今天 25,000 美元/BTC 与 45 美元/BSV 的价格，铸造相同的 NFT 便宜 >100,000 倍！
2. 数据大小：BSV 在 Genesis 升级后已经恢复了完整的 Script，所以同样的原理可以工作。 它还具有更多将数据嵌入交易的方法。 例如，[OP_RETURN](https://wiki.bitcoinsv.io/index.php/History_of_OP_RETURN) 可用于可花费输出。 目前，单笔交易最多可包含 50MB 的数据，比 BTC 的 4MB 限制大 12.5 倍。 这是一个 [27MB](https://test.whatsonchain.com/tx/eba34263bbede27fd1e08a84459066fba7eb10510a3bb1d92d735c067b8309dd) 的交易和另一个 [12MB](https://whatsonchain.com/tx/320ba9fb3826c0bc66beed51edf2463e958b7274921563c5c90be62deabb725f） 的交易。 随着使用量的增长，数据限制预计会不断增加。 更高级的 NFT 协议已经通过将 NFT 分解为多个交易来存储超过 1 GB 的 NFT。


BSV 在不更改协议的情况下这样做。 不需要 SegWit 和 Taproot。

-------------------------------------------------


参考

[1] https://docs.ordinals.com/

[2] https://medium.com/coinmonks/ordinals-an-overview-of-bitcoin-nfts-795c39447e23

[3] https://tara-annison.medium.com/a-comprehensive-explanation-of-ordinals-nfts-on-bitcoin-67b11868e74f

[4] https://www.chain.com/blog/what-are-bitcoin-ordinals



-------------------------------------------------------

[1] 我们比较平均费率。 在 BTC 中，SegWit 可以带来 4 倍的折扣，而在 BSV 中，一些矿工拿 0.01 sat/byte，5 倍的折扣。
