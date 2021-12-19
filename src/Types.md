# 基本类型

lua 有八种基本类型，而且 `nil`、`Boolean`、`userdata`
类型在 [入门#类型](./Getting-Started.md#类型) 部分介绍完了。

此部分介绍其余几种。

## `numbers`

`numbers` 类型分为两类[^numbers]：
1. 整数 (integers)：64 位整数。
2. 浮点型 (floats)：双精度浮点。

[^numbers]: 这个分类是 lua 5.3 之后的情况（在某些平台上，你可以编译成 32 位整数和单精度浮点），
lua 5.2 之前的版本的 numbers 只有单精度浮点型。可以称整数和浮点数为 `numbers` 的子类型 (subtype)。

### 表示方法

1. 十进制整数：0、1、-1 等等。
2. 带小数点的浮点数：`0.`、`1.2`、`-1.3` 等等。
3. 科学计数法（`e` 或 `E`，背后是浮点数）：`1E2`、`1E+2`、`1E-2`、`1e2`、`1e+2`、`1e-2` 等等。
4. 十六进制整数（`0x` 或 `0X` 开头）：`0xff == 16*15+15 --> 255`、`0x1A3 == 16^2*1+16*10+3 --> 419` 等等。
5. 十六进制浮点数（`0x` 或 
   `0X` 开头，带小数位）：`0x0.2 == 2/16 --> 0.125`、`0xa.25 == 10+2/16^1+5/16^2 --> 10.14453125` 等等。
6. 十六进制+二进制的浮点数（`0x` 或 `0X` 开头，带 `p` 或 `P`，其中 `p` 表示 
   `2.^`[^p]）：`0x1p1 == 1*2.^1 --> 2.0`、`0xap-1 == 10*2.^-1 --> 5.0` 等等。
7. 十六进制浮点数（结合小数位和二进制）：`0xa.bp+3 == (10+11/16)*2.^3 --> 85.5` 等等。

但是整数和浮点数在 lua 中都使用 `numbers` 类型，因为两者往往相互转化。

从而，当浮点数可以转化成整数时，两者相等：`0. == 0 --> true`、`1e2 == 100 --> true`。

当你真的需要区分这两类 `numbers`，则使用 `math.type` 函数。

使用 `string.format` 和 `%a` 将一个数字转化为十六进制浮点数表示：
```lua
string.format("%a", 419) --> 0x1.a3p+8 即 0x1A3
string.format("%a", 0.1) --> 0x1.999999999999ap-4
```
虽然这种形式不是很好阅读，但是它保存了任意浮点数的所有精度，而且比基于十进制的转化快。

[^p]: 这从 lua 5.2 之后才被引进。

### 算术运算符

| 算术运算符                       | 计算结果的类型                                              |
|----------------------------------|-------------------------------------------------------------|
| `+`、`-`、`*`、`//`、`%`         | 当两个运算对象都是整数时，结果为整数；否则为浮点数          |
| `/`、`^`                         | 浮点数（因为两个整数相除或幂的结果不一定是整数）            |
| `>`、`<`、`==`、`~=`、`>=`、`<=` | `Boolean`（整数可以和浮点数比较，但同一子类型比较效率更高） |

- `//` (floor division) 的结果总是把商的结果向负半轴方向取整，比如 
  `3 // 2 --> 1`、`-9 // 2 --> -5.0`。此运算符从 lua 5.3 之后被引入。类似于
  `math.floor(-9/2)` 但 `//` 的类型可能为浮点数。
- `%` (modulo) 具有以下性质： `a % b == a - ((a // b) * b)`。其结果总是与第二个运算对象正负号相同。当
  `b` 大于 1 时，其结果总是在 `[0, b-1]` 范围内。

### 数学库运算

lua 5.3 的[数学函数 API](http://www.lua.org/manual/5.3/manual.html#6.7)。

| 函数        | 返回值                                                                                                   |
|-------------|----------------------------------------------------------------------------------------------------------|
| math.random | 无参数时为 `[0,1)` 内的浮点数；一个正整数时为 `[1,n]` 内的整数；两个正整数 m 和 n 时为 `[m, n]` 内的整数 |
| math.floor  | 向负无穷方向取整；当参数为整数时，直接返回整数                                                           |
| math.ceil   | 向正无穷方向取整；当参数为整数时，直接返回整数                                                           |
| math.modf   | 向 0 方向取整；当参数为整数时，直接返回整数；此外，它还返回小数部分，如 `math.modf(-3.3) --> -3 -0.3`    |

- 对 `x` 四舍五入法取整：
	1. 方法一：`math.floor(x + 0.5)`。当数字足够大时（比如 `2^52 + 1`），这个方法会出现问题。
	```lua
	-- 对 1 四舍五入取整为 1
	math.floor(1)    --> 1
	math.floor(1+.5) --> 1
	-- 对 2^52 + 1 四舍五入取整应该为自己
	2^52+1 == 4503599627370497 --> true
	math.floor(2^52+1)   --> 4503599627370497
	math.floor(2^52+1.5) --> 4503599627370498 因为双精度无法正确表示 2^52+1.5
	```
	2. 方法二：对方法一进行完善，当参数为整数时，直接返回整数，不 `+.5`
	```lua
	function round (x)
	  local f = math.floor(x)
	  if x == f then return f
	  else return math.floor(x + 0.5)
	  end
	end
	```

- 对 `x` 进行双数取整 (unbiased rounding、round-to-even)：
```lua
function round (x)
  local f = math.floor(x)
  if (x == f) or (x % 2.0 == 0.5) then
    return f
  else
    return math.floor(x + 0.5)
  end
end
```

### 范围和精度

详细论述见 `Programming in Lua#Representation Limits` 部分。一些要点：


| number | 范围或精度（64 位）                                                           |
|--------|-------------------------------------------------------------------------------|
| 整数   | -2^63 ~ 2^63-1，大约为 -10^19 ~ 10^19                                         |
| 浮点数 | 大约 -10^308 ~ 10^308，精确到小数后 16 位，能精确表示的整数范围 [-2^53, 2^53] |

一些极端例子：
```lua
math.maxinteger    -->  9223372036854775807
0x7fffffffffffffff -->  9223372036854775807
math.mininteger    --> -9223372036854775808
0x8000000000000000 --> -9223372036854775808

math.maxinteger + 1 == math.mininteger   --> true
math.mininteger - 1 == math.maxinteger   --> true
-math.mininteger == math.mininteger      --> true
math.mininteger // -1 == math.mininteger --> true

math.maxinteger + 2   --> -9223372036854775807
math.maxinteger + 2.0 --> 9.2233720368548e+18
math.maxinteger + 2.0 == math.maxinteger + 1.0 --> true
```

### 浮点和整数相互转换

- 整数转成浮点：
  - 让整数与 `0.0` 相加。注意：此方法针对超过 2^53 的整数，会导致相加的结果以确定的双精度浮点数的形式表示，
    从而 `9007199254740993 + 0.0 == 9007199254740993 --> false`。
- 浮点转成整数（只在浮点数能表示成整数时）：
  - 让整数与 `0` 或运算。如 `2^1 | 0 --> 2`。注意：这在无法转换时出现错误。
  - 使用 `math.tointeger` 函数。当无法转换时，返回 `nil`。如
    `math.tointeger(-1.0) --> -1`、`math.tointeger(2^64) --> nil`

## `string`

### 基础要点

- lua 中的 `string` 是字节序列，存储任何二进制数据，不局限于文本、编码。
- `string` 是不可变的值 (immutable values)，修改值就是创建新的 `string`。
- lua 的所有对象是自动内存管理的，包括 `string` —— 无需担心分配和重新分配的问题。
- 基本操作（字节操作，不局限于字符编码）：
    - **字节**长度 `#`：如 `#"a" --> 1`、`b = '中'; print(#b) --> 3`
    - 拼接 `..`：如 `"Hello " .. "World" --> Hello World`、`a = "a"; print(a..0) --> a0`、`'1'..3 --> 13`、`3 .. 5 --> 35`
- 字面字符串 (literal strings)：
    - 使用单引号 `'` 或双引号 `"`：`'a'`、`"B"`、`'"a"'`、`"'B'"`、`'"a"' == "\"a\"" --> true`\
      两种写法是等价的，区别在于使用一种引号的字面值无需转义另一种引号。
    - 支持的转义符号：`\a`、`\b`、`\f`、`\n`、`\r`、`\t`、`\v`、`\\`、`\"`、`\'`
    - 转义序列：
        - `\ddd` 三位十进制数：如 `"\065" --> A`、`"\122" --> z`
        - `\xhh` 两位十六进制数：如 `"\x41" --> A`、`"\x7A" --> z`
        - `\u{h... h}` 表示 UTF-8 字符：如 `"\u{4E2D}" --> 中`
    - 忽略连续空白字符 (white-space characters) `\z`：空白字符包括 `' '`、`'\r'` `'\x08'` 
      之类的符号，但不包括 `\n`。即 `"\z\r \na" --> 输出等价于 "\na"`。
    - 多行文本（这两个功能对[注释](Getting-Started.html#注释符号)也生效）：
        - 使用 `[[` 和 `]]`：

            ```lua
            page = [[
                multi lines [1]
            ]]
            ```

            会识别为

            ```text
                multi lines [1]

            ```

        - 使用 `[=[` 和 `]=]`，其中的等号数量任意，但两边的等号数量必须相等。
          上面的方法无法输入 `[[` 或 `]]`（因为它们是开始和结束的符号），所以添加等号来增强标识。
            
            ```
            page = [=[
            [[[[[[[[[[[[[[[]]]]]]]]]]]]]]=]
            ```

          这段代码可以连续输入任意多的括号，以及中间携带任意多等号的双括号（一个等号除外）。 
          原因相同，lua 会按照开始和结束的符号确定长文本，唯一的限制是文本不能包含开始或结束的符号。


### 强制转换

- 在算数操作符和预计传入数字的地方，`string` 会尝试转化成 `number`。
- 同理，在预计需要字符串的地方，`number` 会转化成 `string`。
- 注意，`1 .. 2` 的结果是 `"12"`，而且 `..` 前面为数字时，必须用空格隔开，因为 lua 会把 `.` 视为数字的一部分。
- 字符串和数字的算数操作中，字符串总是转成浮点型：如 `'1'+1 --> 2.0`。
- 显式把字符串转成整数，使用 `tonumber` 函数。此函数：
    - 在一个参数时，接收任何[有效的数字表示](#表示方法)。
    - 在两个参数时，第二个参数为 2 到 36 的进制方式，且输入的数字不是该进制的正确表示，那么返回 `nil`。
  
    ```lua
    tonumber("0xa")        --> 10
    tonumber("100101", 10) --> 100101
    tonumber("100101", 2)  --> 37
    tonumber("fff", 16)    --> 4095
    tonumber("-ZZ", 36)    --> -1295
    tonumber("987", 8)     --> nil 八进制不含 8 和 9
    tonumber("0xa", 10)    --> nil 十进制不含 a
    ```
- 显式把数字转化成字符串，使用 `tostring` 函数。如 
  `tostring(0) --> "0"`、`tostring(0xa) --> 10`。这个转换总是成功的，但指定数字的呈现格式，须使用
  `string.format`。
- 数字和字符串之间无法比较大小。字符串与字符串之间按照字母表顺序比较大小 (alphabetical order)。如 
  `1>"1" --> 错误`、`"1" < "2" --> true`、`'12' < 'a' --> true`
- 根据 [`Boolean`](Getting-Started.html#boolean) 的运算规则，由于数字和字符串类型不同，当比较相等时，总为
  `false`：如 `"1" == 1 --> false`、`'0' ~= 0 --> true`。

### 标准库函数

在标准库之外，lua 在 `string` 方面只提供创建字面值、拼接、比较、获取字节长度的功能。

而标准库的 `string` 相关函数也只提供基础的功能且假设字符串是单字节的。lua 5.3 之后才引入处理 UTF-8 的功能（见下节）。

几乎所有返回 `string` 的函数都返回新 `string`，lua **貌似**没有切片（视图）的概念。

lua 的索引从 1 开始，而且允许负数索引：-1 表示倒数第一个字节，-2 表示倒数第二个字节，等等。索引区间一般为两边闭区间。

几乎所有的 `string.xx` 函数都可以写成 `s:xx` 的形式，其中 `s` 是变量名，如 `a = "123"; print(a:reverse()) --> 321`。

| 函数名                   | 功能（返回值）                   | 附注                                                  |
|--------------------------|----------------------------------|-------------------------------------------------------|
| string.len(s)            | s 的字节长度                     | 等价于 `#s`                                           |
| string.rep(s, n)         | 把 s 重复 n 次                   | 如 `string.rep("a", 2^20)` 表示创建 1 MB 的 `a` 字符  |
| string.reverse(s)        | 把 s 字节反转                    |                                                       |
| string.lower(s)          | 转化成大写                       |                                                       |
| string.upper(s)          | 转化成小写                       |                                                       |
| string.sub(s, i, j)      | 获取 `[i, j]` 区间的字节         | 返回新 `string`；省略 `j` 表示 `[i, -1]`              |
| string.char(n1)          | 把数字转成对应的字符             | 支持多个数字，如 `string.char(97, 98, 99) --> "abc"`  |
| string.byte(s, i, j)     | 把 `[i, j]` 区间的字节转成数字   | 省略 `j` 表示 `[i, i]`；省略 `i` 和 `j` 表示 `[1, 1]` |
| string.format(..., s)    | 格式化字符串，参考 C 的 `printf` | 比如 `%d`、`%x`、`%f` 分别表示 十进制、十六进制、浮点 |
| string.find(s, pattern)  | 查找模式，返回索引区间或 nil     | 如 `string.find("hello world", "wor") --> 7 9`        |
| string.gsub(s, pat, tar) | 返回替换之后的字节和次数         | 如 `string.gsub("hello", "l", ".") --> he..o 2`       |

### 处理 UTF-8

UTF-8 的 `string` 适用于 `#` 和 `..` 操作，也适用于比较大小操作（按照 Unicode 代码点顺序），但不适用于假定为单字节操作的 
`string.xx` 函数。

以下 `utf8.xx` 函数仅被 lua 5.3 之后的版本支持，`string.xx` 函数不受 lua 5.3 影响。

| 函数名                   | 功能（返回值）                       | 附注                                                    |
|--------------------------|--------------------------------------|---------------------------------------------------------|
| utf8.len(s)              | s 的字节长度                         | 等价于 `#s`                                             |
| utf8.lower(s)            | 转化成大写                           |                                                         |
| utf8.upper(s)            | 转化成小写                           |                                                         |
| utf8.offset(s,n)         | 获取第 n 个 Unicode 的字节索引       | n 可以为负整数，表示倒数第几个字符                      |
| utf8.char(n1)            | 把数字转成对应的字符                 | 支持多个数字，如 `string.char(97, 98, 99) --> "abc"`    |
| utf8.codepoint(s, i, j)  | 把 `[i, j]` 字节区间内容的转成代码点 | 类似于 `string.byte`                                    |
| utf8.codes(s)            | 返回 UTF-8 字符迭代器                | 如 `for index, c in utf8.codes(s)` c 为字节的数字表示   |
| string.rep(s, n)         | 把 s 重复 n 次                       |                                                         |
| string.sub(s, i, j)      | 获取 `[i, j]` 区间的字节             | 如 `string.sub(s, utf8.offset(s, -2))` 获取最后两个字符 |
| string.format(..., s)    | 格式化字符串，参考 C 的 `printf`     | 无法使用 `%c`                                           |
| string.find(s, pattern)  | 查找模式，返回索引区间或 nil         |                                                         |
| string.gsub(s, pat, tar) | 返回替换之后的字节和次数             |                                                         |

UTF-8 编码让每个 Unicode 字符的字节数长度不固定。它让 ASCII 范围内的字符保持单字节，让大部分中文字符为 3 字节长度。具体来说：
- 小于 128 的单个字节与 ASCII 相同；
- 非单字节的首个字节范围为 [194, 244]，其后的字节位于 [128, 191]；
- 两字节字符的首个字节范围为 [194, 223]
- 三字节字符的首个字节范围为 [224, 239]
- 四字节字符的首个字节范围为 [240, 244]

lua 自身不提供复杂的 `string` 处理。不同版本对字符串的处理也不同，参考 
[5.1](https://www.lua.org/manual/5.1/manual.html#5.4)、[5.3](https://www.lua.org/manual/5.3/manual.html#6.4) 手册。

## `table`

### 基础要点
- `table` 是 lua 最主要（唯一）的数据结构机制，可以作为包、模块、数组使用：比如 `math.sin` 是一个 `table`。
- `table` 在 lua 中既不是值 (value)，也不是变量 (variable)，而是对象 (object)。
- `table` 通过构造表达式 (constructor expression) `{}` 创建（初始化）一个表：
    - `a = {}` 创建一个空表
    - `a = {x = 0, y = 0}` 等价于 `a = {["x"] = 0, ["y"] = 0}` 等价于 `a = {}; a.x = 0; a.y = 0;`
    - `a = {"1", 2}` 等价于 `a = {[2] = "1", [2] = 2}` 等价于 `a = {}; a[1] = "1"; a[2] = 2;`
    - 最后的逗号是可选的：`a = {0}` 等价于 `a = {0,}` 
    - 所有 `,` 都可以换成 `;`，两种符号没有区别：`a = {1, 2}` 等价于 `a = {1; 2}`
    - 带字段的语法可以和无字段的语法并存：如 `a = {1, b = 2}`，则 `a[1] --> 1; a.b -->2`
    - 表可以嵌套表：比如 `a = {1}; b = {a}`，则 `b[1][1] --> 1` 又比如 `a = {x = {y = 1}}` 则 `a.x.y --> 1`
    - 表可以存放函数：如 `a = {f = math.floor}; a.f(0.5) --> 0`
- `table` 总是匿名的，变量并不拥有 `table`：`a = {}; b = a` 意味着 a 与 b 指向同一个表，通过任何一个指针修改都会同步修改。
- 可以给变量赋值为 `nil` 来减少 `table` 的引用，当一个表不再有引用时，垃圾回收器最终会删除表，重新使用该块内存：
    - `a = {1}; a = nil` 让 `{1}` 在只有一个引用的情况下变成了无引用；
    - `a = {1}; b = {a}; a = nil` 让 `{1}` 在两个引用的情况下减少一个引用，`b[1][1]` 依然有效。
- lua 程序管理 `table` 的指针（引用），但不隐式复制出新的 `table`。
- `table` 使用索引存储数据，语法为 `table[index]`：
    - 索引的类型并不唯一，可以是字符串、数字等；
    - 无字段初始化的表的索引从 1 开始：如 `a = {"1", 2}` 使用 `a[1] --> "1"`、`a[2] --> 2` 的方式获取数据；
    - 索引未初始化的部分时，返回 `nil`；
    - 通过赋值为 `nil` 来删除表的索引（或者叫做字段、键）：如 `a = {1}; a[1] = nil`；
    - 索引语法糖：**`a.name` 是 `a["name"]` 的语法糖**，而不是 `a[name]` 的语法糖；
    - 不支持 `a.1` 之类的语法糖来进行 `a["1"]` 操作；
    - 由于整数和浮点数都是 `number` 类型，所以 `a[0]` 和 `a[0.]` 是一样的；
    - 索引不同的类型，得到不同的数据：`a[0]` 和 `a["0"]` 是不一样的；
    - 不能在 `nil` 索引上存储数据： `a[nil] = 1` 或者 `b = nil; a[b] = 1` 是不允许的；

### 数组、列表、序列操作

- `table` 无法声明长度（大小）。
- 使用 `#` 符号获取 `table` 的“长度”。长度在 `table` 里是一个不好定义的事物。出于历史原因，可以使用 `n` 字段存放长度信息。
- 带正整数索引的 `table` 被称为数组 (array) 或者列表 (list)。
- 如果一个 `table` 不仅带正整数索引，还带其他类型的索引，那么带正整数索引的那部分被称为列表。
- 由于 `table` 可以有不连续的键（索引），比如 `d = {1, 2}; d[5] = 5` 在 [3, 4] 的值为 `nil`，此时这种中间有
  `nil` 元素的 `table` 被认为有一个洞 (hole)。
- 在 lua 看来，值为 `nil` 的字段和未初始化的字段没有区别：对于 `a = {1, nil, 3}`，`a[2]` 和 `a[4]` 一样。

序列 (sequence)：
- 定义：对于正整数 n，如果表的 `{1, ..., n}` 索引存储的值都不是 `nil`，那么称表的 `{1, ..., n}` 的部分叫做序列。
- 无 `{1, ..., n}` 数值索引的表，被称为长度为 0 的序列
- 比如 `a = {1, 2, 3}`，`a` 是一个长度为 3 的序列（`#a == 3`）

对于有洞的列表，其长度有时不会是你想要的：
- 比如上面定义的 `d`，有 `#d == 2`，而 `#{1, 2, nil, nil, 5} == 5`。
- 甚至有 `#{1, 2, nil, nil} == #{1, 2}`：在列表最末端的那些 `nil`，lua 不会考虑它们的长度。
- 大多数情况下，lua 遇到的列表为序列，此时使用获取的长度是安全的，但遇到有洞的列表，其长度并不可靠。
- 如果你真的需要处理有洞的列表，应该把长度显式地存储在某个地方。

### 遍历 `table`

1. 使用 `pairs` 迭代器遍历 `table` 的所有键值对：

```lua
t = {10, print, x = 12, k = "hi"}
for k, v in pairs(t) do
print(k, v)
end
--> 1 10
--> k hi
--> 2 function: 0x420610
--> x 12
```

遍历的顺序是未定义的，每次运行都可能顺序不同。但可以确定的是，它会把所有键值对遍历一次。

2. 使用长度和 `for` 遍历序列部分：

```lua
t = {10, print, nil, "4", x = 12, k = "hi"}
for k = 1, #t do
print(k, t[k])
end
--> 1	10
--> 2	function: 0x560c95149d90
--> 3	nil
--> 4	4
```

需要注意带洞列表的长度：
```lua
t = {10, x = 12, k = "hi"}
t[4] = "4"
for k = 1, #t do
print(k, t[k])
end
--> 1	10
```

### 安全导航

如果你需要索引一个很深的嵌套表，比如：

```lua
zip = company and company.director and company.director.address and company.director.address.zipcode
```

那么这种做法是低效的，它需要对表做六次查询（看 `.` 的数量）。

在某些语言中（比如 C#），有 `?` 这个安全导航符号 (safe navigation operator)，这种情况可使用：

```c#
zip = company?.director?.address?.zipcode
```

处理。但 lua 奉新极简主义，不打算提供这种语法。因为有绕行的办法：

```lua
zip = (((company or {}).director or {}).address or {}).zipcode

-- 或者
E = {} -- 可在类似的表达式中重复使用
...
zip = (((company or E).director or E).address or E).zipcode
```

此时只需要做三次查询（已经是最小查询次数）。

### 增、删、移动

针对序列的操作，即表（或者说列表）的 `{1, ..., n}` 索引的部分。

增 `table.insert`：
- `table.insert(t, ele)` 等价于 `t[#t+1] = ele`，在序列的最后位置插入元素，即 `table.insert(t, #t, ele)`
- `table.insert(t, pos, ele)`：在第 `pos` 位置上插入元素，它会把后面的元素往后移

删 `table.remove`：
- `table.remove(t)` 等价于 `last = t[#t]; t[#t] = nil; last`，它删除最后位置上的元素，并返回这个元素
- `table.remove(t, pos)`：删除第 `pos` 位置上的元素，并返回被删除的元素，它也会把之后的元素往前移

有了这两个操作，lua 可以实现栈、队列、双向队列。虽然它们效率不高，但其背后是 C
的循环，在数百元素的情况下代价不算太昂贵。

移动 `table.move`：
- 这是 lua5.3 引进的通用函数，对原表执行移动之后，返回新表的引用
- `table.move(a, f, e, p)` 表示把 a 表的 [f, e] 索引内的元素**复制**到位置 p 上
- `table.move(a, f, e, p, b)` 表示把 a 表的 [f, e] 索引内的元素**复制**到 b 表的位置 p 上
- `table.remove(t, pos)` 等价于 `table.move(t, pos+1, #t, pos); t[#t] = nil`
- `table.insert(t, pos, ele)` 等价于 `table.move(t, pos, #t, pos+1); t[pos] = ele`
- `table.move(a, 1, #a, 1, {})` 意味着返回 a 序列所有元素的副本序列
- `table.move(a, 1, #a, #b + 1, b)` 表示把 a 序列的所有元素添加到 b 序列之后

完整的 `table` 操作请参考 [lua5.3-manual#Table Manipulation](https://www.lua.org/manual/5.3/manual.html#6.6)：
- `table.concat` 拼接序列的字符串和数字：如 `table.concat({"a", 1}) == "a" .. 1 --> "a1"`
- `table.pack` 把多个表按照序列顺序存放到一个新表：如 `c = table.pack({a=1,b=2}, {1}); c[1].a == 1; c[2][1] == 1`
- `table.unpack` 把一个表的序列部分拆分成多个元素：如 `d,e = table.unpack(c); d.a == 1; e[1] == 1`
- `table.sort` 对一个表的序列排序（直接修改序列顺序）：如 `a = {1,3,2}; table.sort(a); table.concat(a) --> "123"`

## `function`

### 基础要点

- 调用函数的语法：
    - 在有无参数的情况下，都使用括号：`os.date()`、`print(1, 2)`
    - 当参数只有一个，而且这个参数是字符串字面值或者表构造表达式，可以省略括号，也可以无需空格分隔：
      `print"1"`、`type{} --> table`
    - 方法调用：`o:foo(x)` 其中 `o` 是对象，`foo` 是其方法
    - 调用来自 C 或者宿主应用 (host application) 的函数：#todo#
- 参数：
    - 可以输入与函数定义时数量不同的参数个数：
        
        ```lua
        function f (a, b) print(a, b) end

        -- 以下都是有效调用
        f() --> nil nil
        f(3) --> 3 nil
        f(3, 4) --> 3 4
        f(3, 4, 5) --> 3 4 （5 被舍弃）
        ```
    - 默认参数：
    
        ```lua
        -- 通过调用 `incCount()` 可以达到默认增加 1 的效果
        function incCount (n)
          n = n or 1
          globalCounter = globalCounter + n
        end
        ```
    - 可变长度的参数：使用可变参数表达式 (vararg expression) `...` 作为函数参数，然后在函数内
        - 使用 `{...}` 把可变参数放入表
        - 使用赋值语句获取所需数量的参数
        - 使用 `select(i, ...)` 获取第 i 个及其之后的所有参数；或者使用 `select("#", ...)` 查看长度
        - 使用 `table.pack(...)`[^pack] 把可变参数放入表，并把长度记录到 `.n` 字段里
        - `{...}` 的方式会忽略以 `nil` 结尾的那些参数，但 `select` 和 `table.pack` 不会

        ```lua
        function add (...)
          local s = 0
          for _, v in ipairs{...} do
            s = s + v
          end
          local a = {...}     -- 还可以继续使用
          local _, b, c = ... -- 还可以继续使用
          print(a[1], b, c) 
          print(select(4, ...)) -- 还可以继续使用
          print(#a, select("#", ...), table.pack(...).n)
          return s
        end
        print(add(3, 4, 10, 25, 12)) --> 54 （打印 3 4 10 和 25 12 和 5 5 5 ，返回 54）
        print(add(3, 4, 10, 25, nil)) --> 42 （打印 3 4 10 和 25 nil 和 4 5 5 ，返回 42）
        ```
    - 可以混合固定参数和可变参数：如 `function f (a, b, ...) end`
- 返回值：
    - 当函数作为语句被调用时，丢弃该函数所有返回值
    - 当函数作为普通的表达式被调用时，只保留该函数的第一个返回值
    - 当且仅当函数作为**最后一个或者唯一一个**表达式被调用时，才会得到这个函数的所有返回值。\
      此时函数必须在以下表达式中出现：
        - 多个赋值的表达式
        - 另一个函数的参数
        - `table` 的构造表达式
        - `return` 语句里
    - 对于 `f(g())` 形式的函数调用，如果 `f()` 具有固定长度的参数，那么 `g()` 返回 `f()` 所需的固定长度的值
    - 通过在函数调用的外围添加 `()` 来强制让该函数只返回一个值
    - 返回值也可以是可变长度的：`function f (...) return ... end`、`function f (a, b, ...) return ... end`
    - 使用 `select(i, f())` 获取第 i 个及其之后的所有返回值；或者使用 `select("#", f())` 查看函数返回值的长度

[^pack]: `table.pack` 函数是从 lua5.2 版本才引进的

```lua
-- 返回值综合案例
function foo0 () end -- returns no results
function foo1 () return "a" end -- returns 1 result
function foo2 () return "a", "b" end -- returns 2 results

x, y = foo2() -- x="a", y="b"
x = foo2() -- x="a", "b" is discarded
x, y, z = 10, foo2() -- x=10, y="a", z="b"

x,y = foo0() -- x=nil, y=nil
x,y = foo1() -- x="a", y=nil
x,y,z = foo2() -- x="a", y="b", z=nil

x,y = foo2(), 20 -- x="a", y=20 ('b' discarded)
x,y = foo0(), 20, 30 -- x=nil, y=20 (30 is discarded)

print(foo0()) --> (no results)
print(foo1()) --> a
print(foo2()) --> a b
print(foo2(), 1) --> a 1
print(foo2() .. "x") --> ax (这里不属于那四种表达式，是普通的表达式，所以只返回一个值)

t = {foo0()} -- t = {} (an empty table)
t = {foo1()} -- t = {"a"}
t = {foo2()} -- t = {"a", "b"}

t = {foo0(), foo2(), 4} -- t[1] = nil, t[2] = "a", t[3] = 4 这里并不是最后一个表达式

function foo (i)
  if i == 0 then return foo0()
    elseif i == 1 then return foo1()
    elseif i == 2 then return foo2()
  end
end

print(foo(1)) --> a
print(foo(2)) --> a b
print(foo(0)) -- (no results)
print(foo(3)) -- (no results)

print((foo0())) --> nil
print((foo1())) --> a
print((foo2())) --> a
```

`table.unpack` 进阶用法：
- `table.unpack(t)` 把序列 `t` 一次分解成 [1, #t] 位置上的元素
- `table.unpack(t, start, end)` 把序列 `t` 一次分解成 [start, end] 位置上的元素
- 将其返回值放到函数参数上，形成变长和泛型调用
    ```lua
    print(string.find("hello", "ll"))
    -- 改成动态函数和动态参数的等价写法
    f = string.find
    a = {"hello", "ll"}
    print(f(table.unpack(a)))
    ```
- 该函数为 C 写成，其等价的 lua 写法：
    ```lua
    function unpack (t, i, n)
      i = i or 1
      n = n or #t
      if i <= n then
        return t[i], unpack(t, i + 1, n)
      end
    end
    ```

### 尾调用

lua 的函数实现了尾调用消除 (tail-call elimination) —— 
当进行尾调用时，lua 不使用额外的栈空间，因为最后被调用的函数无需返回调用它的函数。

因此程序可以无限地嵌套尾调用，而不产生栈溢出。

但是要清楚尾调用的形式：`return func(args)` —— 即调用尾函数的函数，在调用尾函数之后，不做任何事情。

如下形式不是尾调用：

```lua
function f (x) g(x) end         -- 无论 g 返回什么，f 都要返回
function f (x) return g(x) + 1  -- 执行了 g 还要执行加法操作
function f (x) return x or g(x) -- 需要把结果调整为 1 个
function f (x) return (g(x))    -- 需要把结果调整为 1 个
```

注意，`func` 和 `args` 可以是复杂的表达式，此时依然是尾调用，如 
`return x[i].foo(x[j] + a*b, i + j)`。





