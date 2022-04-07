# 比特币上的安全多方计算

> 以去中心化彩票为例

[安全多方计算](https://en.wikipedia.org/wiki/Secure_multi-party_computation) (MPC) 协议使多方能够联合各方的输入，共同计算一个函数，同时保持这些输入的私密性。例如，两位百万富翁决定谁更富有，应该为晚餐买单，而不透露他们的实际财富<sup>1</sup>。或者一组员工可以在不披露个人工资的情况下计算该组的平均工资。

MPC 的一个基本限制是它不能强迫各方遵守约定。在百万富翁的例子中，一个人可以在发现自己更富有后拒绝付款。

我们使用比特币来解决这一挑战²，通过使用比特币的智能合约将 MPC 的结果与真实交易联系起来。

我们通过在没有受信任的第三方的情况下实现去中心化彩票来证明这一点。

## 去中心化彩票

![彩票](./lottery.jpeg)


N 个玩家中的每一个选择一个随机数并提交给合约。在他们透露他们的秘密号码后，将选出一名获胜者并拿走所有 N 个比特币。每个玩家获胜的概率相同。

```js

   
// a fair secure multiparty lottery
// each player i chooses a random number n_i and the winner is the w-th player
// where w = (n_0 + n_1 + .. + n_(N-1)) mod N
contract Lottery {
    // number of players
    static const int N = 5;
    // players identified by their addresses
    PubKey[N] players;
    // commitments: the hash of their random numbers
    Sha256[N] nonceHashes;

    public function reveal(int[N] nonces, Sig sig) {
        int i = 0;
        int sum = 0;
        loop (N) {
            // open commit
            require(hash256(pack(nonces[i])) == this.nonceHashes[i]);
            sum += nonces[i];
            i++;
        }

        PubKey winner = this.players[sum % N];

        require(checkSig(sig, winner));
    }
}
```

<center><a href="https://github.com/sCrypt-Inc/boilerplate/tree/master/contracts/lottery.scrypt">Lottery 合约源代码</a></center>

## 实际运用

在实践中，可以采取措施防止玩家不揭示他们的秘密号码。 一种方法是使用定时承诺<sup>2</sup>，如果玩家在截止日期前没有透露，他将失去他的存款。

## 结论

我们以去中心化彩票为例，展示了如何执行 MPC 规则。相同的技术可以推广到其他 MPC 协议，例如[抛硬币](https://blog.csdn.net/freedomhero/article/details/114257034)或[心理扑克](https://en.wikipedia.org/wiki/Mental_poker)。

-----------

[1] [姚的百万富翁问题](https://en.wikipedia.org/wiki/Yao%27s_Millionaires%27_problem)

[2] [比特币上的安全多方计算](https://cacm.acm.org/magazines/2016/4/200175-secure-multiparty-computations-on-bitcoin)