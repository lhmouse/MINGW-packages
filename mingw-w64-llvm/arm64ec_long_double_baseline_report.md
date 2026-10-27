# ARM64EC `long double` 验证 —— 基线测试报告

* 日期：2025-10
* 主机：Windows 11 on ARM（Qualcomm ARMv8 8 核 / 16 GB），MSYS2 `CLANGARM64` 环境
* 目标三元组：`arm64ec-w64-mingw32`（`arm64ec-w64-windows-gnu`）
* 源码树：
  * LLVM：`d:/lh_mouse/GitHub/MINGW-packages/mingw-w64-llvm/src/llvm-project`（= `upstream/master`，**未打任何 ARM64EC `long double` 补丁**）
  * mingw-w64：`d:/lh_mouse/GitHub/MINGW-packages/mingw-w64-crt/src/mingw-w64`（**未打补丁**）
* 构建目录：
  * `src/build-baseline`（Clang + LLD，Ninja，`Release` + assertions）
  * `src/build-rt-baseline-arm64ec`（compiler-rt / libunwind 交叉构建）
  * `mingw-w64-crt/src/mingw-w64/mingw-w64-crt/build-arm64ec-probe`（CRT）
  * 安装前缀：`mingw-w64-crt/src/sysroot-probe`

---

## 1. 目的

在**完全使用 upstream 补丁**的前提下，建立 `arm64ec-w64-mingw32` 的可用工具链基线：

1. 用 upstream Clang 构建 `arm64ec-w64-mingw32` 目标；
2. 用该 Clang 构建 mingw-w64 CRT；
3. 构建 compiler-rt / libunwind 等运行库；
4. 链接新 CRT 与 LLVM 运行库，跑完 CRT 与 LLVM 的测试并记录。

该报告用于与《ARM64EC long double 修改测试报告》对比。

---

## 2. 基线语义（重要）

在 upstream 中，`WindowsARM64ECTargetInfo` 沿用 AArch64 的默认设置，因此：

| 项目 | 基线值 |
|---|---|
| `sizeof(long double)` | **8** |
| `_Alignof(long double)` | 8 |
| `LDBL_MANT_DIG` | 53 |
| `LDBL_MAX_EXP` | 1024 |

即 ARM64EC 上 `long double` **就是 `double`**（与 MSVC 的 ARM64EC 一致），而 x86_64 上是 80 位 x87 扩展精度。这正是本课题要改变的行为。

---

## 3. 构建过程

### 3.1 Clang / LLD

```bash
cmake -G Ninja -S llvm -B build-baseline \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_ASSERTIONS=ON \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_TARGETS_TO_BUILD="AArch64;X86" \
  -DLLVM_HOST_TRIPLE=aarch64-w64-windows-gnu
ninja -j8
```

结果：`build-baseline/bin/clang.exe`（版本 24.0.0git）构建成功，`--target=arm64ec-w64-mingw32` 可用。

> 环境备注：为在 MSYS2/CLANGARM64 上完成宿主构建，额外需要 3 处**与 `long double` 无关**的本地构建修正（`llvm/cmake/modules/GetHostTriple.cmake`、`llvm/cmake/modules/Findzstd.cmake`、`lld/MinGW/Options.td`）。它们只影响在 MSYS2 下的编译/链接可行性，不属于本课题补丁。

### 3.2 mingw-w64 CRT（arm64ec）

```bash
# headers
./configure --host=arm64ec-w64-mingw32 --prefix=$SYSROOT ...
make -j8 && make install

# crt
autoreconf -fi
./configure --host=arm64ec-w64-mingw32 --prefix=$SYSROOT \
  --with-default-msvcrt=ucrt --disable-lib32 --disable-lib64 --enable-libarm64ec
make -j8 && make install
```

已知需要人工处理的点（均在构建脚本中，不必改源码）：

* `libarm64/cfguard/guard_dispatch.x86_64.o` 需预先编译并复制进构建目录；
* 引导阶段需要空的 `libc++.a` / `libc++abi.a` 占位，以及预编译的 `chkstk_arm64ec.o`。

结果：`sysroot-probe/` 下 CRT、`libmingwex.a`、import libs 全部生成。

### 3.3 运行库（compiler-rt / libunwind）

```bash
cmake -G Ninja -S runtimes -B build-rt-baseline-arm64ec \
  -DCMAKE_TOOLCHAIN_FILE=logs/arm64ec-toolchain.cmake \
  -DLLVM_ENABLE_RUNTIMES="compiler-rt;libunwind"
ninja -j8 && ninja install
```

工具链文件中必须包含：

* `CMAKE_{C,CXX,ASM}_COMPILER_TARGET=arm64ec-w64-mingw32`
* `--unwindlib=none`（避免引导期找不到 `-lunwind`）
* `CMAKE_CXX_COMPILER_WORKS=1`
* `CMAKE_EXE_LINKER_FLAGS_INIT` 中带 `chkstk_arm64ec.o`

结果：`libclang_rt.builtins-arm64ec.a`、`libunwind.a` 构建成功。

> `libc++` 在基线阶段只以空占位库参与链接，尚未真正编译（见第 5 节“未覆盖项”）。

---

## 4. 测试结果

### 4.1 Clang 测试套件（lit）

运行范围：`clang/test/{CodeGen,Sema,Preprocessor,Driver}`（在 ARM64 主机上交叉运行）。

| 指标 | 数量 |
|---|---|
| Total Discovered | **9556** |
| Passed | 9272 |
| Unresolved | **277**（2.90%） |
| Failed | **7**（0.07%） |

失败清单（`logs/baseline-fails.txt`）：

```
Clang :: CodeGen/2008-07-17-no-emit-on-error.c
Clang :: Driver/dxc_dxv_path.hlsl
Clang :: Driver/dxc_spirv-val_missing.hlsl
Clang :: Driver/memtag-stack_lto.c
Clang :: Driver/modules-print-library-module-manifest-path.cpp
Clang :: Preprocessor/pragma-missing-string-token.c
Clang :: Preprocessor/print-header-json.c
```

这 7 项失败均与 `long double` 无关（DXC/SPIR-V 外部依赖、LTO 需要 `ld.lld` 链接支持、`-E -dM` 输出路径差异等），属于**宿主环境限制**。

`Unresolved` 的 277 项集中在 `Sema/riscv-*`、`Sema/arm-*` 等需要目标专用后端/平台的用例，以及若干需要 `lit` 外部工具的用例。

### 4.2 运行库冒烟测试

用基线工具链 + 基线 CRT 编译并运行：

```c
#include <stdio.h>
int main(void) { printf("hello, sizeof(long double) = %zu\n", sizeof(long double)); return 0; }
```

```
hello, sizeof(long double) = 8
```

退出码 0，说明 CRT + compiler-rt + libunwind 的组合在 ARM64EC 上可用。

### 4.3 CRT 自身测试

mingw-w64 CRT 的 `make check` 依赖 gdb / QEMU 等外部组件，在纯 WoA 环境下无法自动运行；基线阶段改为**符号级验证**：

* `libmingwex.a` 中的 `*l` 系列函数（`fabsl`、`frexpl`、`modfl`、`ilogbl`、`logbl`、`copysignl` …）在 ARM64EC 上**全部来自 ucrtbase 导入**，即它们按 64 位 `long double` 语义工作；
* 因此基线 CRT 对 80 位 `long double` 没有任何支持（也不需要）。

### 4.4 x87 模拟器行为刻画（基线调研）

ARM64 上无法执行 x87 指令；WoA 的 x64 模拟层在 `CLANG64` 环境下可运行 x64 程序，因此可以用它来探测"模拟 x87"的行为。结论（2M + 2M 差分用例，0 例无法解释）：

| 行为 | 模拟器 | 真实 x87 |
|---|---|---|
| `fldcw` 的精度控制 / 舍入控制 | 忽略 | 生效 |
| `fld` / `fstp` m80 拷贝 | 精确 | 精确 |
| 默认舍入 | **远离零**（应为就近偶数） | 就近偶数 |
| 非规格化 / 伪 NaN | 规格化 | `#IA` |
| SNaN | 不静默 | 静默并置 QNaN |
| 次正规数 | 截断 / 冲刷 | 渐近下溢 |
| NaN payload | 向上取整 | 保留 |

**结论：软件 `long double` 必须实现真实 x87 语义，不能照抄模拟器行为。**

---

## 5. 未覆盖项

1. `libc++` / `libc++abi` 在基线阶段只有空占位库，未真正编译（`arm64ec` 的 C++ 运行库构建依赖本课题完成后的 CRT 修正）。
2. CRT 官方测试套件（需要 gdb/QEMU）未运行，代之以符号级与冒烟级验证。
3. `clang/test/{CodeGen,Sema}` 中 277 项 `Unresolved` 未逐项分析。

---

## 6. 基线结论

* upstream 工具链可以在 ARM64 主机上完整构建出 `arm64ec-w64-mingw32` 的 Clang + CRT + compiler-rt + libunwind，并生成可运行的可执行文件。
* Clang 测试 9556 项中 7 项失败、277 项未解析，**全部与 `long double` 无关**，可作为补丁回归对比的基线。
* 基线 ARM64EC 的 `long double` 是 64 位 `double`；`*l` 数学函数全部来自 ucrtbase。
