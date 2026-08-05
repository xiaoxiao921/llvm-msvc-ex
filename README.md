# `llvm-msvc-ex`

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

## Building

### Supported Build Environment

Personally supported and tested extensively through GitHub Action CI:

- Visual Studio 2022 only
- Windows SDK `10.0.26100`
- MSVC toolset `14.44`
- Architecture: `amd64`

### Build Command

```ps1
mkdir build

pushd build

cmake .. -G Ninja `
  -DCMAKE_CXX_FLAGS="/utf-8" `
  -DCMAKE_C_FLAGS="/utf-8" `
  -DLLVM_USE_RPMALLOC=ON `
  -DLLDB_ENABLE_PYTHON=OFF `
  -DLLVM_INCLUDE_TESTS=OFF `
  -DLLVM_INCLUDE_EXAMPLES=OFF `
  -DLLVM_INCLUDE_BENCHMARKS=OFF `
  -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;lld;lldb;compiler-rt" `
  -DCMAKE_INSTALL_PREFIX="C:/LLVM" ` # Where LLVM will be installed when the cmake --install command will be ran by the user.
  -DLLVM_ENABLE_LIBXML2=OFF `
  -DLLVM_ENABLE_ZLIB=OFF `
  -DLLVM_TARGETS_TO_BUILD=X86 `
  -DLLVM_OBFUSCATION_LINK_INTO_TOOLS=ON `
  -DCMAKE_BUILD_TYPE=RelWithDebInfo `
  -DLLVM_USE_CRT_RELEASE=MT

cmake --build . --config RelWithDebInfo --parallel --verbose

cmake --install . --config RelWithDebInfo

popd
```

---

## Using llvm-msvc-ex

### Installation

Releases are distributed as **ZIP archives**, not installers. Simply extract the archive to any location.

### Visual Studio Integration

- Ensure that the "Clang C++ tools for Windows" workload is installed. You can verify this in the Visual Studio Installer.

- Create a `Directory.Build.props` file in your solution or project directory:

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

[[clang::annotate("split, fla, bcf, x-fla-enh, x-mba, mba-subs, split, sub, x-var-rot")]]
void crypt_func4(uint8_t *var, uint8_t *key, size_t var_size, size_t key_size) {
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
