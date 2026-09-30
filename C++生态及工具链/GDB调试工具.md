# GDB 调试工具

> GDB（GNU Debugger）是 Linux 下 C/C++ 的事实标准调试器，提供断点、单步、变量检查、调用栈、多线程、core dump 分析等完整调试能力。本文件以工程实战级讲解 GDB 的调试原理、高频指令与调试思路。

---

## 目录

1. GDB 的定位与调试原理
2. 编译与启动：如何让程序可调试
3. 启动与基本导航命令
4. 断点：设置、管理、条件断点
5. 单步执行与继续
6. 查看变量、内存与寄存器
7. 调用栈与帧切换
8. 多线程调试
9. 修改程序状态与函数调用
10. core dump 分析
11. 高级：watchpoint、catch、Python 扩展
12. 调试思路与实战流程
13. 常见问题

---

## 1. GDB 的定位与调试原理

**定位**：源代码级调试器，让开发者观察程序运行时的状态、控制执行流程。

**调试原理**：
- 程序需带**调试符号**（`-g`），GDB 才能把机器地址映射回源码行/变量。
- 通过 `ptrace` 系统调用控制目标进程（停、续、读写寄存器/内存）。
- 支持断点（暂停）、单步、堆栈回溯、修改变量与调用函数。

```bash
g++ -g -O0 main.cpp -o app    # -g 调试符号，-O0 关闭优化便于逐行调试
gdb ./app
```

---

## 2. 编译与启动

### 2.1 编译要求

```bash
g++ -g -O0 main.cpp -o app     # 必须 -g；-O0 保证变量可见、行号对应
```

- `-O2` 下调试会“行号漂移”、变量被优化，难以下断点。
- 发布崩溃定位用 **core dump**（见第 10 章）。

### 2.2 启动方式

```bash
gdb ./app                     # 交互启动
gdb ./app core                # 分析 core dump
gdb --args ./app -v --flag    # 传命令行参数
gdb -p 12345                  # 附加到运行中进程
```

- 交互内也可：`set args -v` 设置参数，`run` 启动。

---

## 3. 启动与基本导航

| 命令 | 作用 |
|---|---|
| `run` / `r` | 运行程序 |
| `continue` / `c` | 继续执行到下一个断点/结束 |
| `next` / `n` | 单步（不进入函数） |
| `step` / `s` | 单步（进入函数） |
| `finish` | 运行到当前函数返回 |
| `until` | 运行到指定行 |
| `quit` / `q` | 退出 |

GDB 会自动加载源文件并显示当前行。常用 `list`/`l` 显示源码。

---

## 4. 断点

### 4.1 设置断点

```bash
break main                    # 函数名断点
break main.cpp:42             # 文件:行号断点
break main.cpp:foo            # 文件:函数断点
break *0x401000               # 地址断点
break foo if x > 5            # 条件断点（每次检查条件）
```

### 4.2 管理断点

```bash
info breakpoints              # 列出断点
delete 2                      # 删除断点 2
disable 2                     # 禁用（保留编号）
enable 2                      # 启用
condition 2 x > 10            # 给断点 2 设条件
```

---

## 5. 单步执行与继续

```bash
step        # 进入被调函数
next        # 不进入，跳过
stepi / nexti  # 指令级单步（配合汇编调试）
finish      # 运行到当前函数返回，停在调用点
continue    # 继续到下一断点
```

- 观察**进入/跳过函数**的差异是掌握单步的关键。
- 指令级调试用 `layout asm` 查看汇编，`info registers` 查看寄存器。

---

## 6. 查看变量、内存与寄存器

### 6.1 变量

```bash
print x                    # 打印变量 x
p &x                       # 地址
p my_vector.size()         # 调用表达式
p *ptr                     # 解引用
p *(std::vector<int>*)&v   # 类型转换查看
set var x = 5              # 修改变量（调试用）
```

### 6.2 查看容器/结构

- 默认 `print` 对 STL 容器显示不友好；可用：
  - `p v` 结合 `p v[0]`、`p v._M_impl._M_start`（实现细节）。
  - 启用 pretty-printer（GDB 默认带 STL pretty-print，输出更可读）。
- `ptype T`：显示类型定义。

### 6.3 内存与寄存器

```bash
x/10xb ptr        # 从 ptr 起查看 10 个字节（十六进制）
x/10i $pc         # 反汇编 10 条指令
info registers    # 全部寄存器
info frame        # 当前帧信息
```

`x` 格式：`x/<数量><格式><大小>`，如 `x/20wx`（20 个 4 字节十六进制字）。

---

## 7. 调用栈与帧切换

```bash
bt                # backtrace：打印完整调用栈
bt full           # 含局部变量
frame 2           # 切换到第 2 帧
info locals       # 当前帧局部变量
info args         # 当前帧参数
up / down         # 向调用方/被调方移动帧
```

**核心调试手法**：崩溃/断点处先用 `bt` 看调用链，再切到各帧检查参数与局部变量，定位错误发生的完整路径。

---

## 8. 多线程调试

```bash
info threads                  # 列出线程
thread 3                      # 切换到线程 3
thread apply all bt           # 全部线程打印栈（崩溃必用）
set scheduler-locking on      # 单步时锁定其他线程
set detach-on-fork off        # 调试 fork 出的子进程
```

**多线程崩溃/死锁定位**：`thread apply all bt` 是首选——看每个线程停在哪，判断死锁环或崩溃线程。

---

## 9. 修改程序状态与函数调用

```bash
set var flag = false        # 修改局部/全局变量
call foo(1, "x")            # 手动调用函数
return                      # 让当前函数立即返回
jump *addr                  # 跳到指定地址（危险，谨慎）
```

用于绕过故障分支、复现特定路径、验证假设。

---

## 10. core dump 分析

程序崩溃（段错误等）会生成 **core 文件**，GDB 可离线分析崩溃现场：

```bash
ulimit -c unlimited          # 允许生成 core（先设置）
./app                        # 崩溃 → 生成 core
gdb ./app core               # 打开 core
bt                           # 立即查看崩溃调用栈
info threads                 # 多线程看崩溃线程
```

**最佳实践**：
- 发布版本也保留符号（或单独 debuginfo），便于崩溃后 `bt`。
- core dump 分析无需复现，直接还原崩溃瞬间。

---

## 11. 高级功能

### 11.1 观察点（watchpoint）

```bash
watch x                # x 被修改时停下
rwatch x               # x 被读时停下
awatch x               # 读写都停
```

用于追踪“谁改坏了变量”。

### 11.2 捕获（catch）

```bash
catch throw            # 异常抛出时停下
catch signal SIGSEGV   # 收到段错误信号时停下
```

### 11.3 断点命令脚本与 Python

```bash
commands 1            # 断点 1 命中时自动执行命令
> print x
> continue
> end
```

```python
# Python 扩展：定制命令、pretty-printer
(gdb) python print("hi")
```

### 11.4 TUI 界面

```bash
layout src        # 源码窗口
layout split      # 源码+汇编
```

---

## 12. 调试思路与实战流程

### 12.1 定位崩溃

```bash
gdb ./app core
bt            # 看崩溃栈
frame <n>     # 切到可疑帧
info locals   # 看变量
```

### 12.2 定位逻辑错误

1. 在可疑函数入口下断点，`run` 到断点。
2. `print` 参数，`next`/`step` 逐行跟踪。
3. 用 watchpoint 追踪关键变量变化。
4. 对比预期与实际的中间值，锁定错误分支。

### 12.3 排查死锁/卡死

```bash
gdb -p <pid>          # 附加到卡死进程
thread apply all bt   # 各线程栈 → 找等待环
```

---

## 13. 常见问题

| 问题 | 对策 |
|---|---|
| 断点无法命中 | 未 `-g`、优化导致行号漂移、断点打在已删除行 |
| 变量“optimized out” | 用 `-O0` 重编译 |
| 找不到源文件 | 用 `directory`/`set substitute-path` 指源码路径 |
| 多线程单步混乱 | `set scheduler-locking on` |
| 无 core | 检查 `ulimit -c` |
| 符号缺失 | 确保 `-g` 且未 strip（或保留 debuginfo） |

**配套工具**：崩溃分析亦可用 `addr2line`、`objdump`；数据竞争用 ThreadSanitizer（见编译器章节）。

---

> **自检**：能否用 `bt`+`frame`+`info locals` 定位一次崩溃？区分 `next` 与 `step`？如何用 watchpoint 追踪变量被谁修改？下一步请阅读 `多平台开发环境配置.md`。
