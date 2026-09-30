# 包管理 Vcpkg / Conan

> 第三方依赖管理是现代 C++ 工程的刚需。**vcpkg**（微软）与 **Conan**（JFrog）是两大主流 C++ 包管理器，解决“库从哪来、怎么构建、怎么接入工程”的问题。本文件以工程实战级讲解两者的安装、配置与工程集成。

---

## 目录

1. 为什么需要 C++ 包管理
2. Vcpkg：定位与体系
3. Vcpkg 安装与基本命令
4. Vcpkg 与 CMake 集成
5. Vcpkg manifest 模式（工程级依赖）
6. Conan：定位与体系
7. Conan 安装与基本命令
8. Conan 与 CMake 集成
9. 双平台与交叉编译
10. 对比与选型
11. 常见问题与最佳实践

---

## 1. 为什么需要 C++ 包管理

- 手动下载/编译第三方库：版本混乱、平台差异、链接错误频发。
- 包管理器提供：**统一获取、按平台/编译器构建、依赖解析、版本锁定、CMake 集成**。
- 对比：vcpkg 偏向“库直接可用 + CMake 友好”；Conan 偏向“灵活配方 + 跨平台可复现”。

---

## 2. Vcpkg：定位与体系

**定位**：微软开源的跨平台 C++ 包管理器，强调“以源码构建 + CMake 原生集成”。

**体系要点**：
- **端口（port）**：每个库对应一个构建配方（`ports/<name>/portfile.cmake`）。
- **三元组（triplet）**：描述目标平台/架构/运行时，如 `x64-linux`、`x64-windows`、`x64-windows-static-md`、`arm64-osx`。
- **manifest 模式（推荐）**：在工程里用 `vcpkg.json` 声明依赖，构建期自动安装。
- 默认**从源码构建**；也支持预编译缓存。

---

## 3. Vcpkg 安装与基本命令

### 3.1 安装

```bash
git clone https://github.com/microsoft/vcpkg
cd vcpkg
./bootstrap-vcpkg.sh        # Linux/macOS
.\bootstrap-vcpkg.bat       # Windows
./vcpkg integrate install   # （Windows）全局集成 MSBuild
```

### 3.2 核心命令

```bash
vcpkg search fmt                            # 搜索库
vcpkg install fmt:x64-linux                 # 安装（指定三元组）
vcpkg list                                  # 已安装库
vcpkg remove fmt                            # 移除
vcpkg update                                # 更新端口索引
vcpkg x-update-baseline                     # 更新基线（锁定版本）
vcpkg version fmt                           # 查看库版本
```

- 指定三元组：`vcpkg install <lib>:<triplet>`。
- 查看可用三元组：`vcpkg help triplet`。

---

## 4. Vcpkg 与 CMake 集成

### 4.1 传统（工具链文件）方式

```bash
# 方式一：命令行传入工具链文件
cmake -S . -B build \
      -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake

# 方式二：CMakePresets.json 或 CMAKE_TOOLCHAIN_FILE 变量
```

### 4.2 工具链文件作用

vcpkg 的工具链文件自动：设置 `CMAKE_PREFIX_PATH`、`CMAKE_LIBRARY_PATH`，把已安装库的路径注入 find_package，从而：

```cmake
find_package(fmt CONFIG REQUIRED)
target_link_libraries(app PRIVATE fmt::fmt)
```

**现代做法**：`find_package(fmt CONFIG REQUIRED)` 直接找到 vcpkg 安装的 `fmt` 并链接其导出目标。

---

## 5. Vcpkg manifest 模式（工程级依赖）

### 5.1 `vcpkg.json`

```json
{
  "name": "my-app",
  "version-string": "0.1.0",
  "dependencies": [
    "fmt",
    "spdlog",
    { "name": "openssl", "version>=": "3.0.0" }
  ],
  "builtin-baseline": "abcdef0123456789..."
}
```

### 5.2 使用

```bash
# 构建时自动按 manifest 安装依赖
cmake -S . -B build \
      -DCMAKE_TOOLCHAIN_FILE=$VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake \
      -DVCPKG_MANIFEST_INSTALL=ON
cmake --build build
```

- **`builtin-baseline`**：锁定 vcpkg 端口索引版本，保证**可复现构建**。
- 平台差异可用 `"platform": "windows"` 等字段限制依赖。

### 5.3 项目化组织

```text
project/
├─ CMakeLists.txt
├─ vcpkg.json          # 依赖声明
└─ src/
```

---

## 6. Conan：定位与体系

**定位**：JFrog 开源的**去中心化、配方驱动**的 C++ 包管理器，强调跨平台可复现与二进制缓存。

**体系要点**：
- **配方（recipe，`conanfile.py`）**：描述包的构建方式、依赖、选项。
- **远程仓库（remote）**：拉取/上传包（默认 ConanCenter）。
- **settings**：操作系统、编译器、架构、构建类型（决定二进制变体）。
- **二进制度（binary compatibility）**：不同 settings 组合对应不同二进制包。
- **conanfile.txt / conanfile.py**：工程依赖声明。

---

## 7. Conan 安装与基本命令

### 7.1 安装

```bash
pip install conan
conan --version
```

### 7.2 核心命令

```bash
conan profile detect                       # 生成默认 profile
conan search fmt --remote=conancenter      # 搜索
conan install --requires=fmt/11.1.1 --output-folder=build   # 安装到 build
conan list "*"                             # 列出包
conan remove fmt/*                         # 移除
```

- `conanfile.txt`（简单依赖声明）：

```text
[requires]
fmt/11.1.1
spdlog/1.14.1

[generators]
CMakeDeps
CMakeToolchain
```

---

## 8. Conan 与 CMake 集成

### 8.1 生成器

- **`CMakeToolchain`**：生成 CMake 工具链文件（定义编译设置）。
- **`CMakeDeps`**：生成 `find_package` 所需的 config 文件（导出目标）。

### 8.2 集成流程

```bash
# 1. 安装依赖并生成 CMake 集成文件
conan install . --output-folder=build --build=missing

# 2. 配置 CMake（引用生成的工具链 + 前缀路径）
cmake -S . -B build \
      -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake \
      -DCMAKE_PREFIX_PATH=build

# 3. 构建
cmake --build build
```

CMake 中：

```cmake
find_package(fmt CONFIG REQUIRED)
target_link_libraries(app PRIVATE fmt::fmt)
```

- `--build=missing`：若某个 settings 组合无预编译二进制，则现场构建。

---

## 9. 双平台与交叉编译

### 9.1 settings 决定二进制

Conan 依据 profile（OS、编译器、架构、build_type）生成/匹配二进制包：

```bash
conan profile show default        # 查看当前 settings
```

不同架构：`conan install . -s arch=armv8 ...`；交叉编译通过自定义 profile 指定编译器与 sysroot。

### 9.2 vcpkg 三元组

```bash
vcpkg install fmt:x64-linux       # Linux
vcpkg install fmt:x64-windows     # Windows
vcpkg install fmt:arm64-osx       # macOS ARM
```

静态/动态运行时：Windows 三元组 `x64-windows`（动态 CRT）vs `x64-windows-static`（静态 CRT）。

---

## 10. 对比与选型

| 维度 | vcpkg | Conan |
|---|---|---|
| 维护方 | 微软 | JFrog |
| 模式 | 端口（port）+ manifest | 配方（recipe）+ profile |
| 与 CMake | 工具链文件原生集成 | CMakeDeps+CMakeToolchain 生成器 |
| 版本锁定 | `builtin-baseline`（全局） | 精确版本 + lockfile |
| 二进制复用 | 源码构建为主 + 缓存 | 强二进制缓存 |
| 学习曲线 | 平缓 | 较陡（配方/profile） |
| 适用 | Windows 生态、快速上手 | 复杂跨平台、CI 可复现 |

**选型建议**：
- Windows + Visual Studio 生态、希望“即装即用” → **vcpkg**。
- 复杂多平台矩阵、CI 严格复现、需要自定义配方 → **Conan**。
- 两者可在不同项目共存。

---

## 11. 常见问题与最佳实践

### 11.1 常见问题

| 问题 | 对策 |
|---|---|
| find_package 找不到库 | 确认工具链文件已设置、库已安装、三元组匹配 |
| 编译报“找不到头文件” | 检查 `find_package` + `target_link_libraries` 传导包含路径 |
| ABI 不匹配 | settings/三元组与编译器、CRT 一致 |
| 版本漂移 | 用 manifest/baseline/lockfile 锁定版本 |
| 重复安装多个变体 | 明确三元组/settings |
| CI 不可复现 | 提交 lockfile、固定包源 |

### 11.2 最佳实践

- **项目级声明依赖**（vcpkg manifest / conanfile），不依赖全局安装。
- **锁定版本**：`builtin-baseline`（vcpkg）、`conan.lock`（Conan）。
- **CI 缓存**二进制，避免每次全量构建。
- **工具链文件/生成器**统一由 CMake 引入，避免手写 `-l`。
- 第三方库的导出目标（`fmt::fmt` 等）是首选链接方式。

---

> **自检**：能否用 manifest 声明并集成 vcpkg 依赖到 CMake？区分 vcpkg 三元组与 Conan settings？说出两者与 CMake 的集成差异？下一步请阅读 `GDB调试工具.md`。
