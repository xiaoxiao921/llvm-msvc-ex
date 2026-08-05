# `llvm-msvc-ex`

**跳转到中文文档：[中文文档](#中文文档)**

---

## Project Status

This repository is a fork of [killvxk](https://github.com/killvxk)'s `llvm-msvc-ex`. Purpose of this fork is to:

- Enable automated GitHub Actions builds
- Provide reliable build infrastructure
- Produce release artifacts

---

## Overview

`llvm-msvc-ex` is a compiler based on LLVM that aims to provide the same experience as MSVC on Windows, while adding support for obfuscation and other security‑related features. Unlike MSVC, it allows you to use naked functions anywhere and provides custom extensions.

<img width="732" height="697" alt="image" src="https://github.com/user-attachments/assets/743ba706-24c4-48f3-92af-cc2f9e9f055f" />

---

## Features

- Compatible with MSVC syntax as much as possible
- Almost fully compatible with SEH (Structured Exception Handling)
- Special intrinsic functions (`__vmx_vmread`/`__vmx_write`)
- Supports x64/ARM64 Windows drivers
- Supports AArch64 Android GKI drivers
- Allows naked X64 inline assembly
- Enables multi‑core compilation
- Supports `/MP` with precompiled headers
- Supports Link‑Time Optimization (LTO) – Whole Program Optimization (`/GL`) compatibility is currently **unverified**

---

## Included Obfuscation Passes

The following obfuscation passes are included:

- Bogus Control Flow (`-bcf`)
- Split Basic Block (`-split`)
- Control Flow Flattening (`-fla`)
- Instruction Substitution (`-sub`)
- MBA Substitution (`-mba-subs`)
- Indirect Call (`-ind-call`)
- String Encryption (`-string-obfus`)
- Constant Variable XOR (`-const-obfus`)
- VM Flattening (`-vm-fla`)

---

## Roadmap

- [x] Anti‑symbolic execution and anti‑memory tracing (`vm-fla-sym`)
- [x] Data encryption for VM flattening (`vm-fla-enc`)
- [x] Indirect global variable access in `vm-fla-enc`
- [x] VM flattening levels 0–7 (7 = strongest, 0 = weakest, default = 7)
- [x] Basic VMP‑like protection
- [x] Function combination (`combine`)
- [x] Enhanced flattening (`x-fla-enh`)
- [x] `x-full` – applies `vm-fla-level=7` to a function
- [x] String encryption with combine support
- [x] Custom function split/merge via `combine_func[tag_number]`
- [x] Custom calling convention (`custom-cc`) for parameter passing and return values
- [x] Anti‑IDA (ported from other OLLVM variants)
- [x] Self‑build support
- [x] `-rich` for `lld-link` to build PE with rich header
- [x] `-x-pic` – Windows `-fPIC` equivalent (`-mllvm -x-pic -mllvm -xpic-global -mllvm -xpic-memset -mllvm -xpic-memcpy`)
- [x] `-x-vmobf` (next‑gen VM; runtime not public)
- [x] `-x-mba` (simplified MBA mode)
- ~~[ ] MBA‑subs bug fixes~~
- ~~[ ] Port xVMP~~
- ~~[ ] `x-var-rot` pending~~
- [ ] New functions

---

## FAQ

### Why this project?

- Clang follows GCC standards, while MSVC has its own unique syntax.
- Some code is too experimental for upstream submission.
- Waiting for official fixes takes too long.

### Can it run on Linux?

Yes.

### Can it run on macOS?

Yes.

---

## Building llvm-msvc-ex

### Supported Build Environment

Currently supported and tested:

- Visual Studio 2022 only
- Windows SDK `10.0.26100`
- MSVC toolset `14.44`
- Architecture: `amd64`

### Build Command

A **single** build step is required:

```batch
mkdir build
cd build
cmake .. -G "Visual Studio 17 2022" -A x64 ^
  -DLLVM_ENABLE_PROJECTS="clang;lld;lldb;compiler-rt" ^
  -DLLVM_ENABLE_RPMALLOC=ON ^
  -DLLDB_ENABLE_PYTHON=OFF ^
  -DLLVM_INCLUDE_TESTS=OFF ^
  -DLLVM_INCLUDE_EXAMPLES=OFF ^
  -DLLVM_ENABLE_LIBXML2=OFF ^
  -DLLVM_ENABLE_ZLIB=OFF ^
  -DLLVM_TARGETS_TO_BUILD=X86 ^
  -DLLVM_OBFUSCATION_LINK_INTO_TOOLS=ON ^
  -DCMAKE_BUILD_TYPE=RelWithDebInfo ^
  -DLLVM_USE_CRT_RELEASE=MT

msbuild /m -p:Configuration=RelWithDebInfo INSTALL.vcxproj
```

> **Note:** `CMAKE_INSTALL_PREFIX` is not hardcoded; it defaults to the build directory for portability.

---

## Using llvm-msvc-ex

### Installation

Releases are distributed as **ZIP archives**, not installers. Simply extract the archive to any location.

### Visual Studio Integration

Create a `Directory.Build.props` file in your solution or project directory:

```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <LLVMInstallDir>C:\Path\To\Extracted\LLVM</LLVMInstallDir>
  </PropertyGroup>
</Project>
```

### Toolset Configuration

The toolset version must be set in your project – either in `.vcxproj` or via the Visual Studio UI.

**Example `.vcxproj` snippet:**

```xml
<PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Debug|Win32'" Label="Configuration">
  <ConfigurationType>Application</ConfigurationType>
  <UseDebugLibraries>true</UseDebugLibraries>
  <CharacterSet>Unicode</CharacterSet>
  <PlatformToolset>ClangCL</PlatformToolset>
  <LLVMToolsVersion>777</LLVMToolsVersion>
</PropertyGroup>
```

> The `LLVMToolsVersion` value depends on the specific release.

---

## Obfuscation Configuration

### Important Notes

- **`[[clang::annotate("...")]]`** – applies only to annotated functions.
- **`-mllvm`** – configures LLVM passes globally.

These are **different** mechanisms; not all command‑line options have an annotation equivalent.

### Optimization Notes

- `/O2` is enabled by default in Release builds.
- Some obfuscation passes rely on optimisation passes or behave better with optimisations enabled.

### `/GL` Warning

**Disable `/GL` before using obfuscation** – Whole Program Optimisation is not verified with these passes.

---

## Obfuscation Examples

### Command‑Line Examples

#### Maximum Protection (may produce binaries >100 MB)

```bash
-mllvm -data-obfus -mllvm -const-obfus -mllvm -string-obfus -mllvm -ind-call -mllvm -vm-fla -mllvm -fla -mllvm -sub -mllvm -sub_loop=1 -mllvm -split -mllvm -split_num=3 -mllvm -bcf -mllvm -bcf_loop=1 -mllvm -bcf_prob=40 -mllvm -vm-fla-level=7 -mllvm -x-fla-enh -mllvm -x-var-rot -mllvm -x-combine
```

#### Lightweight Mode

```bash
-mllvm -data-obfus -mllvm -const-obfus -mllvm -string-obfus -mllvm -ind-call -mllvm -vm-fla -mllvm -vm-fla-level=0 -mllvm -x-fla-enh -mllvm -x-combine -mllvm -x-linear -mllvm -ida-obfus
```

### Function Annotation Examples

```cpp
[[clang::annotate("x-vm,x-full,x-cfg,custom-cc")]]
void crypt_func1(uint8_t *var, uint8_t *key, size_t var_size, size_t key_size) {
    for (auto i = 0; i < var_size; i++) {
        var[i] ^= key[i % key_size];
    }
}

[[clang::annotate("x-cfg,ind-br,alias-access,custom-cc")]]
void crypt_func2(uint8_t *var, uint8_t *key, size_t var_size, size_t key_size) {
    for (auto i = 0; i < var_size; i++) {
        var[i] ^= key[i % key_size];
    }
}

[[clang::annotate("x-cfg,x-vm,ind-br,alias-access,custom-cc")]]
void crypt_func3(uint8_t *var, uint8_t *key, size_t var_size, size_t key_size) {
    for (auto i = 0; i < var_size; i++) {
        var[i] ^= key[i % key_size];
    }
}
```

### Function Combination Example

Functions with the same tag (`combine_func[tag]`) are combined:

```cpp
[[clang::annotate("combine_func[tag1]")]]
int a1(int a, int b) {
    printf("%d , %d\r\n", a, b);
    printf("%x\r\n", a ^ b);
    return a + b;
}

[[clang::annotate("combine_func[tag1]")]]
int a2(int a, int b) {
    std::cout << "hello1" << std::endl;
    for (auto i = std::min(a, b); i < std::max(a, b); i++) {
        printf("%x,", i);
    }
    printf("\n");
    return a * b + a1(a, b);
}

[[clang::annotate("combine_func[tag2]")]]
int a3(int a, int b) {
    printf("%d , %d\r\n", a + 1, b + 2);
    printf("%x\r\n", a ^ b);
    return a + b + a ^ b + a2(a, b);
}
```

---

## Contributing

- [Contribution Practice Guide](https://github.com/HyunCafe/contribute-practice)
- [GitHub Contributing to Projects](https://docs.github.com/en/get-started/quickstart/contributing-to-projects)

---

## Learning LLVM

Check out [awesome-llvm-security](https://github.com/gmh5225/awesome-llvm-security).

---

## Credits

- LLVM
- Various anonymous contributors

---

# 中文文档

## 项目状态

本仓库是 [killvxk](https://github.com/killvxk) 的 `llvm-msvc-ex` 的一个复刻。本复刻的目的在于：

- 启用 GitHub Actions 自动构建
- 提供可靠的构建基础设施
- 提供发布工件（release artifacts）

---

## 概述

`llvm-msvc-ex` 是一个基于 LLVM 的编译器，旨在 Windows 上提供与 MSVC 相同的使用体验，同时增加混淆及其他安全相关特性。与 MSVC 不同，它允许你在任何地方使用 naked 函数，并提供自定义扩展。

<img width="732" height="697" alt="image" src="https://github.com/user-attachments/assets/743ba706-24c4-48f3-92af-cc2f9e9f055f" />

---

## 特性

- 尽可能兼容 MSVC 语法
- 几乎完全兼容 SEH（结构化异常处理）
- 内置特殊 intrinsic 函数（`__vmx_vmread`/`__vmx_write`）
- 支持 x64/ARM64 Windows 驱动程序
- 支持 AArch64 Android GKI 驱动程序
- 允许 naked X64 内联汇编
- 支持多核编译
- 支持使用预编译头时的 `/MP`
- 支持链接时优化（LTO）—— `/GL`（全程序优化）的兼容性**尚未验证**

---

## 内置混淆 Pass

包含以下混淆 Pass：

- 虚假控制流（`-bcf`）
- 基本块分割（`-split`）
- 控制流平坦化（`-fla`）
- 指令替换（`-sub`）
- MBA 替换（`-mba-subs`）
- 间接调用（`-ind-call`）
- 字符串加密（`-string-obfus`）
- 常量变量 XOR（`-const-obfus`）
- VM 平坦化（`-vm-fla`）

---

## 路线图

- [x] 在 `vm-fla-sym` 中添加反符号执行和反内存追踪
- [x] `vm-fla-enc` 对 VM 平坦化的部分数据加密
- [x] 在 `vm-fla-enc` 中使用间接全局变量访问
- [x] `vm-fla-level` 0~7 共 8 个等级，7 最强，0 最弱，默认 7
- [x] 弱鸡 VMP 加入
- [x] 添加 `combine` 功能
- [x] 添加 `x-fla-enh`（增强平坦化）
- [x] `x-full` 功能：在函数上使用 `vm-fla-level=7`
- [x] 字符串加密等类似功能加上 `combine`
- [x] 自定义分割合并 `combine_func[tag_number]` 模式
- [x] 自定义调用约定（`custom-cc`）参数传递和返回值方式
- [x] 反 IDA（从其他 OLLVM 代码移植）
- [x] 自编译（self build）
- [x] 为 `lld-link` 添加 `-rich` 选项，生成带 rich header 的 PE
- [x] Windows 下的 `-fPIC` 等效：`-mllvm -x-pic -mllvm -xpic-global -mllvm -xpic-memset -mllvm -xpic-memcpy`
- [x] `-x-vmobf`（下一代 VM，VM 运行时未公开）
- [x] `-x-mba`（简易 MBA 模式）
- ~~[ ] MBA‑subs 的 bug 修复~~（已划掉）
- ~~[ ] 移植 xVMP~~（已划掉）
- ~~[ ] `x-var-rot` 待处理~~（已划掉）
- [ ] 新功能

---

## 常见问题

### 为什么要做这个项目？

- Clang 遵循 GCC 标准，而 MSVC 有自己的独特语法。
- 部分代码过于实验性，无法向上游提交。
- 等待官方修复耗时太长。

### 可以在 Linux 上运行吗？

可以。

### 可以在 macOS 上运行吗？

可以。

---

## 构建 llvm-msvc-ex

### 支持的构建环境

当前支持并测试：

- 仅 Visual Studio 2022
- Windows SDK `10.0.26100`
- MSVC 工具集 `14.44`
- 架构：`amd64`

### 构建命令

**只需一步**即可完成构建：

```batch
mkdir build
cd build
cmake .. -G "Visual Studio 17 2022" -A x64 ^
  -DLLVM_ENABLE_PROJECTS="clang;lld;lldb;compiler-rt" ^
  -DLLVM_ENABLE_RPMALLOC=ON ^
  -DLLDB_ENABLE_PYTHON=OFF ^
  -DLLVM_INCLUDE_TESTS=OFF ^
  -DLLVM_INCLUDE_EXAMPLES=OFF ^
  -DLLVM_ENABLE_LIBXML2=OFF ^
  -DLLVM_ENABLE_ZLIB=OFF ^
  -DLLVM_TARGETS_TO_BUILD=X86 ^
  -DLLVM_OBFUSCATION_LINK_INTO_TOOLS=ON ^
  -DCMAKE_BUILD_TYPE=RelWithDebInfo ^
  -DLLVM_USE_CRT_RELEASE=MT

msbuild /m -p:Configuration=RelWithDebInfo INSTALL.vcxproj
```

> **注意：** `CMAKE_INSTALL_PREFIX` 未硬编码，默认使用构建目录，便于移植。

---

## 使用 llvm-msvc-ex

### 安装

发布版本以 **ZIP 压缩包** 形式提供，**不是安装程序**。只需解压到任意位置即可。

### Visual Studio 集成

在解决方案或项目目录中创建 `Directory.Build.props` 文件：

```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <LLVMInstallDir>C:\Path\To\Extracted\LLVM</LLVMInstallDir>
  </PropertyGroup>
</Project>
```

### 工具集配置

需要在项目中设置工具集版本——可以在 `.vcxproj` 中直接指定，也可以通过 Visual Studio UI 设置。

**.vcxproj 示例片段：**

```xml
<PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Debug|Win32'" Label="Configuration">
  <ConfigurationType>Application</ConfigurationType>
  <UseDebugLibraries>true</UseDebugLibraries>
  <CharacterSet>Unicode</CharacterSet>
  <PlatformToolset>ClangCL</PlatformToolset>
  <LLVMToolsVersion>777</LLVMToolsVersion>
</PropertyGroup>
```

> `LLVMToolsVersion` 的值取决于具体的发布版本。

---

## 混淆配置

### 重要说明

- **`[[clang::annotate("...")]]`** —— 仅作用于被标注的函数。
- **`-mllvm`** —— 全局配置 LLVM Pass。

这两种机制**不同**，并非所有命令行选项都有对应的注解。

### 优化相关

- Release 构建默认启用 `/O2`。
- 某些混淆 Pass 依赖优化 Pass，或开启优化后效果更好。

### `/GL` 警告

**使用混淆前请关闭 `/GL`** —— 全程序优化尚未与这些混淆 Pass 验证兼容。

---

## 混淆示例

### 命令行示例

#### 最大保护（生成文件可能超过 100 MB）

```bash
-mllvm -data-obfus -mllvm -const-obfus -mllvm -string-obfus -mllvm -ind-call -mllvm -vm-fla -mllvm -fla -mllvm -sub -mllvm -sub_loop=1 -mllvm -split -mllvm -split_num=3 -mllvm -bcf -mllvm -bcf_loop=1 -mllvm -bcf_prob=40 -mllvm -vm-fla-level=7 -mllvm -x-fla-enh -mllvm -x-var-rot -mllvm -x-combine
```

#### 轻量模式

```bash
-mllvm -data-obfus -mllvm -const-obfus -mllvm -string-obfus -mllvm -ind-call -mllvm -vm-fla -mllvm -vm-fla-level=0 -mllvm -x-fla-enh -mllvm -x-combine -mllvm -x-linear -mllvm -ida-obfus
```

### 函数注解示例

```cpp
[[clang::annotate("x-vm,x-full,x-cfg,custom-cc")]]
void crypt_func1(uint8_t *var, uint8_t *key, size_t var_size, size_t key_size) {
    for (auto i = 0; i < var_size; i++) {
        var[i] ^= key[i % key_size];
    }
}

[[clang::annotate("x-cfg,ind-br,alias-access,custom-cc")]]
void crypt_func2(uint8_t *var, uint8_t *key, size_t var_size, size_t key_size) {
    for (auto i = 0; i < var_size; i++) {
        var[i] ^= key[i % key_size];
    }
}

[[clang::annotate("x-cfg,x-vm,ind-br,alias-access,custom-cc")]]
void crypt_func3(uint8_t *var, uint8_t *key, size_t var_size, size_t key_size) {
    for (auto i = 0; i < var_size; i++) {
        var[i] ^= key[i % key_size];
    }
}
```

### 函数合并示例

使用相同标签 `combine_func[tag]` 的函数将被合并：

```cpp
[[clang::annotate("combine_func[tag1]")]]
int a1(int a, int b) {
    printf("%d , %d\r\n", a, b);
    printf("%x\r\n", a ^ b);
    return a + b;
}

[[clang::annotate("combine_func[tag1]")]]
int a2(int a, int b) {
    std::cout << "hello1" << std::endl;
    for (auto i = std::min(a, b); i < std::max(a, b); i++) {
        printf("%x,", i);
    }
    printf("\n");
    return a * b + a1(a, b);
}

[[clang::annotate("combine_func[tag2]")]]
int a3(int a, int b) {
    printf("%d , %d\r\n", a + 1, b + 2);
    printf("%x\r\n", a ^ b);
    return a + b + a ^ b + a2(a, b);
}
```

---

## 贡献指南

- [贡献实践指南](https://github.com/HyunCafe/contribute-practice)
- [GitHub 贡献项目指南](https://docs.github.com/en/get-started/quickstart/contributing-to-projects)

---

## 学习 LLVM

可参考 [awesome-llvm-security](https://github.com/gmh5225/awesome-llvm-security)。

---

## 致谢

- LLVM
- 多位匿名贡献者
