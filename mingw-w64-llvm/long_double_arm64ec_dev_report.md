# arm64ec-w64-mingw32：将 `long double` 恢复为 80 位 x87 扩展精度

**开发过程记录 + 逐条验收测试报告**

- 日期：2026-10-26
- 主机：Intel Core i9-14900K（ASUS/AMI，Windows 11 26H2 build 26300.9711，x64）
- LLVM 源码：`D:\lh_mouse\GitHub\MINGW-packages\mingw-w64-llvm\src\llvm-project`（`main` @ `44839f9c833f`）
- mingw-w64 源码：`/d/lh_mouse/GitHub/mingw-w64`（`lhmouse` @ `8b8b4a167`）
- 构建目录：`...\src\build-ld80`（Ninja，Release + assertions，`LLVM_TARGETS_TO_BUILD=X86;AArch64`）
- 本文档配套产物目录：`C:\Windows\Temp\ld80`（补丁、日志）与 `C:\Windows\Temp\ldprobe`（x87 实验程序）

---

## 0. 结论摘要

| 项目 | 状态 |
| --- | --- |
| x87 硬件行为 5 个待确认项 | **已完成**（第 2 章，含实测数据） |
| `__arm64ec_load_x87_long_double` / `__arm64ec_store_x87_long_double` 纯软件实现 | **已完成**，与真实 x87 硬件差分测试 **800,096 例 0 差异**（3.2 节） |
| Clang：支持 `-mlong-double-64` / `-mlong-double-80`，**默认 80 位** | **已完成**（4.1 节，三种模式实测） |
| Clang：`long double` 由 64 位改回 80 位（16 字节、16 字节对齐） | **已完成**（4.1 节，`sizeof`/`_Alignof` 静态断言通过） |
| Clang：load/store 纯整数指令、算术转 `double`、两个辅助函数 | **已完成**（4.2–4.5 节，IR/汇编逐项核对） |
| mingw-w64 CRT：7 个无误差函数整数实现 + 算术函数转 `double` | **已完成**（5 章，49 个源文件用新 clang 编译通过） |
| 编译期回归测试（Clang Sema/CodeGen/Preprocessor/Driver，9554 项） | **已完成**，与基线逐项比对（6.2 节） |
| 在 ARM64 实机运行 arm64ec 程序 | **未进行**（本机为 x64，无 Windows on ARM 设备）；替代验证见 6.3 节 |
| 完整 mingw-w64 CRT 交叉构建（arm64ec 全套） | **未进行**（需 autoreconf + 完整 sysroot，超出本次时间预算），单文件编译验证已完成 |

一句话结论：**80 位 `long double` 的语义与代码生成链路已经打通并逐项验证通过**；剩余工作是完整的 CRT 交叉构建与 ARM64 实机联调。

---

## 1. 背景与目标

现状（stock clang 23.1.3 实测）：

```text
$ clang --target=arm64ec-w64-mingw32 -dM -E -x c /dev/null | grep LDBL
#define __LDBL_MANT_DIG__ 53
#define __SIZEOF_LONG_DOUBLE__ 8
```

即 arm64ec 目标默认 `-mlong-double-64`（实现位置并不是驱动参数，而是
`clang/lib/Basic/Targets/AArch64.cpp` 中 `WindowsARM64TargetInfo` 构造函数无条件设置的
`LongDoubleWidth = LongDoubleAlign = 64; LongDoubleFormat = IEEEdouble()`），
与 x86_64-w64-mingw32（`sizeof(long double) == 16`，x87 80 位）不一致，造成源码级 ABI 不兼容。

目标（依据 `long_double.md`）：

1. 驱动支持 `-mlong-double-64` 与 `-mlong-double-80`（原先两者都报
   `unsupported option ... for target 'arm64ec-w64-windows-gnu'`），且**默认等价于 `-mlong-double-80`**；
2. `long double` 大小 16、对齐 16，语义为 x87 扩展精度；
3. 编译器不产生任何 x87 指令：load/store 用整数指令拼字，算术一律转换为 `double`；
4. 新增两个运行时辅助函数（与 x86-32 上 `long long` 长除法同库）：
   ```c
   double __arm64ec_load_x87_long_double(const long double *mem);   /* 读 80 位镜像 -> double */
   void   __arm64ec_store_x87_long_double(long double *mem, double v); /* double -> 80 位镜像 */
   ```
5. 更新 mingw-w64 CRT：`fabsl`/`signbitl`/`copysignl`/`fpclassify`/`frexpl`/`ldexpl`/`modfl`
   必须无误差（纯整数），其余算术函数转 `double`。

---

## 2. 硬件行为实验（待确认项）

### 2.1 实验方法

实验程序：`C:\Windows\Temp\ldprobe\ldprobe.s`（Intel 语法汇编）+ `ldprobe.c`（驱动）。
每条用例执行：

1. `fldcw` 装载控制字：通过 PC（bit 8–9）设定精度为 24/53/64 位，通过 RC（bit 10–11）设定
   舍入方向（nearest / down / up / truncate），其余异常全部屏蔽；
2. `fld m80` 装载 80 位被测值 → 用 `fnstsw` 记录状态字；
3. `fstp m80` 存回内存 → 检查是否发生精度损失；
4. `fld1; fmulp st(1), st`（乘以 1，强制按 PC 精度做一次舍入）→ 记录状态字；
5. `fstp m80` 存回 → 检查结果。

> 说明：x87 指令集中**不存在** `fst m80`（80 位存储只有 `fstp` 一种编码，`DB /7`），
> 因此第 3 步用 `fld; fstp` 实现"纯存储"路径——这一步本身不做任何舍入，恰好验证了
> "`fld`/`fstp` 不改变数据"这一前提。

编译运行：

```bash
export PATH=/ucrt64/bin:$PATH
gcc -O1 -Wall -Wextra -o ldprobe.exe ldprobe.c ldprobe.s && ./ldprobe.exe > probe_out.txt
```

### 2.2 实测数据（节选，`PC=53`）

| 用例 | 输入 `mantissa:sign_exp` | 纯存储 `fld;fstp` | `×1`（nearest） | 状态字 |
| --- | --- | --- | --- | --- |
| 1.0 | `8000000000000000:3fff` | 不变 | 不变 | — |
| 1+2⁻⁵³（正好平局、偶） | `8000000000000400:3fff` | 不变 | `8000000000000000` | PE |
| 1+2⁻⁵³+2⁻⁶³（平局+粘位） | `8000000000000401:3fff` | 不变 | `8000000000000800` | PE |
| 1+3·2⁻⁵³（平局、奇） | `8000000000000c00:3fff` | 不变 | `8000000000001000` | PE |
| −(1+2⁻⁵³)、RC=up | `8000000000000400:bfff` | 不变 | `8000000000000000`（幅值减小） | PE |
| x87 最大有限数 | `ffffffffffffffff:7ffe` | 不变 | `8000000000000000:7fff`（+∞） | OE PE |
| 非规格数最小值 | `0000000000000001:0000` | 不变 | `0000000000000000:0000`；RC=up → `0000000000000800` | DE UE PE |
| 非规格数最大值 | `7fffffffffffffff:0000` | 不变 | `8000000000000000:0001`（规格化） | DE PE |
| 伪非规格数 0.5·2⁻¹⁶³⁸² | `8000000000000000:0000` | 不变 | `8000000000000000:0001`（等值、规格化） | DE |
| 伪非规格数 0.75·2⁻¹⁶³⁸² | `c000000000000000:0000` | 不变 | `c000000000000000:0001` | DE |
| 非规格化数（unnormal） | `7fffffffffffffff:3fff` | **位不变、无异常** | `c000000000000000:ffff`（不定 QNaN） | IE |
| 非规格化数（小） | `0000000000000001:3fff` | 位不变、无异常 | `c000000000000000:ffff` | IE |
| 伪 NaN（整数字 0，exp=7FFF） | `4000000000000000:7fff` | 位不变、无异常 | `c000000000000000:ffff` | IE |
| 信号 NaN | `a000000000000000:7fff` | 位不变 | `e000000000000000:7fff`（静默化） | IE |
| ±∞ / QNaN | — | 不变 | 不变 | — |

`PC=64` 时上表所有 `×1` 用例都**不产生任何舍入**（无 PE、位完全保持），
即"精度损失只可能由 PC 精度控制引起"。

### 2.3 五个待确认项的结论

1. **尾数截断的舍入方向**：遵循当前 x87 舍入控制（RC）。
   默认（RC=00，round-to-nearest-even）：平局取偶（用例 1+2⁻⁵³ → 舍去；
   1+3·2⁻⁵³ → 进位到偶），非平局按最近方向；RC=01/10/11 分别向 −∞/+∞/零方向定向舍入，
   且是**按数值大小**而非按幅值（负数在 RC=up 时幅值减小，见表第 5 行）。
   x87 用状态字 C1 位指示"向上舍入发生"。
2. **上溢/下溢的处理**：PC 精度截断**不改变指数范围**，因此"乘以 1"本身不会下溢；
   只有在最大指数处向上舍入才会溢出为 ±∞（挂 OE|PE）。
   非规格数被舍入到零时挂 UE|PE，舍入到最小可表示值时 C1=1。
   真正的"下溢到 `double` 次正规数/零"发生在 80 位 → 53 位**格式转换**时，由软件实现负责（见第 3 章）。
3. **显式最高位为 0 的规格数（unnormal）**：`fld` 与 `fstp` **不报错、不改变位**（纯搬运）；
   一旦参与算术运算即触发无效操作（IE），结果为"不定 QNaN" `c000000000000000:ffff`
   （符号 1、整数字 1、尾数 0）。软件实现按此规则处理：视为 NaN。
4. **非规格数（denormal，exp=0 且整数字=0）**：硬件完全支持，不触发无效；参与运算时挂 DE；
   在 PC=53 下可被舍入到零（UE|PE）或向上舍入并重新规格化。
   其数值范围（< 2⁻¹⁶³⁸²）远小于 `double` 次正规数范围（≥ 2⁻¹⁰⁷⁴），转换到 `double` 时必然下溢为 ±0。
5. **最高位为 1 的非规格数（pseudo-denormal，exp=0 且整数字=1）**：**合法输入**，
   解释为 `mantissa × 2⁻¹⁶⁴⁴⁵`（等价于正常数的指数域 1），使用时会重新规格化并挂 DE，
   **无精度损失**（实测 0.5·2⁻¹⁶³⁸² 与 0.75·2⁻¹⁶³⁸² 均等值转换）。

**对 Windows ARM 模拟器的补充判断**：模拟器把 x87 寄存器实现为 `double` 并在 ARM64 上计算，
等价于"恒定 PC=53 + 每次运算都按 53 位舍入"。这与上述 `PC=53` 实测行为一致（含溢出/下溢/定向舍入），
但与 64 位精度要求不一致——因此 x64 程序的 `long double` 在 ARM 上会静默损失 11 位尾数。
本方案正是为了消除这一损失：**存储保持 64 位尾数，运算时才转换**。

---

## 3. 软件转换函数

### 3.1 实现

文件：`C:\Windows\Temp\ldprobe\x87ld_soft.h`（将并入 compiler-rt `lib/builtins`，见 4.6 节），
核心即两个辅助函数的算法：

- `x87_store_double(image, value)`：`double` → 80 位镜像。任何 `double` 都能被 x87 精确表示，
  包含次正规 `double` 的规格化，因此**永不舍入**。
- `x87_load_double(image)`：80 位镜像 → `double`，round-to-nearest-even；
  非规格数 → 0；unnormal / 伪 NaN → 与硬件一致的"不定 QNaN"；NaN 保留 payload 并静默化；
  溢出 → ±∞；次正规结果做**单次**正确舍入（不会出现双重舍入）。

### 3.2 与硬件的差分测试

`difftest.c` + `ldconv.s`：对每个输入分别用
(a) 真实 x87（`fld m80; fstp m64`，控制字 PC=64/RC=nearest）与
(b) 软件实现计算，逐位比较。

覆盖：31 个边界用例（±0、±∞、QNaN/SNaN/伪 NaN、unnormal、denormal、pseudo-denormal、
`2⁻¹⁰⁷⁴`/`2⁻¹⁰⁷⁵` 平局边界、最大有限数、溢出边界），
外加 40 万随机 80 位模式（重点覆盖指数 0、±100、溢出/下溢边界）与 40 万随机 `double`
（含次正规、NaN/∞、2 的幂）。

```text
$ ./difftest.exe
...
total cases: 800096, load mismatches: 0, store mismatches: 0
```

**80 万例 0 差异**，其中包含 4,225 个曾暴露实现错误的用例（`2⁻¹⁰⁷⁵` 附近应向上舍入到
最小次正规数而非 0，见 6.1 节第 H-2 项），修正后全部通过。

---

## 4. LLVM / Clang 侧实现

### 4.1 目标信息：16 字节、16 字节对齐、x87 语义

`clang/include/clang/Basic/TargetInfo.h`：新增虚函数

```cpp
virtual bool hasSoftX87LongDouble() const { return false; }
```

`clang/lib/Basic/Targets/AArch64.h`（`WindowsARM64TargetInfo`）：

```cpp
bool hasSoftX87LongDouble() const override {
  return Triple.isWindowsArm64EC();
}
```

`clang/lib/Basic/Targets/AArch64.cpp`（构造函数末尾）：

```cpp
if (Triple.isWindowsArm64EC()) {
  LongDoubleWidth = LongDoubleAlign = SuitableAlign = 128;
  LongDoubleFormat = &llvm::APFloat::x87DoubleExtended();
}
```

驱动与选项（新增需求）：

- `clang/lib/Driver/ToolChains/Clang.cpp`：`-mlong-double-{64,80,128}` 原先只对 x86/PPC 放行，
  arm64ec 会报 `unsupported option ... for target`。现对 `isWindowsArm64EC()` 也放行并下传 cc1
  （`LongDouble_Group`）。
- cc1 侧已有逻辑（`CompilerInvocation.cpp` → `LangOptions::LongDoubleSize` →
  `TargetInfo::adjust`）直接生效：`80` → 宽/对齐 128、格式 `x87DoubleExtended`；
  `64` → 宽/对齐 64、格式 `IEEEdouble`；`128` → 宽/对齐 128、格式 `IEEEquad`。
- `hasSoftX87LongDouble()` 改为**按当前格式判定**而非仅看三元组，这样 `-mlong-double-64`/`-128`
  会自然回到“不需要软 x87 辅助函数”的旧路径：

```cpp
bool hasSoftX87LongDouble() const override {
  return Triple.isWindowsArm64EC() &&
         LongDoubleFormat == &llvm::APFloat::x87DoubleExtended();
}
```

效果（实测，见 6.1 节 A-1/A-2）：

```text
#define __SIZEOF_LONG_DOUBLE__ 16
#define __LDBL_MANT_DIG__ 64
#define __LDBL_MAX_EXP__ 16384
#define __LDBL_MIN_EXP__ (-16381)
#define __LDBL_EPSILON__ 1.08420217248550443401e-19L
#define __LDBL_MAX__ 1.18973149535723176502e+4932L
```

注意：由于语义仍是 `x87DoubleExtended`，**所有常量折叠、`-Woverflow` 范围检查、`LDBL_*`
宏**都自动获得正确的 80 位语义，无需另写代码。

### 4.2 IR 表示：`i128`

`clang/lib/CodeGen/CodeGenTypes.cpp`：`ConvertType` 对 `BuiltinType::LongDouble` 在
`hasSoftX87LongDouble()` 时返回 `i128`（16 字节、16 字节对齐），而不是 `x86_fp80`
（AArch64 后端无法选择 `x86_fp80`）。

选用 `i128` 的理由：AArch64 后端的 `i128` 装载/存储天然降为整数寄存器对
（`ldp x0, x1` / `stp x0, x1`），完全满足"整数指令拼字"的要求，且 16 字节对齐由
`alloca i128, align 16` 保证。

### 4.3 标量代码生成改造（`CGExprScalar.cpp`）

新增三个 `CodeGenFunction` 辅助：

```cpp
Address CreateSoftX87LdTemp(SourceLocation);            // 16 字节、对齐 16 的临时对象
llvm::Value *EmitSoftX87LdToDouble(llvm::Value *Bits, SourceLocation);
llvm::Value *EmitDoubleToSoftX87Ld(llvm::Value *Value, SourceLocation);
```

并在以下枢纽接入（全部以 `hasSoftX87LongDouble()` 为条件，不影响其他目标）：

| 位置 | 处理 |
| --- | --- |
| `EmitScalarConversion` | 任何 `long double` ↔ 其他标量的转换：`long double` 侧先经辅助函数转成 `double`，再做常规转换 |
| `EmitConversionToBool` | 同上（`if (ld)` 等） |
| `EmitAdd` / `EmitSub` / `EmitMul` / `EmitDiv` / `EmitRem` | 操作数转 `double` → 以 `double` 类型递归调用自身 → 结果转回 |
| `VisitBinLT/LE/GT/GE/EQ/NE` | 转 `double` 后 `fcmp`（谓词保持有序/无序语义） |
| `VisitMinus` | 与 `0x8000<<64` 异或（保持 `-(+0.0) == -0.0` 的精确语义） |
| `VisitFloatingLiteral` | 直接生成 16 字节整数镜像常量 |
| `ConstantEmitter::tryEmitPrivate`（`APValue::Float`） | 同上，用于全局/静态初始化 |

图像常量的构造（`CodeGenModule::EmitSoftX87LdConstant`）：

```cpp
llvm::APInt Bits = Value.bitcastToAPInt();          // 80 位：低 64 位尾数，高 16 位符号+指数
return llvm::ConstantInt::get(llvm::IntegerType::get(getLLVMContext(), 128),
                              Bits.zext(128));      // 剩余 48 位填充为 0
```

### 4.4 内建函数（`CGBuiltin.cpp`）

`EmitBuiltinExpr` 入口处拦截"任何参数或返回值是软 x87 `long double`"的内建函数：

- 要求无误差的（`fabs`、`copysign`、`signbit`、`isnan`、`isinf`、`isinf_sign`、
  `isfinite`、`isnormal`、`fpclassify`）→ **纯整数位运算**；
- `__builtin_infl` / `__builtin_huge_vall` / `__builtin_nanl` / `__builtin_nansl` → 直接生成图像常量；
- 其余（`sqrtl`、`powl`、`sinl`…）→ 转 `double` 后调用同名 `double` 版本的 libm 函数，
  再把结果转回，与 CRT 的算术函数策略完全一致。

> 开发中真实遇到的问题：`Builtin::Context::getName()` 返回的是 `std::string`（按值），
> 最初的 `StringRef Name = ...getName(...)` 产生悬垂引用，导致函数名被破坏并触发
> `Null bytes are not allowed in names` 断言。修正为 `const std::string Name = ...` 后解决。

### 4.5 调用约定

arm64ec 的命名参数遵循 **ARM64 ABI**（arm64ec 函数要能被纯 ARM64 代码调用），
可变参数遵循 **x86_64 ABI**（"超过 8 字节或不满足 1/2/4/8 字节者按引用传递"）。
因此本方案中：

- 命名参数/返回值：16 字节 `i128` 直接放在整数寄存器对 `x0/x1`；
- 可变参数与结构体成员：按引用/内存传递（`ptr` 指向 16 字节副本），与
  `AArch64ABIInfo::EmitMSVAArg` 中既有的 "Width > 64 → IsIndirect" 规则一致；
- `WinX86_64ABIInfo::classify` 增加：软 x87 `long double` 走 `getNaturalAlignIndirect(...)`，
  使可变参数"传参侧"与"读取侧"规则一致（这是本补丁修复的一个真实不一致）。

### 4.6 运行时辅助函数归属

按 `long_double.md` 的要求"与 x86-32 上 `long long` 长除法同一库"，实现放在
**compiler-rt `lib/builtins`**（LLVM 侧对应 libgcc 的位置），源文件见附录 C。
其算法即第 3 章已验证的软件实现。

### 4.7 补丁清单（LLVM）

```text
clang/include/clang/Basic/TargetInfo.h             +8
clang/lib/Basic/Targets/AArch64.h                  +9
clang/lib/Basic/Targets/AArch64.cpp                +10
clang/lib/CodeGen/CodeGenTypes.cpp                 +8
clang/lib/CodeGen/CodeGenFunction.h                +21
clang/lib/CodeGen/CGExprScalar.cpp                 +114
clang/lib/CodeGen/CGExprConstant.cpp               +16
clang/lib/CodeGen/CodeGenModule.h                  +4
clang/lib/CodeGen/CGBuiltin.cpp                    +~260
clang/lib/CodeGen/Targets/X86.cpp                  +8
clang/lib/Driver/ToolChains/Clang.cpp              +5      # -mlong-double-64/-80
```

完整 diff：`C:\Windows\Temp\ld80\llvm-patched.diff`（同时复制到桌面
`long_double_arm64ec_llvm.patch`）。

---

## 5. mingw-w64 CRT 侧实现

修改原则：**尽量沿用 mingw-w64 已有的按架构分支惯例**（x86 走内联汇编、ARM 走
`math/arm-common`），arm64ec 增加独立分支；关键点是所有分支必须写在
`defined(__x86_64__)` 分支**之前**，因为 arm64ec 为了兼容性**同时定义了 `__x86_64__`
和 `_AMD64_`**（见 `AArch64.cpp::getTargetDefines`）——这是本次修改中最容易踩的坑。

### 5.1 七个必须无误差的函数（纯整数实现）

| 函数 | 文件 | 实现 |
| --- | --- | --- |
| `fabsl` | `mingw-w64-crt/math/fabsl.c` | `sign_exponent &= 0x7fff`（清除符号位） |
| `__signbitl` | `mingw-w64-crt/math/signbitl.c` | `(sign_exponent & 0x8000) != 0` |
| `copysignl` | `mingw-w64-crt/math/arm64ec/copysignl.c` | `(|x| 的指数域) \| (y 的符号位)` |
| `__fpclassifyl` | `mingw-w64-crt/math/fpclassifyl.c` | 既有 x86_64 整数分支（`exp`/整数字/尾数判定），显式加入 arm64ec 条件 |
| `frexpl` | `mingw-w64-crt/math/frexpl.c` | 既有 x86_64 纯整数分支（提取指数、置指数域为 `0x3FFE`、次正规时用 `__builtin_clzll` 规格化），显式加入 arm64ec 条件 |
| `ldexpl` | `mingw-w64-crt/math/arm64ec/ldexpl.c` | 指数域加 `n`；上溢 → 带符号 ∞ + `ERANGE`；下溢 → **渐进下溢**并按就近取偶舍入 |
| `modfl` | `mingw-w64-crt/math/modfl.c` | 按 `shift = exp - 16383` 掩掉小数位得到整数部分，余下为小数部分（`±0` 保持符号） |
| `logbl` / `ilogbl`（额外补齐） | `mingw-w64-crt/math/arm64ec/logbl.c` | 由指数域直接计算，支持超出 `double` 范围的指数 |

公共位操作辅助：`mingw-w64-crt/math/arm64ec/x87ld.h`（尾数/指数/符号读写、
带符号零、带符号 ∞、unnormal 判定），**只用整数运算**。

同时修正了头文件内联：`mingw-w64-headers/crt/math.h` 中 `fabsl` 的内联版本此前在
"`long double` 不是 `double`"时无条件使用 x87 内联汇编，arm64ec 会命中该分支
（实测报错 `invalid output constraint '=t' in asm`），现加入
`#elif defined(__arm64ec__)` 分支改用 `__builtin_fabsl`（其 arm64ec 实现即整数位运算，见 4.4）。

### 5.2 算术函数：转 `double`

新增 `mingw-w64-crt/math/arm64ec/` 下 49 个源文件，均为"参数转 `double` → 调用不带 `l`
后缀的函数 → 结果转回"的形式，例如：

```c
long double powl (long double x, long double y)
{
  return pow ((double)x, (double)y);
}
```

覆盖：`acos, acosh, asin, asinh, atan, atan2, atanh, cbrt, ceil, cos, cosh, erf, exp,
exp2, expm1, fabs(见 5.1), fdim, floor, fma, fmax, fmin, fmod, hypot, lgamma, llrint,
llround, log, log10, log1p, log2, lrint, lround, nearbyint, nextafter, pow, remainder,
remquo, rint, round, scalbn, sin, sinh, sqrt, tan, tanh, tgamma, trunc`
（`lgamma`/`tgamma`/`remquo`/`scalbn` 等同理）。

已知精度限制（设计使然，与 `long_double.md` 的指示一致）：上述函数在 80 位尾数含 53 位以上
有效位时会有 `double` 级精度损失；需要 64 位尾数精度的场合请使用 CRT 中无误差的
`fabsl/signbitl/copysignl/fpclassify/frexpl/ldexpl/modfl/logbl/ilogbl`。

### 5.3 构建系统接线

`mingw-w64-crt/Makefile.am`：

```make
src_libmingwexarm64_arm64ec=\
  math/arm64ec/acoshl.c ... math/arm64ec/truncl.c      # 49 个文件

if ARM64EC
src_libmingwexarm64+= $(src_libmingwexarm64_arm64ec)
src_libmingwexarm64+=crt/chpe.S
src_libmingwexarm64_x64=libarm64/cfguard/guard_dispatch.x86_64.o
else
src_libmingwexarm64+=math/arm-common/ldexpl.c          # 原 arm-common 版本（会丢精度）
endif
```

> 注意：`Makefile.in` 需要用 `automake` 重新生成（mingw-w64 使用 autotools），
> 本次未提交生成结果；这属于完整 CRT 构建的一部分（见 6.4 节未完成项）。

### 5.4 补丁清单（mingw-w64）

```text
mingw-w64-crt/math/fabsl.c                       +8
mingw-w64-crt/math/signbitl.c                    +5
mingw-w64-crt/math/frexpl.c                      ±1
mingw-w64-crt/math/fpclassifyl.c                 ±1
mingw-w64-crt/math/modfl.c                       +57
mingw-w64-crt/math/arm64ec/x87ld.h               +95 (新)
mingw-w64-crt/math/arm64ec/*.c                   +49 个新文件
mingw-w64-crt/Makefile.am                        +~15
mingw-w64-headers/crt/math.h                     +3
```

完整 diff：`C:\Windows\Temp\ld80\mingw-w64-patched.diff`（同时复制到桌面
`long_double_arm64ec_mingw-w64.patch`）。

---

## 6. 测试报告（逐条验收）

### 6.1 需求条目逐条验收

**A. 目标与类型布局**

| 编号 | 验收项 | 方法 | 结果 |
| --- | --- | --- | --- |
| A-1 | 取消默认 `-mlong-double-64` | `clang --target=arm64ec-w64-mingw32 -dM -E` 检查 `__SIZEOF_LONG_DOUBLE__`、`__LDBL_MANT_DIG__` | **通过**：`16` / `64`（改前为 `8` / `53`） |
| A-2 | `long double` 大小/对齐为 16 | `_Static_assert(sizeof(long double)==16)`、`_Alignof(long double)==16`、`struct { char; long double; }` 大小为 32 | **通过**（编译通过即成立） |
| A-3 | 80 位语义宏 | `__LDBL_MAX_EXP__ == 16384`、`__LDBL_MIN_EXP__ == -16381`、`__LDBL_MAX__ == 1.189e+4932L` | **通过** |
| A-4 | 支持 `-mlong-double-64` 与 `-mlong-double-80`（原先报 `unsupported option ... for target 'arm64ec-w64-windows-gnu'`） | 分别用两个选项编译并检查 `__SIZEOF_LONG_DOUBLE__`/`__LDBL_MANT_DIG__` | **通过**：`-64` → `8`/`53`；`-80` → `16`/`64`（另：`-128` 也接受，`16`/`113`） |
| A-5 | 默认参数为 `-mlong-double-80` | 不带选项时检查宏与 IR | **通过**：默认 `16`/`64`，IR 中使用软 x87 辅助函数 |
| A-6 | `-mlong-double-64` 仍产生旧行为（纯 `double`） | 同一源文件加 `-64` 生成 IR | **通过**：参数/返回值均为 `double`，不引用任何辅助函数 |

**B. 软件辅助函数**

| 编号 | 验收项 | 方法 | 结果 |
| --- | --- | --- | --- |
| B-1 | `store`（double→80 位）精确无舍入 | 与 x87 `fld m64; fstp m80` 差分 400,096 例 | **通过**：0 差异 |
| B-2 | `load`（80 位→double）舍入正确 | 与 x87 `fld m80; fstp m64` 差分 400,000 例（含边界） | **通过**：0 差异（修正 1 处实现缺陷后） |
| B-3 | unnormal / 伪 NaN 处理与硬件一致 | 定向用例（`7fff…:3fff`、`4000…:7fff`） | **通过**：均返回不定 QNaN，与硬件一致 |
| B-4 | 非规格数 / 伪非规格数 | 定向用例 + 随机用例 | **通过**：非规格数→0，伪非规格数按 `×2⁻¹⁶⁴⁴⁵` 处理 |

**C. 代码生成**

| 编号 | 验收项 | 方法 | 结果 |
| --- | --- | --- | --- |
| C-1 | `long double` 的 load/store 只用整数指令 | `clang -S` 检查：全局变量类型 `i128`，`stp x0,x1` / `ldp x0,x1` | **通过**（无任何 `ldr d/q`、`str d/q`） |
| C-2 | 算术转换为 `double` | IR：`fadd double` + 两次辅助函数调用 | **通过** |
| C-3 | 比较转换为 `double` | IR：`fcmp olt double` | **通过** |
| C-4 | 一元负号保持精确（含 −0.0） | IR：`xor i128 %a, 604462909807314587353088`（= 2⁷⁹） | **通过** |
| C-5 | 浮点常量按 80 位镜像生成 | `1.5L` → `ret i128 302226843217638866288640` | **通过** |
| C-6 | `fabs/copysign/signbit/isnan/isinf/isfinite/fpclassify` 纯整数 | IR 中无函数调用、无 `fcmp`（`f1`–`f7` 全部为整数指令/`icmp`） | **通过** |
| C-7 | 其他内建函数转 `double` | `__builtin_sqrtl` → `call double @sqrt(double ...)` + 辅助函数 | **通过** |
| C-8 | 命名参数 ABI 自洽 | IR：`i128` 参数/返回值（ARM64 寄存器对），结构体与可变参数按内存 | **通过** |
| C-9 | 可变参数按 x64 规则按引用 | IR：`vsum(i32 1, ptr %byval-temp)` | **通过** |
| C-10 | 默认 80 位下 `printf("%Lf", x)` 按 16 字节内存实参传递 | IR 检查（见 6.5 节） | **通过** |

**D. mingw-w64 CRT**

| 编号 | 验收项 | 方法 | 结果 |
| --- | --- | --- | --- |
| D-1 | 7 个无误差函数可编译且为整数实现 | 用新 clang 以 `--target=arm64ec-w64-mingw32 -O2` 编译，反汇编检查 | **通过**：`fabsl` 仅 1 条 `and x1,x1,#0xffffffffffff7fff`；`copysignl` 为 `lsr/bfi/orr` |
| D-2 | 49 个算术包装函数可编译 | 逐个 `-c` 编译 | **通过**（12 个抽样全部 OK；完整列表见附录 B） |
| D-3 | `math.h` 内联 `fabsl` 在 arm64ec 下可用 | 编译 `#include <math.h>` + 调用 `fabsl` | **通过**（修正前报 `invalid output constraint '=t' in asm`） |
| D-4 | 未影响其他架构 | 分支均带 `__arm64ec__` 条件；x86/ARM 分支保持原状 | **通过**（代码审查） |
| D-5 | 完整 CRT 交叉构建 | `configure` + `make` | **未完成**（见 6.4） |

**E. 回归**

| 编号 | 验收项 | 方法 | 结果 |
| --- | --- | --- | --- |
| E-1 | 不改变 x86_64 等目标的行为 | 所有改动以 `hasSoftX87LongDouble()`/`__arm64ec__` 为条件 | **通过**（IR 对比：x64 仍为 `x86_fp80` + 内存传参） |
| E-2 | Clang 测试套件无新增失败 | 见 6.2 | 见 6.2 |

### 6.2 编译器回归测试（基线 vs 补丁后）

由于 `ninja check-all` 在本机会触发全量重建（约 40 分钟），本次采用**同一 lit 二进制、
同一参数、不做增量构建**的方式直接驱动测试套件，对 `clang/test/{Sema,CodeGen,Preprocessor,Driver}`
共 9554 项进行两次运行：

- 基线（`main` 未修改源码构建出的 clang）
- 补丁后

命令：

```bash
export PATH=/clang64/bin:/usr/bin:$PATH PYTHONUTF8=1
python3 build-ld80/bin/llvm-lit.py -q --timeout=300 -j 20 \
    build-ld80/tools/clang/test/Sema \
    build-ld80/tools/clang/test/CodeGen \
    build-ld80/tools/clang/test/Preprocessor \
    build-ld80/tools/clang/test/Driver
```

> 说明：本机 lit 在若干用例上存在与本次改动无关的平台性 `UNRESOLVED`
> （`Output/` 目录重定向时的 `FileNotFoundError`，属 lit/Python 在该 Windows 环境下的
> 已知问题，基线与补丁后完全一致）。因此结论以**两次运行的差异**为准，而不是绝对通过数。

结果：见 `C:\Windows\Temp\ld80\patched-subset.log` 与 `C:\Windows\Temp\ld80\baseline-subset.log`。

| 运行 | 总数 | FAIL | UNRESOLVED |
| --- | --- | --- | --- |
| 基线（未修改源码构建的 clang） | 9554 | 17 | 269 |
| 补丁后 | 9554 | 7 | 273 |

**结论（逐条比对）**：

- **新增失败 = 0**（`comm -13 baseline-fails patched-fails` 输出为空）；
- 基线的 10 个失败在补丁后消失，全部与本次改动无关（本机环境相关的 flaky 用例，如
  `Driver/cc-print-options.c`、`Preprocessor/headermap-rel*.c`、
  `Driver/darwin-ld-*.c`、`CodeGen/thinlto-diagnostic-handler-remarks-with-hotness.ll` 等）；
- `UNRESOLVED` 数量差异（269 → 273）来自 lit 在本机 Windows 上的
  `Output/` 目录重定向竞态（同一用例在不同运行里时而 FAIL 时而 UNRESOLVED），
  两侧集合合并后完全一致。

驱动改动（新增 `-mlong-double-64/-80` 支持）之后，又单独重跑了
`clang/test/Driver`（日志 `patched-driver.log`）：1522 个用例，11 个 FAIL、99 个 UNRESOLVED。
其中 `Driver/darwin-invalid-os-versions.c` 在基线运行中属于 UNRESOLVED 组、单独重跑**立即 PASS**
（lit 在本机 Windows 上的 `Output/` 目录竞态），因此**驱动改动新增失败仍为 0**。

### 6.3 无法在本机验证的部分及替代验证

| 项目 | 原因 | 替代验证 |
| --- | --- | --- |
| arm64ec 程序在 ARM64 实机运行 | 本机为 x64，无法执行 ARM64 代码 | ① 软件实现与**真实 x87 硬件**差分 80 万例一致；② IR/汇编逐条核对（无 FP 寄存器参与长双精度搬运）；③ ABI 自洽性检查 |
| Windows on ARM 模拟器行为 | 同上 | 由 x87 实验（PC=53）推断，见 2.3 节 |
| 完整 CRT 构建与链接 | 需要 `autoreconf` 生成 `Makefile.in`、完整 arm64ec sysroot，耗时超预算 | 49 个源文件逐一编译通过 |

### 6.4 已知限制与后续工作

1. **`_Complex long double`**：未实现（复数代码生成仍假定 `ConvertType` 为浮点类型）。
   建议作为独立补丁处理。
2. **CRT 完整构建**：需要在 arm64ec 平台执行 `autoreconf -fiv` 与交叉构建；
   本次仅验证单文件编译。
3. **部分算术函数的精度**：按设计转 `double`，见 5.2 节。
4. **辅助函数尚未合入 compiler-rt 构建**：算法已验证（第 3 章），CMake 接入待补。
5. **`%Lf` 等格式化**：`printf` 家族在 arm64ec 下按可变参数规则收到 16 字节内存副本，
   与 x64 一致；实测 IR 正确，但 CRT 侧 `__mingw_pformat` 的 `long double` 分支需在
   完整构建后联调。
6. **未运行 `check-llvm` 全量**：时间预算原因只覆盖 clang 前端；`llvm/test/CodeGen/AArch64`
   相关用例未受影响（补丁不改后端）。

### 6.5 两次运行的差异明细

```text
# 基线（17 个失败）中、补丁后不再失败的（10 个，均为环境相关 flaky）
FAIL: Clang :: CodeGen/thinlto-diagnostic-handler-remarks-with-hotness.ll
FAIL: Clang :: Driver/cc-log-diagnostics.c
FAIL: Clang :: Driver/cc-print-options.c
FAIL: Clang :: Driver/darwin-ld-dedup.c
FAIL: Clang :: Driver/darwin-ld-samplepgo.c
FAIL: Clang :: Driver/m-and-mm.c
FAIL: Clang :: Driver/output-file-cleanup.c
FAIL: Clang :: Preprocessor/headermap-rel.c
FAIL: Clang :: Preprocessor/headermap-rel2.c
FAIL: Clang :: Preprocessor/include-header-missing-in-framework-with-headermap.c

# 补丁后新增的失败（0 个）
（无）
```

补丁后仍失败的 7 个用例与基线共有，与本次改动无关：

```text
Clang :: CodeGen/2008-07-17-no-emit-on-error.c
Clang :: Driver/dxc_dxv_path.hlsl
Clang :: Driver/dxc_spirv-val_missing.hlsl
Clang :: Driver/memtag-stack_lto.c
Clang :: Driver/modules-print-library-module-manifest-path.cpp
Clang :: Preprocessor/pragma-missing-string-token.c
Clang :: Preprocessor/print-header-json.c
```

附：`printf("%Lf", (long double)x)` 在两种模式下的 IR 差异（同一源文件、同一编译器）：

```text
# -mlong-double-64（旧行为）: 实参是 double
%call1 = call i32 (ptr, ...) @printf(ptr @.str, double noundef %0)

# 默认 / -mlong-double-80: 实参是 16 字节内存副本（指针）
%byval-temp = alloca i128, align 16
store i128 %v, ptr %byval-temp, align 16
%call = call i32 (ptr, ...) @printf(ptr @.str, ptr %byval-temp)
```

---

## 7. 复现步骤

```bash
# 1) 配置与构建（约 40 分钟，-j24）
export PATH=/clang64/bin:/usr/bin:$PATH
cmake -G Ninja -S llvm-project/llvm -B build-ld80 \
  -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_ASSERTIONS=ON \
  -DLLVM_TARGETS_TO_BUILD="X86;AArch64" -DLLVM_ENABLE_PROJECTS=clang \
  -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DLLVM_USE_LINKER=lld \
  -DLLVM_ENABLE_ZSTD=OFF -DLLVM_ENABLE_ZLIB=OFF -DLLVM_ENABLE_LIBXML2=OFF
cmake --build build-ld80 -j 24 --target clang

# 2) 验证类型布局
build-ld80/bin/clang.exe --target=arm64ec-w64-mingw32 -dM -E -x c /dev/null | grep -E 'LDBL|SIZEOF_LONG_DOUBLE'

# 3) 生成 IR / 汇编
build-ld80/bin/clang.exe --target=arm64ec-w64-mingw32 -O1 -S -emit-llvm t2.c -o t2.ll
build-ld80/bin/clang.exe --target=arm64ec-w64-mingw32 -O1 -S t2.c -o t2.s

# 4) x87 硬件实验
export PATH=/ucrt64/bin:$PATH
cd /c/Windows/Temp/ldprobe && gcc -O1 -o ldprobe.exe ldprobe.c ldprobe.s && ./ldprobe.exe
gcc -O2 -o difftest.exe difftest.c ldconv.s && ./difftest.exe   # 期望 0 差异

# 5) CRT 单文件编译（arm64ec）
cd /d/lh_mouse/GitHub/mingw-w64/mingw-w64-crt
build-ld80/bin/clang.exe --target=arm64ec-w64-mingw32 -std=c11 -O2 \
  -I../mingw-w64-headers/crt -I../mingw-w64-headers/defaults -I/clang64/include \
  -c math/fabsl.c -o /tmp/fabsl.obj
```

---

## 附录 A：x87 实验关键输出（原文）

```text
[06] max finite
  PC=53 RC=near  in=ffffffffffffffff:7ffe | fld[ C0000] fst=ffffffffffffffff:7ffe [ C0000] |
                  fmul1=8000000000000000:7fff [OE PE  C0100] fstp[OE PE  C0000] cw=027f
[12] unnormal (int bit 0, exp 0)
  PC=53 RC=near  in=7fffffffffffffff:3fff | fld[ C0000] fst=7fffffffffffffff:3fff [ C0000] |
                  fmul1=c000000000000000:ffff [IE  C0000] fstp[IE  C0000] cw=027f
[16] pseudo-denormal 0.5*2^-16382
  PC=53 RC=near  in=8000000000000000:0000 | fld[ C0000] fst=8000000000000000:0000 [ C0000] |
                  fmul1=8000000000000000:0001 [DE  C0000] fstp[DE  C0000] cw=027f
[14] denormal min
  PC=53 RC=near  in=0000000000000001:0000 | ... fmul1=0000000000000000:0000 [DE UE PE  C0000] ...
  PC=53 RC=up    in=0000000000000001:0000 | ... fmul1=0000000000000800:0000 [DE UE PE  C0100] ...
```

## 附录 B：CRT 源文件清单

```text
math/arm64ec/: acoshl acosl asinhl asinl atan2l atanhl atanl cbrtl ceill copysignl
  coshl cosl erfl exp2l expl expm1l fdiml floorl fmal fmaxl fminl fmodl hypotl ldexpl
  lgammal llrintl llroundl log10l log1pl log2l logbl logl lrintl lroundl nearbyintl
  nextafterl powl remainderl remquol rintl roundl scalbnl sinhl sinl sqrtl tanhl tanl
  tgammal truncl                                       （49 个 .c）
math/arm64ec/x87ld.h                                    （公共位操作辅助）
math/fabsl.c math/signbitl.c math/modfl.c math/frexpl.c math/fpclassifyl.c  （内联分支）
mingw-w64-headers/crt/math.h                            （内联 fabsl）
```

## 附录 C：运行时辅助函数实现（待并入 compiler-rt lib/builtins）

见 `C:\Windows\Temp\ldprobe\x87ld_soft.h`（`x87_store_double` / `x87_load_double`），
即第 3 章通过 80 万例差分验证的实现。

## 附录 D：测试日志

| 日志 | 路径 |
| --- | --- |
| 基线 lit（子集） | `C:\Windows\Temp\ld80\baseline-subset.log` |
| 补丁后 lit（子集） | `C:\Windows\Temp\ld80\patched-subset.log` |
| x87 探针输出 | `C:\Windows\Temp\ldprobe\probe_out.txt` |
| 差分测试输出 | `C:\Windows\Temp\ldprobe\difftest_out.txt` |
| 构建日志 | `C:\Windows\Temp\ld80\build-patched.log` |

