# I/O 功能

lua 只提供标准 ISO C 提供的功能，其标准库仅实现了基础的 I/O。大部分 I/O 功能是由外部库或者宿主语言提供的。

## 简单 I/O 模型

基于当前输入/输出流：

- `io.input(filename)` 读取 `filename` 文件的内容
- `io.write(a, b, c)` 与 `print(a, b, c)` 类似，但 `io.write`
  不会增加额外的符号（制表符、换行等），而且可以重定向输出
- `io.read()` 从当前输入流输入读取一行，等价于 `io.read("l")`
  （可以读取标准输入也可以通过重定向输入读取文件）。此函数参数如下：

| 参数[^read-arg]    | 说明[^read-res]                        |
|--------------------|----------------------------------------|
| "a"                | 读取整个文件的内容[^read-all]          |
| "l"                | 读取下一行（丢弃换行符）               |
| "L"                | 读取下一行（保留换行符）               |
| "n"                | 读取一个数字[^read-n]                  |
| bytes （非负整数） | 读取 bytes 字节大小的数据[^read-bytes] |

[^read-arg]: 在 lua5.2 及之前的版本，该参数需要使用 `*` 前缀，比如 `io.read("*a")`

[^read-res]: 除了 `n`，其他参数情况下都返回 `string` 类型

[^read-all]: 一般用于读文件，而不是读取标准输入（比如从终端输入的内容）

[^read-n]: 在跳过空格之后，如果还没找到数字，那么返回 nil，找到数字返回 `number` 类型

[^read-bytes]: `io.read(0)` 返回空串（表示文件还有待读取的内容）或者 `nil`（表示无内容待读）

按行读取文件：

```lua
-- 方法一：io.read("L")
for count = 1, math.huge do
  local line = io.read("L")
  if line == nil then break end
  io.write(string.format("%6d ", count), line)
end

-- 方法二：io.lines()
local count = 0
for line in io.lines() do
  count = count + 1
  io.write(string.format("%6d ", count), line, "\n")
end
```

按块（固定大小）读取文件：
```lua
while true do
  local block = io.read(2^13) -- block size is 8K
  if not block then break end
  io.write(block)
end
```

`io.read` 可以接收可变长度的参数，来以不同方式读取同一个文件的不同区域：
```lua
-- 对于以下 xx.txt 文件内容
-- 6.0 -3.23 15e12
-- 4.3 234 1000001
-- 以下代码写在 xx.lua 文件中
-- 使用 `lua xx.lua < xx.txt` 命令运行的打印结果
-- 6.0	-3.23	 15e12	4.3 234 1000001
local n1, n2, n3, n4 = io.read("n", "n", "l", "l")
print(n1, n2, n3, n4)
```

## 完整的 I/O 模型

- `io.open(filename, mode)` 打开文件，等价于 C 的 `fopen` 函数
    - 其中 `mode` 有
        - `"r"` 读权限
        - `"w"` 写权限（擦除原文件所有内容）
        - `"a"` 追加写
        - `"b"` 以二进制方式打开
    - 该函数返回新的文件流
    - 遇到错误时，返回 `nil`、错误信息和依赖于系统的错误码
    - 一般使用 `assert(io.open(filename, mode))` 来检查错误
    - 正常打开文件之后，调用 `read` 或者 `write` 方法来读写
        ```lua
        local f = assert(io.open(filename, "r"))
        local t = f:read("a")
        f:close()
        ```
- 标准输入/输出流：
    - `io.stderr:write(message)` 把错误信息打印到标准错误流
    - `io.input():read(args)` 的简写是 `io.read(args)`
    - `io.input():lines(args)` 接收与 `io.read` 相同的参数[^read-lines]，以读取方式打开一个文件流，返回迭代器
        ```lua
        -- 迭代 8K 大小的块，把输入流的内容复制到输出流
        for block in io.input():lines(2^13) do
          io.write(block)
        end
        ```
    - `io.output():write(...)` 的简写是 `io.write(...)`
        ```lua
        local fname = "a.txt"
        local a = io.output(fname) -- 如果文件不存在，则创建
        print(a:write(1)) -- 覆盖写入
        print(a:write(2))
        print(a:close())
        local a = io.input(fname)
        print(a:read()) -- 打印 12
        ``` 
- `io.tmpfile()` 返回临时文件流，当程序结束时，该文件自动被删除
- `io.flush()` 或 `f:flush()` 把剩余内容写进流
- `f:setvbuf(mode, size)` 设置流的缓冲方式和大小：
    - `"no"` mode 不设置缓冲
    - `"full"` mode 当缓冲满了或者显式调用 `f:flush()` 之后才写入流
    - `"line"` mode 在换行或者终端新输入之前缓冲
    - `size` 在 `"full"` 和 `"line"` mode 下是可选的
    - 在大多数操作系统上，stderr 是无缓冲的；stdout 是行缓冲的，如果对标准输出写入不完整的行，可能需要 
      `flush` 才能看到输出结果
- `f:seek(whence, offset)` 获取并设置文件流的当前位置（位置 0 表示文件的起点，所以 offset 最低为 0）
    - `f:seek("set", n)` 设置当前位置为 n（距离起点 n 字节）
    - `f:seek("cur")` 或者 `f:seek()` 获取当前位置
    - `f:seek("end")` 返回文件的终点位置，即文件的字节大小，此时落在终点位置
    - `f:seek("cur", n)` 和 `f:seek("end", n)` 意义不明，**可能**与 `f:seek("set", n)`
      一样（可变参数的缺点：如果不列举完全一个函数在不同参数下的功能，那么它的用法很难搞懂）
        ```lua
        -- 获取文件大小，但不移动当前位置
        function fsize (file)
         local current = file:seek()   -- save current position
         local size = file:seek("end") -- get file size
         file:seek("set", current)     -- restore position
         return size
        end
        ```

[^read-lines]: 这在 lua5.2 之后才支持。

## `os` 函数

- `os.rename` 重命名文件
- `os.remove` 删除文件
- `os.exit` 终止程序执行：
    - 第一个参数是可选的：0 （或者 `true`）表示正常执行
    - 第二个可选参数为 `true` 表示调用 finalizers，根据状态释放所有内存。通常无需 
      finalization，因为大多数操作系统在进程退出时释放该进程的所有资源。
- `os.getenv` 获取环境变量。如 `os.getenv("HOME")` 获取主目录路径。如果环境变量不存在，函数返回 `nil`。
- `os.execute("xx cmd")` 执行系统命令，等价于 C 的 `system` 函数，它执行命令之后，返回三个部分：
    - `Bolean`：`true` 表示执行成功
    - `string`：`exit` 表示正常退出；`signal` 表示被中断
    - `number`：程序返回的状态码
- `io.popen` 与 `os.execute` 会在执行命令的时候创建流：
    - `io.popen(cmd, "r")` 表示读取命令的结果，等价于 `io.popen(cmd)`
    - `io.popen(cmd, "w")` 表示往命令的流里写数据
        ```lua
        -- 获取 `ls` 命令的结果
        local f = io.popen("ls .", "r")
        local dir = {}
        for entry in f:lines() do
          dir[#dir + 1] = entry
        end

        -- 往命令的流里写数据
        local f = io.popen(cmd, "w")
        f:write([[xxx]])
        f:close()
        ```

`os.execute` 和 `io.popen` 是依赖于操作系统的。

对于基础的目录和文件操作，可使用 `LuaFileSystem` 库；需要 POSIX.1 标准的功能则使用 `luaposix` 库。
