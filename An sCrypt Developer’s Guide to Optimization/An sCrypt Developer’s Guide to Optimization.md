# sCrypt开发人员优化指南

## 第一部分：手动优化

sCrypt编程与使用Javascript或Python进行的传统编程不同，因为当将封装它的交易提交给比特币网络时，编译后的脚本大小直接决定了它们的运行成本。因此，最终的脚本应该尽可能小，以节省交易费用¹。下面我们列出了一些技巧，供开发人员手动优化其sCrypt合约的脚本输出。

![logo](logo.png)

### 1. 合并相同的函数调用

所有函数调用均通过内联实现。函数主体会被复制到调用它的位置。如果同一个函数在不同的地方多次调用，则可以通过合并它们来节省代码大小。在下面的示例中，将两个 `Tx.checkPreimage()` 调用合并为一个。

```javascript

// 优化前
contract Unoptimized {
    public function unlock0(SigHashPreimage txPreimage, bool arg1) {
        require(Tx.checkPreimage(txPreimage));
        
        require(arg1);
    }

    public function unlock1(SigHashPreimage txPreimage, int arg2) {
        require(Tx.checkPreimage(txPreimage));
        
        require(arg2 == 0);
    }
}

// 优化后
contract Optimized {
    // idx is 0 when calling the original unlock0, unlock1 otherwise
    public function unlock(SigHashPreimage txPreimage, int idx, bool arg1, int arg2) {
        require(Tx.checkPreimage(txPreimage));
        
        require(idx == 0 ? arg1 : arg2 == 0);
    }
}

```

### 2. 嵌入原始脚本

[内联汇编](https://scryptdoc.readthedocs.io/en/latest/asm.html)允许将原始比特币脚本直接嵌入sCrypt代码中，以实现更细粒度的控制。如果您知道自定义脚本比sCrypt生成的脚本短，则可以改用它。


```javascript


// 优化前
contract P2PKH {
  Ripemd160 pubKeyHash;

  public function unlock(Sig sig, PubKey pubKey) {
      require(hash160(pubKey) == this.pubKeyHash);
      require(checkSig(sig, pubKey));
  }
}

// 优化后
contract P2PKH {
  public function unlock(Sig sig, PubKey pubKey) {
      asm {
        op_dup
        op_hash160
        $pubKeyHash
        op_equalverify
        op_checksig
      }
  }
}

```

### 3. 使用隐式/默认构造函数

构造函数通常仅使用其参数来初始化每个属性。如果是这种情况，请使用默认构造函数以优化脚本大小。

```javascript

// 优化前
contract Unoptimized {
    int x1;
    bytes x2;
    bool x3;

    constructor(int x1, bytes x2, bool x3) {
        this.x1 = x1;
        this.x2 = x2;
        this.x3 = x3;
    }

    public function equal(int y) {
      // TODO
    }
}

// 优化后
contract Optimized {
    int x1;
    bytes x2;
    bool x3;

    public function equal(int y) {
      // TODO
    }
}

```

### 4. 避免写变量

[比特币虚拟机](https://blog.csdn.net/freedomhero/article/details/106801904)（BVM）完全在堆栈上运行。与许多其他虚拟机不同，它没有其他类型的内存，例如寄存器或RAM，因此其它虚拟机在内存上写入变量的复杂度是O(1)，而比特币虚拟机写入变量的复杂度是用O(n)，其中n是变量在堆栈中的深度²。

```javascript

// 优化前
function unoptimized(bool flag): int {
  int i = 0;
  
  if (flag) {
    i = -1;
  } else {
    i = 1;
  }

  // ...
}


// 优化后
function optimized(bool flag): int {
  int i = flag ? -1 : 1;

  // ...
}

```

sCrypt合约的静态属性是一种特殊的情况。他们存储在堆栈的底部，因此更改他们的成本特别高。如果确实需要更改它们，请检查看是否可以将它们设置为非静态属性。

### 5. 在循环中使用归纳变量

循环中，使用[归纳变量](https://scryptdoc.readthedocs.io/en/latest/loop.html#induction-variable)比使用递增的常规变量更加优化。

```

// 优化前
int i = 0;
loop (10) {
  // use i
  
  i++;
}

// 优化后
loop (10) : i {
  // use i
}

```

[1] 当前，交易费用仅与脚本/事务的大小成正比，未来会将演变为考虑脚本的复杂性。因此，优化目标也有望在未来发展演变。

[2] 读取变量始终为O(1)。


