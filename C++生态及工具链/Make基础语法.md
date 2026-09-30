# Make 基础语法

> Make 是最经典的**增量构建工具**：通过描述文件（`Makefile`）声明“目标-依赖-命令”关系，只在源文件变化时重建必要产物，避免全量重编译。本文件以工程实战级讲解 Make 的核心工作机制、基础语法与高频写法。

---

## 目录

1. Make 的定位与工作机制
2. Makefile 核心结构
3. 规则、目标、依赖与命令
4. 变量（make 变量）
5. 自动变量
6. 常用内置函数
7. 模式规则与隐式规则
8. 条件与伪目标
9. 实战：C++ 工程 Makefile
10. 常见问题与调试

---

## 1. Make 的定位与工作机制

**定位**：增量构建工具，依据**文件时间戳**判断是否需要重建。

```mermaid
flowchart LR
    SRCC[源文件变化] -->|比目标新| R[触发规则命令]
    R -->|更新| OBJ[对象文件]
    OBJ -->|比可执行新| LINK[链接命令]
```

**增量原理**：Make 比较**目标**与**依赖**的时间戳。若任一依赖比目标新（或目标不存在），则执行该规则的命令重建目标。

**为什么需要 Make**：
- 大型工程全量重编译昂贵；Make 只重编译变化的文件。
- 声明式描述构建依赖，可移植（GNU Make 跨平台）。
- 相比 CMake：更底层、更贴近命令；适合中小项目与学习构建本质。

---

## 2. Makefile 核心结构

```makefile
# 注释
目标: 依赖...
	命令（必须以 Tab 缩进）
```

```makefile
app: main.o util.o
	g++ main.o util.o -o app

main.o: main.cpp
	g++ -c main.cpp -o main.o

util.o: util.cpp
	g++ -c util.cpp -o util.o

clean:
	rm -f app *.o
```

- **目标（target）**：要生成的产物（文件或伪目标）。
- **依赖（prerequisite）**：构建目标所需的前置文件。
- **命令（recipe）**：构建动作，**每行必须以 Tab 开头**（Make 的经典约束）。

---

## 3. 规则、目标、依赖与命令

### 3.1 执行流程

```bash
make app
```

1. 目标 `app` 不存在，或其依赖 `main.o`/`util.o` 比它新 → 执行命令。
2. 依次递归检查 `main.o` 是否需重建（`main.cpp` 比 `main.o` 新则重建）。
3. 全部依赖就绪后，执行 `app` 的命令。

### 3.2 默认目标

**第一个目标**是默认目标。执行 `make` 等价于 `make 第一个目标`。

### 3.3 伪目标（.PHONY）

声明不生成文件的目标（如 `clean`）：

```makefile
.PHONY: clean
clean:
	rm -f *.o app
```

没有 `.PHONY` 时，若恰好存在名为 `clean` 的文件，Make 会误判“已是最新”而不执行。

### 3.4 命令前缀

| 前缀 | 含义 |
|---|---|
| `-` | 忽略本命令错误继续执行 |
| `@` | 不回显命令本身 |

```makefile
clean:
	@rm -f *.o app      # 静默执行
	-rm -f extra.tmp    # 出错也继续
```

---

## 4. 变量（make 变量）

### 4.1 定义与使用

```makefile
CXX    = g++
CXXFLAGS = -std=c++17 -Wall -O2
LDFLAGS  = -L.
LIBS     = -lmylib

app: main.o
	$(CXX) $(CXXFLAGS) $^ $(LDFLAGS) $(LIBS) -o $@
```

- 变量以 `$()` 或 `${}` 引用。
- `=` 为**递归展开**（引用时求值）；`:=` 为**简单展开**（定义时求值）；`?=` 仅在未定义时赋值；`+=` 追加。

### 4.2 变量赋值差异

```makefile
A = $(B)      # 递归：使用 A 时才展开 B
B = world
A := $(B)     # 简单：立即展开
```

### 4.3 常用变量命名约定

`CC`（C 编译器）、`CXX`（C++ 编译器）、`CFLAGS`、`CXXFLAGS`、`CPPFLAGS`（预处理）、`LDFLAGS`、`LDLIBS`。

---

## 5. 自动变量

在规则命令中使用的**由 Make 自动填充**的变量：

| 变量 | 含义 |
|---|---|
| `$@` | 当前目标的文件名 |
| `$<` | 第一个依赖 |
| `$^` | 全部依赖（去重） |
| `$?` | 比目标新的依赖 |
| `$*` | 去除后缀的目标名（模式规则中） |

```makefile
%.o: %.cpp
	$(CXX) $(CXXFLAGS) -c $< -o $@
```

---

## 6. 常用内置函数

### 6.1 文件名函数

| 函数 | 作用 |
|---|---|
| `$(wildcard pattern)` | 展开匹配的文件（如 `$(wildcard src/*.cpp)`） |
| `$(patsubst pattern,replacement,text)` | 模式替换 |
| `$(notdir names)` | 去掉目录 |
| `$(basename name)` | 去后缀 |

### 6.2 文本函数

| 函数 | 作用 |
|---|---|
| `$(subst from,to,text)` | 子串替换 |
| `$(foreach v,list,text)` | 循环生成 |
| `$(sort list)` | 排序去重 |
| `$(filter pattern,list)` | 过滤 |
| `$(word n,text)` | 取第 n 个词 |
| `$(call name,args...)` | 调用自定义函数 |

### 6.3 实战组合：从源文件自动推导对象文件

```makefile
SRCS := $(wildcard src/*.cpp)
OBJS := $(patsubst src/%.cpp, build/%.o, $(SRCS))
```

---

## 7. 模式规则与隐式规则

### 7.1 模式规则

用 `%` 通配匹配一类文件：

```makefile
build/%.o: src/%.cpp
	$(CXX) $(CXXFLAGS) -Iinclude -c $< -o $@
```

### 7.2 隐式规则

Make 内置规则自动推导：`.cpp` → `.o` 使用变量 `CXX`/`CXXFLAGS`；`.o` → 可执行使用 `LINK.o` 等。可直接写出依赖而不写命令：

```makefile
app: main.o util.o      # 隐式规则完成 .o 生成与链接
```

- 隐式规则简化 Makefile，但显式模式规则更可控（推荐）。

### 7.3 `.PHONY` 与常用目标

```makefile
.PHONY: all clean install
all: app
clean: ...
install: ...   # 拷贝到安装目录
```

---

## 8. 条件与 include

### 8.1 条件判断

```makefile
ifeq ($(DEBUG),1)
CXXFLAGS += -g -O0
else
CXXFLAGS += -O2 -DNDEBUG
endif
```

### 8.2 包含其他 Makefile

```makefile
include common.mk
```

### 8.3 `-j` 并行

```bash
make -j4      # 4 个并行作业，加快增量构建
```

---

## 9. 实战：C++ 工程 Makefile（完整示例）

```makefile
# 工程变量
CXX      := g++
CXXFLAGS := -std=c++17 -Wall -Wextra -O2
CPPFLAGS := -Iinclude
LDFLAGS  := -Llib
LDLIBS   := -lmylib -lpthread

# 源文件与对象文件推导
SRCDIR := src
BUILDDIR := build
SRCS := $(wildcard $(SRCDIR)/*.cpp)
OBJS := $(patsubst $(SRCDIR)/%.cpp,$(BUILDDIR)/%.o,$(SRCS))

TARGET := app

.PHONY: all clean install

all: $(TARGET)

# 目标链接
$(TARGET): $(OBJS)
	$(CXX) $^ $(LDFLAGS) $(LDLIBS) -o $@

# 模式规则：编译对象
$(BUILDDIR)/%.o: $(SRCDIR)/%.cpp | $(BUILDDIR)
	$(CXX) $(CXXFLAGS) $(CPPFLAGS) -c $< -o $@

# 目录不存在的订单依赖
$(BUILDDIR):
	mkdir -p $(BUILDDIR)

# 依赖文件（可选，自动生成头文件依赖）
$(BUILDDIR)/%.d: $(SRCDIR)/%.cpp | $(BUILDDIR)
	$(CXX) $(CPPFLAGS) -MM -MT '$(BUILDDIR)/$*.o' $< -o $@

clean:
	rm -rf $(BUILDDIR) $(TARGET)

install:
	install -m 755 $(TARGET) /usr/local/bin/
```

### 9.1 使用

```bash
make          # 构建默认目标
make clean    # 清理
make -j8      # 并行构建
make DEBUG=1  # 覆盖变量（若 Makefile 支持覆盖）
```

---

## 10. 常见问题与调试

| 问题 | 原因/对策 |
|---|---|
| `missing separator. Stop.` | 命令未用 **Tab** 缩进 |
| `No rule to make target` | 依赖文件不存在/路径错 |
| 目标总是重建 | 时间戳比较；检查伪目标 |
| 头文件变更不触发重建 | 依赖未包含头文件 → 用自动依赖（`-MM` 生成 `.d`） |
| 变量值不对 | `=` vs `:=` 展开时机 |
| 找不到库 | `LDFLAGS`/`LDLIBS` 未设置 |

**调试工具**：

```bash
make -n        # 只打印要执行的命令，不执行（dry run）
make -d        # 输出详细决策（为何重建/不重建）
make -p        # 打印 Makefile 数据库（变量、规则）
make -B        # 强制全部重建
```

---

> **自检**：能否解释 Make 的增量判定原理？写出 `$@/$</$^` 的语义？用 `wildcard`/`patsubst` 推导对象文件？下一步请阅读 `CMake核心指令.md`。
