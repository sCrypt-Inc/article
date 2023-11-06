# 将 Ordinals 与比特币智能合约集成：第 4 部分

> 控制 BSV-20 代币的分配

在[上一篇文章](https://github.com/sCrypt-Inc/article/blob/master/Integrate%20Ordinals%20with%20Smart%20Contracts%20on%20Bitcoin%20-%20Part%203/README.md#L1)中，我们展示了智能合约可以在铸造后控制 [BSV-20](https://docs.1satordinals.com/bsv20) 代币的转移。 今天，我们演示如何控制此类代币的分发/发行。

## 无Tick模式

BSV-20 在 V2 中引入了无Tick模式，并采用了与 V1 不同的方法。

## 部署 (Deploy)

要部署和铸造供应量为 21000000 的代币，请写入以下 JSON（ContentType：application/bsv-20）：

```json

{ 
  "p": "bsv-20",
  "op": "deploy+mint",
  "amt": "21000000",
  "sym": "sCrypt",
  "dec": "10"
}
```

请注意，与 V1 不同，没有指定的刻度字段（因此无刻度）。

`sym` 字段仅代表代币名称，不用于建立代币的索引。

## 发行 (Issue)

要发行上述 `10000` 个代币，您可以使用以下 JSON 创建转移铭文:

```json
{ 
  "p": "bsv-20",
  "op": "transfer",
  "id": "3b313338fa0555aebeaf91d8db1ffebd74773c67c8ad5181ff3d3f51e21e0000_1"
  "amt": "10000",
}
```

代币不是通过 `tick` 来标识，而是通过 `id` 字段来标识，该字段由交易 ID 和部署代币的输出索引组成，格式为 <txid>_<vout>。


此外，第一个发行交易必须从部署交易中支出，因为整个供应量是立即铸造的，而铸造交易不会从中支出，并且它们在 V1 中是分开的。 这意味着代币的每笔交易都可以追溯到该代币的创世部署，并且每笔交易都位于植根于创世交易的 DAG（有向无环图）中。 这使得 BSV-20 索引器能够更有效地扩展，因为它不必扫描整个区块链并排序铸币交易，以强制“[first is first](https://docs.1satordinals.com/bsv20#specification)”铸币。

有关 BSV-20 代币 V2 工作原理的更多详细信息，请阅读[官方文档](https://docs.1satordinals.com/bsv20)。

## 公平发布

与 ERC-20 代币相比，BSV-20 V1 代币的一个显着特点是公平发行。 具体来说，一旦有人在 BSV-20 上部署代币交易，每个人都有相同的机会领取代币。 发行人不能免费预留一部分，即没有预挖。

如果在 V2 无代码模式下部署时一次性铸造出总供应量，是否可以保持公平发布？

答案是肯定的。 *我们在部署时没有将整个供应锁定在标准发行人地址（P2PKH 脚本）中，而是将其锁定在智能合约中。* 任何人都可以调用智能合约，并且可以在其中执行任何分配策略。

![](./1.webp)

在上图中，每个方框代表一个代币 UTXO，堆叠的 UTXO 位于同一笔交易中。 第二笔交易花费了第一笔部署交易的 UTXO（如第一个箭头所示），并创建了两个 UTXO：

- 创世时相同合约的衍生副本，但剩余供应量减少
- 新发行的代币。


交易链一直持续到整个代币供应被发行为止。 请注意，任何人都可以调用该合约。

我们列出了一些分发策略作为示例。

### 速率限制

根据这项策略，任何人都可以领取代币，只要距离最后一次领取的时间超过 5 分钟即可。 合约如下。

```ts
export class BSV20Mint extends BSV20V2 {
    @prop(true)
    supply: bigint

    @prop()
    maxMintAmount: bigint

    @prop(true)
    lastUpdate: bigint

    @prop()
    timeDelta: bigint

    constructor(
        id: ByteString,
        max: bigint,
        dec: bigint,
        supply: bigint,
        maxMintAmount: bigint,
        lastUpdate: bigint,
        timeDelta: bigint
    ) {
        super(id, max, dec)
        this.init(...arguments)

        this.supply = supply
        this.maxMintAmount = maxMintAmount
        this.lastUpdate = lastUpdate
        this.timeDelta = timeDelta
    }

    @method()
    public mint(dest: Addr, amount: bigint) {
        // Check time passed since last mint.
        assert(
            this.timeLock(this.lastUpdate + this.timeDelta),
            'time lock not yet expired'
        )

        // Update last mint timestamp.
        this.lastUpdate = this.ctx.locktime

        // Check mint amount doesn't exceed maximum.
        assert(amount <= this.maxMintAmount, 'mint amount exceeds maximum')

        // Update supply.
        this.supply -= amount

        // If there are still tokens left, then
        // build state output inscribed with leftover tokens.
        let outputs = toByteString('')
        if (this.supply > 0n) {
            outputs += this.buildStateOutputFT(this.supply)
        }

        // Build FT P2PKH output to dest paying specified amount of tokens.
        outputs += BSV20V2.buildTransferOutput(dest, this.id, amount)

        // Build change output.
        outputs += this.buildChangeOutput()

        assert(hash256(outputs) == this.ctx.hashOutputs, 'hashOutputs mismatch')
    }
}
```

[合约源代码](https://github.com/sCrypt-Inc/boilerplate/blob/master/src/contracts/bsv20Mint.ts)

第 6-12 行强制执行速率限制。 第 26-30 行确保不超过供应量。 如果是，第 38-52 行将创建一个包含相同合约但具有更新状态的输出：剩余供应量。 第 55-58 行向目标地址发出令牌。

### 迷你工作量证明

该策略确保任何人都可以领取代币，只要她发现随机数满足某些特定的难度要求，就像比特币的工作量证明（PoW）一样。

```ts
export class Pow20 extends SmartContract {
    // ...

    @method(SigHash.ANYONECANPAY_ALL)
    public mint(
        nonce: ByteString,
        lock: ByteString,
        trailingOutputs: ByteString
    ) {
        if (len(this.id) == 0n) {
            this.id =
                this.ctx.utxo.outpoint.txid +
                int2ByteString(this.ctx.utxo.outpoint.outputIndex, 4n)
  }
        this.pow = this.validatePOW(nonce)
        const reward = this.calculateReward()
        this.supply -= reward
        let stateOutput = toByteString('')
        if (this.supply > 0n) {
            stateOutput = this.buildStateOutput(1n)
        }
        const rewardOutput = Utils.buildOutput(
            this.buildInscription(lock, reward),
            1n
        )

        const outputs: ByteString = stateOutput + rewardOutput + trailingOutputs
        assert(
            hash256(outputs) == this.ctx.hashOutputs,
            `invalid outputs hash ${stateOutput} ${rewardOutput} ${trailingOutputs}`
        )
    }

    @method()
    calculateReward(): bigint {
        let reward = this.reward
        if (this.supply < this.reward) {
            reward = this.supply
        }
        return reward
    }

    @method()
    validatePOW(nonce: ByteString): ByteString {
        const pow = hash256(this.pow + nonce)
        const test = rshift(Utils.fromLEUnsigned(pow), 256n - this.difficulty)
        assert(test == 0n, pow + ' invalid pow')
        return pow
    }
}
```

<center>归功于: David Case</center>

### ICO


可以实施一项策略，以便任何人都可以通过以无需信任的方式将比特币发送到特定地址来接收代币，类似于首次代币发行（ICO）。 在上图中，添加了第三个输出用于比特币支付，该输出在合约中进行验证。


```ts
export class BSV20Mint extends SmartContract {
    // ...

    @method()
    public mint(dest: Addr, amount: bigint) {
        // If first mint, parse token id and store it in a state var
        if (this.isFirstMint) {
            this.tokenId =
                BSV20Mint.txId2Ascii(this.ctx.utxo.outpoint.txid) +
                toByteString('_', true) +
                BSV20Mint.int2Ascii(this.ctx.utxo.outpoint.outputIndex)
            this.isFirstMint = false
        }

        // Check if tokens still available.
        assert(
            this.totalSupply - this.alreadyMinted >= amount,
            'not enough tokens left to mint'
        )

        // Update already minted amount.
        this.alreadyMinted += amount

        let outputs = toByteString('')

        if (this.alreadyMinted != this.totalSupply) {
            // If there are still tokens left, then
            // build state output inscribed with leftover tokens.
            const leftover = this.totalSupply - this.alreadyMinted
            const transferInscription = BSV20Mint.getTransferInsciption(
                this.tokenId,
                leftover
            )
            const stateScript = slice(
                this.getStateScript(),
                this.prevInscriptionLen
            ) // Slice prev inscription
            outputs += Utils.buildOutput(transferInscription + stateScript, 1n)

            // Store next inscription length, so we know how much to slice in the next iteration.
            this.prevInscriptionLen = len(transferInscription)
        }

        // Build P2PKH output to dest paying specified amount of tokens.
        const script1 =
            BSV20Mint.getTransferInsciption(this.tokenId, amount) +
            Utils.buildPublicKeyHashScript(dest)
        outputs += Utils.buildOutput(script1, 1n)

        // Calculate total price for minted amount and enforce payment to ICO address.
        const priceTotal = this.icoPrice * amount
        outputs += Utils.buildPublicKeyHashOutput(this.icoAddr, priceTotal)

        // Build change output.
        outputs += this.buildChangeOutput()

        assert(hash256(outputs) == this.ctx.hashOutputs, 'hashOutputs mismatch')
    }
}
```