# 入门

[lua 官网](http://www.lua.org/) | [Reference 手册](https://www.lua.org/manual) | [Getting started][Getting started]

> 该笔记系列主要整理自 Roberto Ierusalimschy 的 *Programming in Lua* （2017，第四版） 一书。以 Lua5.3 为主。

## 版本历史

| 正式版本号   | 发布年份[^year] | *Programming in Lua* |
|--------------|-----------------|----------------------|
| 1.0          | 1993            |                      |
| 2.1          | 1995            |                      |
| 3.0          | 1997            |                      |
| 4.0          | 2000            |                      |
| 5.0[^LuaJIT] | 2003            | 第一版 (2003)        |
| 5.2          | 2011            | 第三版 (2016)        |
| 5.3          | 2014            | 第四版 (2017)        |
| 5.4          | 2020            |                      |

[^year]: 年份信息整理自：[lua 官网](https://www.lua.org/news.html)

[^LuaJIT]: 最新 LuaJIT 2.0+ 所对应的版本

[Getting started]: http://www.lua.org/start.html

## 安装 lua

参考：[安装说明][Getting started]。

例如，在 ubuntu 上输入 lua，会提醒你没有安装 lua，并且列出安装命令。在我的发行版本上（ubuntu
20），使用 `apt install lua5.3` 安装。如果需要最新版本，则下载源码编译安装。

## 运行 lua

### 基本方式

```console
usage: lua [options] [script [args]]
Available options are:
  -e stat  execute string 'stat'
  -i       enter interactive mode after executing 'script'
  -l name  require library 'name'
  -v       show version information
  -E       ignore environment variables
  --       stop handling options
  -        stop handling options and execute stdin
```

- 进入命令行交互模式：`lua` 或者 `lua -i`。
- 运行脚本文件：`lua xx.lua`。
- 运行脚本文件之后进入命令行交互模式：`lua -i xx.lua`，它会把 `xx.lua` 中的内容（变量、函数等）带入交互会话。
- 执行单条语句：`lua -e "print(1)"` 则打印 1。也可以执行多条语句：`lua -e "a=1" -e "print(a)"`，也打印 1。

### 捕获参数

在 `print.lua` 中写入以下内容：
```lua
print(arg[-4])
print(arg[-3])
print(arg[-2])
print(arg[-1])
print(arg[0])
print(arg[1])
print(type(arg[1]))
print(arg[2])
print(a)
```

并使用以下命令运行：
```console
lua -i -e "a=100" print.lua 1 "s"
```

会打印如下内容，并进入 lua 的交互模式：
```text
lua
-i
-e
a=100
print.lua
1
string
s
100
```

注意：
1. `-e` 参数后的表达式是有效定义的，在交互模式下，`a` 的值为 100；
2. `arg` 是 lua 预定义的[全局变量][全局变量]，`arg[0]` 总是脚本名，
   `arg` 的正整数索引为其后所跟的参数，类型为 `string`。

[全局变量]: #全局变量

### 加载库

针对上面 `print.lua` 脚本，你也可以使用 `lua -l print` 加载运行（无需 `.lua` 后缀）。

但是由于此时我们调用了与 `print` 函数同名的库，这导致
`print` 函数不可用。（你可以使用 `lua -l print -i` 进入交互模式尝试一下）

### 交互模式

#### 可以省略的 `=`

在 lua5.2 之前的版本中，命令行需要在表达式之前输入 `=`：
```lua
a = 0
= a+1 --> 1
= a-1 --> -1
```

而从 lua5.3 开始的版本中，可以无需输入 `=`：
```lua
a = 0
a+1 --> 1
a-1 --> -1
```
当然，为了保持兼容性，你输入 `=` 也不会有报错。

#### 运行块

lua 交互模式并不直接支持以块 (chunk) 为单位运行代码，也就是说，你无法在交互中以块的方式换行 —— 按下 
`Enter` 即执行语句[^Enter]。

[^Enter]: 但你如果在语句未结束之前按 `Enter`，并不会执行语句，而是语句内的换行：
```lua
> c = 1; print( -- 回车
>> c)
1
```

除了把多个语句写到一行之外，你还可以采取以下一些方式运行一个代码块：
- 把代码写到 `.lua` 的文件中，使用 `lua -i xx.lua`，执行脚本然后自动进入交互模式，并继续使用脚本内定义的内容。
- 进入交互模式，使用 `dofile("xx.lua")`，中途执行脚本，并继续使用脚本内定义的内容。\
  `xx.lua` 可以是绝对路径也可以是相对路径，但是无法识别 `~` 开头的 HOME 目录。

> lua 这种按块加载脚本的做法，在调试或测试代码时是方便的。
>
> 你可以在两个窗口中工作：一个编辑器窗口用来写 lua 代码；另一个命令行窗口进入交互模式，执行刚写的代码。

## 命名规则

1. 可以组合大小写英文字母、数字、`_` 来命名，但是不能以数字开头。
2. 不支持 UTF8 字符，即不支持中文字符作为（变量、函数等）名字。
3. 避免使用 `_` 后跟大写字母的命名方式，比如 `_VERSION` 这个名字被 lua 所使用，具有特殊含义。
4. 以下名字被 lua 所保留，不能当作普通名字被使用。
    ```
    and break do else elseif end false for function goto
    if in local nil not or repeat return then true until while
    ```
5. lua 里的名字是大小写敏感的，比如 `and` 是被保留的名字，但 `And` 和 `AND` 是可被使用的名字。

## 注释符号

1. 单行注释符号为 `--`，从 `--` 开始到一行结束的内容都被认为是注释。\
   因此，你可以在一行的开头，或者一行代码的末尾用此符号写注释。
2. 区域（多行）注释符号为 `--[[]]`，从 `--[[` 到 `]]` 之间的内容被认为是注释。
3. 如果在多行注释内输出 `[[` 或 `]]`，参考 [`string` 的多行文本](Types.html#string)

注意，区域注释不一定是多行：
```lua
--[[ print(1) ]] --> 等价于 `-- print(1)` 当行注释
--[[ print(1) ]]print(2) --> 依然会执行 `]]` 之后的内容，即打印 2

---[[
print(1) --> 这个语句会被执行，因为 `---[[` 会优先被识别为单行注释，从而 `--]]` 也是单行注释
--]]
```

## 语句分隔符 `;` 

lua 在语句之间并不需要 `;` 分隔符，而且换行也不具有语句分隔的作用。

以下四种形式都可以分隔两个语句。

```lua
-- 常用
a = 1
b = a * 2

-- 你可以在末尾加 `;` 
a = 1;
b = a * 2;

a = 1; b = a * 2 -- 把多个语句写到一行时常用

a = 1 b = a * 2 -- 有效语句
```

在交互模式下，调用函数会直接打印结果，而在函数后添加 `;` 可以阻止打印结果。

# 变量与类型

## 全局变量

全局变量 (Global Variables)：
1. 无需声明即可使用。未声明的全局变量都是 `nil`。所以使用未声明的全局变量并不会产生错误。
2. 可以把全局变量赋值为 `nil`。lua **不区分**全局变量的 `nil` 是人为赋值，还是未初始化。
3. 把全局变量赋值为 `nil` 之后，lua 最终可能<abbr title="reclaim">回收</abbr>全局变量所使用的的内存。
4. 全局变量存放在 `_ENV` 表中，比如 `a` 无论是否被定义，都可以使用 `_ENV.a` 来表示，且 `a==_ENV.a --> true`。


## 类型

lua 是<abbr title="dynamically-typed language">动态类型语言</abbr>，所以没有类型定义（你无法定义一个类型）。

但是，<abbr title="Each value carries its own type.">每个值都带有各自的类型。</abbr>

lua 有八种基本类型：
1. `nil`
2. `Boolean`
3. `number`
4. `string`
5. `userdata`
6. `function`
7. `thread`
8. `table`

使用 `type` 函数可以知道一个值的类型，`type` 函数的返回值是 `string` 类型。

```lua
type(nil)           --> nil
type(X)             --> nil
type(true)          --> boolean
type(1)             --> number
type(0.1)           --> number
type("Hello world") --> string
type(io.stdin)      --> userdata
type(type)          --> function
type({})            --> table
type(type(X))       --> string
```

变量可以存放任何类型的值：

```lua
type(a) --> nil ('a' 尚未被初始化)

a = 1
type(a) --> number

a = "a string!!"
type(a) --> string

a = nil
type(a) --> nil
```

### `nil`

`nil` 类型只有一个值 `nil`，用来表示与其他值不同、或者没有值。

`nil` 类型可以比较相等，且只与 `nil` 相等。

```lua
nil == nil --> true
nil == 0   --> false
nil == 1   --> false
nil == ""  --> false
```

全局变量的值在第一次赋值之前，默认为 `nil`。通过赋值 `nil` 给全局变量来把它删除。

### `Boolean`

`Boolean` 类型只有两个值：`true` 和 `false`。
```lua
not true == false --> true
not false == true --> true
false ~= true     --> true
```

但 `Boolean` 值不是唯一表示条件的值。

> 在条件语句和逻辑操作中，有以下规定：
>
> 1. `false` 和 `nil` 都表示 `false`，其他都表示 `true`。尤其 `0`、`0.`、`""` 都表示 `true`。
> 2. `and` 运算符返回的结果：当第一个运算对象表示 `false` 时，返回第一个运算对象，否则返回第二个运算对象。
> 3. `or` 运算符返回的结果：当第一个运算对象表示 `true` 时，返回第一个运算对象，否则返回第二个运算对象。
>
> 注意，`and` 和 `or` 返回的是运算对象 (operand)，而不是 `Boolean` 值。

比如：

```lua
nil   and 1    --> nil
false and 1    --> false
0     and 1    --> 1
1     and 0    --> 0
false or "hi"  --> "hi"
nil   or false --> false
0     or 1     --> 0
1     or 0     --> 1
```

---

`and` 和 `or` 都是<abbr title="short-circuit evaluation">短路求值</abbr>：当第二个运算对象必须求值时才计算。

`x = x or v` 与 `if not x then x = v end` 等价，可以用来把原本为 `false` 或者 `nil` 的 `x` 设置成 `v` 的值。

`(a and b) or c` 与 `a and b or c` 等价，与 C 语言中的 `a ? b : c` 等价，表示满足 a 时，返回 b，否则返回 c。\
比如 `(x > y) and x or y` 表示取数字 x 和 y 中的较大值。当 `x > y` 时，`and` 语句返回 `x`，而 
`x` 为数字，表示 `true`，所以在 `or` 语句中被返回。当不满足 `x > y` 时，`and` 语句返回 `false`，而
`y` 为数字，表示 `true`，所以在 `or` 语句中被返回。

`>`、`<`、`==`、`~=`、`>=`、`<=`、`not` 运算符总是返回 `Boolean` 值：

```lua
"1" > "0"   --> 等价于 1 > 0，返回 true
"-1" > "0"  --> 等价于 -1 > 0，返回 false
not nil     --> true
not false   --> true
not 0       --> false
not not 0   --> true
not not nil --> false
```

两个不同的类型一般不能比较大小，但是可以比较相等：两个不同的类型比较相等时，总是返回 
`false`。比如 `1 == nil --> false`、`false ~= nil --> true`。

\#TODO\# `>` 和 `<` 比较字符串。



### `userdata`

userdata 可以把任意 C 的类型存储在 lua 变量中，并在 lua 中，除赋值和比较相等之外，无预定义的操作。

它表示外部程序或 C 库中创建的新类型。比如标准库 I/O 用它表示打开文件。所以它在 C API 方面涉及较多。

