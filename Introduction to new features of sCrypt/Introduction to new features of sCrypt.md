# sCrypt 新功能介绍 (v1.9.0)

今天我们发布了 sCrypt IDE 的新版本 [v1.9.0](https://marketplace.visualstudio.com/items?itemName=bsv-scrypt.sCrypt)。 新版本支持在被导入的文件中执行 **REPL**， 同时带来了更加强大的 [内联汇编](https://scryptdoc.readthedocs.io/zh_CN/latest/asm.html) 语法。

## 在被导入的文件中执行 **REPL**

在之前版本的IDE，当你在调试sCrypt 合约时， 如果调试器是停在被导入的文件中，这个时候在 **REPL** 中执行表达式，会提示以下错误。

![unsupport_repl](./unsupport_repl.gif)

`util.scrypt` 是被导入的文件，当调试器停在这里时，在 **REPL** 中无法执行表达式。

新版本的IDE 解决了这个问题， 你可以在任何地方执行表达式。

![repl](./repl.gif)


## 内联汇编新语法

1. 现在支持在内联汇编中使用 `loop` 循环了，这在内联汇编出现大量逻辑相同的操作码时非常有用，它可以增加代码的可读性。

    ```js
    public function unlock(int x) {
        asm {
            OP_DUP
            loop (N) : i {
                loop (N) : j {
                    i
                    j
                    OP_ADD
                    OP_ADD
                }
            }
            $xxx
            OP_NUMEQUAL
            OP_NIP
        }
    }
    ```

    与之等价的sCrypt代码是： 

    ```js

    public function unlock(int x) {
        int sum = x;
        loop (N) : i {
            loop (N) : j {
                sum += (i + j);
            }
        }
        require(sum == 19);
    }

    ```

    `i` 和 `j` 是[归纳变量](https://scryptdoc.readthedocs.io/zh_CN/latest/loop.html#induction-variable)，
    `$xxx` 是[汇编变量](https://scryptdoc.readthedocs.io/zh_CN/latest/asm.html#assembly-variable)。

2. 支持在内联汇编中使用字符串字面量:

   ```js
   contract AsmString {
        static function equalImpl(bytes msg) : bool {
            asm {
                "你好world! 😊"
                OP_EQUAL
            }
        }
        public function unlock(bytes msg) {
            require(AsmString.equalImpl(msg));
        }
    }
   ```

欢迎体验新功能!

