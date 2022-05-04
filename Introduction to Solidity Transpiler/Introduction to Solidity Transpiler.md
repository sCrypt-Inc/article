# Introduction to Solidity Transpiler

sol2scrypt is a transpiler program. It can convert Solidity code into equivalent sCrypt code, which is convenient for developers to quickly learn or migrate code.

Before introducing the principle of the transpiling, let's review the main differences between Ethereum's account model and BSV's UTXO model:

- The account model of the Ethereum application actually saves an independent global state for each user or contract address, and triggers its state changes through contract calls. The advantage of this is that a globally unique address can be used for fast addressing and message delivery, and it is closer to the mental model of traditional database application developers, but its biggest disadvantage is the performance problem, which can only be processed serially.

- The UTXO model used by BSV actually maintains a set of UTXOs for each user or contract address, and uses the aggregated information of this set to represent the state of the user or contract. The advantage of this is that transactions can be processed as independently as possible, maximizing parallel processing capabilities, but the inability to use a single address for addressing and messaging makes writing certain contracts more difficult.

The original intention of the sol2scrypt transpiler is to implement an automatic conversion tool from Solidity smart contracts to sCrypt smart contracts. It provides a good starting point for developers who are less familiar with smart contracts based on the UTXO model. On the one hand, it allows developers to intuitively see different implementations of the same business logic in two languages, and on the other hand, it saves them from writing equivalent sCrypt contracts from scratch. Of course, due to the fundamental difference between the account model and the UTXO model mentioned above, this tool will also have certain limitations and cannot achieve 100% automatic conversion of any type of Solidity contract.

## Transpiling Principle

If you summarize the basic principle of the transpiling in one sentence: use a single UTXO to simulate a single contract account of Ethereum, and perform the equivalent transformation of the contract code on this premise. Next, we take the different life cycles of the contract as observation points, and briefly introduce the mapping principle.

### Contract Deployment

First use a single UTXO to map the initial state of a contract instance in the EVM;

### Contract Call

Utilize [Stateful Smart Contracts on Bitcoin SV](https://medium.com/coinmonks/stateful-smart-contracts-on-bitcoin-sv-c24f83a0f783) to map those of the contract on the EVM that change the contract's internal state method calls. Whenever the internal state of the contract needs to be changed, a new transaction is generated that spends the current UTXO of the contract and generates a UTXO with the new state, and so on.

### Contract Destruction

The destruction of the contract instance of Ethereum needs to call the selfdestruct method to mark its internal state as invalid, and on BSV we only need to spend the current contract UTXO of the contract and no longer generate a new contract UTXO.

The above is the basic principle of the entire transpiling. It should be noted that here is only the most simple and direct transpiling idea. We only use a single UTXO to map a single contract. In fact, for a specific Solidity contract, there may be other transpiling methods that are more complex but can improve parallel performance, but this may require more manual design by developers.

## Transpiling of Syntax

 
Let's take a look at some specific syntax transpiling details.
 

### Contract State Variables
 
Since Solidity's contract state variables (state variables) are the specific carriers of the contract state, they may change as the contract is called, so they will all be transpiled into sCrypt's stateful contract properties (`@state` contract properties)

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

 
#### 1. Basic Data Type

Solidity's basic data types include `bool` / (`int` / `uint` series) / (`bytes` series / `string`) / `address`, these basic types will be directly transpiled into `bool` in sCrypt language / `int` / `bytes` / `PubKeyHash` type.

#### 2. Structure Type

Solidity's structure `struct` will be transpiled into sCrypt structure, and all properties contained in it will also be transpiled into the corresponding sCrypt type. However, if a structure contains properties of contract type, it cannot be directly transpiled.
 
#### 3. Array Type

Solidiy's array type is also directly transpiled into sCrypt arrays, but there is a limitation here: dynamic-length arrays cannot be transpiled, because sCrypt's native arrays must be fixed-length.
 
#### 4. Mapping Type

The `mapping` type in Solidiy is also a relatively common data structure, it will be transpiled into the `HashedMap` data type of sCrypt, but this transpilation process is special.

First of all, Solidiy's `mapping` is a common hash table implementation. It supports the use of basic types as keys, arbitrary types as values, and supports operations such as addition, deletion, modification, and search. But the `HashedMap` in sCrypt is not such an ordinary hash table implementation, it has several very different characteristics:


1. What is stored in `HashedMap` is not the original `key` and `value` values in the `mapping` data structure, but their respective hash values, which can be understood as the hash of the entire `mapping` structure result. Or to put it another way, it's a credential that ensures the key-value pair exists in the `mapping`.

2. When the program needs to use one of the key-value pairs, it first needs to pass in the specific value from the external parameter, and then use `HashedMap` to verify it. If the verification pass indicates that the incoming parameters are legal, then in the following program, you can safely use them for operation; otherwise, it indicates that this is an invalid key-value pair, and the program should not continue to execute.


**The main reason why `HashedMap` uses this design is to consider the limit of the script size and the number of loops must be constant. For more instructions, please refer to [this article](https://blog.csdn.net/freedomhero/article/details/121395939)**.

 
This verification-based model is also a distinguishing feature of BSV blockchain smart contracts different from other smart contracts.
 
Suppose we already have an instance `mapping(uint256 => uint256) m;`, the read and write access to this instance will be transformed as follows:
 
1. Insert the real `val` value and an `index` value in the input parameter of the function (this is the index value of `key` after all key values are sorted);

2. Verify that this `val` is indeed the value corresponding to `key`;
 
3. Replace the access expression with the incoming `val` parameter;
 
4. If it is a write operation, the update to `HashedMap` needs to be inserted at the end;
 
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

In addition, when dealing with nested `mapping` types, such as `mapping (address => mapping (address => uint)) `, the nested `keys` are defined as a sCrypt structure, and use it as the key type of the `HashedMap`. for example:
 
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
 
* `key` cannot be reassigned within a function, otherwise there may be unexpected results;
 
 ```solidity
 a = m[key];
 key = xxx;
 m[key] = b;
 ```
 
* `mapping` of nested types cannot be **partially read and written**, e.g. `nm[a] = anotherMapping;`

### Operators


Most of Solidity's operators can be directly transpiled into the same operators in sCrypt, such as the common `+`, `-`, `*`, `/` and so on. But there are also operators that do not support transpilation, such as the exponentiation operator `**`.

In addition, be very careful when transpiling bitwise operators, because integers on BSV are encoded in little endian, and negative numbers are not encoded in Solidity's two's complement form. So although the transpiled expressions have the same form, the execution result may not be the same.

### Conditional Statements
 
sCrypt also supports `if` / `else` statements, which can be directly transpiled.
 
### Loop Statements


Solidity supports three common loop statements, namely `for` / `while` / `do ... while`, the bottom layer is implemented by jump instructions. However, since there is no jump instruction in the underlying opcode of the BVM virtual machine, the above loop statement cannot be directly implemented. And sCrypt can implement a constant number of loop statements by introducing the `loop` statement. Of course this also requires the number of loops to be a compile-time constant.


So when transpling Solidiy's loop statements, we uniformly use the `loop` statement for mapping. Since the specific number of loops is closely related to the program logic, the transpiler cannot automatically give a reasonable value, so a placeholder variable such as `__LoopCount__0` is used instead. This requires developers to manually modify the transpiled sCrypt code and replace it with a reasonable value, otherwise the transpiled sCrypt contract cannot be compiled. The number of cycles can generally be deduced according to the gas limit.


#### `for` Statement

For example, for the following Solidity code `for` loop:
 
```solidity
uint sum = 0;
for(uint i=0; i<n2; i++) {
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
 
`do ... while` is slightly different in that it is executed at least once, so the corresponding Solidity code:
 
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


As we just mentioned, there is no direct jump in sCrypt, so the first instinct is that we cannot transplie the `break` or `continue` statements that are common in loops. But in fact, we can still combine the conditional statements `if`/`else` to implement these two logics, but need to add a boolean type `FLAG`.
 
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

Similar to how `break` is handled, the only thing to note is that `continue` just skips the following statements within this loop iteration; `break` skips all remaining loop iterations after that.
 
Please see this example:
 
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


Functions as the interface for interacting with smart contracts are undoubtedly another focus of transpilation, and due to the differences between EVM and BVM, the transpilation method will not be as intuitive and easy to understand as some of the syntax units introduced earlier. Next, we will introduce the relevant transpilation principles in detail.

 
The first thing to note is that Solidity functions have 4 types of visibility:
 
* `private`: can only be called by the contract itself;
* `internal`: Only this contract and its subclass contracts can be called;
* `external`: can only be called by sending a message from the outside (including sending TX);
* `public`: can be called either in `internal` or `external`;


Depending on whether they can be called by sending Tx, they can be divided into two categories: `external` and `public` are allowed, while `private` and `internal` are not. According to this standard, we transplate the former into the `public` function of sCrypt, which is the external interface of the contract; and the latter into the `private` and `default` functions, which are the specific implementation inside the contract. For details, see:
 
| Solidity | sCrypt |
| :----: | :----: |
| `private` | `private` |
| `internal` | `default` |
| `external` | `public` |
| `public` | `public` |

#### `private` / `internal` functions


Since these two types of functions are only called inside the contract, considering the complexity of the transpiler implementation, it is agreed that they must satisfy a constraint: that is, the number of parameters cannot be changed when transpling the function body, which will lead to changes in the function signature. An example of function transpilation:


```solidity
function set(uint x) external {
    storedData = x;
}

function get() internal view returns (uint) {
    return storedData;
}
```

The transpiling result is:

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

#### `return` statement in `private` / `internal` functions

**Note here: Special handling may be required for `return` statements in `private` / `internal` functions.** Specifically:
 
1. **without return value**


Since Solidity allows function without return value but sCrypt does not. it will be transpilated into return a default `bool` value in this case. For example, the following Solidity code:
 
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

2. **Return in middle of the function**

As mentioned earlier, the bottom layer of BVM does not have jump instructions, so sCrypt does not support returning in the middle of a function. If this happens in Solidity code, special handling is required. Of course, the principle is also very simple, similar to the previous `break` / `continue` processing method, adding FLAG and `if` / `else` for equivalent transformation. For example for Solidity code:

 
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


Both types of functions can be called by sending Tx, so we transplate them into sCrypt's `public` function. But there are two more complicated cases to be aware of:


1. return value

The `public` function of sCrypt actually implicitly returns a boolean type, indicating whether the contract function call succeeded or not. When Solidity's `public` / `external` function has a return value, it is necessary to make a transformation on the function during transpilation: that is, add a parameter of the original return value type, and verify that this value is equal to the original return value. Take this example:


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

Here we emphasize again: the function of sCrypt's `public` function is to **verify whether the unlock condition is satisfied**, not to give a certain result through calculation.


2. propagate states


Another one that is not intuitively easy to understand is the transmission of [the state change of the contract](https://scryptdoc.readthedocs.io/en/latest/state.html). We also mentioned before that sCrypt's stateful contract is based on the UTXO model, and every time its `public` method is called, we need to ensure that the state is correctly passed to the new contract UTXO. This is why `public` functions transpilated from Solidity will basically end with a `require(this.propagateState(...)));` statement. Of course, a common `SigHashPreimage` type parameter (ie transaction preimage) needs to be added to the parameter list of the function.

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

Two of the more commonly used built-in object properties in Solidity functions are: `msg.sender` and `msg.value`. The former returns the address of the current function caller; the latter returns the amount of ether (unit: `wei`) carried in the message.


When transpiling, we map the two to the caller address of the public function and the amount of BSV (unit: `satoshi`) added to the contract UTXO itself through this call.


For `msg.sender`, we add the corresponding `PubKey` type and `Sig` type parameters in the public method, and add the following statement to the function, which make sure the contract method is called by verifing the validity of the signature and ensure that the address is processed:


```js
PubKeyHash msgSender = hash160(pubKey);
require(checkSig(sig, pubKey));
```


For `msg.value`, we add an `int` type parameter `msgValue` to the public method, requiring the caller to actively pass in its value as a parameter. At the same time, through the [OP_PUSH_TX](https://xiaohuiliu.medium.com/op-push-tx-3d3d279174c1) technology, you can know the original locked balance of the contract. The new balance after the contract is unlocked should be: `SigHash.value(txPreimage) + msgValue`. Finally, use the aforementioned `propagateState` method inside the function to verify that the balance in the new contract UTXO does indeed contain the increment brought by this value.


```js
public function xxx(... int msgValue, SigHashPreimage txPreimage) {
 ...
 int contractBalance = SigHash.value(txPreimage) + msgValue; // add `msgValue` to contract balance
 require(msgValue >= 0); // basic validation
 ...
 require(this.propagateState(txPreimage, contractBalance)); // check the new balance is real
}
```

There is a special case to note here. If Solidity accesses `msg.value` in the constructor, it will automatically add an `initBalance` property to the contract during transpilation, and use the `msg.value` pair in the constructor its assignment. In addition, because the constructor of sCrypt is special, it cannot verify whether the value passed in as a parameter is forged or not, so it is necessary to verify that the initial value is correct in other public methods. For example, for Solidity code:


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
```


## Other limitations

At present, the transpiler has some other Solidity grammars that cannot be transpiled normally, mainly including the following:


* Tuples and multiple assignment expressions are not supported;
* does not support enumeration;
* Interface is not supported;
* Does not support interface inheritance;
* Does not support exception handling mechanism `try` / `catch`;
* The use of `modifier` is not supported;
* The use of `receive` / `fallback` functions is not supported;
* The use of other built-in methods and properties, such as `block.*` / `tx.*` / ... is not supported;
* Inter-contract calls are not supported;
* Ignore all event definitions and `emit()` function calls;
* `assembly` statement is not supported;

 
## Summarize

As we mentioned at the beginning, the main function of this transpiler is to help developers quickly transpilation from Solidity contracts to sCrypt contracts, so that they can use the BSV blockchain to build Web3 applications with a more efficient and low-cost network. I hope it can help everyone. If you have any questions or suggestions, please join our sCrypt [slack](https://join.slack.com/t/scryptworkspace/shared_invite/enQtNzQ1OTMyNDk1ODU3LTJmYjE5MGNmNDZhYmYxZWM4ZGY2MTczM2NiNTIxYmFhNTVjNjE5MGYwY2UwNDYxMTQyNGU2NmFkNTY5MmI1MWM) discussion group or github [repository](https://github.com/sCrypt-Inc/sol2scrypt)
