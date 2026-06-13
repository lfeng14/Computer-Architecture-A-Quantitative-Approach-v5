# 第一章 Computer Architecture: A Quantitative Approach 知识点梳理

> 章节：**Chapter 1: Fundamentals of Quantitative Design and Analysis**  
> 主题：计算机体系结构的定量设计与分析基础

---

## 第一章总纲：体系结构不是“拍脑袋设计”，而是定量权衡

这一章的核心思想可以压缩成一句话：

> **Computer Architecture = 在成本、功耗、性能、可靠性、可扩展性等约束下，用定量方法设计计算机。**

所以本章反复强调几个关键词：

| 关键词 | 含义 |
|---|---|
| Performance | 性能，不只是“频率高”，而是执行时间、吞吐率、响应时间 |
| Power / Energy | 功耗与能耗，现代架构最大约束之一 |
| Cost | 成本，包括制造成本、芯片面积、良率、封装、测试、运营成本 |
| Dependability | 可靠性/可用性，尤其对服务器和云计算非常关键 |
| Parallelism | 并行性，现代性能提升的核心 |
| Quantitative Design | 用数据、公式、实验、benchmark 做架构决策 |

---

# 1.1 Introduction：为什么需要定量设计？

## 1. 计算机性能提升来自两股力量

第一股是 **工艺技术进步**，比如晶体管变小、集成度提高、频率提升。

第二股是 **体系结构创新**，比如 RISC、流水线、cache、多发射、乱序执行、多核等。

书中提到，早期性能增长大约是每年 25%；微处理器兴起后，性能增长约每年 35%；后来由于 RISC、流水线、cache 等架构创新，曾出现过每年超过 50% 的性能增长。

## 2. RISC 的意义

RISC，也就是 Reduced Instruction Set Computer，简化指令集计算机。

它的重点不是“指令少”这么简单，而是：

| 设计思想 | 目的 |
|---|---|
| 指令简单 | 容易流水线化 |
| load-store 架构 | 内存访问和计算分离 |
| 大量寄存器 | 减少访存 |
| 固定长度指令 | 简化译码 |
| 编译器配合 | 让硬件更规整 |

RISC 架构推动了两个关键性能技术：

1. **Instruction-Level Parallelism，ILP，指令级并行**
2. **Cache，高速缓存**

这两个东西几乎贯穿整本书。

## 3. 为什么 2003 年后性能增长变慢？

原因主要是两个：

1. **功耗墙 Power Wall**  
   频率继续提高会导致功耗和散热难以承受。

2. **ILP 墙 Instruction-Level Parallelism Wall**  
   单线程内部可挖掘的并行性有限，乱序、多发射、预测执行不可能无限扩展。

所以架构重点从过去的：

```text
单核更快、更深流水、更高频率、更强 ILP
```

转向：

```text
多核、DLP、TLP、RLP、GPU、warehouse-scale computing
```

---

# 1.2 Classes of Computers：计算机的五大类别

这一节非常重要，因为不同类型计算机的优化目标不同。不能拿手机 CPU、服务器 CPU、嵌入式 MCU 用同一套指标评价。

书中把主流计算环境分成五类：PMD、Desktop、Server、Cluster/WSC、Embedded。

## 1. Personal Mobile Device，PMD，个人移动设备

比如手机、平板。

核心关注：

| 目标 | 原因 |
|---|---|
| 成本 | 消费电子价格敏感 |
| 能耗 | 电池供电 |
| 响应性 | 用户交互、视频、音频 |
| 体积 | 设备小、散热弱 |
| 软实时 | 视频帧、音频播放不能频繁卡顿 |

PMD 的关键不是极限性能，而是：

> **在有限电池、有限散热、有限成本下，让用户感觉流畅。**

## 2. Desktop Computing，桌面计算

比如 PC、工作站、笔记本。

核心关注：

| 目标 | 说明 |
|---|---|
| Price-performance | 性价比 |
| 图形性能 | UI、游戏、图形软件 |
| 通用性能 | 编译、办公、浏览器、生产力 |
| 交互响应 | 用户直接感知 |

桌面计算的典型指标是：

> **花同样的钱，谁跑得更快。**

## 3. Server，服务器

比如数据库服务器、Web 服务器、企业系统。

核心关注：

| 目标 | 说明 |
|---|---|
| Availability | 可用性，不能轻易宕机 |
| Scalability | 可扩展性 |
| Throughput | 吞吐率 |
| I/O 能力 | 网络、存储、内存带宽 |
| Reliability | 可靠性 |

服务器不只是看单个请求快不快，更看：

> **单位时间能处理多少请求，以及能不能 7×24 小时稳定运行。**

## 4. Clusters / Warehouse-Scale Computers，集群/仓库级计算机

比如 Google、Amazon、Meta 这种数据中心。

核心关注：

| 目标 | 说明 |
|---|---|
| Price-performance | 数量巨大，成本敏感 |
| Power | 电费和散热成本极高 |
| Throughput | 海量请求处理 |
| Availability | 单机可以坏，服务不能挂 |
| Scalability | 横向扩展 |
| Energy proportionality | 负载低时功耗也应下降 |

WSC 和传统服务器不同：

| 传统服务器 | WSC |
|---|---|
| 单机可靠性强 | 单机可以普通，但系统层面容错 |
| 依赖高端硬件 | 依赖软件冗余 |
| 垂直扩展 | 水平扩展 |
| 强调单系统可靠 | 强调整体服务可靠 |

一个很关键的思想：

> **在超大规模系统里，硬件一定会坏，所以架构必须假设故障是常态。**

## 5. Embedded Computers，嵌入式计算机

比如微波炉、洗衣机、打印机、汽车控制器、交换机芯片。

核心关注：

| 目标 | 说明 |
|---|---|
| 成本 | 很多嵌入式芯片极度便宜 |
| 功耗 | 低功耗常常很重要 |
| 实时性 | 控制系统有 deadline |
| 应用专用性 | 为特定任务优化 |
| 体积 | 芯片/板级资源有限 |

嵌入式设计经常不是追求最高性能，而是：

> **刚好满足性能要求，同时成本最低、功耗最低。**

---

# 1.2 后半：并行性的分类

本章把并行性分成两个应用层面的来源：

## 1. Data-Level Parallelism，DLP，数据级并行

意思是：

> 很多数据可以执行同样或类似的操作。

例子：

```text
for i in range(N):
    C[i] = A[i] + B[i]
```

每个元素加法彼此独立，可以 SIMD、向量化、GPU 并行。

## 2. Task-Level Parallelism，TLP，任务级并行

意思是：

> 多个任务可以相互独立执行。

例子：

| 场景 | 并行任务 |
|---|---|
| Web 服务器 | 多个用户请求 |
| 编译系统 | 多个源文件并行编译 |
| 数据库 | 多个查询 |
| 操作系统 | 多个进程/线程 |

---

# Flynn 分类：SISD、SIMD、MISD、MIMD

这是经典并行计算分类。

| 类型 | 全称 | 含义 | 例子 |
|---|---|---|---|
| SISD | Single Instruction Single Data | 单指令流、单数据流 | 传统单核顺序处理器 |
| SIMD | Single Instruction Multiple Data | 单指令流、多数据流 | 向量处理器、GPU、SIMD 指令 |
| MISD | Multiple Instruction Single Data | 多指令流、单数据流 | 基本没有商业主流机器 |
| MIMD | Multiple Instruction Multiple Data | 多指令流、多数据流 | 多核 CPU、集群、服务器 |

重点：

- **SIMD** 适合 DLP。
- **MIMD** 适合 TLP，也可以处理 DLP，但开销更大。
- 现代系统常常是混合体：比如 CPU 是 MIMD，每个核内部又有 SIMD。

---

# 1.3 Defining Computer Architecture：什么是计算机体系结构？

这一节很关键，因为它纠正一个误解：

> 体系结构不是只有 ISA，真正的 Computer Architecture 包括 ISA、organization / microarchitecture、hardware。

## 1. ISA：Instruction Set Architecture

ISA 是程序员可见的指令集架构，是软件和硬件之间的接口。

比如：

| ISA | 代表 |
|---|---|
| x86 / x86-64 | Intel、AMD |
| ARM | 手机、嵌入式、Apple Silicon |
| MIPS | 教学、网络设备、早期 RISC |
| RISC-V | 开源 ISA |

ISA 决定：

- 有哪些寄存器
- 有哪些指令
- 如何访问内存
- 指令怎么编码
- 数据类型有哪些
- 分支/调用如何实现
- 程序员/编译器能看到什么

## 2. ISA 的七个维度

书中用 MIPS、ARM、x86 对比，列出 ISA 的七个维度。

### 维度一：ISA 类型

主要有：

| 类型 | 特点 |
|---|---|
| Stack architecture | 操作数隐含在栈顶 |
| Accumulator architecture | 累加器作为隐含操作数 |
| Register-memory | 指令可以直接操作内存 |
| Load-store | 只有 load/store 访问内存，计算只在寄存器间进行 |

现代 RISC 大多是 **load-store**。

也就是：

```asm
load  r1, [mem]
load  r2, [mem]
add   r3, r1, r2
store [mem], r3
```

而不是：

```asm
add [mem1], [mem2]
```

### 维度二：Memory Addressing，内存寻址

关键概念：

| 概念 | 含义 |
|---|---|
| byte addressing | 按字节编址 |
| alignment | 对齐访问 |
| unaligned access | 非对齐访问，可能慢或异常 |

对齐的意思是：

> 大小为 s 字节的数据，地址 A 如果满足 A mod s = 0，就是对齐访问。

例如 4 字节 int 放在地址 0x1000 是对齐，放在 0x1001 就是不对齐。

### 维度三：Addressing Modes，寻址模式

常见寻址方式：

| 寻址模式 | 例子 | 含义 |
|---|---|---|
| Register | R1 | 操作数在寄存器 |
| Immediate | #100 | 常数立即数 |
| Displacement | 100(R1) | 地址 = R1 + 100 |
| PC-relative | PC + offset | 分支常用 |
| Indexed | base + index | 数组访问 |
| Scaled indexed | base + index × scale | x86 常见 |

MIPS 比较简洁，x86 寻址模式更复杂。

### 维度四：Operand Types and Sizes，操作数类型与大小

常见类型：

| 类型 | 大小 |
|---|---|
| byte | 8 bit |
| half word | 16 bit |
| word | 32 bit |
| double word | 64 bit |
| single precision float | 32 bit |
| double precision float | 64 bit |

### 维度五：Operations，操作类型

主要包括：

| 类型 | 例子 |
|---|---|
| 数据传送 | load、store、move |
| 算术 | add、sub、mul、div |
| 逻辑 | and、or、xor、shift |
| 控制流 | branch、jump、call、return |
| 浮点 | fadd、fmul、fdiv |
| 系统 | trap、exception return |

### 维度六：Control Flow，控制流

包括：

- 条件分支
- 无条件跳转
- 函数调用
- 返回
- 异常/中断

不同 ISA 的差别：

| ISA | 分支判断方式 |
|---|---|
| MIPS | 比较寄存器 |
| ARM / x86 | 使用 condition codes / flags |
| MIPS / ARM | 返回地址放寄存器 |
| x86 | 返回地址压栈 |

### 维度七：Instruction Encoding，指令编码

两大类：

| 类型 | 优点 | 缺点 |
|---|---|---|
| 固定长度 | 译码简单，适合流水线 | 代码密度较低 |
| 变长 | 代码更紧凑 | 译码复杂 |

MIPS/ARM 经典指令常为 32-bit 固定长度；x86 是 1 到 18 字节不等的变长编码。

---

# ISA、微架构、硬件的区别

这是第一章非常容易考的点。

| 层次 | 英文 | 含义 | 例子 |
|---|---|---|---|
| 指令集架构 | ISA | 程序员可见接口 | x86、ARM、MIPS、RISC-V |
| 组织/微架构 | Organization / Microarchitecture | 如何实现 ISA | 流水线、cache、乱序、多发射 |
| 硬件实现 | Hardware | 具体电路、工艺、封装 | 7nm、晶体管、电源网络、封装 |

例如：

> Intel Core i7 和 AMD Opteron 都可以实现 x86 ISA，但微架构不同。  
> 同一个 ISA 可以有不同实现。

这就是为什么：

```text
ISA 相同 ≠ 性能相同
```

---

# 1.4 Trends in Technology：技术趋势

这一节说明：体系结构设计必须跟着技术趋势走。

## 1. 五类关键技术

书中列出五类对现代计算机影响巨大的技术：

| 技术 | 作用 |
|---|---|
| Integrated circuit logic | 逻辑晶体管，决定 CPU/GPU 能力 |
| DRAM | 主存 |
| Flash | 非易失存储，手机/SSD |
| Magnetic disk | 大容量存储 |
| Network | 集群/WSC 通信基础 |

## 2. Moore’s Law，摩尔定律

核心意思：

> 单位面积晶体管数量持续增长。

书中说，晶体管密度大约每年增长 35%，芯片上晶体管数量大约每年增长 40% 到 55%，大概每 18 到 24 个月翻倍。

但注意：

> 摩尔定律主要说的是晶体管数量，不等于性能、频率、能效都同步翻倍。

这是现代芯片的关键矛盾。

## 3. Bandwidth over Latency：带宽提升快于延迟

这一章特别强调：

> 技术发展中，带宽提升通常远快于延迟改善。

| 指标 | 含义 |
|---|---|
| Bandwidth / Throughput | 单位时间完成多少工作 |
| Latency / Response time | 一个任务从开始到完成要多久 |

例子：

| 设备 | 带宽提升 | 延迟提升 |
|---|---|---|
| CPU | 很大 | 相对小 |
| 内存 | 带宽提升明显 | 延迟改善有限 |
| 磁盘 | 容量/带宽提升大 | 寻道/旋转延迟改善慢 |
| 网络 | 带宽提升巨大 | 延迟改善没那么大 |

这对架构设计的启发是：

> 既然延迟难降，就要用并行、流水线、预取、缓存、批处理等方式隐藏延迟。

## 4. 晶体管和导线的趋势不同

晶体管变小以后通常：

- 数量增加
- 开关速度提升
- 单位面积能力提高

但导线不一样：

- 导线延迟不一定随工艺缩小而变好
- 长距离通信越来越贵
- 片上互连成为瓶颈

所以现代芯片不是“晶体管越多越爽”，还要处理：

- wire delay
- clock distribution
- power distribution
- interconnect congestion
- NoC / mesh interconnect

---

# 1.5 Trends in Power and Energy：功耗与能耗趋势

这一节是现代体系结构的核心。

## 1. Power 和 Energy 的区别

| 概念 | 公式 | 含义 |
|---|---|---|
| Power | Energy / Time | 功率，单位 W |
| Energy | Power × Time | 能量，单位 J |

非常重要：

> 对一个固定任务，比较处理器效率时，Energy 通常比 Power 更合理。

例如：

- A 处理器功率比 B 高 20%
- 但 A 执行时间只有 B 的 70%

那么 A 的能耗：

```text
1.2 × 0.7 = 0.84
```

说明 A 反而更省能量。

这就是书中强调的：

> Power 常常是约束，Energy 才更适合衡量完成某任务的代价。

## 2. TDP：Thermal Design Power

TDP 是热设计功耗，主要用于散热设计。

注意三个概念不要混：

| 概念 | 含义 |
|---|---|
| Peak Power | 峰值功耗 |
| TDP | 散热系统需要长期处理的功耗 |
| Average Power | 某个 workload 实际平均功耗 |

TDP 不是峰值，也不一定等于真实平均功耗。

## 3. CMOS 动态能耗公式

动态能耗来自晶体管翻转。

单次 0→1 或 1→0 近似为：

```text
Energy ∝ 1/2 × Capacitive Load × Voltage²
```

动态功耗：

```text
Power ∝ 1/2 × Capacitive Load × Voltage² × Frequency switched
```

核心结论：

> 电压 V 是平方项，所以降压对能耗/功耗非常有效。

## 4. 为什么降频不一定省能量？

对于固定任务：

- 降频会降低功率
- 但任务执行时间变长
- 所以总能耗不一定下降

这句话很要命：

> **降低频率可以降低 Power，但不一定降低 Energy。**

如果同时降压，就可能明显降低能耗。

## 5. DVFS：Dynamic Voltage-Frequency Scaling

DVFS 是动态电压频率调整。

负载低时：

- 降低频率
- 降低电压
- 降低功耗和能耗

常见于：

- 手机
- 笔记本
- 服务器
- CPU governor
- Turbo / power states

## 6. 现代节能技术

书中列了几个方向：

| 技术 | 含义 |
|---|---|
| clock gating | 空闲模块关闭时钟 |
| power gating | 空闲模块断电 |
| DVFS | 动态调电压频率 |
| typical-case design | 为常见情况优化 |
| multicore | 不靠无限提频，而靠并行 |
| energy proportionality | 负载低时功耗也低 |

---

# 1.6 Trends in Cost：成本趋势

成本这节非常适合理解芯片为什么贵。

## 1. 成本不只是“硅片材料钱”

芯片成本包括：

- 晶圆成本
- die 面积
- 良率
- 测试
- 封装
- 再测试
- 掩膜版 mask
- 研发
- 设备折旧
- 运营成本

书中强调，die 做出来后还要测试、封装、封装后再测试，这些都会增加显著成本。

## 2. Die 面积越大，成本急剧上升

原因有两个：

1. 同一片 wafer 上能切出的 die 更少。
2. die 越大，包含缺陷的概率越高，良率越低。

所以大芯片不是线性变贵，而是经常 **超线性变贵**。

## 3. 良率 Yield

良率指：

> 一片 wafer 上，最终可用 die 的比例。

影响因素：

| 因素 | 影响 |
|---|---|
| die 面积 | 面积越大，越容易碰到缺陷 |
| 缺陷密度 | 工艺越成熟，缺陷越低 |
| 工艺复杂度 | 越先进越难 |
| 设计冗余 | 可修复结构能提高有效良率 |

这和“芯片良品率为什么用泊松分布”是连起来的：如果缺陷随机落在晶圆上，die 上缺陷数可以近似建模为泊松分布。

## 4. Mask cost，掩膜版成本

现代芯片需要很多层 mask。对于现代高密度工艺，mask 成本可以超过 100 万美元，尤其低产量芯片会受影响很大。

所以：

| 高产量芯片 | 低产量芯片 |
|---|---|
| mask 成本可摊薄 | mask 成本很重 |
| 适合 ASIC | 可能更适合 FPGA / gate array |
| 单颗成本低 | 单颗成本高 |

## 5. Cost vs Price

Cost 是制造/运营成本。

Price 是卖价。

中间差额要覆盖：

- R&D
- 市场营销
- 销售
- 设备维护
- 厂房
- 融资成本
- 利润
- 税

工程师容易只看制造成本，但商业上还要考虑完整成本结构。

## 6. Manufacturing Cost vs Operating Cost

传统计算机主要看购买成本。

但是 WSC / 数据中心还要看运营成本：

- 电费
- 散热
- 电力基础设施
- 机房建设
- 设备折旧
- 维护

所以架构师必须重视能效。

---

# 1.7 Dependability：可靠性、可用性、容错

这一节要区分几个概念。

## 1. Dependability 是总概念

Dependability 可以理解为系统可信赖程度，包括：

| 概念 | 含义 |
|---|---|
| Reliability | 可靠性，系统持续正确工作的能力 |
| Availability | 可用性，系统在某时刻可提供服务的概率 |
| MTTF | Mean Time To Failure，平均无故障时间 |
| MTTR | Mean Time To Repair，平均修复时间 |
| Fault | 故障原因 |
| Error | 错误状态 |
| Failure | 对外服务失败 |

## 2. Reliability 和 Availability 不一样

Reliability 更关注：

> 多久不坏？

Availability 更关注：

> 坏了能不能很快恢复，所以大部分时间仍然可用？

公式：

```text
Availability = MTTF / (MTTF + MTTR)
```

## 3. FIT

FIT = Failures In Time。

通常表示：

```text
每 10^9 小时发生多少次失败
```

如果故障率越高，MTTF 越低。

## 4. 系统可靠性通常由最弱环节限制

假设组件独立失效，系统整体 failure rate 可以近似相加。

例如：

```text
Failure rate_system = Σ Failure rate_component
```

也就是：

```text
1 / MTTF_system = Σ (1 / MTTF_component)
```

结论：

> 组件越多，系统越容易坏。单个组件很可靠，不代表系统很可靠。

## 5. Redundancy，冗余

提升可靠性的核心方法是冗余：

| 冗余类型 | 例子 |
|---|---|
| 时间冗余 | 重试、重复计算 |
| 空间/资源冗余 | RAID、双电源、备份节点 |
| 信息冗余 | ECC、校验码 |
| 软件冗余 | WSC 中故障隔离和任务迁移 |

但是如果系统还有单点故障，整体可靠性仍会被限制。

---

# 1.8 Measuring, Reporting, and Summarizing Performance：性能如何衡量？

这一节特别重要，基本是后面所有性能分析的基础。

## 1. Performance 的两个视角

| 指标 | 英文 | 含义 | 典型场景 |
|---|---|---|---|
| 响应时间 | Response time / Execution time | 完成单个任务要多久 | 个人电脑、交互应用 |
| 吞吐率 | Throughput / Bandwidth | 单位时间完成多少任务 | 服务器、数据中心 |

例子：

- 手机打开 App：看响应时间
- Web 服务器每秒处理请求数：看吞吐率
- 编译一个文件：看执行时间
- 数据库每秒事务数：看吞吐率

## 2. 性能与执行时间互为倒数

如果比较同一个任务：

```text
Performance = 1 / Execution Time
```

所以：

```text
Performance_X / Performance_Y = ExecutionTime_Y / ExecutionTime_X
```

如果 X 的执行时间是 5 秒，Y 是 10 秒：

```text
X 比 Y 快 2 倍
```

## 3. CPU Time 公式

经典公式：

```text
CPU Time = Instruction Count × CPI × Clock Cycle Time
```

也可以写成：

```text
CPU Time = Instruction Count × CPI / Clock Rate
```

其中：

| 项 | 含义 |
|---|---|
| Instruction Count | 指令数 |
| CPI | Cycles Per Instruction，每条指令平均周期数 |
| Clock Cycle Time | 时钟周期 |
| Clock Rate | 时钟频率 |

所以提升性能可以从三方面来：

| 方向 | 例子 |
|---|---|
| 降低指令数 | 更好的 ISA、编译器优化 |
| 降低 CPI | 流水线、cache、预测、乱序 |
| 提高频率 | 工艺、流水线、物理设计 |

但是三者相互影响。

比如：

- CISC 指令数少，但 CPI 可能高。
- RISC 指令数多一点，但流水线容易，CPI 低。
- 提高频率可能导致流水线更深，分支代价更高。
- cache 变大可能降低 miss，但访问延迟增加。

所以不能只看一个指标。

## 4. Benchmark

Benchmark 是性能评估程序。

常见思想：

| 类型 | 说明 |
|---|---|
| Real programs | 真实应用，最可信 |
| Kernels | 抽取关键代码 |
| Toy benchmarks | 小程序，不可靠 |
| Synthetic benchmarks | 人工合成，谨慎使用 |
| Benchmark suites | 多个程序组成，如 SPEC |

SPEC 是书中使用的重要 benchmark 系列。

## 5. 为什么不能只用 MIPS / MFLOPS？

MIPS = Million Instructions Per Second。

问题：

1. 不同 ISA 的一条指令做的事不同。
2. 同一程序不同编译器指令数不同。
3. MIPS 高不代表执行时间短。
4. 可能优化了“指令条数”，但实际没变快。

MFLOPS 也类似，只适合浮点计算密集任务，不适合通用程序。

## 6. 如何汇总多个 benchmark？

算术平均、调和平均、几何平均都有不同适用场景。

体系结构比较中常用 **geometric mean，几何平均**，尤其是比较归一化后的性能比值。

如果有 n 个性能比：

```text
GM = (Π ratio_i)^(1/n)
```

原因是：

- 对比值公平
- 不容易被单个大数支配
- 与参考机器选择关系较小

---

# 1.9 Quantitative Principles of Computer Design：定量设计原则

这一节是第一章的灵魂。

## 原则一：Take Advantage of Parallelism，利用并行性

并行性存在于多个层次：

| 层次 | 例子 |
|---|---|
| bit-level | 加法器并行处理位 |
| instruction-level | 流水线、乱序、多发射 |
| data-level | SIMD、向量、GPU |
| thread-level | 多核、多线程 |
| request-level | Web 请求、集群服务 |

流水线能工作的关键是：不是每条指令都依赖前一条指令，因此可以部分重叠执行。

## 原则二：Principle of Locality，局部性原理

程序有局部性：

> 最近用过的东西，很可能马上还会用；地址相近的东西，也可能很快一起用。

两种局部性：

| 类型 | 英文 | 含义 | 例子 |
|---|---|---|---|
| 时间局部性 | Temporal locality | 最近访问过的未来还会访问 | 循环变量、函数代码 |
| 空间局部性 | Spatial locality | 地址相近的数据会被连续访问 | 数组遍历、顺序取指 |

Cache、TLB、预取、分层存储都依赖局部性。

## 原则三：Focus on the Common Case，优化常见情况

这是架构设计第一原则之一：

> 设计取舍时，优先优化经常发生的情况，而不是罕见情况。

例子：

| 常见情况 | 优化方式 |
|---|---|
| 指令 fetch/decode 经常发生 | 优先优化前端 |
| cache hit 比 miss 常见 | hit path 要极快 |
| 整数加法比除法常见 | 加法器更关键 |
| 无溢出比溢出常见 | 溢出走慢路径 |
| 分支通常可预测 | 做分支预测 |

这条原则不仅适用于性能，也适用于功耗和可靠性。

## 原则四：Amdahl’s Law，阿姆达尔定律

阿姆达尔定律说明：

> 优化某一部分带来的整体加速，受这部分在原执行时间中占比的限制。

公式：

```text
Speedup_overall = 1 / ((1 - Fraction_enhanced) + Fraction_enhanced / Speedup_enhanced)
```

其中：

| 项 | 含义 |
|---|---|
| Fraction_enhanced | 被优化部分原来占总时间的比例 |
| Speedup_enhanced | 这部分本身加速了多少 |
| Speedup_overall | 整体加速比 |

例子：

如果某部分占 40%，你把它加速 10 倍：

```text
整体加速 = 1 / (0.6 + 0.4/10)
        = 1 / 0.64
        = 1.5625
```

也就是说，局部快 10 倍，整体只快 56%。

阿姆达尔定律最狠的地方是：

> 如果一部分本来占比很小，优化它再多也没用。

所以架构优化前必须测量：

- 时间花在哪里？
- miss 花在哪里？
- stall 花在哪里？
- 功耗花在哪里？
- 请求瓶颈在哪里？

## 原则五：Processor Performance Equation

也就是前面的 CPU Time 公式：

```text
CPU Time = IC × CPI × Cycle Time
```

它告诉你：性能优化不是单变量问题。

比如一个优化：

- 指令数减少 20%
- 但 CPI 增加 10%
- 频率降低 5%

那最后未必更快，要乘起来算。

## 原则六：Memory Hierarchy

虽然第一章只是铺垫，但局部性直接导出存储层次：

| 层次 | 特点 |
|---|---|
| Register | 最快、最小、最贵 |
| L1 Cache | 很快 |
| L2/L3 Cache | 较大、较慢 |
| DRAM | 主存，大但慢 |
| SSD/HDD | 更大但更慢 |
| Remote storage | 更远、更慢 |

核心思想：

> 用小而快的存储保存最近/邻近会用的数据，用大而慢的存储保存全部数据。

---

# 1.10 Putting It All Together：性能、价格、功耗放在一起看

这一节的核心是：

> 架构设计不能只看性能，要同时看 performance、price、power。

常见综合指标：

| 指标 | 含义 |
|---|---|
| Performance / Price | 性价比 |
| Performance / Watt | 每瓦性能 |
| Energy / Task | 每个任务消耗能量 |
| Cost of Ownership | 总拥有成本 |
| Energy proportionality | 能耗是否随负载下降 |

例如：

一个处理器性能高 20%，但功耗高 80%，价格高 50%，它未必是更好的设计。

对不同市场：

| 市场 | 更重视 |
|---|---|
| 手机 | Energy / Task、电池寿命、响应 |
| PC | Price-performance |
| 服务器 | Throughput、availability、TCO |
| 数据中心 | Performance/Watt、power、cooling |
| 嵌入式 | 成本、实时性、功耗 |

---

# 1.11 Fallacies and Pitfalls：常见谬误和陷阱

这一节很适合考试和工程实践。

## Fallacy 1：硬件增强一定提升能效

错误。

性能提升不一定能效更好。

正确理解：

> 更快不一定更省电。要看 Energy per task。

## Fallacy 2：Benchmark 永久有效

错误。

benchmark 会过时，因为：

- 应用变化
- 编译器专门优化 benchmark
- 体系结构针对 benchmark 调优
- benchmark 被“刷分”
- 小 kernel 不代表真实负载

正确理解：

> benchmark 是工具，不是真理。要看它是否代表真实 workload。

## Fallacy 3：磁盘 MTTF 很高，所以几乎不会坏

错误。

单块磁盘 MTTF 可能标得很高，但大规模系统有成千上万块磁盘，整体故障会非常频繁。

正确理解：

```text
组件越多，系统故障率越高
```

## Pitfall 1：忽视 Amdahl’s Law

优化一个很少发生的部分，即使局部加速巨大，整体收益也很小。

正确做法：

1. 先 profile
2. 找热点
3. 估算占比
4. 用 Amdahl’s Law 算上限
5. 再决定是否优化

## Pitfall 2：单点故障

系统如果有单个不可替代组件，比如唯一风扇、唯一电源、唯一网络链路，那么这个组件会限制整体可靠性。

## Pitfall 3：只看平均值

平均性能可能掩盖尾延迟。

尤其在：

- Web 服务
- 数据中心
- 交互系统
- 实时系统

要关注：

| 指标 | 含义 |
|---|---|
| mean | 平均 |
| median | 中位数 |
| p95 | 95 分位延迟 |
| p99 | 99 分位延迟 |
| worst-case | 最坏情况 |

用户感知经常更接近 tail latency，而不是平均值。

---

# 1.12 Concluding Remarks：本章结论

第一章最后的精神是：

> 计算机架构已经从“靠更高频率和更复杂单核自动变快”，转向“在功耗、成本、可靠性约束下显式利用并行性”。

过去：

```text
工艺进步 + ILP + cache + 高频
```

现在：

```text
多核 + DLP + TLP + RLP + 能效 + 可靠性 + 成本控制
```

这也是为什么现代架构师必须同时懂：

- 编译器
- 操作系统
- 电路
- 存储层次
- 并行程序
- benchmark
- 功耗
- 可靠性
- 成本模型

---

# 第一章公式汇总

## 1. 性能与执行时间

```text
Performance = 1 / Execution Time
```

```text
Performance_X / Performance_Y = ExecutionTime_Y / ExecutionTime_X
```

## 2. CPU Time

```text
CPU Time = Instruction Count × CPI × Clock Cycle Time
```

```text
CPU Time = Instruction Count × CPI / Clock Rate
```

## 3. 动态能耗

```text
Energy_dynamic ∝ 1/2 × Capacitive Load × Voltage²
```

## 4. 动态功耗

```text
Power_dynamic ∝ 1/2 × Capacitive Load × Voltage² × Frequency switched
```

## 5. Energy

```text
Energy = Power × Time
```

## 6. Availability

```text
Availability = MTTF / (MTTF + MTTR)
```

## 7. Failure Rate

```text
Failure rate = 1 / MTTF
```

系统多个独立组件：

```text
Failure rate_system = Σ Failure rate_component
```

也就是：

```text
1 / MTTF_system = Σ (1 / MTTF_i)
```

## 8. Amdahl’s Law

```text
Speedup_overall =
1 / ((1 - Fraction_enhanced) + Fraction_enhanced / Speedup_enhanced)
```

## 9. 几何平均

```text
Geometric Mean = (Π ratio_i)^(1/n)
```

---

# 第一章最重要的 10 个知识点

1. **体系结构设计是定量权衡，不是单纯追求频率。**
2. **计算机分为 PMD、Desktop、Server、WSC、Embedded，不同类别目标不同。**
3. **ISA 只是体系结构的一部分，体系结构还包括微架构和硬件实现。**
4. **RISC 的价值在于简化硬件、便于流水线、cache、编译器优化。**
5. **2003 年后单核性能增长变慢，主要因为功耗墙和 ILP 墙。**
6. **现代性能提升依赖 DLP、TLP、RLP、多核、GPU、集群。**
7. **带宽提升通常远快于延迟改善，所以要隐藏延迟。**
8. **Energy 比 Power 更适合衡量固定任务的能效。**
9. **性能分析必须用执行时间、CPI、IC、clock rate、benchmark，而不是只看频率。**
10. **Amdahl’s Law 告诉你：优化前先看占比，否则很容易白忙。**

---

# 学习这章时的主线

你可以把第一章理解成一张架构师的“决策地图”：

```text
应用场景是什么？
        ↓
目标是什么？性能 / 成本 / 功耗 / 可靠性 / 可扩展性？
        ↓
瓶颈在哪里？执行时间 / CPI / memory / I/O / power / failure？
        ↓
常见情况是什么？
        ↓
优化这部分最多能带来多少收益？Amdahl’s Law
        ↓
会不会增加成本、功耗、复杂度？
        ↓
用 benchmark 和真实 workload 验证
```

这就是这本书所谓的 **Quantitative Approach**。第一章就是在告诉你：以后看任何架构优化，都不要只问“这个技术牛不牛”，而要问：

> **它优化了什么？占比多大？代价是什么？整体收益是多少？适合哪类计算机？**

***

# 良品率为什么使用泊松分布？怎么用泊松分布？

- 现实中不是不用二项分布，而是很多工程问题满足“机会很多、概率很小”的特点，此时泊松分布既准确又简单，因此成为工程界的首选模型。
- 使用Yield = P(X=0) = e^(-6.75)

## 1. X 表示什么？

在芯片制造的泊松模型中：

$$
X = \text{Die 上出现的缺陷数量}
$$

因此：

- `X = 0`：没有缺陷
- `X = 1`：有 1 个缺陷
- `X = 2`：有 2 个缺陷
- `X = 3`：有 3 个缺陷
- 以此类推

---

## 2. 为什么良品率是 P(X=0)？

对于最简单的良品率模型，假设：

> 只要出现一个缺陷，芯片就报废。

那么只有一种情况是良品：

$$
X = 0
$$

也就是：

> 该 Die 上一个缺陷都没有。

因此：

$$
Yield = P(X=0)
$$

**Yield（良品率）本质上就是“零缺陷概率”。**

---

## 3. 泊松分布公式

如果：

$$
X \sim Poisson(\lambda)
$$

则：

$$
P(X=k)=\frac{\lambda^k e^{-\lambda}}{k!}
$$

其中：

- $\lambda$：平均缺陷数
- $k$：实际缺陷数

---

## 4. 求 X=0 的概率

令：

$$
k=0
$$

代入泊松分布公式：

$$
P(X=0)=\frac{\lambda^0 e^{-\lambda}}{0!}
$$

因为：

$$
\lambda^0 = 1
$$

并且：

$$
0! = 1
$$

所以：

$$
P(X=0)=e^{-\lambda}
$$

因此：

$$
\boxed{Yield=e^{-\lambda}}
$$

---

## 5. 芯片缺陷例子

假设：

- Die 面积：$A = 225\ mm^2$
- 缺陷密度：$D_0 = 0.03/mm^2$

平均缺陷数为：

$$
\lambda = A \times D_0
$$

代入：

$$
\lambda = 225 \times 0.03 = 6.75
$$

于是：

$$
Yield = P(X=0)=e^{-6.75}
$$

计算结果：

$$
e^{-6.75} \approx 0.00117
$$

也就是：

$$
Yield \approx 0.117\%
$$

含义是：

> 平均 1000 颗芯片中，大约只有 1 颗完全没有缺陷。

---

## 6. 直观理解

可以把缺陷想象成“随机落下的雨滴”：

- $\lambda = 6.75$ 表示平均会落下 6.75 个缺陷
- $X = 0$ 表示一滴缺陷都没有落到这块 Die 上

而良品率恰好就是：

> 一滴缺陷都没落到芯片上的概率。

因此：

$$
\boxed{Yield=P(X=0)}
$$

并且：

$$
\boxed{Yield=e^{-A D_0}}
$$

这就是芯片制造中经典的泊松良品率模型。
