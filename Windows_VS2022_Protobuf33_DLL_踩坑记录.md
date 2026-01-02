# Windows（VS2022）下编译 Protobuf 33.2.0（DLL）踩坑全记录

> **关键词**：Protobuf 33.2.0 / Windows / VS2022 / DLL / map / absl / vcpkg  
> **适用人群**：Windows 下使用 C++ / VS 集成 protobuf 的开发者  
> **结论先行**：Protobuf 33.x + `map<>` = **必须正确引入并运行期匹配 Abseil**

---

## 一、背景说明

- 操作系统：Windows 10 / 11  
- IDE：Visual Studio 2022  
- Protobuf 版本：**3.3.2（33.2.0）**
- 使用方式：
  - protobuf：源码编译，生成 **DLL**
  - 依赖库（zlib / abseil）：vcpkg
- 使用场景：
  - VS C++ 项目
  - `.proto` 中包含 `map<string, int32>` 字段

---

## 二、最初的 CMake 编译命令

```bat
cmake -S . -B build ^
  -DCMAKE_INSTALL_PREFIX=../install ^
  -Dprotobuf_BUILD_SHARED_LIBS=ON ^
  -DCMAKE_CXX_STANDARD=17 ^
  -G "Visual Studio 17 2022"
```

### 参数说明

- `protobuf_BUILD_SHARED_LIBS=ON`  
  👉 生成 `libprotobuf.dll`
- `CMAKE_CXX_STANDARD=17`  
  👉 protobuf 33.x 要求 C++17
- VS2022 对应生成器：`Visual Studio 17 2022`

⚠️ **注意**：在 Windows 下，单独执行以上命令一定会遇到依赖问题。

---

## 三、问题一：找不到 ZLIB

### 报错信息

```text
Could NOT find ZLIB (missing: ZLIB_LIBRARY ZLIB_INCLUDE_DIR)
```

### 原因分析

- Windows 系统默认 **没有 zlib**
- protobuf 在 Windows 下依赖 zlib

### 解决方案：使用 vcpkg 安装 zlib

```bat
git clone https://github.com/microsoft/vcpkg.git
cd vcpkg
bootstrap-vcpkg.bat
vcpkg install zlib:x64-windows
```

---

## 四、问题二：加入 map 字段后异常

### 现象

- `.proto` **不包含 map** → 正常运行
- `.proto` **包含 map<string, int32>** → 编译或链接失败

### 常见报错

```text
error C4996: google::protobuf::RepeatedPtrField ...
```

或

```text
LNK2001: unresolved external symbol absl::hash_internal::MixingHashState
```

---

## 五、根本原因：Protobuf 33.x 强依赖 Abseil

从 **Protobuf 33.x 开始**：

- `map<>` 内部实现基于 `absl::flat_hash_map`
- **Abseil（absl）不再是可选依赖**
- 只有 absl 头文件 ❌ 不够
- **必须链接 absl 的 lib / dll**

---

## 六、一个常见误区：Abseil 不是 header-only

❌ 错误理解：

> “我已经有 absl_xxx.h 文件了”

✅ 正确认知：

- Abseil 是 **源码库**
- 需要编译生成 `.lib / .dll`
- 否则一定会在 `map<>` 处链接失败

---

## 七、推荐方案：vcpkg 管理 Abseil

### 1️⃣ 安装 abseil（动态库）

```bat
vcpkg install abseil:x64-windows
```

- 默认使用 `/MD`
- 自动生成 `.lib + .dll`

---

### 2️⃣ 使用 vcpkg toolchain 编译 protobuf

```
cmake -S . -B build ^
  -A x64 ^
  -G "Visual Studio 17 2022" ^
  -DCMAKE_TOOLCHAIN_FILE=D:\vcpkg\scripts\buildsystems\vcpkg.cmake ^
  -DCMAKE_INSTALL_PREFIX=../install ^
  -DCMAKE_CXX_STANDARD=17 ^
  -Dprotobuf_BUILD_SHARED_LIBS=ON ^
  -Dprotobuf_BUILD_TESTS=OFF ^
  -Dprotobuf_WITH_ZLIB=ON ^
  -DCMAKE_MSVC_RUNTIME_LIBRARY=MultiThreadedDLL ^
  -DCMAKE_CXX_FLAGS="/wd4996"
```
编译
```bat
cmake --build build --config Release
```
安装
```
cmake --install build --config Release
```
✔ protobuf 自动找到 absl  
✔ `map<>` 编译通过

---

## 八、问题三：C4996（MSVC 警告当错误）

### 报错示例

```text
error C4996: 'RepeatedPtrField': Use Arena::Create instead
```

### 原因

- protobuf 33.x 在 MSVC 下存在 **已知 C4996 警告**
- VS 项目启用了 `/WX`（警告视为错误）

### 解决方式（推荐）

```cpp
#define _SILENCE_CXX17_ALLOCATOR_VOID_DEPRECATION_WARNING
#define _SILENCE_CXX17_ITERATOR_BASE_CLASS_DEPRECATION_WARNING
```

或关闭单条警告：

```
/wd4996
```

❌ 不建议修改 protobuf 源码

---

## 九、问题四：运行时报“无法定位程序输入点”

### 报错示例

```text
无法定位程序输入点
?combine_raw@MixingHashState@hash_internal@lts_20250814@absl@@
```

### 特点

- 编译、链接均成功
- 启动 exe 直接失败
- 报错符号包含 `absl::lts_YYYYMMDD`

---

## 十、问题本质：Abseil ABI 不兼容

Abseil 的特点：

- **ABI 不稳定**
- 命名空间带日期版本号：

```cpp
absl::lts_20250814
absl::lts_20250512
```

### 出错原因

- 编译期链接的是 **A 版本 absl**
- 运行期加载的是 **B 版本 absl**
- Windows DLL 搜索顺序导致加载了“错误版本”

---

## 十一、最终解决方案（强烈推荐）

### ✅ 所有 DLL 放到 exe 同目录

```text
ProtoBufDemo.exe
libprotobuf.dll
absl_hash.dll
absl_raw_hash_set.dll
absl_city.dll
absl_base.dll
absl_strings.dll
absl_throw_delegate.dll
```

Windows DLL 搜索顺序：

1. exe 所在目录（最高优先级）
2. 当前目录
3. PATH
4. System32

👉 放在 exe 同目录可 **100% 避免版本错配**

---

## 十二、运行库必须统一

| 模块 | 运行库 |
|----|----|
| protobuf | /MD |
| absl | /MD |
| Demo 工程 | /MD |

❌ `/MT` 与 `/MD` 混用会导致隐蔽崩溃

---

## 十三、最终经验总结

- Protobuf 33.x + `map<>` = **必须 Abseil**
- Abseil 在 Windows 下：
  - 版本必须一致
  - 编译 / 运行必须一致
  - DLL 路径必须可控
- **exe 同目录放 DLL** 是最稳妥方案
- vcpkg 是 Windows 下最省心的依赖管理工具

---

## 十四、建议

> 如果不是强制 DLL，  
> **优先考虑静态编译 protobuf + absl**，可彻底规避 ABI / DLL 地狱。

---

**踩坑一次，受益终身。**  
希望这篇记录能帮到后来者。  
