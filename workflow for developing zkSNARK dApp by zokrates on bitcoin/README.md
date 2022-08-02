# 使用 Zokrates 在比特币上创建您的第一个 zkSNARK 证明


这个 [ZoKrates](https://github.com/Zokrates/ZoKrates) [Fork库](https://github.com/sCrypt-Inc/zokrates)  是比特币上 zkSNARKs 的工具箱。它可以帮助您在应用程序中使用可验证的计算，从高级语言的编写的电路到生成计算证明，再到在 sCrypt 中验证这些证明。

## 安装

下载并使用我们发布的二进制包:

https://github.com/sCrypt-Inc/zokrates/releases/latest

或者从源码编译：

```bash
git clone https://github.com/sCrypt-Inc/ZoKrates
./build_release.sh
cd target/release
```

## 工作流程

整个工作流程与原始 ZoKrates 相同，除了验证部分。

1. 编写电路程序，创建文本文件 `root.zok` 并实现以下的程序。在这个例子中，我们将证明知道数字 `b` 的平方根为 `a` ：

```
def main(private field a, field b) {
    assert(a * a == b);
    return;
}
```

2. 编译电路

```
zokrates compile -i root.zok
```

3. 执行设置

```
zokrates setup
```

4. 计算见证人

```
zokrates compute-witness -a 337 113569
```

5. 生成证明

```
zokrates generate-proof
```

6. 导出验证者智能合约 `verifier.scrypt`, 同时会提供一个 `verifier.js` 文件

```
zokrates export-verifier-scrypt
```

7. 执行 `verifier.js` 验证 zkSNARK 证明<sup>1</sup>。如果本地验证成功，则会将 `verifier.scrypt` 验证者合约部署到测试网<sup>2</sup>，并调用部署的验证者合约

```
node --max-old-space-size=8192 verifier.js
```

------------------------

- [1] 在此之前，你需要确保安装了 `scryptlib` 和 `axios` *nodejs* 模块。 由于验证者合约较大，增加 `--max-old-space-size=8192` 确保 *nodejs* 内存充足

- [2] 需要填写测试网私钥
