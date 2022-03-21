# sCrypt 1.0.0 正式版正式发布

经过1年多的努力，40几个版本的迭代更新，sCrypt 今天正式发布 1.0.0 版本。


1.0.0 版本包含以下重大更新：


1. 完整的 **IDE** 使用 [文档](https://scrypt-ide.readthedocs.io/zh/latest/getting_started.html)

2. 高级付费功能 [代码优化](https://scrypt-ide.readthedocs.io/zh/latest/premium.html#optimize)
3. 更加细致和安全的代码格式化器。
4. 更加完善的语法高亮效果。
5. 新增 `:hex2Asm`, `:parsePreimage`, `:diffoutputs` 三个内置的[调试器命令](https://scrypt-ide.readthedocs.io/zh/latest/repl.html#id3)
6. 新增 `sCrypt: Launch Debugger` [命令](https://scrypt-ide.readthedocs.io/zh/latest/testing.html#launch-debugger)，以便快速启动调试器。



## 代码优化

代码优化是将一段代码转换为其他功能上等效的代码，以提高一个代码运行速度和减少脚本大小，节省交易费用。当使用 [发布编译](https://scrypt-ide.readthedocs.io/zh/latest/compiling.html#release-compiling) 来构建合约时，编译器会自动开启优化，优化可以显著减小脚本的体积并提高其性能, 部分合约最高可达 `50%`。以下是部分合约的优化结果：

| 合约        | 优化前    |  优化后  |
| --------   | -----:   | :----: |
| `p2pkh.scrypt`    | 26 bytes | 12 bytes |
| `rpuzzle.scrypt` | 71 bytes | 40 bytes |
| `advancedCounter.scrypt` | 1556 bytes | 1152 bytes |
| `advancedTokenSale.scrypt` | 1642 bytes | 1214 bytes |




## 购买许可证

每个许可证 `199` 美元，请查看我们的 **IDE** 文档，了解如何 [购买和使用许可证](https://scrypt-ide.readthedocs.io/zh/latest/premium.html#buy-license)


## 下载安装

欢迎下载使用 `1.0.0` 正式版本， [下载](https://marketplace.visualstudio.com/items?itemName=bsv-scrypt.sCrypt)