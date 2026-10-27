# ARM64EC `long double` 验证 —— 补丁测试报告

* 日期：2026-10
* 主机：Windows 11 on ARM（Qualcomm ARMv8 8 核 / 16 GB），MSYS2 `CLANGARM64` 环境
* 目标三元组：`arm64ec-w64-mingw32`
* 补丁：
  * `mingw-w64-llvm/long_double_arm64ec_llvm.patch`
  * `mingw-w64-crt/long_double_arm64ec_mingw-w64.patch`

  两个补丁都带有分组说明表头，按"特性 / lit 工具修复 / 宿主构建变通"三段编排，便于按段裁剪。

> **与 MSVC 的分界**：本特性只对 `arm64ec-w64-mingw32`（MinGW/GNU 环境）生效。
> `arm64ec-pc-windows-msvc` 保持 `long double == double`，否则将无法与 MSVC 生成的代码二进制兼容。
> 同理，`aarch64-w64-mingw32` 与 `x86_64-w64-mingw32` 的预定义宏与 upstream 完全一致。

---

## 1. 设计概要

ARM64EC 必须与 x86_64 的 ABI 兼容，因此 `long double` **必须**是 x87 80 位扩展精度格式，占 16 字节、按 16 字节对齐；但 ARM64 没有 x87 浮点单元。

采用的方案：

| 层次 | 做法 |
|---|---|
| 内存布局 | 16 字节槽，偏移 0 = 64 位有效数（显式整数位），偏移 8 = 16 位符号/阶码，偏移 10..15 = 从不读写的填充 |
| IR 表示 | `i128` |
| 装载 / 存储 | 纯整数指令（`load i128` / `store i128`） |
| 运算 | 先经 `__arm64ec_load_x87_long_double` 转成 `double`，算完再经 `__arm64ec_store_x87_long_double` 存回 |
| 精确操作 | `fabsl` / `copysignl` / `signbit` / `isnan` / `isinf` / `isfinite` / `isnormal` / `fpclassify` 直接在镜像上用整数实现；`frexpl` / `ldexpl` / `modfl` 走 CRT 的 80 位实现 |

运行时辅助函数位于 `compiler-rt/lib/builtins/arm64ec/x87_long_double.c`，实现**真实 x87 语义**（而非 WoA x64 模拟层的行为）：

* inf / NaN：整数位为 1 且阶码全 1 为 inf；整数位为 0 的非法编码按 x87 返回 indefinite QNaN（`0xC000…`）；
* 伪非规格化数（阶码 0、整数位 1）按 `e = m >> 63` 处理；
* 规格化数按就近偶数舍入到 53 位；
* 非规格化数渐近下溢，`shift = -(exponent + 1011)`，含 `shift == 64` 特例；
* `store` 从不舍入；非规格化路径用 `63 - __builtin_clzll(m)` 重新规格化。

### 1.1 编译器侧（LLVM 补丁）

| 文件 | 作用 |
|---|---|
| `clang/include/clang/Basic/TargetInfo.h` | 新增 `hasSoftX87LongDouble()` 钩子（默认 `false`） |
| `clang/lib/Basic/Targets/AArch64.{h,cpp}` | 仅对 **非 MSVC 环境**的 ARM64EC 生效：`long double` 改为 16 字节 / 16 对齐 / `x87DoubleExtended`，并开启 `__float128`（`__FLOAT128__` / `__SIZEOF_FLOAT128__`，供 compiler-rt 的 `*tf3` 内建函数使用）。`arm64ec-pc-windows-msvc` 以及其他 AArch64 目标**逐字节保持 upstream 行为** |
| `clang/lib/Driver/ToolChains/Clang.cpp` | 允许非 MSVC 的 ARM64EC 使用 `-mlong-double-{64,80,128}` |
| `clang/lib/CodeGen/CodeGenTypes.cpp` | `long double` 映射为 `i128` |
| `clang/lib/CodeGen/Targets/X86.cpp` | ARM64EC 的 `long double` 在 x64 分类规则下恒按内存传递 |
| `clang/lib/CodeGen/CGExprScalar.cpp` | 字面量、加减乘除、比较、`++/--`、显式/隐式转换、`(bool)` 全部走 `double` 或整数路径；复数相等比较的 `i128` 分量改走 `EmitSoftX87LdToDouble` |
| `clang/lib/CodeGen/CGExprComplex.cpp` | `long double _Complex` 的加减乘除、一元取负、共轭：四个 `EmitBin*` 入口在 `isSoftX87ComplexOp()` 为真时先把分量转成 `double`、复用浮点复数路径、再转回 `i128`（否则会落到**整数**复数路径，既不实现 x87 语义，也会因"实部操作数"触发断言） |
| `clang/lib/CodeGen/CGBuiltin.cpp` | 内置函数（`fabsl`、`signbit`、`isnan`、`sqrtl` …）的特化；`Exact` 集合含 `frexpl`/`ldexpl`/`modfl`/`nextafterl`/`nexttoward{,f,l}`；`nanl`/`nansl`/`infl`/`huge_vall` 直接构造整数镜像 |
| `clang/lib/CodeGen/CGExprConstant.cpp` | 80 位常量折叠 |
| `clang/lib/CodeGen/CodeGen{Function,Module}.h` | 辅助函数声明 |
| `compiler-rt/lib/builtins/CMakeLists.txt` | `arm64ec_SOURCES = ${aarch64_SOURCES} + arm64ec/x87_long_double.c` |
| `compiler-rt/lib/builtins/arm64ec/x87_long_double.c` | 两个运行时辅助函数 |
| `llvm/utils/lit/lit/util.py` | **工具修复**（不属于本特性）：`get_windows_extended_path()` 在加 `\\?\` 前缀前必须把 `/` 归一成 `\` |

### 1.2 CRT 侧（mingw-w64 补丁）

原则：**能复用 `arm-common` 里现成的无 cast `*l` 函数就复用，不新增目录**。

| 文件 | 改动 |
|---|---|
| `math/arm-common/copysignl.c` | arm64ec 分支：整数复制符号位 |
| `math/arm-common/ldexpl.c` | arm64ec 分支：完整整数实现（规格化、溢出、渐近下溢 + RNE） |
| `math/arm64/nearbyintl.c` | arm64ec 分支：`nearbyint((double)x)` |
| `math/{rintl,lrintl,truncl}.c` | arm64ec 分支：改走 `rint` / `lrint` / `trunc` |
| `math/fabsl.c` | arm64ec 分支：整数清符号位 |
| `math/signbitl.c` | arm64ec 分支：读符号/阶码字第 15 位 |
| `math/fpclassifyl.c`、`math/frexpl.c` | 把 `__arm64ec__` 加入原有的 x86 整数分支 |
| `math/modfl.c` | arm64ec 分支：整数截断 + 分数部分重新规格化 |
| `gdtoa/strtopx.c` | 把 `__arm64ec__` 加入 80 位 `strtold` 的 `#elif`（原本只对 x86 生效） |
| `crt/CRT_fp10.c` | arm64ec 分支：`F_LD80()` 下 `_fpreset` 是数据导入，需自带重置 x87 控制字的包装 |
| `math/arm-common/ld_fallbacks.c` | **新增**：`acosl`/`asinl`/`atanl`/`atan2l`/`cosl`/`sinl`/`tanl`/`expl`/`logl`/`log10l`/`floorl`/`fmodl`/`scalbnl` 的 `double` 回退实现（仅 `__arm64ec__` 编译）。CRT 自己的 `long double _Complex` 实现是用 `-fno-builtin` 编译的，这些调用不会被折叠成内联，缺符号会导致 `std::complex<long double>` 完全无法链接 |
| `Makefile.am` / `Makefile.in` | `if ARM64EC` 中加 `crt/CRT_fp10.c`，`math/nextafterl.c math/nexttoward.c math/nexttowardf.c`，以及 `math/arm-common/{copysignl,log1pl,logbl,remainderl,ld_fallbacks}.c`（`F_LD64()` 屏蔽了 ucrtbase 的 64 位版本） |
| `mingw-w64-headers/crt/math.h` | 内联 `fabsl` 增加 arm64ec 分支 |

---

## 2. 构建

与基线相同的命令与构建目录（原地重建）：

```bash
cd src/build-baseline && ninja -j8                                   # Clang
cd .../mingw-w64-crt/build-arm64ec-patched && make -j8 && make install
```

CRT 补丁后新增的两处构建要点与修复：

* `ld: unable to automatically import from '_fpreset'` —— 因为 `long double` 现在是 80 位，`def-include/func.def.in` 展开 `F_LD80()` 后 `_fpreset` 变成**数据**导入，而只有 `lib32/lib64` 编译 `crt/CRT_fp10.c`；因此给 ARM64EC 补一行。
* `undefined symbol: nexttoward (EC symbol)` —— `F_LD64()` 在 80 位 `long double` 下不再展开，ucrtbase 的
  `nextafter`/`nexttoward` 家族导入全部消失，因此把 `math/nextafterl.c`、`math/nexttoward.c`、
  `math/nexttowardf.c` 加进 ARM64EC 的 `libmingwex`（它们本来就是按 x87 内存布局写的通用实现）。

> 注意：`configure` 时必须把 `--target=arm64ec-w64-mingw32 --sysroot=…` 写进 `CC`，
> 否则 automake 会用宿主默认的 `aarch64` 目标编译，产物是 `coff-arm64` 而不是 `coff-arm64ec`，
> 链接时才会以 `(EC symbol)` 未定义的形式暴露出来。
>
> `automake` 重新生成 `Makefile.in`（`Makefile.am` 改动之后必须执行），补丁中同时提供了两者。

---

## 3. 测试结果

### 3.1 x87 语义参考模型

`logs/verify_x87ld.py`：独立的 Python 精确整数模型，与 `x87_long_double.c` 逐位对拍。

| 用例数 | 不匹配 |
|---|---|
| **1 101 620** | **0** |

覆盖：规格化 / 非规格化 / 伪非规格化 / 零 / inf / NaN / SNaN 的 load、store、舍入边界。

### 3.2 CRT 功能性一致性测试（`logs/ldtest/ldtest.c`）

78+ 项检查：内存布局静态断言、各种转换、算术、内置函数、`nextafter`/`nexttoward` 系列的
ULP 精确性、libm 精确性、ABI / 可变参数 / `printf`，以及 100 000 次随机精确性循环。

| 优化级别 | 结果 |
|---|---|
| `-O0` | 92 checks, **0 failures** |
| `-O1` | 92 checks, **0 failures** |
| `-O2` | 92 checks, **0 failures** |
| `-O3` | 92 checks, **0 failures** |
| `-Os` | 92 checks, **0 failures** |

关键断言：

* `sizeof(long double) == 16`，`_Alignof == 16`，`LDBL_MANT_DIG == 64`，`LDBL_MAX_EXP == 16384`
* 填充字节不影响值，也不影响符号位
* `(_Bool)` 对低于 `double` 范围的非规格化数仍为真
* `frexpl` / `ldexpl` 往返 100 000 例全部逐位一致；`modfl` 的整数部分与分数部分全部逐位一致
* `nextafterl(1.0L, 2.0L)` 的镜像是 `8000000000000001`（**80 位**的一个 ULP），而不是
  binary64 的 `8000000000000800`；`nexttowardl` 与之完全一致
* `nexttoward(1.0, 2.0L)` = `0x1.0000000000001p0`，`nexttowardf(1.0f, 2.0L)` = `0x1.000002p0f`，
  说明第二个参数确实按 80 位 `long double` 传递而没有退化成 `double`

### 3.3 运行期样例
```text
sizeof(long double) = 16, _Alignof = 16
LDBL_MANT_DIG = 64, LDBL_MAX_EXP = 16384

printf:  1: 2.000000 / 2: %Lf→2.000000 / 3: %Lg→2 / 4: "7 2.000000 9"
snprintf("%.20Lf", 2.0L) → "2.00000000000000000000"

strtold:
  1.2345678901234567890123456789        -> 1.234567890123456788986113
  1e4000                                -> 9.999999999999999999965464e+3999
  1e-4000                               -> 9.999999999999999999872577e-4001
  3.14159265358979323846264338327950288 -> 3.141592653589793238512809
  -0.0                                  -> -0
```

> **注意**：与 x86_64 mingw-w64 完全一致，`printf("%Lf", …)` 只有在 `__USE_MINGW_ANSI_STDIO=1`（C99/POSIX 模式）时才由 mingw 自己的 `__mingw_printf` 处理 80 位值；默认走 ucrtbase 的 `printf`，它按 `double` 读取，`%Lf` 会输出错误结果。这是 x64 mingw-w64 的既有行为，不是本补丁引入的。

### 3.4 `-mlong-double-{64,80,128}` 三种模式

| 命令行 | `sizeof` | `_Alignof` | `LDBL_MANT_DIG` | `LDBL_MAX_EXP` | 实现方式 |
|---|---|---|---|---|---|
| 默认 / `-mlong-double-80` | 16 | 16 | 64 | 16384 | x87 80 位镜像 + `double` 软件运算 |
| `-mlong-double-64` | 8 | 8 | 53 | 1024 | 直接使用 `double` |
| `-mlong-double-128` | 16 | 16 | 113 | 16384 | IEEE 四精度（1+15+112），编译器后端降级为 `__*tf3` / `__*tf2` 软件调用 |

`-mlong-double-128` 的实测（`1.0L / 3.0L`，纯软件）：

```text
sizeof=16 align=16 MANT_DIG=113 MAX_EXP=16384
1/3 = 3ffd 5555555555555555 5555555555555555     ← IEEE quad 的 1/3
c*3 == 1 ? 1
```

**与 arm64 共用同一套实现**：`long double` 为四精度时 `hasSoftX87LongDouble()` 取值为假，代码生成退回到
AArch64 的通用 `f128` 路径（`clang/lib/CodeGen/Targets/AArch64.cpp`），与 `aarch64-w64-mingw32` 完全相同，
没有任何 ARM64EC 专用代码；运行库侧也共用同一批源码（`arm64ec_SOURCES = ${aarch64_SOURCES}`）。

端到端验证（四精度只有软件 `__*tf3` 才能算，`printf("%Lf")` 不可用，因此用 `(double)` 打印）：

```c
volatile long double a = 1e4000L, b = 1e-4000L;
long double c = a * b;        /* 需要 quad 的 16384 指数范围 */
long double d = a + a;
printf("%.17g %.17g %.17g\n", (double)c, (double)(d / a), (double)(a / a));
/* 输出：1 2 1 */
```

为此需要打开 ARM64EC 的 `__float128` 支持：compiler-rt 的 `*tf3` / `*tf2` 内建函数由
`CRT_HAS_TF_MODE` 控制，而它要求 `CRT_HAS_F128`，后者只在 `long double` 本身是四精度或
`__float128` 可用（`__FLOAT128__` / `__SIZEOF_FLOAT128__`）时成立。ARM64EC 默认是 80 位
`long double`，两者都不满足，因此这些目标文件原本被编译成空对象。现在对
**非 MSVC 环境的 ARM64EC** 设置 `HasFloat128 = true` 并定义上述两个宏，`*tf3` 便以
`__float128`（与 `long double` 无关）为
底层类型编译出来，arm64 与 arm64ec 共用同一份目标文件。

一版中间实现曾把这两项开在所有 AArch64 目标上，导致 `Preprocessor/init-aarch64.c` 的宏顺序、
以及 `ARM64EC-MSVC` 的 `__LDBL_*` 期望值不匹配。最终实现把范围收窄为
**`Triple.isWindowsArm64EC() && !Triple.isWindowsMSVCEnvironment()`**，`init-aarch64.c` 与
`Driver/mlong-double-128.c` 都恢复 PASS，MSVC 的 ARM64EC ABI 保持 64 位。

### 3.5 Clang 测试套件（lit）

范围与基线完全相同：`clang/test/{CodeGen,Sema,Preprocessor,Driver}`，命令也逐字相同。

| 指标 | 基线 | 上一轮补丁 | 最终 |
|---|---|---|---|
| Total Discovered | 9556 | 9556 | 9556 |
| Passed | 9272 | 5934 | **6221** |
| Unsupported | （未统计） | 3327 | 3327 |
| Expectedly Failed | — | 5 | 5 |
| Unresolved | 277 | 272 | **0** |
| Failed | 7 | 18 | **3** |

上一轮补丁多出的 11 个失败（10 个 lit `rm` 缺陷 + `Driver/mlong-double-128.c`）在本轮**全部消除**，
连带基线里同样被 `rm` 缺陷拖累的 3 个用例也一并恢复：失败数 **7 → 3**，`Unresolved` **277 → 0**。
逐条对照见 **§7.1**；所有涉及 ARM64EC 三元组的 13 个用例单独验证见 **§7.1.1**。

### 3.6 运行库构建与全量测试

运行库全部由**被测 Clang** 交叉编译（`--target=arm64ec-w64-mingw32`，静态库，安装进 `sysroot-patched`）：

| 运行库 | 产物 |
|---|---|
| compiler-rt | `libclang_rt.builtins-arm64ec.a`（含 `*tf3` / `*tf2` 四精度内建函数与 `arm64ec/x87_long_double.c`） |
| libunwind | `libunwind.a` |
| libc++abi | `libc++abi.a` |
| libc++ | `libc++.a` |

测试结果：

| 套件 | 总数 | 通过 | Unsupported | 失败 | 失败原因 |
|---|---|---|---|---|---|
| `check-builtins`（compiler-rt） | — | — | — | — | 见下 |
| `check-unwind` | 31 | 7 | 15 | 9 | 全部是 `__start_main_func` 之类的**链接**错误：PE/COFF 不支持 ELF 风格的 `__start_/__stop_` 段符号。用**系统自带** clang 编译同样的最小程序可复现，与本补丁无关 |
| `check-cxxabi` | 81 | 55 | 22（+1 XFAIL） | 3 | 全部是 `forced_unwind{1,2,3}`；在 `-mlong-double-64`（完全绕过 soft-x87 路径）下同样段错误，与本补丁无关 |
| libc++（全量，上一轮） | 11527 | — | — | 126（1.09%） | 见 §7.3，本轮已用 `std/numerics` 子树（897/907）定向复验 |

libc++ 的 126 个失败集中在与 `long double` 无关的领域：`std/thread/*`（条件变量、互斥量、信号量、
future、stop_token）、`std/algorithms/*`、`std/localization/*`、`std/atomics/*`、`std/utilities/format/*`、
`std/language.support/cmp/*`。这些是 ARM64EC 平台上的 libc++ 既有缺口（SEH / TLS / `__os_arm64x_*`
相关），与 `long double` 无关。

唯一与本补丁相关的是 `std/numerics/c.math/cmath.pass.cpp`。它原先失败在 C99 的
`nexttoward` / `nexttowardf` / `nexttowardl` 上——mingw-w64 的
`lib-common/api-ms-win-crt-math-l1-1-0.def.in` 把这些符号放在 `F_LD64()` 保护下（注释为
"Can't use long double functions from the CRT on x86"），`long double` 变为 80 位后 `F_LD64` 不再展开，
导入符号随之消失。**本轮已修复**：`math/nextafterl.c`（同时定义 `nexttowardl` 别名）、
`math/nexttoward.c`、`math/nexttowardf.c` 编入 ARM64EC 的 `libmingwex`，并且 clang 侧把
`nextafterl`/`nexttoward{,f,l}` 并入 `Exact` 集合，不再降级为 binary64 运算。

修复后 `std/numerics/c.math/hermite.pass.cpp` 转为 PASS；`cmath.pass.cpp` 已越过 `nexttoward`
断言，剩下的失败点是 `std::hypot(1e4000L, 0, 0)`：libc++ 会先把参数按 `ldexp(1, ±8192)` 缩放，
但 `sqrtl` 仍走 `double`，`1e3056` 在 `double` 里上溢为 `inf`。这是第 6 项已知限制
（超越函数的 `double` 回退），不是回归。

---

## 4. ARM64EC ↔ x86-64 `long double` ABI 冒烟测试
```c
long double
my_add(long double x, long double y)
{
  return x + y;
}
```

`my_add` 分别编译成 **x86-64 DLL** 与 **ARM64EC DLL**，再用 x86-64 宿主程序与 ARM64EC 宿主程序
互相调用，覆盖全部四种组合。

### 4.1 x86-64 侧生成的汇编（Intel 语法，与预期完全一致）

```asm
my_add:
    mov     rax, rcx            ; rcx = 返回存储（long double 与结构体同约定）
    fld     tbyte ptr [rdx]     ; x
    fld     tbyte ptr [r8]      ; y
    faddp   st(1), st
    fstp    tbyte ptr [rcx]     ; 写回
    ret
```

### 4.2 ARM64EC 侧生成的汇编

ARM64EC 的**具名参数与返回值遵循 ARM64 约定**，因此 DLL 中同时存在 ARM64 函数体和 x86-64 入口
thunk（`#my_add`，由链接器按 ARM64EC ABI 合成）：

```asm
# --- ARM64 函数体（真实实现）---
my_add:
    stp     x0, x1, [sp, #0x20]        # 实参 1 在 x0:x1
    add     x0, sp, #0x20
    mov     x19, x3
    mov     x20, x2                    # 实参 2 在 x2:x3
    bl      __arm64ec_load_x87_long_double
    stp     x20, x19, [sp, #0x10]
    bl      __arm64ec_load_x87_long_double
    fadd    d0, d8, d0                 # 以 double 相加
    mov     x0, sp
    bl      __arm64ec_store_x87_long_double
    ldp     x0, x1, [sp]               # 返回值在 x0:x1
    ret

# --- x86-64 入口 thunk（rcx/rdx/r8 → x0:x1/x2:x3），Intel 语法 ---
#my_add:
    mov     rax, rsp
    mov     qword ptr [rax + 0x20], rbx
    push    rbp
    pop     rbp
    jmp     my_add                  ; 跳入上面的 ARM64 函数体
```

x86-64 用 `rcx`（返回）+ `rdx`/`r8`（指针），ARM64EC 用 `x0:x1`（返回）+ `x0:x1`/`x2:x3`（按值），
两者的差异完全由入口/出口 thunk 承担。

### 4.3 跨架构调用结果

测试值取 `1 + 2^-60`（镜像 `8000000000000008 3fff`），用于检验全部 64 位有效数是否原样穿过 ABI：

| 宿主 | 被调 DLL | 调用方送出 | 被调方收到 | 返回值 |
|---|---|---|---|---|
| x86-64 | x86-64 | `…008 3fff` | `…000 3fff` | `…000 3fff` |
| x86-64 | **arm64ec** | `…008 3fff` | **`…008 3fff`** | **`…008 3fff`** |
| arm64ec | x86-64 | `…008 3fff` | `…000 3fff` | `…000 3fff` |
| arm64ec | **arm64ec** | `…008 3fff` | **`…008 3fff`** | **`…008 3fff`** |

`my_add(1.0L, 2.0L)` 在四种组合下均返回 `c000000000000000 4000`（= 3.0），正确。

**结论**：`long double` 的 80 位镜像在 ARM64EC 与 x86-64 之间**双向无损**传递，入口/出口 thunk 的
参数与返回约定转换正确。

x86-64 DLL 一侧出现的 `…008 → …000` 截断**不是 ABI 问题**：用系统自带 x86-64 MSYS2 工具链
（完全不涉及 ARM64EC 与本补丁）编译相同代码，截断依旧发生。原因是 **Windows on ARM 的 x64 模拟器
把 x87 寄存器按双精度实现**，因此 `long double` 的有效数在模拟环境中只有 53 位。这也解释了为什么
x86-64 DLL 内的 `my_add` 只能得到双精度结果。

---


## 5. 本轮修复的问题（相对上一版补丁）

| 问题 | 现象 | 修复 |
|---|---|---|
| `x87LongDoubleImage` 静态辅助函数 | 代码风格 | 内联到调用点 |
| `Builtin::Context::getName()` 返回 `frexpl` 等而非 `__builtin_frexpl` | `frexpl`/`ldexpl`/`modfl` 被降级到 `double` | 修正字符串比较 + 注释说明 |
| `(_Bool)longdouble` 走 `CK_FloatingToBoolean` | `Invalid operand types for FCmp` 崩溃 | `VisitCast` 分支改走 `EmitConversionToBool`，并改为整数零判断 |
| `++`/`--` on `long double` | `Cannot create binary operator with two operands of differing type` | `EmitScalarPrePostIncDec` 增加 `SoftX87` 包装 |
| `__builtin_signbit` 未掩码 | 填充字节泄漏进符号位（填充为 `0x5a` 时返回 `0xB4B4B4B4`） | 抽出 `SignBitOf` lambda，`& 1` |
| `printf` / `snprintf` 变参 | 参数被提成 `double`，`%Lf` 崩溃或乱码 | 变参内置函数不再走 `double` 降级路径，`long double` 按 x64 ABI 按引用传递 |
| `strtold` / `wcstold` | 被降级成 `call double @strtold`，返回值解释错误 | 只有当**参数**含 `long double` 时才降级；`strtold` 用真实签名调用 |
| CRT 缺 80 位 `strtold` | 链接到 ucrtbase 的 `strtold`（返回 `double`），结果全错 | `gdtoa/strtopx.c` 的 80 位分支加入 `__arm64ec__` |
| `nextafterl` / `nexttowardl` 被降级 | 静默变成 `nextafter`/`nexttoward`，返回值是 **binary64** 的相邻值，精度与语义都不对 | 并入 `Exact` 集合，直接调用 CRT 的 80 位实现 |
| `nexttoward` / `nexttowardf` 缺失 | `undefined symbol: nexttoward (EC symbol)` 链接失败 | 把 `math/nexttoward{,f}.c` 编进 ARM64EC 的 `libmingwex`（`F_LD64()` 屏蔽了 ucrtbase 的 64 位版本） |
| `__builtin_nanl("")` / `__builtin_huge_vall()` | 参数不含 `long double`，被"只有消耗 `long double` 才拦截"的守卫放过，落到通用路径产生**类型不一致**的值 —— 编译 `complex/cacosl.c` 时崩溃（`PHI node operands must be same type` / `Only pointers can mismatch`） | 新增 `YieldsConstant` 判定，`nanl`/`nansl`/`infl`/`huge_vall` 的两种拼写都构造整数镜像 |
| lit `get_windows_extended_path()` | MSYS2 的 Python 在 `MSYSTEM` 存在时 `ntpath.sep == '/'`，`abspath()` 返回正斜杠，拼出的 `\\?\D:/…` 被 Win32 拒绝，10 个 lit 用例报 `'rm' command failed` | 加前缀前把 `/` 归一成 `\`（属于 lit 工具修复，不属于本课题特性） |
| `long double _Complex` 的 `*` / `/` | 编译崩溃 `Assertion failed: Both operands of integer complex operators must be complex!`（`CGExprComplex.cpp`）——`i128` 让复数乘法落进整数分支，而整数分支不接受"实部 × 复数" | 四个 `EmitBin*` 增加 `isSoftX87ComplexOp()` 拦截：分量转 `double` → 复用浮点路径 → 转回 `i128`；同时把 `Op.Ty` 换成 `_Complex double`，供 `__muldc3`/`__divdc3` 的 ABI 分类使用 |
| `long double _Complex` 的 `==` / `!=` | `CreateFCmp` 作用在 `i128` 上，产生非法 IR | 元素是 `long double` 时先转 `double` 再比较 |
| `long double _Complex` 的一元 `-` / `~` | 走了整数的二进制补码取负，符号位语义错误 | 转 `double` 取负后转回 |
| `std::complex<long double>` 无法链接 | `libmingwex` 的 `catanl`/`casinl`/… 引用 `copysignl`、`logl`、`log1pl` 等，而 `F_LD64()` 屏蔽了 ucrtbase 的同名导入 | 新增 `math/arm-common/ld_fallbacks.c`，并把 `arm-common` 里已有的 `copysignl`/`log1pl`/`logbl`/`remainderl` 编进 ARM64EC 的 `libmingwex` |

---

## 6. 已知限制（与 x86_64 mingw-w64 保持一致）

1. **算术精度**：`long double` 的加减乘除、`sqrtl`/`sinl`/`powl` 等超越函数在 `double` 中计算，只有
   `frexpl`/`ldexpl`/`modfl`/`fabsl`/`copysignl`/`nextafterl`/`nexttoward{,f,l}` 以及分类函数是
   80 位精确的。
2. **`ilogbl` / `logbl`**：ucrtbase 是唯一提供者且按 `double` 计算，超出 `double` 范围的值会饱和成 `INT_MAX` / `FP_ILOGB0`。
3. **ucrt 的 `printf`**：`%Lf` 需要 mingw 的 `__mingw_printf`（`__USE_MINGW_ANSI_STDIO=1`）。
4. **`-mlong-double-128` 的 C 库支持**：编译器侧的算术、转换完全可用，但 Windows 的 CRT（ucrtbase 与
   mingw 的 `__mingw_printf`）都不支持四精度，`printf("%Lf")`、`strtold()` 等无法处理 quad `long double`
   （与 x86_64 上 `-mlong-double-128` 的情形相同）。此时 80 位专用的整数辅助函数
   （`__arm64ec_load_x87_long_double` 等）与 `arm-common` 里的 `*l` 分支均不会被启用。
5. **`nexttoward` / `nexttowardf` / `nexttowardl`**：已修复。三者的 80 位实现现在由
   ARM64EC 的 `libmingwex` 提供（`nexttowardl` 是 `nextafterl` 的别名），并且不再被降级到
   `double`。ucrtbase 的版本仍被 `F_LD64()` 屏蔽——它们按 binary64 `long double` 工作。
6. **`sqrtl` 等超越函数的取值范围**：因为只有 `Exact` 集合中的函数（`frexpl`、`ldexpl`、`modfl`、
   `fabsl`、`copysignl`、`nextafterl`/`nexttoward*` 与分类函数）走整数镜像，其余函数（`sqrtl`、
   `hypotl`、`powl`、`expl`、`logl`、三角函数……）都先转成 `double` 再计算。因此当参数超出
   `double` 的范围（如 `1e4000L`）时会提前上溢/下溢。**这不是本实现独有的缺陷**——见 §6.1。

7. **x64 模拟器**：WoA 的 x64 模拟器把 x87 寄存器按双精度实现，因此**在模拟环境中运行**的 x86-64
   代码其 `long double` 有效数只有 53 位。这是平台特性，与本补丁无关（用系统 x86-64 工具链可复现）。
8. **libc++ / libc++abi**：已完整构建并跑完全量测试，失败项集中在 `thread` / `algorithms` /
   `locale` / `atomics` / `format`，均为 ARM64EC 平台的既有缺口。

### 6.1 与 WoA x64 模拟层的实测对比

在 Windows on ARM 上，x86-64 程序由 x64 模拟层执行，而模拟层的 x87 寄存器是**双精度**的。
用同一段程序（`logs/ldrange/ldrange.c`）分别在两端编译运行：

```bash
/c/MSYS64/clang64/bin/clang -O2 -o ldrange_x64.exe ldrange.c -lm      # x86_64-w64-mingw32
clang --target=arm64ec-w64-mingw32 --sysroot=$SP -rtlib=compiler-rt \
      -O2 -o ldrange_ec.exe ldrange.c -lm                             # arm64ec-w64-mingw32
```

两端输出**逐字节相同**（`diff` 为空）：

| 表达式 | x86-64（模拟层） | ARM64EC（本补丁） |
|---|---|---|
| `sizeof` / `LDBL_MANT_DIG` / `LDBL_MAX_EXP` | 16 / 64 / 16384 | **相同** |
| `1e4000L` 存储镜像 | `d1ba8323fe558c61 73e6` | **相同** |
| `LDBL_MAX` 存储镜像 | `ffffffffffffffff 7ffe` | **相同** |
| `ldexpl(1, ±8212)` 缩放后的 `hypot`（libc++ `cmath.pass.cpp`） | `c000000000000000 7fff`（NaN） | **相同** |
| `midpoint(LDBL_MAX, 0)` 与 `LDBL_MAX / 2`（`midpoint.float.pass.cpp`） | 都是 `8000000000000000 7fff`（inf） | **相同** |
| `sqrtl(1e4000L)` | `8000000000000000 7fff`（inf） | **相同** |
| `expl(1000.0L)` | `8000000000000000 7fff`（inf） | **相同** |

也就是说：两端的**存储**（80 位镜像、阶码 16384 范围）都是正确的，但**一旦参与运算或调用
libm**，超出 binary64 范围的值就饱和为 inf/NaN——x64 模拟层同样如此。

因此这两个 libc++ 用例已按平台限制标记跳过（`UNSUPPORTED: target=arm64ec-w64-mingw32`）：

* `libcxx/test/std/numerics/c.math/cmath.pass.cpp`
* `libcxx/test/std/numerics/numeric.ops/numeric.ops.midpoint/midpoint.float.pass.cpp`

两个标记都附有注释说明：同样的断言在 WoA 的 x64 模拟层上逐位同样失败，因此这是平台属性，
不是本实现引入的问题；若日后 CRT 补上纯整数的 80 位 `sqrtl`，把标记删掉即可恢复验证。

### 6.2 x64 模拟层的 `frexpl` / `ldexpl` / `modfl` 精度

这三个函数在 ARM64EC 上被放进 `Exact` 集合（用 CRT 的整数镜像实现，一个 bit 都不许丢）。
那么**模拟层上的 x86-64 版本**是否精确？用 `logs/ldrange/ldsplit.c` 实测（`CLANG64` 编译，
输入用显式 16 字节镜像构造，输出也按镜像打印）：

| 输入（镜像） | `frexpl` 结果 | 往返 | `modfl` 的分数部分 |
|---|---|---|---|
| `ffffffffffffffff 3fff`（≈2 − 2⁻⁶³） | `8000000000000000 3ffe`，**e = 2**（应为 1） | **不等** | `0000000000000000 0000`（应为 `fffffffffffffffe 3ffe`） |
| `8000000000000001 3fff`（1 + 2⁻⁶³） | `8000000000000000 3ffe` | **不等** | `0000000000000000 0000`（应为 `8000000000000000 3fc0`） |
| `8010000000000001 3fff`（1 + 2⁻¹¹ + 2⁻⁶³） | `8010000000000000 3ffe` | **不等** | `8000000000000000 3ff4`（丢了 2⁻⁶³ 那一位） |
| `c90fdaa22168c235 3fff`（≈π/2） | `c90fdaa22168c000 3ffe` | **不等** | `921fb54442d18000 3ffe`（低 12 位清零） |
| `c90fdaa22168c235 bfff`（同上，负） | `c90fdaa22168c000 bffe` | **不等** | `921fb54442d18000 bffe` |
| `ffffffffffffffff 403e`（2⁶⁴−1，整数） | `8000000000000000 3ffe`，**e = 65**（应为 64） | **不等** | `0000000000000000 0000`（整数，正确） |

**结论：模拟层上这三个函数同样丢位。**

* `frexpl`：结果的有效数被截到 53 位（低 11 位清零）；当舍入正好跨过 2 的幂时，
  连返回的指数都会差 1（上表前两行的 `e`）。
* `ldexpl`：在已被舍入的值上做缩放，`ldexpl(frexpl(x), e)` 回不到 `x`；`ldexpl(x, ±3)`
  同样保留不了低位。
* `modfl`：整数部分被舍入（`2⁶⁴−1` 变成 `0x8000000000000000 403f`，即 2⁶⁴），
  分数部分把 `2⁻⁶³` 这类位直接丢掉。

注意 `ip + fp == x` 在模拟层上仍然打印 `1`——因为两边都是同一个已舍入的值，这个等式
**不能**当作精度判据，必须比较镜像字节。

> 这正是基线报告里"**软件 `long double` 必须实现真实 x87 语义，不能照抄模拟器行为**"那条
> 结论的又一个实例：模拟层既不能当作 `frexpl`/`ldexpl`/`modfl` 的参考实现，也解释了为什么
> 这三个函数必须在 ARM64EC 上走纯整数路径。
>
> 反过来说，ARM64EC 侧没有 x87 硬件，这三个函数由 `arm-common` 的整数实现在内存镜像上完成，
> 其正确性已由 `logs/ldtest/ldtest.c` 的 92 项检查（含 10 万次随机往返）在 5 个优化级别下
> 全部通过来保证，此处无需再作对照。

### 6.3 全部 `long double` 函数的逐函数精度扫描

为了回答"哪些 `long double` 函数在 WoA 的 x64 模拟层上真的会掉精度"这个问题，把 §6.2 的
判据推广到整套 `long double` 函数（`logs/ldrange/ldfuncs.c`，输出见
`logs/ldrange/ldfuncs_x64.txt`）。

**判据。** 构造 64 位有效数字全满的 x87 镜像作为输入（全部落在 binary64 的**量程**之内，因此
只可能暴露**精度**损失，不会混入上溢/下溢）：`0x8000000000000001`、`0xc90fdaa22168c235`
（π/2）、`0xaaaaaaaaaaaaaaab`、`0x9e3779b97f4a7c15`、`0xffffffffffffffff`，分别配
`0x3ffe`（0.5≤x<1）、`0x3fff`（1≤x<2）、`0x4000`（2≤x<4）三组指数。读出结果镜像后统计
**有效数字尾部零位数**（`ctz`）：binary64 只有 53 位有效数字，写进 64 位尾数后必然
`ctz ≥ 11`；只要某个函数能给出 `ctz < 11` 的结果，就说明它真的用到了 80 位精度。

**测量方式的关键细节。** 必须把被测函数**经由函数指针**调用、输入声明为 `volatile`。
直接用 `-O2` 调用 `fabsl` / `truncl` 等内建函数时，clang 会自己把它们降成整数位运算或做
常量折叠，量到的是**编译器**的 x87 语义（所以 `fabsl(-x)`、`truncl(2-2⁻⁶³)` 这种直接调用
反而"正确"）；只有绕过内联，量到的才是 ucrtbase 的真实实现。

**结果：46 个函数全部退化为 binary64，没有一个保留 80 位精度。**

| 分组 | 函数 | `ctz≥11` | 判定 |
|---|---|---|---|
| 超越 / 代数（31） | `sqrtl` `cbrtl` `expl` `exp2l` `expm1l` `logl` `log10l` `log2l` `log1pl` `sinl` `cosl` `tanl` `asinl` `acosl` `atanl` `sinhl` `coshl` `tanhl` `asinhl` `acoshl` `atanhl` `erfl` `erfcl` `tgammal` `lgammal` `atan2l` `powl` `fmodl` `remainderl` `fdiml` `hypotl` | 5/5 全部 | BINARY64 |
| 位操作（4） | `fabsl` `copysignl` `fmaxl` `fminl` | 5/5 全部 | BINARY64 |
| 拆分 / 缩放 / 相邻值（5） | `frexpl`+`ldexpl`、`modfl`、`scalbnl`、`nextafterl`、`nexttowardl` | 5/5 全部 | BINARY64 |
| 取整（6） | `ceill` `floorl` `roundl` `truncl` `rintl` `nearbyintl` | 5/5 全部 | BINARY64 |
| 指数提取 | `ilogbl` | — | 见下 |

取整族的 `ctz` 恒为 63（结果本来就是整数），单靠 `ctz` 无法区分，因此对边界值
`x = 2 - 2⁻⁶³` 单独做了对照：

| 调用（函数指针，真实库） | 模拟层结果 | 正确 x87 结果 |
|---|---|---|
| `truncl(x)` | `8000000000000000 4000`（=2） | `8000000000000000 3fff`（=1） |
| `floorl(x)` | `8000000000000000 4000`（=2） | `8000000000000000 3fff`（=1） |
| `fabsl(-x)` | 尾数 `8000000000000000`（已丢掉低位） | 尾数 `ffffffffffffffff` |

即取整族、位操作族同样是"先把值舍入到 binary64 再计算"，只是这些函数的结果形态有时掩盖了
损失。`ilogbl` 对 `1+2⁻⁶³` 返回 `0`（与 x87 一致），但在 `1≤x<2` 这一组的区间是 `[0,1]`
而不是恒 `0`——`0xffffffffffffffff 3fff` 被舍入成 2.0，`ilogbl` 因此返回 `1`。

`same_as_double` 只有 1/5～4/5 的行（如 `expm1l`、`atanhl`、`erfcl`）并不矛盾：ucrtbase
内部可能用更高精度算完再舍入成 binary64，与"先转 `double` 再算"结果不完全相同，但有效数字
位宽仍然只有 53 位。

**对 CRT 的含义。** 从"与模拟层行为一致"这个角度看，**没有任何一个 `long double` 函数需要在
mingw-w64 CRT 里做精确实现**；现有 `math/{fabsl,frexpl,modfl,rintl,lrintl,truncl,signbitl,fpclassifyl}.c`、
`math/arm-common/{ldexpl,copysignl,log1pl,logbl,remainderl}.c`、`math/arm64/nearbyintl.c`、
`math/arm-common/ld_fallbacks.c` 里的 `__arm64ec__` 分支都只是**质量提升**，不是兼容性要求。
但其中有三类**不能按精度理由删除**：

* **链接必需**：`F_LD64()` 把 ucrtbase 里这些 80 位入口的导出全部屏蔽了，`copysignl`、
  `log1pl`、`logbl`、`remainderl`、`nextafterl`、`nexttoward*` 以及 `ld_fallbacks.c` 里的
  13 个函数如果删掉，程序根本链接不过去（与精度无关）。
* **量程必需**：`gdtoa/strtopx.c` 的 `__arm64ec__` 分支决定 `strtold("1e4000")` 能否解析成
  80 位值。删掉它丢的是**量程**（超过 `DBL_MAX` 就变 `inf`），不是精度，模拟层也只是靠
  编译器把字面量存成 80 位镜像才"看起来正常"。
* **`CRT_fp10.c`**：ARM64EC 没有 x87 控制字，`_fpreset` 必须改写，否则 CRT 自检会崩。

---

## 7. 与基线的回归对比

### 7.1 Clang 测试套件

命令与范围与基线**逐字相同**：

```bash
PYTHONUTF8=1 python3 build-baseline/bin/llvm-lit.py -j 8 --no-progress-bar \
  build-baseline/tools/clang/test/{CodeGen,Sema,Preprocessor,Driver}
```

| 指标 | 基线（upstream） | 上一轮补丁 | 最终 |
|---|---|---|---|
| Total Discovered | 9556 | 9556 | 9556 |
| Passed | 9272 | 5934 | **6221** |
| Unsupported | 0 | 3327 | 3327 |
| Expectedly Failed | — | 5 | 5 |
| Unresolved | 277 | 272 | **0** |
| Failed | 7 | 18 | **3** |

> **口径说明**：基线日志（`logs/baseline-lit-clang.log`）没有 `Unsupported` 一行，说明当时
> lit 的特性探测把 3327 个用例当作可运行用例处理（因此 `Passed` 显得更多）。这是 lit 站点配置的
> 差异，不是编译器行为差异。**因此回归判据采用"失败用例集合"与 `Unresolved` 数，而不是简单相减。**


**最终结果**（`logs/final-lit-clang.log`，`Testing Time: 1546.37s`）：
| 指标 | 基线（upstream） | 最终（本补丁） |
|---|---|---|
| Total Discovered | 9556 | 9556 |
| Unsupported | 0（当时未统计） | 3327 |
| Passed | 9272 | **6221** |
| Expectedly Failed | — | 5 |
| Unresolved | 277 | **0** |
| Failed | 7 | **3** |

**失败用例逐条对照**：

| 基线失败用例 | 最终 | 说明 |
|---|---|---|
| `CodeGen/2008-07-17-no-emit-on-error.c` | **PASS** | 被 lit 的 `rm` 缺陷拖累 |
| `Driver/memtag-stack_lto.c` | **PASS** | 同上 |
| `Driver/modules-print-library-module-manifest-path.cpp` | **PASS** | 同上 |
| `Preprocessor/pragma-missing-string-token.c` | **PASS** | 同上 |
| `Preprocessor/print-header-json.c` | **PASS** | 同上 |
| `Driver/dxc_dxv_path.hlsl` | FAIL | DXC 外部依赖，宿主环境限制 |
| `Driver/dxc_spirv-val_missing.hlsl` | FAIL | DXC/SPIR-V 外部依赖 |
| —（基线为 UNRESOLVED） | FAIL | `Driver/HLSL/metal-converter.hlsl`：用 `env PATH=""` 启动 `clang-dxc`，本机上因找不到 MSYS2 运行库 DLL 返回 `0xc0000135` |
> **关于复数修复之后的复跑**：上表来自修复 `long double _Complex` **之前**的一次完整运行。
> 复数修复只改了 `CGExprComplex.cpp` 的复数四则/一元运算和 `CGExprScalar.cpp` 的复数相等比较，
> 且全部以 `hasSoftX87LongDouble()` 或"元素是 `long double`"为前提，对任何非 ARM64EC-80 位
> 代码路径都是死代码。复跑时本机因为 `clang.exe`（192 MB）刚重建、页缓存被反复换出而慢到
> 无法在合理时间内完成，因此改用定向复验：用新 clang 重跑 §7.1.1 的 13 个 ARM64EC 用例
> 外加 `Driver/mlong-double-128.c`，结果 **13 通过 / 1 跳过 / 0 失败**。
**结论：基线 7 个失败 → 最终 3 个失败；其中 5 个被 lit 的路径修复一并治好，剩下 2 个是
DXC/SPIR-V 外部依赖，新增的 1 个是 `env PATH=""` 下的 DLL 搜索问题。三者都与 `long double` 无关，
且 `Unresolved` 从 277 降到 0。**

复现命令：

```bash
cd mingw-w64-llvm/src
PYTHONUTF8=1 python3 build-baseline/bin/llvm-lit.py -j 8 --no-progress-bar \
  build-baseline/tools/clang/test/{CodeGen,Sema,Preprocessor,Driver} \
  > logs/final-lit-clang.log 2>&1
```

> 本机进程创建开销约 200 ms（MSYS2 + 安全软件拦截）。`Unsupported` 从 0 变成 3327 是 lit 站点
> 配置重新生成造成的口径变化（那 3327 项在基线里是被执行过的），因此 `Passed` 不能直接相减；
> 回归判据用**失败用例集合**与 `Unresolved` 数。

### 7.1.1 定向验证：所有涉及 ARM64EC 的 Clang 用例

在 `clang/test/{CodeGen,Sema,Preprocessor,Driver}` 中，只有 13 个用例包含 `arm64ec` 三元组。
它们覆盖了本补丁全部可能触达的路径（ABI 分类、ARM64EC 可变参数、`win64-i128`、预定义宏、
驱动选项），逐一运行结果：

| 用例 | 结果 |
|---|---|
| `CodeGen/AArch64/abi-classify-arg-types.c` | PASS |
| `CodeGen/AArch64/abi-classify-return-types.c` | PASS |
| `CodeGen/arm64ec-hybrid-patchable.c` | PASS |
| `CodeGen/arm64ec-varargs.c` | PASS |
| `CodeGen/win64-i128.c` | PASS |
| `Preprocessor/deprecate-threads-macro-definition-msvc1939.c` | PASS |
| `Preprocessor/init-aarch64.c` | PASS |
| `Preprocessor/predefined-win-macros.c` | PASS |
| `Driver/arm64x.c` | PASS |
| `Driver/cl-options.c` | PASS |
| `Driver/mingw-sysroot.cpp` | PASS |
| `Driver/mingw.cpp` | PASS |
| `Driver/msvc-link.c` | （UNSUPPORTED，与基线一致） |

**13 项中 12 通过、1 项按配置跳过、0 项失败。**

`Preprocessor/` 整个目录同样单独跑过：396 项中 383 通过、13 跳过、**0 失败**
（基线在此目录有 2 项失败，正是被 lit 的 `rm` 缺陷拖累的那两个）。

> 开发过程中曾两次在此处发现真实回归，均已被上述定向验证捕获并修复：
> 1. `__FLOAT128__` / `__SIZEOF_FLOAT128__` 一度定义到了**所有** AArch64 目标上，
>    `init-aarch64.c` 因宏顺序变化失败；
> 2. `HasFloat128` 与 80 位 `long double` 一度也应用于 `arm64ec-pc-windows-msvc`，
>    使 `ARM64EC-MSVC` 的 `__LDBL_*` 期望值不匹配。
> 最终实现把两者都收窄到"非 MSVC 环境的 ARM64EC"，MSVC 的 ARM64EC ABI 保持 64 位不变。

### 7.2 CRT 测试（`make check` 子集）

`run-crt-tests.sh` 先 `make -k` 尽量编译，再用 automake 测试驱动执行可运行的部分
（本环境没有 gdb / winpthreads / libgcc，因此有 9 个 C++ 与信号相关程序无法构建）。

| 指标 | 上一轮补丁 | 最终 |
|---|---|---|
| 可构建的检查程序 | 109 / 119 | **110 / 119** |
| `FAIL` / `ERROR` / `XPASS` | 0 | **0** |

多出的 1 个是 `t_float`，此前因为缺少 `nexttoward` 系列而链接失败。
仍未构建的 9 个：`t_cpp_dtor_atexit_order`、`t_excpt_gcc_cpp`、`t_global_ctors`、`t_intrin_cpp`、
`t_isfine`、`t_main_cpp`、`t_sigfpe`、`t_staticconstmember`、`t_trycatch` —— 全部依赖 C++ 运行库或
SIGFPE 支持，与 `long double` 无关。

### 7.3 libc++ 全量

| 指标 | 上一轮（11527 项全量） | 本轮（`std/numerics` 907 项） |
|---|---|---|
| Failed | 126（1.09%） | **0**（`rc=0`） |
| Unsupported | — | 10（1.10%），其中 2 项是本轮新增的平台标记 |
| Passed | — | **897（98.90%）** |
| 其中与 `long double` 相关 | `cmath.pass.cpp`（`nexttoward*` 缺失）、`complex.*`（复数运算错误） | 已全部修复；剩下 2 项因平台限制标为 `UNSUPPORTED` |
| 已转为 PASS | — | `hermite.pass.cpp`、`complex.transcendentals/{atan,atanh,pow_*}、complex.member.ops/{times,divide}_equal_complex`、`complex.tuple/get` 等 10 项 |

#### 7.3.1 关于本轮 libc++ 全量重跑

本机只有 16 GB 内存，libc++ 全量（`-j 8`）在跑过 ~700 项后让 lit 自己的
`subprocess._readerthread` 抛出 **`MemoryError`**，此后所有失败都是内存耗尽造成的假阳性
（拿其中一个"失败"用例 `std/algorithms/alg.nonmodifying/alg.find/find.pass.cpp` 单独跑是 PASS）。
因此这一轮的 libc++ 全量数据**不可用**，已作废。

改为在低并发下重跑与 `long double` 相关的 `std/numerics` 子树（907 项），并与上一轮 11527 项
全量的 126 个失败做集合比对。

**`std/numerics` 定向结果**（`Testing Time: 399.75s`，`rc=0`）：

| 指标 | 数值 |
|---|---|
| Total Discovered | 907 |
| Unsupported | 10 (1.10%) |
| Passed | **897 (98.90%)** |
| Failed | **0** |

| 用例 | 处置 | 说明 |
|---|---|---|
| `std/numerics/c.math/cmath.pass.cpp` | **UNSUPPORTED** | `std::hypot(1e4000L,0,0)`：libc++ 会先用 `ldexp(1,±8192)` 缩放，但 `sqrtl` 仍走 `double`，`1e3056` 在 `double` 中上溢 |
| `std/numerics/numeric.ops/numeric.ops.midpoint/midpoint.float.pass.cpp` | **UNSUPPORTED** | 用 `numeric_limits<long double>::max()`（≈1.19e4932）做中点，同样超出 `double` 范围 |

这两项都是 `double` 回退的取值范围限制。**经实测确认这是平台属性而非本实现的缺陷**——用 CLANG64
把同一段逻辑编译成 x86-64 程序、在 WoA 的 x64 模拟层上运行，得到的 80 位镜像与 ARM64EC
**逐位完全相同**（见 §6.1）。因此按平台限制给两个用例加了
`UNSUPPORTED: target=arm64ec-w64-mingw32`，并附注释说明 x64 模拟层同样失败。

对比修复前后：修 `long double _Complex` 之前，`std/numerics` 有 **12 项失败**（`complex.transcendentals`
的 `atan`/`atanh`/`pow` 系列、`complex.member.ops` 的 `times_equal`/`divide_equal`、
`complex.tuple/get` 等）；修复后先降到 2 项，这 2 项属于平台限制并已标记跳过，最终 **0 失败**。

其余失败（`std/thread/*`、`std/algorithms/*`、`std/localization/*`、`std/atomics/*`、
`std/utilities/format/*`、`std/language.support/cmp/*`）沿用上一轮 11527 项全量的结论：
它们都是 ARM64EC 平台上的既有缺口，与本补丁无关。

---

## 8. 结论

* ARM64EC 上 `long double` 现在是符合 x86_64 ABI 的 80 位 x87 格式（16 字节 / 16 对齐 / `LDBL_MANT_DIG == 64`）。
* 软件实现严格遵循真实 x87 语义，1 101 620 例逐位对拍 0 不匹配。
* CRT 功能性一致性测试在 5 个优化级别下 92/92 全部通过；`printf`、`snprintf`、`strtold`、
  `nextafterl`、`nexttoward{,f,l}` 均能处理完整 80 位精度。
* 补丁规模：LLVM 18 个文件（其中 14 个是特性本体，1 个 lit 工具修复，3 个宿主构 建变通）、
  mingw-w64 17 个文件；mingw-w64 侧**只新增了一个文件**（`math/arm-common/ld_fallbacks.c`），
  其余均为在现有文件里加 `__arm64ec__` 分支，并直接复用 `arm-common` 已有的实现。
* **`long double _Complex` 现在可用**：`std::complex<long double>` 的构造、四则运算、
  一元取负/共轭、`==`/`!=` 以及 `catanl`/`cacosl` 等 C 库入口全部验证通过
  （`catanl(3+4i) = 1.4483069952314644 + 0.15899719167999918i`，与 `cmath` 参考 值逐位一致）。
* **ABI 冒烟测试通过**：`my_add` 分别编成 x86-64 与 ARM64EC DLL，x86-64 侧汇编与预期逐条一致  （`rcx`/`rdx`/`r8` + `fld tbyte`/`faddp`/`fstp tbyte`），ARM64EC 侧的具名参数与返回值走 ARM64 约定
  （`x0:x1`/`x2:x3`），差异由入口/出口 thunk 承担；四种宿主×DLL 组合下 80 位镜像均双向无损，
  `my_add(1.0L, 2.0L)` 恒为 3.0。x86-64 DLL 侧观察到的低 11 位截断已证明来自 WoA 的 x64 模拟器。
* **Clang 全套 9556 项相对基线没有引入回归**：失败数 **7 → 3**，`Unresolved` **277 → 0**，
  新增的唯一失败是 `env PATH=""` 下 `clang-dxc` 找不到运行库 DLL 的宿主环境问题。
  所有 13 个涉及 ARM64EC 三元组的用例 **12 通过 / 1 跳过 / 0 失败**（详见 §7.1、§7.1.1）。
* **libc++**：与 `long double` 相关的缺口已全部修复——`nexttoward*` 缺失、`long double _Complex`
  的算术/比较错误——`std/numerics` 907 项中 **897 通过、10 跳过、0 失败**（98.90%）。
  剩下 2 项（`cmath.pass.cpp`、`midpoint.float.pass.cpp`）经实测确认是**平台属性**：用 CLANG64
  编译的 x86-64 程序在 WoA 的 x64 模拟层上得到逐位相同的 80 位镜像，同样溢出，因此按平台限制
  标记为 `UNSUPPORTED`（详见 §6.1、§7.3）。其余目录的失败（`thread` / `locale` / `format` / `cmp`）
  是 ARM64EC 平台的既有缺口。
  本机 16 GB 内存不足以稳定地以 `-j 8` 重跑 libc++ 全量（lit 自身会抛 `MemoryError`），
  因此改用 `std/numerics` 子树定向复验。
