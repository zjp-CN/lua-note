# 语法细节

## `local` 和 block

lua 的变量默认为全局变量，即非 `local` 声明的变量。

这意味着，如果一个脚本里有全局变量，当这个脚本被加载之后，脚本内的全局变量被带入当前作用域。

而 `local` 声明的局部变量则把变量的作用域限制在变量所声明的 block 内。

block 指：
- 控制结构的主体（及其内部）
- 函数体（及其内部）
- `do ... end`
- chunk：变量所声明的文件（及其内部）或者 `string`。在 lua 的交互模式中，每一个完整执行的语句都是一个 
  chunk：一般按下回车键运行完一个完整语句，那么 lua 进入下一个 chunk。所以如下代码：
    ```lua
    local a = 1
    print(a)
    ```
  如果在交互模式中一行行输入（或者复制粘贴运行），打印结果为 `nil`。  如果放在脚本里，使用
  `lua xx.lua`，打印结果为 1。

  将这段代码放入 `do` `end` 中，则可以在交互模式下正常运行：
  ```lua
  do
  local a = 1
  print(a)
  end
  ```

尽可能地使用局部变量的是良好的 lua 编程方式，因为：
- 避免把不必要的变量引入全局环境
- 避免与程序的其他部分的名字发生冲突
- 获取/访问局部变量比全局变量快
- 局部变量在离开作用域之后消失，有利于 GC 释放

关于全局变量的技巧：
1. 引入 [`strict`](https://github.com/lua-stdlib/strict) ，如果未声明变量而使用，那么产生错误。
2. 使用 `local foo = foo` 来引入全局变量到局部作用域，这有助于加快访问速度，而且即便其他函数修改全局 
   `foo`，在此局部作用域中保留一份变量，即在局部作用域这行语句之后使用的 `foo` 都为局部声明的。
3. 尽可能以初始化的方式声明变量，而不是先声明，后赋值。

## 控制结构

- 任何类型都可以作为条件：`false` 和 `nil` 两个值表示 `false` 含义，其余值都表示 `true` 含义（见
  [`Boolean`](Getting-Started.md#boolean)）
- `if` 用于条件执行：`if cond1 then ... (elseif cond then ...)* (else ...)? end`
- 循环控制：
    - `while cond do ... end`
    - `repeat ... until cond` 注意 repeat 和 until 之间的局部变量在 cond 位置依然可以使用
    - `for` 语句有两种方式：
        - 数值：`for var = start, end, step do ... end` 其中 `var`、`start`、`end`、`step` 均为一个数值，
          `step`[^step] 可以省略，因为默认为 1
        - 泛型：`for var1, var2  = iter do ... end` 其中 `=` 右侧的变量数量可不与 `iter` 返回数量严格相等[^generic]
    - `break` 语句用来结束所处的内层循环，但不结束外层循环。

[^step]: 这有个奇怪的例子：`for i = 1,2,0.1 do print(i) end` 最后打印的数字为 1.9，而
         `for i = 1,2 do print(i) end` 最后打印的数字为 2。

[^generic]: 因为 lua 的返回值也可以是[变长](Types.md#function)的

## `return`

`return` 语句用于从函数中返回值或者结束函数。

每个函数的结尾都有隐式返回 `nil`，此时不必显示写出 `return` 或者 `return nil`。

lua 的语法要求 `return` 只能作为 block 的最后一个语句。你可以在 `end`、`else`、`until` 之前使用
`return`，但不能在 block 中间（`return` 后还有语句）使用 `return`。

有时为了调试一个函数，我们想阻止一个函数执行，那么你可以把 `return` 放到 `do ... end` block 中：

```lua
function foo ()
  return --<< 语法错误
  -- `return` 是下面这个 block 的最后一个语句，可用来结束 foo 函数
  do return end -- OK
  other statements
end
```

# `goto`

`goto label` 用于跳转到程序所对应的 label 再执行，跳转到的 `label` 的语法为 `::label::`。 

lua 对跳转的 label 有以下限制：
1. label 遵循正常的可见性规则，一个 block 内的 label 不被 block 外部可见，所以不能跳转到到
   block 外面，也不能从函数外跳入函数内。
2. 不能跳出函数。
3. 不能跳进局部变量的作用域。

lua 没有 `continue`、多层 `break`、多层 `continue`、`redo`、局部错误处理，但 `goto` 可以充当这些角色。

比如 `continue` 就是 `goto` 循环末端的 label，`redo` 就是跳到循环的开始：

```lua
while some_condition do
  ::redo::
  if some_other_condition then goto continue
  else if yet_another_condition then goto redo
  end
  some code
  ::continue::
end
```

局部变量的作用域在其被定义的 block 的最后非空语句 (non-viod statement) 结束，而 label 是空 (viod) 语句。

```lua
do
goto continue

local var = 1
print(var)

::continue::
--[[
print("end") 
上面这行打印语句意味着局部变量 var 的作用域未结束（即便没有使用 `var`），
但这依然违反了 label 的第三条限制
lua 会告诉你 `jumps into the scope of local 'var'`
--]]
end
```

你可以把 `var` 放入 `do ... end` block，这样 `::continue::` 不落入 `var` 的作用域（此时程序也不会执行 `var` 相关的代码）。下面的语句也是符合语法的：

```lua
do
goto continue

local var = 1
print(var)

::continue:: -- 局部变量 `var` 在此之前结束作用域，而且程序不会执行 `var` 有关的代码
end
```
