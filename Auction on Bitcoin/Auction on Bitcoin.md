# BSV区块链上的拍卖合约

我们在比特币网络上设计并实现了一个安全的拍卖合约。 它是公开透明的，每个人都可以参加，竞标结束后出价最高的竞标者将中标。 投标人受其出价的约束，而拍卖人则受拍卖结果的约束。

![Auction](./auction.jpeg)

## 实现

```js
// Auction: highest bid before deadline wins
contract Auction {
    @state
    PubKeyHash bidder;

    PubKey auctioner;
    int auctionDeadline;

    // bid with a higher offer
    public function bid(PubKeyHash bidder, int bid, int changeSats, SigHashPreimage txPreimage) {
        require(Tx.checkPreimage(txPreimage));

        int highestBid = SigHash.value(txPreimage);
        require(bid > highestBid);

        PubKeyHash highestBidder = this.bidder;
        this.bidder = bidder;

        // auction continues with a higher bidder
        bytes stateScript = this.getStateScript();
        bytes auctionOutput = Utils.buildOutput(stateScript, bid);

        // refund previous highest bidder
        bytes refundScript = Utils.buildPublicKeyHashScript(highestBidder);
        bytes refundOutput = Utils.buildOutput(refundScript, highestBid);

        bytes changeScript = Utils.buildPublicKeyHashScript(bidder);
        bytes changeOutput = Utils.buildOutput(changeScript, changeSats);

        bytes output = auctionOutput + refundOutput + changeOutput;

        require(hash256(output) == SigHash.hashOutputs(txPreimage));
    }

    // withdraw after bidding is over
    public function close(Sig sig, SigHashPreimage txPreimage) {
        require(Tx.checkPreimage(txPreimage));
        require(SigHash.nLocktime(txPreimage) >= this.auctionDeadline);
        require(checkSig(sig, this.auctioner));
    }
}
```

<center><a href="https://github.com/sCrypt-Inc/boilerplate/blob/master/contracts/auction.scrypt" >Auction 合约源码</a></center>

`bid` （竞价）函数逻辑：如果找到更高的出价，则更新当前的中标者，并退款给之前的最高出价者。
`close` （成交）函数逻辑：拍卖者可以在到期后关闭拍卖并接受要约。


## 可能的扩展
有很多方法可以扩展此合同。 例如，如果拍卖的物品被标记并存储在 UTXO（例如 NFT）中，可以要求交易 Tx 的一个输入是通证的 UTXO，通过一个输出将其转移给买主，从而使得成交操作是原子性的，不可能作弊。