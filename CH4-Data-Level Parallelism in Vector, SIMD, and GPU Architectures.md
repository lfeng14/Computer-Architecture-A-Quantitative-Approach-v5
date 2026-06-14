## Assistant

下面按《Computer Architecture: A Quantitative Approach》第四章 **Data-Level Parallelism in Vector, SIMD, and GPU Architectures** 梳理。核心主题是：**如何利用数据级并行 DLP，让一条指令或一组线程同时处理大量相似数据**。本章覆盖三类主要形态：**向量体系结构 Vector、媒体 SIMD 扩展、GPU/SIMT**。fileciteturn1file0

---

# 第四章：数据级并行 DLP 总览

## 1. 什么是数据级并行 DLP？

**数据级并行 Data-Level Parallelism，DLP** 指的是：  
程序中存在大量相互独立、可执行同一操作的数据元素。

典型例子：

```c id="2xf0ag"
for (i = 0; i < n; i++)
    Y[i] = a * X[i] + Y[i];
```

每个 `i` 的计算互不依赖，所以可以并行。

它和线程级并行 TLP 不同：

| 类型 | 并行来源 | 典型形式 |
|---|---|---|
| ILP | 单线程内部多条指令并行 | 流水线、乱序执行、超标量 |
| DLP | 多个数据元素做同一操作 | Vector、SIMD、GPU |
| TLP | 多个线程/任务并行 | 多核、多线程 |
| MIMD | 多个指令流、多个数据流 | 多核 CPU |
| SIMD | 单指令、多数据 | Vector、SSE/AVX、GPU 的部分执行模型 |

本章重点是 **SIMD 风格的 DLP**。

---

# 2. 为什么 DLP/SIMD 重要？

本章给出的核心判断是：  
对于有大量数据并行的程序，SIMD/DLP 往往比 MIMD 更节能、更高效。

原因：

1. **一条指令控制多个数据操作**  
   取指、译码、调度的开销被多个数据元素分摊。

2. **程序员可以保持近似顺序思维**  
   代码看起来仍然像普通循环，但底层并行执行。

3. **能耗效率高**  
   MIMD 需要为每个线程/核心维护独立指令流；SIMD 只需要一个控制流。

4. **适合科学计算、图像、音频、视频、机器学习等场景**  
   这些领域大量操作都是数组、矩阵、向量上的重复计算。

---

# 3. 本章三种 SIMD/DLP 形态

| 类型 | 代表 | 特点 |
|---|---|---|
| Vector Architecture | Cray、VMIPS、RISC-V Vector 思想 | 指令操作长向量，编译器友好 |
| SIMD ISA Extension | MMX、SSE、AVX、NEON | 短向量，固定宽度，常见于 CPU |
| GPU Architecture | CUDA、OpenCL、SIMT | 大量线程，吞吐优先，隐藏延迟 |

这三种都利用 DLP，但抽象层次不同。

---

# 4. 向量体系结构 Vector Architecture

## 4.1 向量机的基本思想

向量处理器一次处理一组数据：

```text id="yrir5s"
load vector X
load vector Y
vector multiply
vector add
store vector Y
```

不是每个元素执行一次标量指令，而是一条向量指令处理多个元素。

例如：

```c id="2pnlk1"
Y = a * X + Y
```

标量 MIPS 需要对每个元素循环执行 load、mul、load、add、store、分支等。  
而 VMIPS 向量代码只需要：

```asm id="obf9hy"
L.D      F0,a
LV       V1,Rx
MULVS.D  V2,V1,F0
LV       V3,Ry
ADDVV.D  V4,V2,V3
SV       V4,Ry
```

差别很大：  
标量代码执行大量循环控制指令；向量代码用少量指令完成整段向量运算。

---

## 4.2 VMIPS 的组成

VMIPS 是书中用于讲解的假想向量机，类似 MIPS + 向量扩展。

主要部件：

| 部件 | 作用 |
|---|---|
| Vector Registers | 存放向量，每个寄存器含多个元素 |
| Vector Functional Units | 向量加法、乘法、除法等流水线单元 |
| Vector Load/Store Unit | 向量加载和存储 |
| Scalar Registers | 标量寄存器，提供地址或标量操作数 |
| VLR | Vector Length Register，控制向量长度 |
| VM | Vector Mask Register，控制条件执行 |

VMIPS 中常见指令：

| 指令 | 含义 |
|---|---|
| `ADDVV.D` | 向量 + 向量 |
| `ADDVS.D` | 向量 + 标量 |
| `MULVV.D` | 向量 × 向量 |
| `MULVS.D` | 向量 × 标量 |
| `LV` | 连续向量 load |
| `SV` | 连续向量 store |
| `LVWS` | 带 stride 的向量 load |
| `SVWS` | 带 stride 的向量 store |
| `LVI` | indexed load / gather |
| `SVI` | indexed store / scatter |
| `CVM` | 设置 mask 全为 1 |
| `POP` | 统计 mask 中 1 的数量 |

---

# 5. 向量执行时间模型

本章讲了三个非常重要的概念：

## 5.1 Convoy

**Convoy**：可以同时开始执行的一组向量指令。

要求：

1. 没有结构冒险；
2. 可以通过 chaining 处理数据依赖；
3. 通常被当作一个执行批次。

例如：

```asm id="jbnm3f"
LV       V1,Rx
MULVS.D  V2,V1,F0
LV       V3,Ry
ADDVV.D  V4,V2,V3
SV       V4,Ry
```

可以划分为：

```text id="6hknos"
Convoy 1: LV + MULVS.D
Convoy 2: LV + ADDVV.D
Convoy 3: SV
```

所以一共 3 个 convoy。

---

## 5.2 Chime

**Chime**：执行一个 convoy 所需的时间单位。

如果一个向量长度为 `n`，共有 `m` 个 convoy，那么大致执行时间：

```text id="blcb0e"
Time ≈ m × n cycles
```

前提是单 lane，每周期处理一个元素。

例如 DAXPY：

```text id="2ha5zt"
3 convoys
2 FLOPs per element
```

所以：

```text id="30w7j8"
cycles per FLOP = 3 / 2 = 1.5
```

注意：这是理想估算，忽略了启动开销。

---

## 5.3 Chaining

**Chaining** 类似流水线 forwarding。

假设：

```asm id="f58qpn"
LV       V1,Rx
MULVS.D  V2,V1,F0
```

`MULVS.D` 不必等整个 `LV` 完成，只要第一个元素 load 出来，就可以开始处理第一个元素。

所以向量机的数据依赖不是“一整条指令完成后再执行下一条”，而是可以按元素流水化。

这就是向量机快的关键之一。

---

# 6. 向量启动开销 Start-up Overhead

向量功能单元是流水线的。  
第一批结果出来前需要等待若干周期，这叫 **start-up time**。

书中 VMIPS 假设：

| 操作 | 启动开销 |
|---|---|
| FP add | 6 cycles |
| FP multiply | 7 cycles |
| FP divide | 20 cycles |
| Vector load | 12 cycles |

重点：

> 向量越长，启动开销越容易被摊薄；向量太短，向量化不一定划算。

这也是一个常见坑：  
**只看峰值性能，不看启动开销，会高估向量机性能。**

---

# 7. 多 Lane：提高向量吞吐

## 7.1 Lane 是什么？

一个 lane 可以理解为一条并行的数据通路。

如果只有 1 个 lane：

```text id="jmc554"
每周期处理 1 个元素
64 个元素需要 64 cycles
```

如果有 4 个 lane：

```text id="2l2aw6"
每周期处理 4 个元素
64 个元素需要 16 cycles
```

所以：

```text id="a44nkz"
执行时间 ≈ 向量长度 / lane 数
```

## 7.2 Lane 的优势

增加 lane：

1. 提高吞吐；
2. 控制复杂度增加不大；
3. 不需要改变指令集；
4. 可以通过硬件扩展提升性能。

## 7.3 Lane 的限制

多 lane 有用的前提是：

1. 程序有足够长的向量；
2. 内存带宽能跟上；
3. 指令供给不能成为瓶颈；
4. 数据排列适合并行访问。

---

# 8. Vector Length Register 与 Strip Mining

## 8.1 问题：程序中的向量长度不一定等于硬件最大长度

比如 VMIPS 每个向量寄存器最多 64 个 double。  
但程序可能是：

```c id="epzgcp"
for (i = 0; i < n; i++)
    Y[i] = a * X[i] + Y[i];
```

这里 `n` 可能运行时才知道，也可能大于 64。

## 8.2 VLR：向量长度寄存器

**VLR** 用来告诉硬件本次向量指令处理多少个元素。

例如：

```text id="7qsq2v"
VLR = 10
```

则向量指令只处理 10 个元素。

## 8.3 MVL：最大向量长度

**MVL = Maximum Vector Length**

例如 VMIPS：

```text id="lf8c32"
MVL = 64
```

VLR 不能超过 MVL。

---

## 8.4 Strip Mining

当 `n > MVL` 时，需要把大循环切成多个向量块。

例如 `n = 200`，`MVL = 64`：

```text id="56t39m"
200 = 8 + 64 + 64 + 64
```

先处理余数 8 个，再处理 3 个完整向量块。

伪代码：

```c id="swgqtf"
low = 0;
VL = n % MVL;

for (j = 0; j <= n / MVL; j++) {
    for (i = low; i < low + VL; i++)
        Y[i] = a * X[i] + Y[i];

    low = low + VL;
    VL = MVL;
}
```

重点：

> Strip mining 是编译器把任意长度循环转换成硬件可处理向量长度的技术。

---

# 9. Vector Mask：处理循环中的 if

## 9.1 问题

有条件判断的循环不容易向量化：

```c id="fdmvob"
for (i = 0; i < 64; i++)
    if (X[i] != 0)
        X[i] = X[i] - Y[i];
```

因为不是每个元素都执行减法。

## 9.2 Vector Mask Register

向量机使用 **mask register** 控制哪些元素参与计算。

例如：

```text id="90c6d7"
VM[i] = 1 表示第 i 个元素执行
VM[i] = 0 表示第 i 个元素不执行
```

VMIPS 代码大致是：

```asm id="8fx2p7"
LV       V1,Rx
LV       V2,Ry
L.D      F0,#0
SNEVS.D  V1,F0
SUBVV.D  V1,V1,V2
SV       V1,Rx
```

其中：

```asm id="5gb2m2"
SNEVS.D V1,F0
```

表示比较 `V1[i] != 0`，结果写入 mask。

## 9.3 Mask 的代价

即使 mask 中某些元素为 0，硬件通常仍然要经过执行流程。  
所以：

```text id="e3y9oh"
mask 不会免费
```

如果大量元素被 mask 掉，实际 GFLOPS 会下降。

但是它避免了分支，通常仍然比标量执行好。

---

# 10. 内存带宽与 Memory Banks

## 10.1 向量机非常依赖内存带宽

向量处理器计算能力强，如果内存供不上数据，就会饿死。

典型向量 load/store 希望：

```text id="6mxp92"
启动后每周期提供一个或多个元素
```

为了做到这一点，需要多个 memory banks。

## 10.2 为什么需要多个 memory banks？

因为：

1. 向量 load/store 会连续访问大量数据；
2. 多个向量单元可能同时访问内存；
3. 非连续访问需要独立 bank 地址控制；
4. 多处理器共享内存时访问压力更大。

---

## 10.3 Bank Conflict

如果多个访问落到同一个 memory bank，就会冲突。

书中给出判断思路：

```text id="hdoaqr"
如果 stride 和 bank 数不匹配，可能频繁访问同一个 bank
```

最糟糕情况：

```text id="b8sfi5"
stride 是 memory bank 数量的倍数
```

例如：

```text id="4nbw79"
8 个 memory banks
stride = 32
```

每次访问都可能落到同一个 bank，严重降低带宽。

---

# 11. Stride：处理多维数组

## 11.1 什么是 stride？

**Stride** 是向量相邻元素在内存中的距离。

例如二维矩阵：

```c id="ilglll"
A[i][j] = A[i][j] + B[i][k] * D[k][j];
```

在 C 语言中，数组按 **row-major** 存储：

```text id="qzl3ir"
同一行连续
同一列不连续
```

访问 `B[i][k]` 可能是 stride = 1。  
访问 `D[k][j]` 可能是 stride = 行长度。

## 11.2 向量机支持 strided load/store

VMIPS 有：

```asm id="ak1egt"
LVWS
SVWS
```

含义：

```text id="ocrkrl"
base + i × stride
```

这样即使数据不连续，也可以加载成向量寄存器中的连续逻辑向量。

## 11.3 Stride 的意义

这是向量体系结构比普通短 SIMD 更强的地方之一：

| 访问类型 | 向量机 | 普通 SIMD |
|---|---|---|
| 连续访问 | 支持 | 支持 |
| stride 访问 | 支持较好 | 通常较弱 |
| gather/scatter | 支持 | 传统 SIMD 较弱 |

---

# 12. Gather-Scatter：处理稀疏矩阵

## 12.1 稀疏矩阵问题

稀疏矩阵中，大部分元素是 0。  
为了节省空间，只存非零元素及其索引。

例如：

```c id="cy6q29"
for (i = 0; i < n; i++)
    A[K[i]] = A[K[i]] + C[M[i]];
```

这里访问地址由索引数组决定，不是连续，也不是固定 stride。

## 12.2 Gather

**Gather**：根据索引向量，从多个地址取数据，压缩成一个密集向量。

```asm id="jpjxgp"
LVI Va, (Ra + Vk)
```

含义：

```text id="ho9gug"
Va[i] = Memory[Ra + Vk[i]]
```

## 12.3 Scatter

**Scatter**：把向量结果根据索引写回多个地址。

```asm id="krwgmj"
SVI (Ra + Vk), Va
```

含义：

```text id="23ent5"
Memory[Ra + Vk[i]] = Va[i]
```

## 12.4 Gather-Scatter 的代价

Gather/scatter 通常比连续 load/store 慢，因为：

1. 每个元素地址不同；
2. bank conflict 难以预测；
3. 缓存局部性差；
4. 无法简单合并成连续访问。

但它极大扩展了可向量化程序范围，尤其是稀疏矩阵。

---

# 13. 向量编程与编译器

向量体系结构的一个优点是：

> 编译器比较容易判断某个循环能否向量化。

不能向量化的常见原因：

1. 循环迭代之间有真实数据依赖；
2. 指针别名 aliasing；
3. 分支复杂；
4. 间接访存复杂；
5. 函数调用无法分析；
6. 内存访问模式不规则。

编译器可能给出提示，比如：

```text id="krxsn2"
loop not vectorized because of dependence
loop not vectorized because memory access is not consecutive
```

程序员可以通过重写代码、加编译器 hint、restrict、pragma 等方式帮助向量化。

---

# 14. Multimedia SIMD 指令扩展

## 14.1 SIMD Extension 是什么？

CPU 中常见的 SIMD 扩展：

| 架构 | SIMD 扩展 |
|---|---|
| x86 | MMX、SSE、AVX、AVX2、AVX-512 |
| ARM | NEON、SVE |
| Power | AltiVec/VSX |

本章主要讲 x86 的 MMX、SSE、AVX。

---

## 14.2 SIMD 扩展的基本思想

例如一个 256-bit 寄存器，可以看成：

| 数据类型 | 并行元素数 |
|---|---|
| 8-bit | 32 个 |
| 16-bit | 16 个 |
| 32-bit | 8 个 |
| 64-bit | 4 个 |

一条指令同时对多个窄数据执行操作。

例如 AVX 256-bit double：

```text id="o8crdw"
4 个 double 同时加法
4 个 double 同时乘法
```

---

## 14.3 SIMD 扩展与 Vector Architecture 的区别

这是本章重点。

| 对比项 | Vector Architecture | Multimedia SIMD |
|---|---|---|
| 向量长度 | 可变，VLR 控制 | 固定在 opcode/寄存器宽度中 |
| 最大长度 | MVL 抽象 | 宽度变化通常要加新 ISA |
| 编译器友好性 | 强 | 较弱 |
| stride 访问 | 原生支持 | 传统 SIMD 较弱 |
| gather/scatter | 原生支持 | 传统 SIMD 较弱，后续扩展才增强 |
| mask | 架构级支持 | 早期较弱，AVX-512 才更系统 |
| 主要用途 | 科学计算、长向量 | 多媒体、短向量、库优化 |
| 寄存器 | 大向量寄存器 | 短 SIMD 寄存器 |

一句话：

> Vector 更像“完整的向量计算模型”；SIMD 扩展更像“在标量 CPU 上加宽 ALU”。

---

## 14.4 为什么 SIMD 扩展仍然流行？

虽然不如向量体系结构优雅，但 SIMD 扩展流行，因为：

1. 增加硬件成本低；
2. 容易集成进已有 CPU；
3. 上下文切换状态较少；
4. 对多媒体、图像、音频很有效；
5. 可以逐步扩展；
6. 程序员可以通过库、intrinsics 使用；
7. 不需要完整向量机那种高内存带宽系统。

---

# 15. Roofline 性能模型

## 15.1 Roofline 是干什么的？

Roofline 用来判断程序性能受什么限制：

1. 受计算能力限制；
2. 受内存带宽限制。

它把性能上限画成一条“屋顶线”。

---

## 15.2 Arithmetic Intensity

**Arithmetic Intensity，算术强度**：

```text id="e8j4ug"
Arithmetic Intensity = FLOPs / Memory Bytes
```

也就是：

```text id="aealo3"
每访问 1 byte 内存，做多少次浮点运算
```

例如：

| 程序类型 | 算术强度 |
|---|---|
| 向量加法 | 低 |
| DAXPY | 较低 |
| 稀疏矩阵 | 低 |
| FFT | 中等 |
| Dense Matrix Multiply | 高 |

---

## 15.3 Roofline 公式

```text id="hphzam"
Attainable Performance =
min(Peak FLOP/s, Peak Memory Bandwidth × Arithmetic Intensity)
```

含义：

如果：

```text id="9aque3"
Bandwidth × Arithmetic Intensity < Peak FLOP/s
```

说明受内存带宽限制。

如果：

```text id="n5wprd"
Bandwidth × Arithmetic Intensity >= Peak FLOP/s
```

说明受计算峰值限制。

---

## 15.4 Ridge Point

**Ridge Point** 是屋顶线拐点。

```text id="m8bltl"
Ridge Point = Peak FLOP/s / Peak Memory Bandwidth
```

它表示要达到峰值计算性能，程序至少需要多少算术强度。

Ridge point 越低，说明机器越容易跑满计算单元。  
Ridge point 越高，说明只有高算术强度程序才能跑满。

---

# 16. GPU 架构

## 16.1 GPU 为什么适合 DLP？

GPU 最初为图形渲染设计，天然处理大量像素、顶点、纹理。  
这些工作都有共同特点：

```text id="x8074u"
大量数据
相同或相似操作
高吞吐需求
单个任务延迟不敏感
```

所以 GPU 发展成高吞吐数据并行处理器。

---

## 16.2 GPU 是异构计算

GPU 通常不是单独工作，而是和 CPU 配合：

| 角色 | 作用 |
|---|---|
| CPU / Host | 控制程序、启动 kernel、管理任务 |
| GPU / Device | 执行大规模并行计算 |
| System Memory | CPU 主存 |
| GPU Memory | GPU 显存 |

问题是：

```text id="1xo59y"
CPU 和 GPU 之间的数据传输有成本
```

所以 GPU 加速不仅要看 kernel 快不快，也要看数据搬运是否划算。

---

## 16.3 CUDA 编程模型

CUDA 把 GPU 程序组织为大量线程。

常见术语：

| CUDA 概念 | 含义 |
|---|---|
| Thread | 最小执行单位 |
| Thread Block | 一组线程，可共享 shared memory |
| Grid | 多个 thread block |
| Kernel | 在 GPU 上执行的函数 |
| Host | CPU |
| Device | GPU |
| `blockIdx` | block 编号 |
| `threadIdx` | block 内线程编号 |
| `blockDim` | 每个 block 的线程数 |

CUDA 调用形式：

```c id="tu3839"
kernel<<<dimGrid, dimBlock>>>(...);
```

例如 DAXPY：

```c id="wx1bnx"
__global__ void daxpy(int n, double a, double *x, double *y) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n)
        y[i] = a * x[i] + y[i];
}
```

---

# 17. SIMT：Single Instruction, Multiple Thread

## 17.1 SIMT 是什么？

NVIDIA 把 GPU 编程模型称为 **SIMT**：

```text id="d3ehq3"
Single Instruction, Multiple Thread
```

它看起来像很多独立线程，但硬件会把一组线程合在一起执行同一条指令。

## 17.2 Warp

在 NVIDIA GPU 中，通常：

```text id="99vacb"
1 warp = 32 threads
```

同一个 warp 的线程通常执行同一条指令，只是操作不同数据。

## 17.3 SIMT 和 SIMD 的关系

| 对比 | SIMD | SIMT |
|---|---|---|
| 程序员视角 | 明确写向量指令 | 写标量线程 |
| 硬件视角 | 多 lane 执行同一指令 | warp 内线程 lockstep 执行 |
| 分支处理 | mask/predicate | warp divergence + mask |
| 编译模型 | 向量化 | 线程映射 |

一句话：

> SIMT 是把 SIMD 的硬件执行包装成线程编程模型。

---

# 18. GPU 的硬件结构

本章中的 GPU 结构可以理解为：

```text id="g7elm7"
GPU
 ├── 多个 multithreaded SIMD processors
 │    ├── SIMD lanes
 │    ├── warp scheduler
 │    ├── registers
 │    ├── local/shared memory
 │    └── load/store units
 └── GPU memory
```

不同术语对应关系：

| NVIDIA CUDA | 体系结构理解 |
|---|---|
| Thread | 一个数据元素的逻辑执行 |
| Warp | 一组 lockstep 执行的线程 |
| Thread Block | 可被调度到一个 SIMD processor 的线程组 |
| SIMD Processor / SM | 执行多个 warp 的硬件核心 |
| Grid | 整个并行任务 |

---

# 19. GPU 如何隐藏延迟？

CPU 通常依赖：

1. 大 cache；
2. 分支预测；
3. 乱序执行；
4. speculative execution。

GPU 更依赖：

1. 大量线程；
2. 快速线程切换；
3. warp scheduler；
4. 高内存带宽。

当一个 warp 等待内存时，GPU 切换到另一个 ready warp 执行。

这叫：

```text id="kvjnwi"
latency hiding by multithreading
```

所以 GPU 不追求单线程低延迟，而是追求整体吞吐。

---

# 20. GPU 内存层次

GPU 常见内存空间：

| 内存类型 | 特点 |
|---|---|
| Global Memory | 大、慢、所有线程可访问 |
| Shared Memory / Local Memory | block 内共享，快 |
| Registers | 每线程私有，最快 |
| Constant Memory | 只读，适合广播 |
| Texture Memory | 图形历史遗留，适合特定访问模式 |

性能优化重点：

1. 尽量合并 global memory 访问；
2. 使用 shared memory 提高复用；
3. 避免 bank conflict；
4. 避免 register pressure 过高；
5. 提高 occupancy，但不是越高越好。

---

# 21. Coalescing：GPU 连续访存合并

GPU 中每个线程看起来都在做自己的 load/store。  
但如果一个 warp 的线程访问连续地址，硬件可以合并成少量内存事务。

例如：

```text id="hnz65c"
thread 0 -> A[0]
thread 1 -> A[1]
thread 2 -> A[2]
...
thread 31 -> A[31]
```

这是好模式。

坏模式：

```text id="23ubji"
thread 0 -> A[random0]
thread 1 -> A[random1]
...
```

这会导致大量分散访问，性能很差。

---

# 22. GPU 的分支发散 Warp Divergence

## 22.1 问题

如果同一个 warp 内线程走不同分支：

```c id="r0dshu"
if (x[i] > 0)
    y[i] = a;
else
    y[i] = b;
```

有些线程走 then，有些走 else。

## 22.2 GPU 的处理方式

GPU 通常会：

1. 先执行 then 分支，mask 掉不走 then 的线程；
2. 再执行 else 分支，mask 掉不走 else 的线程；
3. 最后合流。

所以分支发散会降低并行效率。

## 22.3 优化原则

尽量让同一 warp 内线程走相同路径。

例如：

```text id="nzh9fv"
相邻 thread 处理相似数据
减少数据相关分支
把分支改写成 predicate / select
```

---

# 23. PTX：GPU 的中间指令集

PTX 是 NVIDIA 的虚拟 ISA / 中间表示。

它的好处：

1. 编译器面向 PTX；
2. 硬件可以在不同代际变化；
3. PTX 到真实硬件指令由驱动/JIT 完成；
4. 提高架构演进灵活性。

这点和 CPU ISA 很不同。  
CPU 的 x86 指令集长期保持二进制兼容，硬件改动受到更大约束。  
GPU 通过 PTX 多了一层抽象，因此架构创新更自由。

---

# 24. Vector、SIMD、GPU 对比

## 24.1 编程模型对比

| 项目 | Vector | SIMD Extension | GPU/SIMT |
|---|---|---|---|
| 程序员视角 | 向量操作 | intrinsics/自动向量化/库 | 大量线程 |
| 编译器友好性 | 高 | 中等/偏低 | 依赖 CUDA/OpenCL |
| 控制流 | mask | 早期较弱，后来增强 | warp divergence |
| 内存访问 | stride、gather/scatter 强 | 传统较弱 | 需要 coalescing |
| 目标 | 长向量科学计算 | CPU 内短向量加速 | 高吞吐大规模并行 |

## 24.2 硬件目标对比

| 项目 | CPU SIMD | GPU |
|---|---|---|
| 优化目标 | 低延迟 + 通用性 | 高吞吐 |
| 控制逻辑 | 复杂 | 相对简单 |
| Cache | 大而复杂 | 较小，更多靠线程隐藏延迟 |
| 分支预测 | 强 | 较弱 |
| 单线程性能 | 强 | 弱 |
| 大规模并行 | 中等 | 强 |
| 内存带宽 | 较低 | 高 |

---

# 25. 检测和增强 Loop-Level Parallelism

本章后面讨论如何判断循环能否并行/向量化。

## 25.1 循环依赖

循环能否向量化，关键看不同迭代之间是否有依赖。

例如可向量化：

```c id="urpynx"
for (i = 0; i < n; i++)
    A[i] = B[i] + C[i];
```

每次迭代互不依赖。

不可直接向量化：

```c id="t5x0nw"
for (i = 1; i < n; i++)
    A[i] = A[i-1] + B[i];
```

因为 `A[i]` 依赖上一轮 `A[i-1]`。

这叫：

```text id="5rb34j"
loop-carried dependence
```

---

## 25.2 数据依赖类型

| 类型 | 名称 | 例子 | 是否影响并行 |
|---|---|---|---|
| RAW | true dependence | 先写后读 | 真依赖，影响 |
| WAR | anti-dependence | 先读后写 | 可通过重命名解决 |
| WAW | output dependence | 两次写 | 可通过重命名解决 |
| Control dependence | 控制依赖 | if/branch | 可用 predication/mask 缓解 |

最关键是真依赖 RAW。

---

## 25.3 Dependence Distance

依赖距离表示跨多少次迭代。

例如：

```c id="pq9xs2"
A[i] = A[i-1] + B[i];
```

依赖距离是 1。

```c id="qx3ic3"
A[i] = A[i-4] + B[i];
```

依赖距离是 4。

依赖距离越大，越可能通过重排、分块、软件流水等方式优化，但不能简单无脑向量化。

---

## 25.4 编译器增强 DLP 的方法

常见技术：

1. loop interchange：循环交换；
2. loop unrolling：循环展开；
3. loop distribution：循环分布；
4. loop fusion：循环融合；
5. blocking/tiling：分块；
6. software pipelining：软件流水；
7. if-conversion：把分支变成 mask/predicate；
8. dependence analysis：依赖分析；
9. alias analysis：别名分析；
10. compiler directives：程序员提示。

---

# 26. Crosscutting Issues：跨体系结构问题

本章的交叉主题主要有：

## 26.1 Memory Bandwidth 是 DLP 的生命线

无论 Vector、SIMD、GPU，都怕：

```text id="rafpld"
计算单元很多，但数据送不上来
```

性能调优时要看：

```text id="d92oh0"
实际瓶颈是 compute bound 还是 memory bound
```

Roofline 就是为这个服务。

---

## 26.2 数据布局很重要

例如 AoS 和 SoA：

```c id="svrav9"
struct Point {
    float x;
    float y;
    float z;
};
Point p[N];
```

这是 AoS。

对 SIMD/GPU 来说，有时更喜欢：

```c id="ruvrd1"
float x[N];
float y[N];
float z[N];
```

这是 SoA。

因为 SoA 更容易连续加载、合并访存、向量化。

---

## 26.3 控制流规则性很重要

SIMD/GPU 喜欢：

```text id="gelehz"
所有元素做同一件事
```

不喜欢：

```text id="ezspf5"
每个元素走不同分支
```

所以复杂分支会伤害 DLP。

---

## 26.4 访存规则性很重要

最佳访问模式：

```text id="ssriiv"
unit stride / contiguous / coalesced
```

较差访问模式：

```text id="t5ycl5"
large stride
random gather/scatter
pointer chasing
```

---

# 27. Mobile GPU、Server GPU、Tesla、Core i7 对比思想

本章用不同平台比较说明：

1. GPU 峰值性能高，但并不保证所有程序都快；
2. CPU 的 cache、分支预测、单线程性能强；
3. GPU 在规则、大规模、吞吐型任务上优势明显；
4. 对不规则内存访问、同步频繁、分支复杂任务，GPU 可能不占优势；
5. 移动 GPU 更受功耗和带宽限制；
6. 服务器 GPU 更强调吞吐、显存带宽、并行规模。

重点不是死记具体型号，而是理解：

```text id="eol80z"
架构优势必须和程序特征匹配。
```

---

# 28. Fallacies and Pitfalls：谬误与陷阱

## 28.1 谬误：GPU 因为是协处理器所以天然吃亏

不完全对。

虽然 CPU/GPU 分离带来数据传输成本，但 GPU 作为相对独立设备也有优势：

1. 架构演进更自由；
2. PTX 提供抽象层；
3. 可以更激进地尝试新硬件机制；
4. 不必像 CPU ISA 那样背负长期二进制兼容包袱。

---

## 28.2 陷阱：只看向量峰值性能，忽略启动开销

短向量可能不划算。

例如：

```text id="vapyft"
vector length 太短
start-up overhead 太大
```

实际性能可能不如标量。

---

## 28.3 陷阱：只看 GPU 峰值 FLOPS

GPU 峰值 FLOPS 很高，但实际性能受很多因素影响：

1. 数据传输；
2. memory coalescing；
3. warp divergence；
4. shared memory bank conflict；
5. occupancy；
6. register pressure；
7. cache 命中率；
8. 同步开销；
9. 原子操作；
10. 算术强度。

---

## 28.4 陷阱：以为 SIMD 宽度越大越好

SIMD 越宽，要求越高：

1. 数据必须足够多；
2. 内存要对齐；
3. 分支要少；
4. 访存要连续；
5. 编译器要能向量化；
6. tail handling 更复杂；
7. register pressure 可能上升。

---

## 28.5 陷阱：忽略 gather/scatter

很多真实程序不是完美连续访问。  
如果硬件或编译器对 gather/scatter 支持不好，SIMD 利用率会明显下降。

---

# 29. 本章核心公式与指标

## 29.1 向量执行时间近似

```text id="754d1z"
Time ≈ number of chimes × vector length
```

如果有多 lane：

```text id="y7ce4a"
Time ≈ number of chimes × vector length / number of lanes
```

还要加：

```text id="sif811"
start-up overhead
```

---

## 29.2 Cycles per FLOP

```text id="6r0b3n"
cycles per FLOP =
number of chimes / FLOPs per element
```

例如 DAXPY：

```text id="paql3t"
3 chimes
2 FLOPs per element
cycles per FLOP = 3 / 2 = 1.5
```

---

## 29.3 Roofline

```text id="9mies2"
Attainable Performance =
min(Peak FLOP/s, Peak Memory Bandwidth × Arithmetic Intensity)
```

---

## 29.4 Arithmetic Intensity

```text id="6ecqiz"
Arithmetic Intensity =
FLOPs / Bytes transferred from memory
```

---

## 29.5 Ridge Point

```text id="k3e4ki"
Ridge Point =
Peak FLOP/s / Peak Memory Bandwidth
```

---

# 30. 本章知识点压缩版

你可以把本章压缩成下面这张“脑图式提纲”：

```text id="qq4uxq"
第四章 Data-Level Parallelism
│
├── 1. DLP 基本概念
│   ├── 同一操作作用于大量数据
│   ├── SIMD 比 MIMD 更节能
│   └── 科学计算、多媒体、GPU 都依赖 DLP
│
├── 2. Vector Architecture
│   ├── Vector register
│   ├── Vector functional unit
│   ├── Vector load/store
│   ├── VLR
│   ├── VM
│   ├── Convoy
│   ├── Chime
│   ├── Chaining
│   ├── Start-up overhead
│   ├── Multiple lanes
│   ├── Strip mining
│   ├── Stride
│   └── Gather-scatter
│
├── 3. Multimedia SIMD
│   ├── MMX/SSE/AVX
│   ├── 固定宽度短向量
│   ├── 适合多媒体和库优化
│   ├── 编译器支持较难
│   └── 不如 Vector 抽象优雅
│
├── 4. Roofline Model
│   ├── Arithmetic intensity
│   ├── Peak FLOP/s
│   ├── Memory bandwidth
│   ├── Compute bound
│   └── Memory bound
│
├── 5. GPU Architecture
│   ├── Host/device
│   ├── CUDA
│   ├── Thread
│   ├── Thread block
│   ├── Grid
│   ├── Warp
│   ├── SIMT
│   ├── Shared memory
│   ├── Global memory
│   ├── Coalescing
│   └── Warp divergence
│
├── 6. Loop-Level Parallelism
│   ├── Loop-carried dependence
│   ├── Dependence analysis
│   ├── If-conversion
│   ├── Loop transformation
│   └── Compiler hints
│
└── 7. Fallacies and Pitfalls
    ├── 只看峰值性能
    ├── 忽略启动开销
    ├── 忽略内存带宽
    ├── 忽略分支发散
    ├── 忽略 gather/scatter
    └── 误以为 SIMD/GPU 自动加速所有程序
```

---

# 31. 考试/面试重点

重点掌握这些问题：

## 31.1 Vector 为什么比 SIMD extension 更通用？

因为 Vector 有：

1. VLR；
2. MVL；
3. mask register；
4. stride load/store；
5. gather/scatter；
6. 更适合编译器自动向量化。

而传统 SIMD extension：

1. 宽度固定；
2. 指令集随宽度增加而扩张；
3. 访存模式支持弱；
4. 条件执行支持弱；
5. 更依赖手写 intrinsics 或库。

---

## 31.2 GPU 为什么能隐藏内存延迟？

因为 GPU 有大量线程。  
当一个 warp 等内存时，scheduler 切换到另一个 ready warp。

不是让单个线程变快，而是让整体吞吐保持高。

---

## 31.3 GPU 什么时候不快？

当程序有：

1. 小数据量；
2. CPU/GPU 数据传输占比高；
3. 分支发散严重；
4. 随机访存；
5. 同步频繁；
6. 原子操作多；
7. 算术强度低；
8. occupancy 低；
9. register pressure 高。

---

## 31.4 Roofline 怎么用？

先算：

```text id="syrtwg"
Arithmetic Intensity = FLOPs / Bytes
```

再看：

```text id="2hs534"
Peak BW × Arithmetic Intensity
```

如果它小于 peak FLOP/s，则 memory bound。  
如果它大于 peak FLOP/s，则 compute bound。

---

## 31.5 什么样的循环容易向量化？

容易向量化：

```c id="f4d1jn"
for (i = 0; i < n; i++)
    A[i] = B[i] + C[i];
```

不容易向量化：

```c id="k25tkv"
for (i = 1; i < n; i++)
    A[i] = A[i-1] + B[i];
```

核心区别：

```text id="0xul6g"
是否存在 loop-carried dependence
```

---

# 32. 和性能调优的联系

你平时做 CPU 性能调优，可以把本章落到这些判断上：

## 32.1 看 SIMD 是否利用起来

可以关注：

```text id="w2nzif"
是否生成 AVX/NEON/SVE 指令
是否有 vectorized loop
是否有 scalar remainder loop
是否因为 alias 没向量化
是否因为 alignment 没向量化
```

## 32.2 看内存是否瓶颈

如果程序是：

```text id="83x9qn"
低 arithmetic intensity
大量 streaming load/store
cache miss 高
memory bandwidth 接近上限
```

那继续堆 SIMD 可能收益有限。

## 32.3 看数据布局

如果 AoS 导致 SIMD gather 很多，可以考虑改 SoA。

例如：

```c id="7dqxd3"
struct Tick {
    double price;
    double volume;
    int type;
};
```

如果只处理 `price`，AoS 会导致跨字段访问。  
SoA：

```c id="rwah1z"
double price[N];
double volume[N];
int type[N];
```

更容易向量化和缓存友好。

## 32.4 看分支

如果热循环中分支很多，可以考虑：

1. 拆分数据；
2. 分支外提；
3. 使用 mask/predicate；
4. 转换成查表；
5. 把同类数据聚集处理。

---

# 33. 一句话总结本章

本章的主线是：

> 数据级并行通过“一条指令/一组线程处理多个数据元素”来提升吞吐和能效；Vector 架构提供最完整、编译器友好的 SIMD 模型，CPU SIMD 扩展是更轻量但较受限的实现，GPU 则用 SIMT 和大量线程把 DLP、TLP、SIMD、硬件多线程结合起来；最终性能取决于向量长度、内存带宽、访存规则性、分支发散、算术强度和编译器能否发现并利用循环级并行。

---

## 用户

（空消息）

---

## Assistant

你是想让我**展开讲 Roofline 怎么用**对吧？

可以这样记：

```text
性能上限 = min(计算峰值, 内存带宽 × 算术强度)
```

其中：

```text
算术强度 = FLOPs / 访问内存字节数
```

判断逻辑：

```text
如果 BW × AI < Peak FLOPs
=> memory bound

如果 BW × AI ≥ Peak FLOPs
=> compute bound
```

例子：

```text
Peak = 100 GFLOP/s
BW = 50 GB/s
AI = 1 FLOP/Byte
```

则：

```text
BW × AI = 50 × 1 = 50 GFLOP/s
```

小于 100，所以是 **memory bound**。

如果 AI = 4：

```text
BW × AI = 50 × 4 = 200 GFLOP/s
```

超过 100，所以最多只能到 100，是 **compute bound**。

最关键一句：

> 算术强度低，优先优化访存；算术强度高，优先优化计算单元利用率。
