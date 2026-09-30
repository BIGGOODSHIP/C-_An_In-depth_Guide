# CMake 核心指令

> CMake 是当今 C++ 工程**事实标准**的跨平台构建系统生成器：它不直接编译，而是生成各平台的构建文件（Makefile、Ninja、Visual Studio 工程）。本文件以工程实战级讲解 CMake 的定位、工程构建逻辑、核心指令与完整工程组织。

---

## 目录

1. CMake 的定位与工作流程
2. 核心概念：目标（target）、属性、作用域
3. 最小工程与基础指令
4. 变量与选项
5. 目标构建指令（add_library / add_executable）
6. 依赖与链接（target_link_libraries）
7. 包含路径与编译选项
8. 条件与生成器表达式
9. 多目录组织（子工程）
10. 查找依赖（find_package）
11. 常用模块与工具链
12. 完整工程实战
13. 调试与常见问题

---

## 1. CMake 的定位与工作流程

**定位**：跨平台、跨编译器的构建系统生成器。

```mermaid
flowchart LR
    CMakeLists -->|configure| BuildFiles[Makefile/Ninja/VS工程]
    BuildFiles -->|build| Targets[可执行文件/库]
```

**工作流程（两步）**：
1. **配置（configure）**：解析 `CMakeLists.txt`，检测工具链、生成构建文件。
2. **构建（build）**：调用底层工具（`make`/`ninja`/MSBuild）实际编译链接。

```bash
# 配置（在 build 目录，保持源码目录干净）
cmake -S . -B build

# 构建
cmake --build build

# 安装/测试
cmake --install build
ctest --test-dir build
```

---

## 2. 核心概念

### 2.1 目标（target）

**目标是 CMake 的核心抽象**：一个可执行文件（`add_executable`）或库（`add_library`）。目标是依赖图节点，属性、编译选项、链接关系都挂在目标上。

### 2.2 属性（property）

目标/目录/源码文件携带属性：`CXX_STANDARD`、`INCLUDE_DIRECTORIES`、`COMPILE_DEFINITIONS` 等。属性决定编译与链接行为。

### 2.3 作用域与目录

每个 `CMakeLists.txt` 是一个**目录作用域**。子目录通过 `add_subdirectory` 引入，父目录变量默认**不向子目录传递**（需 `set(... PARENT_SCOPE)` 或缓存变量）。

---

## 3. 最小工程与基础指令

```cmake
cmake_minimum_required(VERSION 3.20)     # 最低版本
project(MyApp CXX)                        # 工程名与语言

add_executable(app main.cpp)              # 生成可执行文件 app
```

### 基础指令一览

| 指令 | 作用 |
|---|---|
| `cmake_minimum_required` | 声明最低 CMake 版本 |
| `project(name LANGUAGES)` | 定义工程名、语言 |
| `set(var value)` | 定义变量 |
| `message(...)` | 输出信息（STATUS/WARNING/FATAL_ERROR） |
| `add_executable(name src...)` | 生成可执行目标 |
| `add_library(name src...)` | 生成库目标 |
| `add_subdirectory(dir)` | 引入子目录工程 |
| `target_include_directories` | 目标头文件路径 |
| `target_link_libraries` | 目标链接依赖 |
| `find_package(name)` | 查找第三方库 |
| `option(name desc default)` | 定义布尔选项 |

---

## 4. 变量与选项

### 4.1 普通变量

```cmake
set(SOURCES main.cpp util.cpp)
set(CMAKE_CXX_STANDARD 17)        # 全局 C++ 标准
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)     # 不用 GNU 扩展
```

### 4.2 缓存变量与选项

```cmake
option(BUILD_TESTS "Enable tests" ON)      # 可在命令行 -DBUILD_TESTS=OFF
set(MY_PATH "..." CACHE PATH "doc")        # 缓存变量，供用户覆盖
```

命令行覆盖：`cmake -S . -B build -DBUILD_TESTS=OFF -DCMAKE_BUILD_TYPE=Release`

### 4.3 生成器表达式（generator expression）

`$<...>` 在构建期按配置求值：

```cmake
target_compile_options(app PRIVATE $<$<CONFIG:Debug>:-g>)   # 仅 Debug 加
target_include_directories(app PRIVATE $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>)
```

---

## 5. 目标构建指令

### 5.1 `add_executable`

```cmake
add_executable(app
    main.cpp
    core/logic.cpp
)
```

### 5.2 `add_library` 的三种类型

| 类型 | 语义 |
|---|---|
| `STATIC` | 静态库 `.a/.lib` |
| `SHARED` | 动态库 `.so/.dll` |
| `INTERFACE` | 仅接口（无源文件，只传播属性，如头文件-only 库） |
| `OBJECT` | 对象库（编译不链接，供复用） |

```cmake
add_library(mylib STATIC math.cpp util.cpp)
add_library(header_only INTERFACE)      # 纯头文件库
target_include_directories(header_only INTERFACE include/)
```

### 5.3 `add_library` 从已有对象/导入库

`add_library(name IMPORTED)`：引入已存在的库（不构建）。

---

## 6. 依赖与链接

### 6.1 `target_link_libraries`

```cmake
add_executable(app main.cpp)
target_link_libraries(app PRIVATE mylib pthread)
```

可见性关键字：`PRIVATE`（仅自己）、`PUBLIC`（自己+使用者）、`INTERFACE`（仅使用者）。**链接关系决定依赖属性的传播**：

- `PRIVATE`：依赖仅用于实现，不传导给下游。
- `PUBLIC`：依赖是接口一部分，传导（如头文件包含它的头）。
- `INTERFACE`：仅下游使用（自身实现不依赖）。

### 6.2 传递依赖（transitive dependencies）

CMake 把 `target_link_libraries` 的依赖自动传递给消费该目标的下游目标，实现**依赖图自动传播**，是 CMake 现代工程的核心优势。

---

## 7. 包含路径与编译选项

```cmake
# 头文件路径（PUBLIC 传导给使用者）
target_include_directories(mylib
    PUBLIC  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    PRIVATE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
)

# 编译选项与宏
target_compile_options(app PRIVATE -Wall -Wextra -Werror)
target_compile_definitions(app PRIVATE NDEBUG VERSION=2)
```

---

## 8. 条件与生成器表达式

### 8.1 条件

```cmake
if(MSVC)
  target_compile_options(app PRIVATE /W4)
elseif(UNIX)
  target_compile_options(app PRIVATE -Wall)
endif()

if(BUILD_TESTS)
  add_subdirectory(tests)
endif()
```

常用判断：`MSVC`、`UNIX`、`WIN32`、`CMAKE_BUILD_TYPE`、`CMAKE_CXX_COMPILER_ID`（`GNU`/`Clang`/`MSVC`）。

### 8.2 平台/配置差异

```cmake
set(CMAKE_BUILD_TYPE Release)   # 或命令行指定
# Release 默认 -O3（GCC/Clang）、/O2（MSVC）
```

---

## 9. 多目录组织（子工程）

```text
project/
├─ CMakeLists.txt        # 顶层
├─ src/CMakeLists.txt    # 应用
├─ lib/CMakeLists.txt    # 库
└─ tests/CMakeLists.txt  # 测试
```

顶层：

```cmake
cmake_minimum_required(VERSION 3.20)
project(MyProject CXX)
add_subdirectory(lib)
add_subdirectory(src)
option(BUILD_TESTS "tests" ON)
if(BUILD_TESTS) add_subdirectory(tests) endif()
```

各子目录自行 `add_*`，通过目标名跨目录引用（无需传变量）。

---

## 10. 查找依赖（find_package）

### 10.1 模块模式 vs 配置模式

```cmake
find_package(OpenSSL REQUIRED)        # 模块/配置模式
find_package(fmt CONFIG REQUIRED)     # 强制配置模式（现代库提供 CMake config）
```

- **模块模式**：找 `FindXXX.cmake` 模块，定义变量 `XXX_INCLUDE_DIR`/`XXX_LIBRARIES`（传统）。
- **配置模式**：库自带 `XXXConfig.cmake`，导出**目标**（现代，推荐），如 `fmt::fmt`。

```cmake
find_package(fmt CONFIG REQUIRED)
target_link_libraries(app PRIVATE fmt::fmt)   # 现代目标式链接
```

### 10.2 设置搜索路径

```cmake
set(CMAKE_PREFIX_PATH "/opt/mylib")
find_package(mylib CONFIG REQUIRED)
```

---

## 11. 常用模块与工具链

| 模块 | 用途 |
|---|---|
| `CTest` | 测试框架（`enable_testing()`、`add_test`） |
| `CPack` | 打包安装 |
| `FetchContent` | 构建期拉取第三方源码 |
| `CheckIncludeFile` 等 | 特性检测 |
| 工具链文件 | `-DCMAKE_TOOLCHAIN_FILE=...`（交叉编译、vcpkg） |

```cmake
include(CTest)
enable_testing()
add_test(NAME unit COMMAND app --test)
```

```cmake
include(FetchContent)
FetchContent_Declare(fmt GIT_REPOSITORY https://github.com/fmtlib/fmt ...)
FetchContent_MakeAvailable(fmt)
```

---

## 12. 完整工程实战

```cmake
# 顶层 CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(MyApp CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

option(BUILD_TESTS "Build tests" ON)
option(BUILD_SHARED_LIBS "Build shared libs" OFF)

# 核心库
add_library(core
    src/core/math.cpp
    src/core/logic.cpp
)
target_include_directories(core PUBLIC include)
target_compile_options(core PRIVATE -Wall -Wextra)

# 可执行
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE core)

# 测试
if(BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()
```

```cmake
# tests/CMakeLists.txt
add_executable(unit test_core.cpp)
target_link_libraries(unit PRIVATE core)
add_test(NAME core COMMAND unit)
```

**构建命令**：

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j8
ctest --test-dir build
```

---

## 13. 调试与常见问题

| 问题 | 对策 |
|---|---|
| 找不到头文件/库 | 检查 `target_include_directories`、`find_package`、`CMAKE_PREFIX_PATH` |
| 标准版本不对 | `set(CMAKE_CXX_STANDARD ...)` + REQUIRED |
| 链接顺序问题 | 用 `target_link_libraries` 显式依赖，避免手写 `-l` |
| 变量不传递到子目录 | `set(... PARENT_SCOPE)` 或缓存变量 |
| 找不到 `FindXXX` | 模块未提供/路径未设 |
| 生成器表达式不生效 | 仅用于 build/install 接口，检查写法 |

**调试工具**：

```bash
cmake -S . -B build --trace        # 追踪指令执行
cmake --build build --verbose      # 打印完整编译命令
cmake -LA build                    # 列出缓存变量
cmake --build build --target help  # 列出可用目标
```

---

> **自检**：能否解释 CMake“配置→构建”两步与 target 依赖传播？`PUBLIC/PRIVATE/INTERFACE` 的区别？用 `find_package` + 目标式链接？下一步请阅读 `包管理Vcpkg-Conan.md`。
