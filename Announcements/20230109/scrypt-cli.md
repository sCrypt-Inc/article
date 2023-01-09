# 正式发布 **sCrypt CLI** 命令行工具

**sCrypt CLI** 使 sCrypt 的开发更快更容易的 CLI 工具。

## 安装

```bash
npm install -g scrypt-cli
```

## 如何使用

```bash
scrypt project my-proj
```

或者

```bash
scrypt p my-proj
```

该命令创建一个新目录 my-proj，其中包含一个 demo sCrypt 智能合约以及所需的脚手架。

同时包含一个项目 `README.md` 文件， 阅读它以获取有关如何测试和部署生成的智能合约的更多信息。

您还可以使用以下命令生成有状态的智能合约项目：

```bash
scrypt p --state my-proj
```

最后，您可以使用以下选项创建一个 sCrypt 库项目：

```bash
scrypt p --lib my-lib
```

## 编译项目

```ts
scrypt compile
```

这将在当前项目中搜索继承 `SmartContract` 的类并编译它们。这将为每个编译类生成一个合约制品文件。这些文件将存储在 `artifacts` 目录下。

该命令需要在项目的根目录下运行。

## 发布项目

```ts
scrypt publish
```

这将检查当前项目的结构并将其发布到 NPM 上。

## 获取系统信息

经常提交问题时，提供有关您的系统的信息很有用。您可以使用以下命令获取此信息：


```ts
scrypt system
```