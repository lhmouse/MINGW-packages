# 任务概述

目前 Clang 对于 arm64ec-w64-mingw32 目标默认启用 `-mlong-double-64` 参数。该行为与 x86_64-w64-mingw32
不一致，造成 ABI 不兼容。

# 可行性分析

## 数据格式

标准的 IEEE-754 双精度浮点数格式为：

```c++
struct alignas(8) ieee754_double
  {
    uint64_t mantissa_without_leading_one : 52;
    uint64_t shifted_exponent : 11;
    uint64_t sign : 1;
  };
```

x87 扩展精度格式为：

```c++
struct alignas(16) x87_long_double
  {
    uint64_t mantissa;
    uint16_t shifted_exponent : 15
    uint16_t sign : 1;
    uint8_t padding[6];
  };
```

硬件并不读取或写入 `padding` 字段。

注意 x87 扩展精度不使用 hidden bit，因此尾数为 64 位。对于规格数，尾数的最高位必须为 1；对于欠规格数
（denormalized number），尾数的最高位必须为 0。不符合此条件的数为非法输入，视为 NaN。

## 调用约定

在 {x86_64,arm64ec}-w64-mingw32 上，存在两种调用约定：

1. 在 Microsoft ABI 中，`long double` 的调用约定与上述 `struct` 相同，即参数和返回值都通过内存传递。
2. 在 System V ABI 中，`long double` 参数通过内存传入，而返回值通过 x87 寄存器 `st(0)` 返回。由于
   ARM64 没有 80 位寄存器，它是通过将之拆成两部分实现的：高 16 位存储于 `x16` 的低 16 位，而低 64 位
   存储于 `lr` 中；因此处理函数返回变得复杂，可能需要将 `lr` 暂时移入其它寄存器以免破坏。

不过，Clang 对于 x86_64-w64-mingw32 上 `attribute((sysv_abi))` 函数返回 `long double` 的处理似乎
不正确。此实现无参考意义，应以 GCC 为准。

## 实现细节

上述关于寄存器拆分的描述适用于所有的 x87 寄存器，见文档：
https://learn.microsoft.com/zh-cn/cpp/build/arm64ec-windows-abi-conventions?view=msvc-170

在 Windows 11 on ARM 上运行 x86-64 程序时，模拟器默认为性能而优化。在此情况下，模拟器将 x87 寄存器
转换为双精度格式，并使用 ARM64 硬件进行计算，再将结果转回（静默截断）。此行为符合 x87 的 53 位精度
模式，并且与 MSVCRT 兼容，但是与 mingw-w64 的 64 位精度要求不一致。

## 待确认项

需要在本机 Intel i9-14900K 上编写程序分析硬件行为。按以下流程测试并整理报告（需要编写汇编时，应当使用
Intel 语法）：

1. 使用 `fldcw` 设置 x87 精度为 53 位、舍入模式为 nearest。
2. 使用 `fld` 加载一个 `long double`，要求尾数包含 64 位有效数字。
3. 使用 `fld st(0); fstp` 将之存入内存，检查是否有精度损失。
4. 使用 `fld1; fmulp st(1), st` 将该数乘以 1，这将导致有效数字被截断到 53 位。
5. 使用 `fstp` 将之存入内存并清空 x87 栈，检查是否有精度损失。

待确认点：

1. 尾数截断时向什么方向舍入？
2. 如果截断时发生上溢或下溢，如何处理？
3. 尾数显式最高位为 0 的规格数如何处理？
4. 欠规格数如何处理？
5. 尾数显式最高位为 1 的欠规格数如何处理？

# 修改 LLVM 源代码

在修改 LLVM 代码以前，应当首先以 arm64ec-w64-mingw32 为默认目标构建上游 `main` 分支，并运行全部测试。
将测试日志保存供后续查阅。

1. 应当支持 `-mlong-double-64` 和 `-mlong-double-80`。目前会导致错误：
   ```text
   clang: error: unsupported option '-mlong-double-64' for target 'arm64ec-w64-windows-gnu'
   clang: error: unsupported option '-mlong-double-80' for target 'arm64ec-w64-windows-gnu'
   ```
2. `-mlong-double-128` 应当选择四精度浮点，用软件实现。
3. 默认参数应当为 `-mlong-double-80`。
4. 需设定 `long double` 类型大小为 16 字节、对齐为 16 字节。
5. 增加两个辅助函数。具体需要在哪里增加？注意到 x86-32 上 `long long` 长除法也是软件实现，应该在同一个
   库中增加。这两个函数用于实现 `long double` 与 `double` 互转，转换规则见[待确认项](#待确认项)，必须
   用纯软件实现：
   ```c++
   extern "C" double __arm64ec_load_x87_long_double(const long double& mem);
   extern "C" void __arm64ec_store_x87_long_double(long double& mem, double value);
   ```
6. 编译器进行 load/store 操作必须完全用整数指令拼字，避免破坏数据。算术操作则将操作数转换为 `double`
   进行计算，得到结果后转回。

修改完成后，应当使用相同参数再次构建并运行测试，并比较两次测试的差异。如果有新增的测试失败，需要加以分析
并修复。例如某些测试依赖了旧的布局，则需要修改测试源代码以符合新布局。

# 修改 mingw-w64 源代码

使用新编译的 Clang 编译 mingw-w64 CRT。检查预定义宏 `__arm64ec__`，源代码中某些使用内联汇编的 x87
操作需要修改为软件实现：

1. `fabsl()`, `signbitl()`, `copysignl()`, `fpclassify()`, `frexp()`, `ldexp()`, `modf()` 必须
   为无误差，因此只能使用整数指令实现。
2. 算术操作改为将参数转换为 `double` 后调用不带 `l` 后缀的函数。例如：
   ```c
   long double
   powl(long double base, long double exponent)
     {
       // ... ...
   #if defined(__arm64ec__)
       return pow((double) base, (double) exponent);
   #else
       // ... ...
     }
   ```
