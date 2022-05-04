# Solidity 转译器简介

sol2scrypt 是一个转译器程序。它可以将 Solidity 合约代码转换为等价的 sCrypt 合约代码，方便开发者进行快速学习或代码迁移。

在介绍转译器的原理之前，我们再来回顾一下以太坊的账户模型和 BSV 的 UTXO 模型的主要差异：

- 以太坊应用的账户模型实际上为每个用户或合约地址都保存了一个独立的全局状态，并通过合约调用的方式触发其状态变更。这样做的好处是可以使用全局唯一的地址来进行快速寻址和消息传递，也更贴近传统数据库应用开发者的思维模型，但其最大缺点是性能问题，只能串行处理。

- BSV 使用的 UTXO 模型实际上为每个用户或合约地址维护了一个 UTXO 的集合，使用这个集合的聚合信息来表示用户或合约的状态。这样的好处是可以让事务处理尽可能独立，最大化并行处理能力，但无法使用单一地址进行寻址和消息传递，使得编写某些合约难度更高。

sol2scrypt 转译器开发的初衷就是实现从 Solidity 智能合约到 sCrypt 智能合约的自动转换工具。它为那些不太熟悉基于 UTXO 模型智能合约的开发者们提供了一个很好的起点。一方面它可以让开发者直观的看到同样的业务逻辑在两种语言的不同实现，另一方面也可以让他们免于从零开始编写等价的 sCrypt 合约。当然因为受限于前面提到的账户模型与 UTXO 模型的根本性差异，这个工具也会有一定的局限性，无法实现任意类型 Solidity 合约的100%全自动转换。

## 转译原理

如果用一句话概括转译器的基本原理就是：使用单个 UTXO 来模拟以太坊的单个合约账户，并以此为前提进行合约代码的等价变换。下面我们以合约的不同生命周期为观察点，简单介绍一下其映射原理。
 
### 合约部署
首先使用一个单一 UTXO 来映射 EVM 中的某个合约实例的初始状态；
 
### 合约调用
利用 [UTXO 有状态合约的工作原理](https://blog.csdn.net/freedomhero/article/details/107307306) 来映射 EVM 上合约的那些会改变合约内部状态方法调用。每当需要变更合约内部状态时，产生一个新的交易，它会花费掉合约当前的 UTXO 并产生一个带有新状态的 UTXO，以此类推。

### 合约销毁

以太坊的合约实例销毁需要调用 selfdestruct 方法使其内部状态标记为失效，而在 BSV 上我们只需要将合约当前合约 UTXO 花费掉并不再产生新的合约 UTXO 即可。

以上是整个转译器的基础原理，需要说明的是，这里只是应用了一种最简单直接的转译思路，我们仅仅使用了一个单一的 UTXO 来映射单一合约。实际上，针对特定的 Solidity 合约来说，可能存在其他更加复杂但能够提高并行性能的转译方式，只不过这可能需要更多地开发者人为设计。

## 语法转译

 
下面我们来看看一些具体的语法转译细节。
 

### 合约状态变量
 
由于 Solidity 的合约状态变量（state variables）是合约状态的具体载体，可能会随着合约的被调用发生改变，所以都会被转译为 sCrypt 的 带状态的合约属性（`@state` contract properties）
```solidity
contract A {
 int x;
 bool y;
 uint z;
 ...
}
```
 
转译结果为：
 
```js
contract A {
 @state
 int x;
 @state
 bool y;
 @state
 int z;
}
```
有下面两种例外情况：
 
* Solidity 的 `constant` 状态变量将转译为 sCrypt 的 `static const` 属性；
* Solidity 的 `immutable` 状态变量将转译为 sCrypt 的 `const` 属性；
 
举例来说，针对 Solidiy 代码：
 
```solidity
contract A {
 uint constant x = 1;
 uint immutable y;
 ...
}
```
 
转译结果为：
 
```js
contract A {
 static const int x = 1;
 const int y;
}
```

### 数据类型

 
#### 1. 基本数据类型

Solidity 的基本数据类型包括 `bool` / (`int` / `uint` 系列) / (`bytes` 系列/ `string`) / `address`，这些基本类型会被直接转译为 sCrypt 语言的 `bool` / `int` / `bytes` / `PubKeyHash` 类型。

#### 2. 结构体类型
 
Solidity 的结构体 `struct` 会被转译为 sCrypt 的结构体，其内部包含的所有属性也会被转译至相应的 sCrypt 类型。但是，如果某个结构体中包含合约类型的属性，则无法直接转译。
 
#### 3. 数组类型
 
Solidiy 的数组类型也是直接转译为 sCrypt 的数组，但这里有个限制条件：无法转译动态长度的数组，因为 sCrypt 的原生数组必须是定长的。
 
#### 4. Mapping 类型
 
Solidiy 中的 `mapping` 类型也是比较常见的一种数据结构，它会被转译为 sCrypt 的 `HashedMap` 数据类型，但这个转译过程比较特殊。

首先，Solidiy 的 `mapping` 就是一种常见的哈希表实现，它支持使用基本类型最为键，任意类型作为值，并且支持增删改查等操作。但是 sCrypt 中的 `HashedMap` 却不是这种普通的哈希表实现，它有几个非常不一样的特点：
 
1. `HashedMap` 中保存的并不是 `mapping` 数据结构中原始的 `key` 和 `value` 值，而是它们各自的哈希值，可以把它理解为是整个 `mapping` 结构的哈希结果。或者换一种说法，它是确保键值对存在于 `mapping` 中的一个凭证。
 
2. 当程序需要使用其中的某个键值对时，首先需要从外部参数传入具体的值，然后利用 `HashedMap` 来对其进行验证。如果验证通过说明传入参数是合法的，那么在下面的程序中就可以放心使用它们进行运算；反之则说明这是无效的键值对，程序不应该继续往下执行。

**`HashedMap` 使用这种设计的主要原因是考虑到脚本体积以及循环次数必须为常数的限制，更多说明还可以参考[这篇文章](https://blog.csdn.net/freedomhero/article/details/121395939)**。

 
这种基于验证的模式也是 BSV 区块链智能合约不同于其他智能合约的一个显著特点。
 
假设我们已有一个实例 `mapping(uint256 => uint256) m;`，针对该实例的读写访问将主要进行如下变换：
 
1. 在函数的输入参数中插入真正的 `val` 值和一个 `index` 值 （这个为 `key` 在所有键值排序后的索引值）；

2. 验证这个 `val` 是确实是 `key` 所对应的值；
 
3. 将访问表达式替换为传入的 `val` 参数; 
 
4. 如果是写操作，最后还需要插入对 `HashedMap` 的更新；
 
比如，针对如下 Soidity 代码：
 
```solidity
mapping(uint256 => uint256)  m;
 
function xxx(...) {
 ...
 a = m[key];
 m[key] = 1;
 ...
}
```
 
转译后的 sCrypt 代码为：
 
```javascript
@state
HashedMap<int, int> m;

function xxx(... int val, int idx) { // parameters injection
    ...
    require((!this.m.has(key, idx) && val == 0) || this.m.canGet(key, val, idx));  // validation
    a = val;   // for read case, replace `m[key]` with 'val'
    val = 1;   // for write case, also replace `m[key]` with 'val'
    require(this.m.set(key, val, idx)); // update
    ...
}
 
```

另外在处理嵌套的 `mapping` 类型时，比如 `mapping (address => mapping (address => uint)) `，会把嵌套的 `keys` 都定义为一个 sCrypt 的结构体，并使用它作为 `HashedMap` 的键类型。比如：
 
```solidity
mapping (address => mapping (address => uint)) nm;
 
...
 
x = nm[a][b];
```
 
会被转译为：
 
```javascript
struct MapKeyST0 {
   PubKeyHash key0;
   PubKeyHash key1;
}
 
HashedMap nm;
 
...
 
require((!this.nm.has({a, b}, idx) && val == 0) || this.nm.canGet({a, b}, val, idx));
x = val;
```

**需要特别注意的是，目前针对 `mapping` 的转换还存在一些限制，包括：**
 
* `key` 在函数内不能被重新赋值，否则可能会有非预期结果；
 
 ```solidity
 a = m[key];
 key = xxx;
 m[key] = b;
 ```
 
* 嵌套类型的 `mapping` 不能进行**部分读写**，比如 `nm[a] = anotherMapping;`

### 运算符

 
Solidity 的大部分操作符都可以直接转译到 sCrypt 相同的运算符上，比如常见的 `+`， `-`， `*`， `/` 等等。但也有不支持转译的操作符，比如幂操作符 `**`。
 
除此之外，在转译位运算的操作符时也要**非常小心**，因为 BSV 上的整数采用的是小端编码方式，且负数值编码也不是 Solidity 的二进制补码形式。所以虽然转译后的表达式形式一致，但执行结果可能并不相同。

### 条件语句
 
sCrypt 同样支持 `if` / `else` 语句， 直接转译即可。
 
### 循环语句
 
Solidity 中支持 3 种常见的循环语句，即 `for` / `while` / `do ... while`，其底层都是通过跳转指令来实现的。但是，由于 BVM 虚拟机的底层操作码中没有跳转指令，故并不能直接实现上述循环语句。而 sCrypt 通过引入 `loop` 语句，可以实现常数次的循环语句。当然这也要求循环次数是一个编译时常量。

所以在转译 Solidiy 的循环语句时，我们统一使用 `loop` 语句进行映射。由于具体的循环次数是和程序逻辑紧密相关的，转译器无法自动给定一个合理的取值，所以这里使用占位符变量比如 `__LoopCount__0` 来替代。这就要求开发者手动修改转译后的 sCrypt 代码，将其替换为合理的取值，否则转译后的 sCrypt 合约无法通过编译。循环次数一般可以根据gas limit反推出来。


#### `for` 语句
举例来说，针对下面的 Solidity 的 `for` 循环：
 
```solidity
uint sum = 0;
for(uint i=0; i<n2; i++) {
   sum += i;
}
```
 
根据其语义进行转译后的 sCrypt 代码为：
 
```js
int sum = 0;
int i = 0;
loop (__LoopCount__0) {
   if (i < n) {
       sum += i;
       i++;
   }
}
```

#### `while` 语句
 
应用类似的原理，可以处理 Solidity 的 `while` 语句：
 
```solidity
uint sum = 0;
int i = 0;
while(i < 10) {
   sum += i;
   i++;
}
```
 
转译结果为：
 
```js
int sum = 0;
int i = 0;
loop (__LoopCount__0) {
   if (i < 10) {
       sum += i;
       i++;
   }
}
```

#### `do ... while` 语句
 
`do ... while` 稍有区别的地方在于至少执行一次，所以对应 Solidity 代码：
 
```solidity
do {
   sum += i;
   i++;
} while (i < 100);
```
 
其转译结果为：
 
```js
{
   sum += i;
   i++;
}
loop (__LoopCount__0) {
   if (i < 100) {
       sum += i;
       i++;
   }
}
```

#### `break` 语句
 
正如我们刚才提到的，sCrypt 中无法直接跳转，所以第一直觉上我们是无法转译循环中常见的 `break` 或 `continue` 语句。但是实际上，我们还是可以结合条件语句 `if`/`else` 来实现这两个逻辑，只不过需要加入一个布尔类型的 `FLAG`。
 
比如，针对以下 Solidity 代码：
 
```solidity
for(uint i=0; i<n; i++) {
   sum += i;
   if(sum > 10)
       break;
}
```
 
转译后的结果为：
 
```js
bool loopBreakFlag0 = false;
int i = 0;
loop (__LoopCount__0) {
   if (!loopBreakFlag0 && i < n) {
       sum += i;
       if (sum > 10)
           loopBreakFlag0 = true;
       i++;
   }
}
```

#### `continue` 语句
 
与 `break` 的处理方式类似，唯一需要注意的是 `continue` 只是跳过了本次循环迭代内后面的语句；而 `break` 是跳过了之后剩余的所有循环迭代。
 
请看这个例子：
 
```solidity
do {
   sum += i;
   if (sum < 20)
       continue;
   i++;
} while (i < 100);
```
 
其转译结果为：
 
```js
{
   sum += i;
   i++;
}
loop (__LoopCount__0) {
   if (i < 100) {
       bool loopContinueFlag0 = false;
       sum += i;
       if (sum < 20) {
           {
               loopContinueFlag0 = true;
           }
       }
       if (!loopContinueFlag0) {
           i++;
       }
   }
}
```

### 函数

函数作为和智能合约交互的界面无疑是转译的另一个重点，而且由于 EVM 和 BVM 的差异，转译方式不会像之前介绍的一些语法单元那样直观易理解。下面我们来详细介绍一下相关的转译原理。
 
首先需要说明的是 Solidity 的函数有4种可见性（visibility）：
 
* `private`：只能被该合约自己调用；
* `internal`：只有该合约以及其子类合约可以调用；
* `external`：只能通过从外部发送消息（包括发送 TX）的方式进行调用；
* `public`：既可以使用 `internal` 方式调用，也可以 `external` 方式调用；
 
根据能否通过发送 Tx 来进行调用，可以把它们分为两类: `external` 和 `public` 是可以的，而 `private` 和 `internal` 则不能。根据这个标准，我们将前者其转译为 sCrypt 的 `public` 函数，是合约的对外接口；而将后者转译为 `private` 和 `default` 函数，是合约内部的具体实现。具体可参见：
 
| Solidity | sCrypt |
| :----: | :----: |
| `private` | `private` |
| `internal` | `default` |
| `external` | `public` |
| `public` | `public` |

#### `private` / `internal` 函数
 
这两类函数因为仅仅会在合约内部被调用，所以从转译器实现的复杂度考虑，约定其必须满足一个约束：即转译函数体时不能改变其参数的个数，从而导致函数签名的变化。一个函数转译的例子:


```solidity
function set(uint x) external {
    storedData = x;
}

function get() internal view returns (uint) {
    return storedData;
}
```
其转译结果为：
```js
public function set(int x, SigHashPreimage txPreimage) {
    this.storedData = x;
    require(this.propagateState(txPreimage, SigHash.value(txPreimage)));
}

function get() : int {
    return this.storedData;
}

function propagateState(SigHashPreimage txPreimage, int value) : bool {
    require(Tx.checkPreimage(txPreimage));
    bytes outputScript = this.getStateScript();
    bytes output = Utils.buildOutput(outputScript, value);
    return hash256(output) == SigHash.hashOutputs(txPreimage);
}
```

#### `private` / `internal` 函数中的 `return` 语句

**这里要注意的是：对于 `private` / `internal` 函数中的 `return` 语句，可能需要进行特殊处理。** 具体来说：
 
1. **无返回值**
 
因为 Solidity 允许无返回值但 sCrypt 不行，所以遇到这种情况时，会转译为返回一个默认的 `bool` 值。比如下面这段 Solidity 代码：
 
 ```solidity
 function a() private {
   return;
 }
 ```
 
 转译结果为：
 
 ```js
 private function a() : bool {
   return true;
 }
 ```

2. **中途返回**
 
前面提到过 BVM 底层是没有跳转指令的，所以 sCrypt 也不支持在函数中途进行返回。如果 Solidity 代码中出现了这种情况，就需要进行特殊处理。当然原理也很简单，类似之前 `break` / `continue` 的处理方法，增加 FLAG 配合 `if` / `else` 进行等价变换。比如针对 Solidity 代码：
 
```solidity
function get(uint amount) internal view returns (uint) {
   if (amount > 0)
       return amount;
   return 0;
}
```
 
转译结果为：
 
```js
function get(int amount) : int {
   int ret = 0;
   bool returned = false;
   if (amount > 0) {
       {
           ret = amount;
           returned = true;
       }
   }
   return returned ? ret : 0;
}
```

#### `public` / `external` 函数
 
这两类函数都可以通过发送 Tx 的方式进行调用，所以我们将其转译为 sCrypt 的 `public` 函数。但这其中有两个比较复杂的情况需要注意：

1. 返回值

sCrypt 的 `public` 函数实际上隐式地返回布尔类型，表示合约函数调用成功与否。Solidity 的 `public` / `external` 函数有返回值时，需要在转译时对函数做一个变换：即增加一个原返回值类型的参数，并且校验这个值与原返回值相等。比如这个例子：

```solidity
function get() external pure returns (uint) {
  return 1 + 1; 
}
```

转译结果为：

```js
public function get(int retVal) {
  require(1 + 1 == retVal);
}
```

这里我们再次强调：sCrypt 的 `public` 函数的功能是**验证解锁条件是否满足**，而不是通过计算给出某个结果。

2. 状态传递

另一个直观上不太容易理解的就是[合约的状态变化](https://scryptdoc.readthedocs.io/zh_CN/latest/state.html)的传递。之前我们也提到过，sCrypt 的有状态合约是基于 UTXO 模型的，每次调用其 `public` 方法后，我们都需要保证状态正确传递到新的合约 UTXO。这也是为什么从 Solidity 转译过来的 `public` 函数基本都会以一个 `require(this.propagateState(...)));` 语句结尾。当然函数的参数列表中也因此需要增加一个常见的 `SigHashPreimage` 类型参数（即交易原像）。

```solidity
function set(uint x) external {
    storedData = x;
}
```

转译结果为：
```js
public function set(int x, SigHashPreimage txPreimage) {
    this.storedData = x;
    require(this.propagateState(txPreimage));
}
``` 

#### 内置对象属性： `msg.sender` 与 `msg.value`
 
在 Solidity 的函数中比较常用的两个内置对象属性是：`msg.sender` 和 `msg.value`。前者返回当前的函数调用者地址；后者返回该消息中携带的 ether 数量 (单位：`wei`)。
 
在转译时，我们将二者分别映射为公共函数的调用者地址以及通过本次调用为合约 UTXO 自身增加的 BSV 数量（单位： `satoshi`）。
 
对于 `msg.sender` 来说，我们在公共方法中增加其对应的 `PubKey` 类型和 `Sig` 类型参数，并在函数中增加以下语句，通过验证签名的有效性进而保证是该地址进行了合约方法调用：


```js
PubKeyHash msgSender = hash160(pubKey);
require(checkSig(sig, pubKey));
```
 
对于 `msg.value` 来说，我们在公共方法中增加一个 `int` 类型参数 `msgValue`，要求调用者将其值主动作为参数传入。与此同时，通过[OP_PUSH_TX](https://blog.csdn.net/freedomhero/article/details/107306604)技术，可以知道合约原来锁定的余额。合约解锁后新的余额应为： `SigHash.value(txPreimage) + msgValue`。最后在函数内部使用前面提到的 `propagateState` 方法验证新的合约 UTXO 中的余额确实包含了这个值带来的增量。


```js
public function xxx(... int msgValue, SigHashPreimage txPreimage) {
 ...
 int contractBalance = SigHash.value(txPreimage) + msgValue; // add `msgValue` to contract balance
 require(msgValue >= 0); // basic validation
 ...
 require(this.propagateState(txPreimage, contractBalance)); // check the new balance is real
}
```


这里有个特殊情况需要注意，如果 Solidity 在构造函数中访问了 `msg.value`，那么在转译时会自动为合约增加一个 `initBalance` 的属性，并且在构造函数中使用 `msg.value` 对其赋值。另外，由于 sCrypt 的构造函数比较特殊，无法在其内部验证作为参数传入的值是否伪造，故需要在其他公共方法中验证这个初始值无误。例如，对于 Solidity 代码：

```solidity
constructor (...) {
 a = msg.value;
 ...
}
```
 
其转译结果为：
 
```js
const int initBalance;

constructor(... int msgValue) {
  this.a = msgValue;
  ...
  this.initBalance = msgValue;
}
 
public function xxx(... SigHashPreimage txPreimage) {
  require(this.checkInitBalance(txPreimage));
  ...
}
```


## 其他局限性
 
目前转译器还有一些其他无法正常进行转译的 Solidity 语法，主要包括以下内容：
 
* 不支持元组以及多元赋值表达式；
* 不支持枚举；
* 不支持接口；
* 不支持接口继承；
* 不支持异常处理机制 `try` / `catch`；
* 不支持使用 `modifier`；
* 不支持使用 `receive` / `fallback` 函数；
* 不支持使用其他内置方法和属性, 例如 `block.*` / `tx.*` / ...；
* 不支持合约间调用；
* 忽略所有事件定义以及 `emit()` 函数调用；
* 不支持 `assembly` 语句；
 
## 总结
 
诚如一开始我们提到的，这个转译器的主要作用是帮助开发者从 Solidity 合约快速过度到 sCrypt 合约，以便能够利用 BSV 区块链更加高效低成本的网络构建 Web3 应用。希望能够对大家有所帮助。如果有任何疑问或建议，请加入我们的 [sCrypt slack 讨论组](https://join.slack.com/t/scryptworkspace/shared_invite/enQtNzQ1OTMyNDk1ODU3LTJmYjE5MGNmNDZhYmYxZWM4ZGY2MTczM2NiNTIxYmFhNTVjNjE5MGYwY2UwNDYxMTQyNGU2NmFkNTY5MmI1MWM) 或者 [github 仓库](https://github.com/sCrypt-Inc/sol2scrypt)。




