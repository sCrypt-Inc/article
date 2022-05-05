# Introduction to Solidity -> sCrypt Transpiler

`sol2scrypt` is a transpiler that converts Solidity code into equivalent sCrypt code, making it easier for Solidity developers to migrate code and quickly learn sCrypt.

Before introducing the innerworkings of the transpiler, let us review the main differences between Ethereum's account model and Bitcoin's UTXO model:

- The account model of Ethereum maintains a separate state for each contract and updates it through contract calls. The advantage of this is that a globally unique address can be used for fast lookup of a contract and it is closer to the model of traditional databases. But its biggest disadvantage is contracts can only be processed sequentially, hindering performance.

- The UTXO model used by Bitcoin maintains a set of UTXOs for each contract and uses the aggregated set to represent the state of the contract. The advantage of this is that transactions can be processed as independently as possible, maximizing parallelization. The downside is that the inability to use a single fixed address for addressing makes writing certain contracts more difficult.

The `sol2scrypt` transpiler aims to be an automatic conversion tool from Solidity to sCrypt smart contracts. It provides a good starting point for developers who are less familiar with smart contracts based on the UTXO model. On the one hand, it allows developers to intuitively see different implementations of the same business logic in two languages. On the other hand, it saves them from writing equivalent sCrypt contracts from scratch.

## Transpiling Principles

At high level, we use a single UTXO to represent a snapshot of an Ethereum contract and perform the equivalent conversion of the contract code on this premise. We take the different life cycles of the contract as observation points and briefly introduce their mapping principle.

### Contract Deployment

We use a single UTXO to store the initial state of a contract instance in the Ethereum Virtual Machine (EVM).

### Contract Call

We use [Stateful Smart Contracts](https://medium.com/coinmonks/stateful-smart-contracts-on-bitcoin-sv-c24f83a0f783) to map function calls that change a contract's state. Whenever the state of the contract needs to be changed, a new transaction is generated that spends the current UTXO of the contract and generates another UTXO with the new state.

### Contract Destruction

The destruction of the contract instance of Ethereum needs to call the `selfdestruct` method to mark its internal state as invalid. On Bitcoin we only need to spend the current UTXO of the contract and no longer generate a new contract UTXO, instead.

The above is the basic principle of the entire transpilation. It should be noted that it is the most straightforward way to transpile. We only use a single UTXO to represent a single contract. In fact, for a specific Solidity contract, there may be other transpiling methods that are more complex but can improve parallel performance, but this may demand more manual intervensions by developers.

## Syntax Transpiling

 
Let's take a look at syntax transpiling details.
 

### Contract State Variables
 
Since Solidity's contract state variables carry the contract state and they may change as the contract is called, they will all be transpiled into sCrypt's stateful contract properties.

```solidity
contract A {
 int x;
 bool y;
 uint z;
 ...
}
```
 
The transpiling result is:
 
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

There are two special cases:
 
* Solidity's `constant` state variables are transpiled into `static const` properties of sCrypt;
* Solidity's `immutable` state variables are transpiled into `const` properties of sCrypt;
 
For example, for Solidiy code:
 
```solidity
contract A {
 uint constant x = 1;
 uint immutable y;
 ...
}
```
 
The transpiling result is:
 
```js
contract A {
 static const int x = 1;
 const int y;
}
```

### Type of Data

 
#### 1. Basic Data Types

Solidity's basic data types include `bool` / (`int` & `uint`) / (`bytes` & `string`) / `address`. These basic types will be directly transpiled into `bool`/`int`/`bytes`/`PubKeyHash` type.

#### 2. Struct Type

Solidity's `struct` will be transpiled into sCrypt `struct` and each member contained in it will be transpiled into the corresponding sCrypt type. However, if a structure contains properties of contract type, it cannot be directly transpiled currently.
 
#### 3. Array Type

Solidiy's array type is also directly transpiled into sCrypt arrays, with a limitation: dynamic-length arrays cannot be transpiled, because sCrypt's native arrays must be of fixed-length.
 
#### 4. Mapping Type

The `mapping` type in Solidiy is also a popular data structure. It will be transpiled into the `HashedMap` data type of sCrypt, but special care must be taken in this transpilation process.

First of all, Solidiy's `mapping` is a regular hash table implementation. It supports the use of basic types as keys, arbitrary types as values, and supports operations such as addition, deletion, modification, and search. But the `HashedMap` in sCrypt is not such an ordinary hash table implementation and has several very unique characteristics:


1. What is stored in `HashedMap` is not the original `key` and `value` values in the `mapping` data structure, but their hashes. It can be roughly regarded as the hash of the entire `mapping`. Or to put it another way, it is a proof that ensures the key-value pair exists in the `mapping`.

2. When a program needs to use one of the key-value pairs, it first needs to pass in the original values as external arguments and uses `HashedMap` to verify their authenticity. If the verification passes,  the passed arguments are legitmate and can be used safely; otherwise, it indicates the key-value pair is invalid and the program should not use them.


**The main reason why `HashedMap` is designed this way is due to the limit of the script size and thus the lack of unbounded loops. For more explantion, please refer to [this article](https://blog.csdn.net/freedomhero/article/details/121395939)**.

 
This verification-based model is a distinguishing feature of Bitcoin smart contracts compared to those on other blockchains.
 
Suppose we already have an instance `mapping(uint256 => uint256) m;`, access to this instance will be transformed as follows:
 
1. Append the orignal `val` value and an `idx` value (the index of `key` after all key hashes are sorted) in the input parameters of the function ;

2. Verify that this `val` is indeed the value corresponding to `key`;
 
3. Replace any read operation with the new `val` parameter;
 
4. If there is any write operation, the update to `HashedMap` needs to be appended;
 
For example, for the following Solidity code:
 
```solidity
mapping(uint256 => uint256)  m;
 
function xxx(...) {
 ...
 a = m[key];
 m[key] = 1;
 ...
}
```
 
The transpiling result is:
 
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

In addition, when dealing with nested `mapping` types, such as `mapping (address => mapping (address => uint)) `, the nested keys are defined as an sCrypt struct, used as the key type of the `HashedMap`. for example:
 
```solidity
mapping (address => mapping (address => uint)) nm;
 
...
 
x = nm[a][b];
```
 
The transpiling result is:
 
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

**It is important to note that there are currently some restrictions on the conversion of `mapping`, including:**
 
* `key` cannot be reassigned within a function, otherwise the result is undefined;
 
 ```solidity
 a = m[key];
 key = xxx;
 m[key] = b;
 ```
 
* `mapping` of nested types cannot be **partially read and written**, e.g. `nm[a] = anotherMapping;`

### Operators


Most of Solidity's operators can be directly transpiled into the same operator in sCrypt, such as `+`, `-`, `*`, and `/`. There is only one exception, the exponentiation operator `**` is not currently supported.

In addition, care must be taken when transpiling bitwise operators. Integers in sCrypt are encoded in little endian, and negative numbers are not encoded in signed-magnitude, not Solidity's two's complement. Although the transpiled expressions look the same, the evaluation result may not be the same.

### Conditional Statements
 
`if` / `else` statements are directly transpiled verbatim.
 
### Loop Statements


Solidity supports three common loop statements, namely `for`, `while`, and `do ... while`. They are all implemented via jump instructions internally. However, since there is no jump instruction in the Bitcoin virtual machine (BVM), the loop statements cannot be directly implemented. sCrypt's loop construct is implemented as the `loop` statement, whose number of loops has to be known at compile time.


When transpling Solidiy's loop statements, we uniformly map them to sCrypt's `loop`. Since the specific number of loops is closely related to the business logic, the transpiler cannot always automatically give a reasonable estimate, a placeholder variable such as `__LoopCount__0` is used instead. This requires developers to manually modify the transpiled sCrypt code and replace the placeholder. Otherwise the transpiled sCrypt contract cannot be compiled. The number of loops can generally be deduced from the gas limit.


#### `for` Statement

For example, for the following Solidity code `for` loop:
 
```solidity
uint sum = 0;
for(uint i=0; i<n; i++) {
   sum += i;
}
```
 
The transpiling result is:
 
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

#### `while` Statement
 
Applying a similar principle, Solidity's `while` statement can be handled:
 
```solidity
uint sum = 0;
int i = 0;
while(i < 10) {
   sum += i;
   i++;
}
```
 
The transpiling result is:
 
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

#### `do ... while` Statement
 
`do ... while` is slightly different in that its loop body is executed at least once, so for the following Solidity code:
 
```solidity
do {
   sum += i;
   i++;
} while (i < 100);
```
 
The transpiling result is:
 
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

#### `break` Statement


As we just mentioned, there is no direct jump in sCrypt, it seems impossible to transplie the `break` or `continue` statements common in loops at first glance. But in fact, we can still combine the conditional statements `if` and `else` to implement these two logics, with the help of an additional boolean flag.
 
For example, for the following Solidity code:
 
```solidity
for(uint i=0; i<n; i++) {
   sum += i;
   if(sum > 10)
       break;
}
```
 
The transpiling result is:
 
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

#### `continue` Statement

Tranpiling `continue` is similar to how `break` is handled, the only notable difference is that `continue` only skips the following statements within the current loop iteration, while `break` skips all remaining loop iterations afterwards.
 
Let us look at this example:
 
```solidity
do {
   sum += i;
   if (sum < 20)
       continue;
   i++;
} while (i < 100);
```
 
The transpiling result is:
 
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

### Function


Functions act as the interface to interacting with smart contracts. Due to the differences between EVM and BVM, the transpilation will not be as intuitive and easy to understand as some of the syntaxs introduced earlier. We will cover the relevant transpilation principles in detail.

 
The first thing to note is that Solidity functions have 4 types of visibility:
 
* `private`: can only be called by the contract itself;
* `internal`: can only be called by this contract and its derived contracts;
* `external`: can only be called by sending a message from outside (including sending a transaction);
* `public`: can be called either in `internal` or `external` mode;


Depending on whether they can be called by sending a transaction, they can be divided into two categories: `external` and `public` are allowed, while `private` and `internal` are not. According to this standard, we transplate the former into the `public` function of sCrypt, which is the external interface of the contract, and the latter into the `private` and `default` functions, which are the specific implementation inside the contract. For details, see:
 
| Solidity | sCrypt |
| :----: | :----: |
| `private` | `private` |
| `internal` | `default` |
| `external` | `public` |
| `public` | `public` |

#### `private` / `internal` functions


Since these two types of functions are only called inside a contract, considering the complexity of the transpiler implementation, it is agreed that they must satisfy a constraint: that is, the number of parameters cannot be changed when transpling the function body, which will lead to changes in the function signature. An example of function transpilation:


```solidity
function f2(uint a, uint b) internal pure returns (uint) {
    return a + b;
}
```

The transpiling result is:

```js
static function f2(int a, int b) : int {
    return a + b;
}
```

##### `return` statement

**Note here: Special handling may be required for `return` statements in `private` / `internal` functions.** Specifically:
 
1. **No return value**


Since Solidity allows function without return value but sCrypt does not, it will be transpilated into returning a default `bool` value in this case. For example, the following Solidity code:
 
 ```solidity
 function a() private {
   return;
 }
 ```
 
The transpiling result is:
 
 ```js
 private function a() : bool {
   return true;
 }
 ```

2. **Return in the middle of a function**

As mentioned earlier, internally BVM does not have jump instructions, thus sCrypt does not support returning in the middle of a function. If this happens in Solidity code, special handling is required. It works similarly to the previous `break` / `continue`, adding a boolean flag and `if` / `else` for equivalent transformation. For example, for Solidity code:

 
```solidity
function get(uint amount) internal view returns (uint) {
   if (amount > 0)
       return amount;
   return 0;
}
```
 
The transpiling result is:
 
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

#### `public` / `external` functions


Both types of functions can be called by sending a transaction, so we transplie them into sCrypt's `public` function. There are two special cases:


1. return value

The `public` function of sCrypt actually implicitly returns a boolean type, indicating whether a contract function call succeeds or not. When Solidity's `public` / `external` function has a return value, it is necessary to make a modification. That is, add a parameter of the original return value type, and verify that the passed value is equal to the original return value. Take this example:


```solidity
function get() external pure returns (uint) {
  return 1 + 1; 
}
```

The transpiling result is:

```js
public function get(int retVal) {
  require(1 + 1 == retVal);
}
```

Here we emphasize again: sCrypt's `public` function is to **verify whether the spending condition is satisfied**, not to return a certain result through direct computation.


2. propagate states


Another one that is not intuitively easy to understand is the propagation of [the state change of a contract](https://scryptdoc.readthedocs.io/en/latest/state.html). sCrypt's stateful contract is based on the UTXO model. Every time its `public` function is called, we need to ensure that the state is correctly passed to the new contract UTXO. This is why `public` functions transpilated from Solidity will always end with a `require(this.propagateState()));` statement. Plus, a common `SigHashPreimage` type parameter (i.e., transaction preimage) needs to be added to the parameter list of the function.

```solidity
function set(uint x) external {
    storedData = x;
}
```

The transpiling result is:

```js
public function set(int x, SigHashPreimage txPreimage) {
    this.storedData = x;
    require(this.propagateState(txPreimage));
}
``` 

#### Built-in object properties: `msg.sender` and `msg.value`

Two of the most commonly used built-in object properties in Solidity are: `msg.sender` and `msg.value`. The former returns the address of the current function caller, and the latter returns the amount of ether (in `wei`) carried in the transaction.


For `msg.sender`, we map it to the Bitcoin address of the caller of the public function. We add the corresponding `PubKey` type and `Sig` type parameters in the public function and add the following statements to the function, which make sure the contract is called by the said caller by verifying his signature:


```js
PubKeyHash msgSender = hash160(pubKey);
require(checkSig(sig, pubKey));
```


For `msg.value`, we map it to the amount of Bitcoin (in `satoshi`) deposited to the contract UTXO through this call. We add an `int` type parameter `msgValue` to the public method, requiring the caller to actively pass in its value as an argument. At the same time, through the [OP_PUSH_TX](https://xiaohuiliu.medium.com/op-push-tx-3d3d279174c1) technique, we can know the original balance of the contract. The new balance after the contract is called should be: `SigHash.value(txPreimage) + msgValue`. Finally, use the aforementioned `propagateState` method inside the function to verify that the balance in the new contract UTXO does indeed contain the expected increment.


```js
public function xxx(... int msgValue, SigHashPreimage txPreimage) {
 ...
 int contractBalance = SigHash.value(txPreimage) + msgValue; // add `msgValue` to contract balance
 require(msgValue >= 0); // basic validation
 ...
 require(this.propagateState(txPreimage, contractBalance)); // check the new balance is real
}
```

There is a special case worth noting here. If Solidity accesses `msg.value` in the constructor, it will automatically add an `initBalance` property to the contract during transpilation, which would be initialized as `msg.value` in the constructor. In addition, because the constructor of sCrypt cannot verify whether the value passed in as an argument is forged or not, it is necessary to verify that the initial value is correct in the first call of public methods. For example, for Solidity code:


```solidity
constructor (...) {
 a = msg.value;
 ...
}
```
 
The transpiling result is:
 
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

function checkInitBalance(SigHashPreimage txPreimage) : bool {
    return !Tx.isFirstCall(txPreimage) || SigHash.value(txPreimage) == this.initBalance;
}
```


## Limitations

Due to the fundamental difference between the account model and the UTXO model, the transpiler has certain limitations and cannot achieve 100% automatic conversion rate for arbitrary Solidity contracts.
At present, there are centain Solidity grammars that cannot be transpiled, including, but are not limited to, the following:


* destructuring assignment
* enum
* interface
* inheritance
* exception handling `try` / `catch`
* `modifier`
* `receive` / `fallback` functions
* some built-in methods and properties, such as `block.*` / `tx.*` / ...
* inter-contract calls
* event definitions and `emit()` function calls
* `assembly` statement

 
## Summary

As we mentioned at the beginning, the main function of this transpiler is to help developers quickly migrate from Solidity contracts to sCrypt contracts, so that they can use the Bitcoin blockchain to build Web3 applications with a more efficient and economic network. If you have any questions or suggestions, please join our sCrypt [slack](https://join.slack.com/t/scryptworkspace/shared_invite/enQtNzQ1OTMyNDk1ODU3LTJmYjE5MGNmNDZhYmYxZWM4ZGY2MTczM2NiNTIxYmFhNTVjNjE5MGYwY2UwNDYxMTQyNGU2NmFkNTY5MmI1MWM) discussion group or Github [repository](https://github.com/sCrypt-Inc/sol2scrypt)
