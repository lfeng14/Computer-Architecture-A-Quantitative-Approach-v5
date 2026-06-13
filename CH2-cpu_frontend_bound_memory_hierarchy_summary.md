## 1. 背景

本轮对话主要围绕三个主题展开：

1. 《Computer Architecture: A Quantitative Approach》第二章 **Memory Hierarchy Design** 的知识点梳理。
2. 性能调优中遇到 `frontend bound 35%`、`bandwidth 10%`、`ooo flush 10%` 时的分析和调优建议。
3. CPU 分支误预测的概念、原因、影响和优化方法。

---

# 2. 第二章：Memory Hierarchy Design 知识点梳理

第二章的核心思想可以概括为：

> CPU 越来越快，而主存和外存相对越来越慢，因此必须通过存储层次结构，让系统看起来像拥有一个又快、又大、又便宜的存储系统。

典型存储层次结构如下：

```text
寄存器 → L1 Cache → L2 Cache → L3 Cache → 主存 DRAM → SSD/Flash/磁盘
```

越靠近 CPU：

```text
速度越快
容量越小
单位成本越高
```

越远离 CPU：

```text
速度越慢
容量越大
单位成本越低
```

---

## 2.1 局部性原理

存储层次结构能够有效工作，根本原因是程序访问内存具有局部性。

### 2.1.1 时间局部性 Temporal Locality

刚访问过的数据，短时间内很可能再次访问。

例如：

```cpp
for (int i = 0; i < 1000; i++) {
    sum += a[i];
}
```

变量 `sum` 会被反复访问。

---

### 2.1.2 空间局部性 Spatial Locality

访问某个地址后，附近地址也很可能被访问。

例如：

```cpp
a[0], a[1], a[2], a[3]
```

数组连续访问就是典型的空间局部性。

---

## 2.2 Cache 基本概念

CPU 访问数据时，通常会先查 Cache：

```text
CPU 查 L1
    命中：直接返回
    未命中：查 L2
        命中：返回并填入 L1
        未命中：查 L3 或主存
```

核心概念：

| 概念 | 含义 |
|---|---|
| Cache block / line | Cache 和内存之间搬运的基本单位 |
| Tag | 标记 Cache line 对应哪个内存地址 |
| Hit | 数据在 Cache 中 |
| Miss | 数据不在 Cache 中 |
| Hit time | 命中访问时间 |
| Miss penalty | 未命中后从下一级取数据的额外代价 |
| Miss rate | 未命中率 |

---

## 2.3 Cache 放置策略

### Direct Mapped Cache

每个内存块只能放到 Cache 中固定位置。

优点：

```text
简单
快
省电
```

缺点：

```text
容易发生 conflict miss
```

---

### Fully Associative Cache

任意内存块可以放到 Cache 任意位置。

优点：

```text
冲突少
```

缺点：

```text
硬件复杂
查找慢
功耗高
```

---

### Set Associative Cache

折中方案。

```text
先根据地址定位到某个 set
然后在 set 里面选择一个 way
```

公式：

```text
Set index = Block address MOD Number of sets
```

如果每个 set 有 n 个 block，就叫 n-way set associative cache。

---

## 2.4 Cache 写策略

### Write-through

写 Cache 的同时，也写下一级存储。

优点：

```text
一致性简单
```

缺点：

```text
写流量大
带宽压力大
```

---

### Write-back

只先写 Cache，等 Cache block 被替换时再写回下一级。

优点：

```text
减少写流量
```

缺点：

```text
一致性更复杂
需要 dirty bit
```

现代处理器大量使用 write-back。

---

## 2.5 Write Buffer

写缓冲可以避免 CPU 等待写内存完成。

```text
CPU 写入 write buffer 后继续执行
write buffer 慢慢把数据写到下一级
```

但它会带来 hazard：

```text
如果后续 read 读取的地址正好还在 write buffer 中，
就必须检查 write buffer，否则可能读到旧值。
```

---

## 2.6 Cache Miss 的 3C 模型

### Compulsory Miss

第一次访问某个 block，不可能在 Cache 中。

也叫 cold miss。

---

### Capacity Miss

Cache 太小，装不下程序的工作集。

---

### Conflict Miss

Cache 总容量可能够，但多个地址映射到同一个 set，互相挤掉。

---

多核场景还有第四类：

### Coherency Miss

多个核心维护 Cache 一致性时，一个核心修改数据，另一个核心的 Cache line 被 invalidation，之后再访问就会 miss。

---

## 2.7 核心评价指标

### Miss rate

```text
Miss rate = Misses / Memory accesses
```

---

### Misses per instruction

```text
Misses per instruction
= Miss rate × Memory accesses per instruction
```

---

### AMAT

AMAT 是本章最重要的公式之一：

```text
AMAT = Hit time + Miss rate × Miss penalty
```

也就是：

```text
平均访存时间 = 命中时间 + 未命中率 × 未命中代价
```

两级 Cache 的 AMAT 可以写成：

```text
AMAT = L1 hit time
     + L1 miss rate × (L2 hit time + L2 miss rate × L2 miss penalty)
```

---

# 3. 第二章中的基础 Cache 优化

## 3.1 增大 block size

利用空间局部性，减少 compulsory miss。

缺点：

```text
miss penalty 可能增加
可能增加 capacity miss
可能增加 conflict miss
可能浪费带宽
```

---

## 3.2 增大 Cache 容量

主要减少 capacity miss。

缺点：

```text
hit time 可能变长
面积增加
功耗增加
```

---

## 3.3 提高相联度

主要减少 conflict miss。

缺点：

```text
tag 比较更多
mux 更复杂
hit time 可能增加
功耗增加
```

所以相联度不是越高越好。

---

## 3.4 多级 Cache

L1 小而快，L2/L3 大而慢。

```text
L1 负责快
L2/L3 负责减少访问主存
```

---

## 3.5 Read miss 优先于 write

读 miss 通常会阻塞程序执行，而写可以先进 write buffer。

因此通常：

```text
读优先级 > 写优先级
```

---

## 3.6 避免地址翻译影响 Cache index

常见优化：

```text
Virtually indexed, physically tagged
```

即：

```text
用虚拟地址中的 page offset 做 index
用物理地址 tag 做最终比较
```

好处：

```text
TLB 查询和 Cache index 可以并行
```

代价：

```text
L1 Cache 大小和结构受到 page size 限制
可能有 alias/synonym 问题
```

---

# 4. 十种高级 Cache 性能优化

第二章中重点介绍了十种高级 Cache 优化。

## 4.1 Small and Simple First-Level Caches

小而简单的 L1 Cache 可以降低 hit time 和功耗。

原因是 Cache hit 的关键路径包括：

```text
1. 根据 index 访问 tag array
2. 比较 tag
3. 选择对应 way 的 data
```

如果 L1 太大、相联度太高，会拖慢访问路径。

---

## 4.2 Way Prediction

在 set associative cache 中预测下一次访问会命中哪个 way。

预测正确：

```text
只访问一个 way
接近 direct-mapped cache 的速度和功耗
```

预测错误：

```text
再查其他 way
多付出额外周期
```

---

## 4.3 Pipelined Cache Access

把 Cache 访问流水化。

优点：

```text
提高 Cache 带宽
支持更高主频
```

缺点：

```text
单次 Cache hit latency 变长
load-use delay 增加
分支预测失败代价增加
```

重点是：

```text
Latency 不一定降低，但 throughput 提高。
```

---

## 4.4 Nonblocking Cache

普通 blocking cache：

```text
一个 miss 发生后，整个 cache 阻塞
后续 hit 也不能服务
```

Nonblocking cache：

```text
miss 处理中，仍然允许后续 hit 继续访问
```

常见形式：

```text
Hit under miss
Hit under multiple misses
Miss under miss
```

它的核心收益是隐藏 miss penalty。

---

## 4.5 Multibanked Cache

把 Cache 分成多个 bank。

多个访问如果落到不同 bank，可以并行服务。

优点：

```text
提高带宽
降低单 bank 压力
可能降低功耗
```

缺点：

```text
多个访问落到同一个 bank 时会发生 bank conflict
```

---

## 4.6 Critical Word First 和 Early Restart

CPU 发生 miss 时，通常马上需要的是某个 word，而不是整个 cache block。

### Critical Word First

先取 CPU 当前急需的 word。

```text
先返回关键字 → CPU 恢复执行 → 后台继续填充剩余 block
```

### Early Restart

按正常顺序取 block，但只要目标 word 到了，就让 CPU 先恢复执行。

---

## 4.7 Merging Write Buffer

连续写可以在 write buffer 中合并。

例如连续写：

```text
Mem[100]
Mem[108]
Mem[116]
Mem[124]
```

没有 merging：

```text
占 4 个 write buffer entry
```

有 merging：

```text
合成一个更大的连续写
```

好处：

```text
减少 write buffer 占用
提高写带宽利用率
减少 stall
```

注意：

```text
I/O memory-mapped register 通常不能随便合并写
```

因为写设备寄存器可能有副作用。

---

## 4.8 Compiler Optimizations

编译器也能优化 Cache。

### Loop Interchange

如果二维数组按 row-major 存储，那么应该优先连续访问行内元素。

不友好写法：

```cpp
for (j = 0; j < 100; j++)
    for (i = 0; i < 5000; i++)
        x[i][j] = 2 * x[i][j];
```

更好写法：

```cpp
for (i = 0; i < 5000; i++)
    for (j = 0; j < 100; j++)
        x[i][j] = 2 * x[i][j];
```

---

### Blocking / Tiling

矩阵乘法中，不要一次处理整行或整列，而是处理一个小块。

```text
把大矩阵拆成 B × B 的小块
让小块能留在 Cache 中反复使用
```

收益：

```text
提升时间局部性
提升空间局部性
减少 capacity miss
```

---

## 4.9 Hardware Prefetching

硬件观察访问模式，提前把可能要用的数据取进 Cache。

适合：

```text
顺序访问
规则 stride 访问
数组扫描
指令流预取
```

风险：

```text
预取没用的数据会浪费带宽
预取太多会污染 Cache
预取错误会增加功耗甚至降低性能
```

---

## 4.10 Compiler-Controlled Prefetching

编译器插入 prefetch 指令。

例如：

```cpp
prefetch(a[i + 7]);
```

含义：

```text
现在不需要 a[i+7]
但过几个循环后会用到
所以现在先取进 Cache
```

要求：

```text
Cache 支持 nonblocking
预取距离合适
CPU 能继续执行
```

---

# 5. 存储技术与优化

## 5.1 SRAM

SRAM 常用于 Cache。

特点：

```text
速度快
不需要 refresh
通常 6 个晶体管存 1 bit
面积大
成本高
容量小
```

---

## 5.2 DRAM

DRAM 常用于主存。

特点：

```text
容量大
成本低
速度慢于 SRAM
需要 refresh
通常 1 个晶体管和 1 个电容存 1 bit
```

DRAM 通常使用行列地址：

```text
RAS: Row Address Strobe
CAS: Column Address Strobe
```

---

## 5.3 Access Time 与 Cycle Time

### Access time

从请求发出到数据返回的时间。

### Cycle time

两次无关访问之间的最小间隔。

DRAM 中 cycle time 往往大于 access time。

---

## 5.4 Memory Bandwidth 优化

常见方法：

```text
增加 memory width
memory banking
burst transfer
SDRAM / DDR
```

---

## 5.5 Flash

移动设备常用 Flash 作为最低层存储。

特点：

```text
比 DRAM 慢
比磁盘快
无机械结构
掉电不丢数据
写入和擦除成本高于读取
有擦写寿命问题
```

---

# 6. 虚拟内存与虚拟机

## 6.1 虚拟内存

虚拟内存让每个进程认为自己拥有独立、连续、巨大的地址空间。

```text
Virtual Address → Physical Address
```

好处：

```text
保护
重定位
共享
扩展
简化编程
```

---

## 6.2 Page 和 Page Fault

虚拟内存按 page 管理。

```text
Virtual page number + page offset
```

如果虚拟页不在物理内存中，会发生 page fault。

page fault 的代价远大于 cache miss。

---

## 6.3 TLB

TLB 是地址翻译专用 Cache。

```text
Translation Lookaside Buffer
```

TLB hit：

```text
快速得到物理地址
```

TLB miss：

```text
需要查页表
```

---

## 6.4 虚拟机

虚拟机中地址翻译可能变成：

```text
Guest virtual address
→ Guest physical address
→ Host physical address
```

因此现代处理器通常提供硬件辅助虚拟化和嵌套页表支持。

---

# 7. 第二章常见误区

## 7.1 Miss rate 低不一定性能好

因为真正要看：

```text
AMAT = Hit time + Miss rate × Miss penalty
```

某个优化可能降低 miss rate，但增加 hit time，最终性能变差。

---

## 7.2 Cache 不是越大越好

大 Cache 可能：

```text
降低 capacity miss
```

但也可能：

```text
增加 hit time
增加功耗
增加面积
降低主频
```

---

## 7.3 相联度不是越高越好

高相联度减少 conflict miss，但也会：

```text
增加比较器
增加 mux 复杂度
增加功耗
可能增加 hit time
```

---

## 7.4 预取不是一定提升性能

预取可能：

```text
浪费带宽
污染 Cache
增加功耗
甚至降低性能
```

预取猜得准才有收益。

---

## 7.5 AMAT 不完全等价于程序执行时间

现代 CPU 有：

```text
乱序执行
nonblocking cache
多线程
预取
memory-level parallelism
```

部分 miss penalty 可能被隐藏。

---

# 8. 性能调优：Frontend Bound 35%、Bandwidth 10%、OOO Flush 10%

## 8.1 指标解读

用户给出的指标：

```text
Frontend Bound = 35%
  ├─ Frontend Bandwidth = 10%
  └─ OOO Flush = 10%
```

可以理解为：

```text
Frontend Bound 35%：
    前端供给不足比较明显。

Frontend Bandwidth 10%：
    取指、解码、uop cache 或代码布局可能存在吞吐压力。

OOO Flush 10%：
    流水线发生冲刷，常见原因包括分支误预测、machine clear、异常/assist 等。
```

这类问题通常不是典型 memory bound，而更像：

```text
前端供给不足
分支/流水线冲刷
代码布局或解码压力
```

---

## 8.2 优先排查方向

### 第一优先级：分支误预测

建议查看：

```bash
perf stat -e cycles,instructions,branches,branch-misses ./your_program
```

关注：

```text
branch-misses / branches
```

如果分支误预测率较高，就要重点优化分支。

---

### 第二优先级：machine clear

可以查看：

```bash
perf stat -e machine_clears.count ./your_program
```

如果支持更细粒度事件，还可以看：

```bash
perf stat -e machine_clears.memory_ordering,machine_clears.smc ./your_program
```

常见原因：

```text
memory ordering violation
self-modifying code
异常或 assist
split lock
unaligned atomic
FP denormal
```

---

### 第三优先级：前端带宽压力

建议定位热点：

```bash
perf record -g --call-graph dwarf ./your_program
perf report
perf annotate
```

重点看：

```text
热点函数是否太大
热点循环是否太大
是否有大量 call
是否有复杂指令
是否有大量分支
是否 inline 过度导致代码膨胀
是否模板展开太多
```

---

## 8.3 调优建议

### 8.3.1 使用 PGO

PGO 可以帮助编译器做：

```text
分支概率判断
代码布局优化
函数布局优化
热冷代码分离
间接调用优化
inline 决策优化
```

GCC 示例：

```bash
g++ -O3 -fprofile-generate ...
./your_program_with_real_workload
g++ -O3 -fprofile-use ...
```

PGO 对 frontend bound、分支预测、代码布局问题通常都比较有效。

---

### 8.3.2 热路径 fall-through

把常见路径写成自然顺序。

较差写法：

```cpp
if (error) {
    slow_path();
} else {
    fast_path();
}
```

更推荐：

```cpp
if (unlikely(error)) {
    slow_path();
    return;
}

fast_path();
```

核心思想：

```text
少见情况提前返回
常见路径直线往下走
```

---

### 8.3.3 likely / unlikely

GCC/Clang：

```cpp
if (__builtin_expect(error, 0)) {
    slow_path();
}
```

C++20：

```cpp
if (error) [[unlikely]] {
    slow_path();
}
```

注意：

```text
只适合概率非常明确的分支
不要乱加
加错可能反而变差
```

---

### 8.3.4 热冷代码分离

把错误处理、日志、慢速 fallback、统计上报从热点函数中拆出去：

```cpp
if (unlikely(error)) {
    return handle_error_slow(...);
}
```

慢路径标记：

```cpp
__attribute__((noinline, cold))
int handle_error_slow(...) {
    ...
}
```

好处：

```text
缩小热代码体积
改善 I-cache
改善 uop cache
改善分支布局
降低 frontend bound
```

---

### 8.3.5 减少间接分支

热点路径中如果有大量：

```cpp
virtual_call();
function_pointer();
std::function();
switch 大跳转表;
```

可能导致 BTB 或间接分支预测压力。

可以考虑：

```text
虚函数去虚化
模板化
CRTP
函数指针改成枚举 + inline fast path
std::function 换成直接 callable 或模板参数
```

---

### 8.3.6 控制 inline

inline 不是越多越好。

inline 的收益：

```text
减少 call/ret
暴露优化机会
```

inline 的代价：

```text
代码体积膨胀
I-cache 压力变大
uop cache 压力变大
前端 bandwidth 变差
```

建议：

```text
热路径小函数 inline
冷路径 noinline
错误处理 noinline
慢路径 noinline
大函数谨慎 inline
```

---

### 8.3.7 减少热点路径指令数

前端带宽压力往往意味着：

```text
CPU 后端还能干，但前端喂不饱。
```

优化方向：

```text
把不必要的检查移出循环
把日志、统计、trace 移出热路径
减少抽象层穿透成本
避免热路径使用过重组件
减少小函数层层调用
```

例如：

```cpp
for (...) {
    if (config.enable_check) {
        check(x);
    }
    hot_compute(x);
}
```

如果 `config.enable_check` 运行期间不变，可以改成：

```cpp
if (config.enable_check) {
    for (...) {
        check(x);
        hot_compute(x);
    }
} else {
    for (...) {
        hot_compute(x);
    }
}
```

---

### 8.3.8 使用 LTO / BOLT

基础选项：

```bash
-O3 -march=native -flto
```

如果是大 C++ 服务，可以考虑 BOLT 做二进制重排。

BOLT 可能改善：

```text
函数布局
基本块布局
I-cache 命中
uop cache 命中
frontend bound
```

---

# 9. CPU 分支误预测

## 9.1 什么是分支误预测？

CPU 的分支误预测，简单说就是：

```text
CPU 遇到 if/else、循环、函数跳转时，会提前猜程序接下来会走哪条路；
如果猜错了，就叫分支误预测。
```

例如：

```cpp
if (x > 0) {
    do_A();
} else {
    do_B();
}
```

CPU 在 `x > 0` 的结果真正出来之前，可能会先猜测走 `do_A()`，并提前取指令、解码甚至执行。

如果实际也走 `do_A()`：

```text
预测正确
流水线不中断
```

如果实际应该走 `do_B()`：

```text
预测错误
错误路径上的工作要丢掉
流水线要被清空
重新从正确路径执行
```

---

## 9.2 为什么 CPU 要预测分支？

现代 CPU 是流水线执行：

```text
取指令 → 解码 → 执行 → 访存 → 写回
```

如果遇到分支就停下来等待结果，流水线会空转，性能很差。

所以 CPU 会预测分支方向，提前执行后续指令。

---

## 9.3 分支误预测的代价

分支误预测后，CPU 需要：

```text
1. 丢掉已经取出的错误路径指令
2. 清空部分流水线
3. 回到正确地址
4. 重新取正确路径的指令
```

这个过程叫：

```text
pipeline flush
流水线冲刷
```

也就是性能指标中可能看到的：

```text
OOO flush
```

现代 CPU 流水线较深，一次误预测可能损失十几个甚至几十个周期。

---

## 9.4 哪些代码容易分支误预测？

### 随机条件

```cpp
if (rand() % 2) {
    x++;
} else {
    y++;
}
```

接近 50/50，CPU 很难预测。

---

### 数据分布不稳定

```cpp
if (price > threshold) {
    buy();
} else {
    sell();
}
```

如果条件一会儿真一会儿假，没有稳定规律，就容易误预测。

---

### 虚函数 / 函数指针

```cpp
obj->handle();
```

如果 `obj` 的真实类型变化很多，CPU 很难预测跳转目标。

---

### switch 分发

```cpp
switch (opcode) {
    case A: ...
    case B: ...
    case C: ...
}
```

解释器、协议解析、行情处理、状态机里很常见。

---

### 链式 if-else

```cpp
if (type == A) {
    ...
} else if (type == B) {
    ...
} else if (type == C) {
    ...
} else {
    ...
}
```

如果类型分布复杂，预测压力会比较大。

---

## 9.5 如何判断是否存在分支误预测？

使用 perf：

```bash
perf stat -e cycles,instructions,branches,branch-misses ./your_program
```

重点看：

```text
branch-misses / branches
```

例如：

```text
branches:        1,000,000,000
branch-misses:      50,000,000
```

误预测率：

```text
50,000,000 / 1,000,000,000 = 5%
```

如果热点程序中分支误预测率达到 3% 到 5% 以上，就值得关注。

---

## 9.6 如何优化分支误预测？

### 让热路径更明显

推荐：

```cpp
if (unlikely(error)) {
    return slow_path();
}

do_fast();
```

核心：

```text
少见情况提前返回
常见路径直线往下走
```

---

### 使用 likely / unlikely

GCC/Clang：

```cpp
if (__builtin_expect(error, 0)) {
    slow_path();
}
```

C++20：

```cpp
if (error) [[unlikely]] {
    slow_path();
}
```

---

### 热冷代码分离

```cpp
if (unlikely(error)) {
    return handle_error_slow();
}
```

```cpp
__attribute__((noinline, cold))
int handle_error_slow() {
    ...
}
```

---

### 用查表代替复杂分支

例如：

```cpp
if (x == 0) return 10;
else if (x == 1) return 20;
else if (x == 2) return 30;
```

可以改成：

```cpp
static int table[] = {10, 20, 30};
return table[x];
```

但查表会引入内存访问，不一定总更快，需要实测。

---

### 分支改成无分支

例如：

```cpp
if (x > 0)
    y = a;
else
    y = b;
```

可能改成：

```cpp
y = (x > 0) ? a : b;
```

编译器有时会生成 `cmov`，避免跳转。

注意：

```text
分支很好预测时，branch 可能更快
分支很难预测时，cmov 可能更快
```

---

# 10. 对当前性能问题的最终建议

结合这轮对话中的指标：

```text
Frontend Bound = 35%
Frontend Bandwidth = 10%
OOO Flush = 10%
```

优先建议如下：

```text
1. 先查 branch-misses / branches
2. 再查 machine_clears.count
3. 用 perf annotate 或 VTune 定位热点函数和汇编
4. 上 PGO，改善代码布局和分支概率
5. 拆冷路径，缩小热路径代码体积
6. 减少虚函数、函数指针、std::function、复杂 switch
7. 控制 inline，避免代码膨胀
8. 减少热点循环中的无关检查、日志、统计
9. 必要时使用 LTO 或 BOLT
10. 所有修改都要基于真实 workload 重新测量
```

这类问题的核心不是单纯 memory bound，而更像：

```text
前端供给不足
分支误预测或流水线冲刷
代码布局不佳
指令/uop 数量偏多
```

最终目标是：

```text
让 CPU 前端更稳定地取到正确指令
减少错误路径执行
减少流水线 flush
缩小热代码体积
提高有效 IPC
```

---

# 11. 一句话总结

本轮对话可以浓缩成一句话：

> 第二章讲的是如何用存储层次结构缓解 CPU 与内存之间的速度鸿沟；而你遇到的 frontend bound 和 ooo flush 问题，说明当前性能瓶颈更偏向 CPU 前端供给、分支预测、代码布局和流水线冲刷，调优时应优先从分支误预测、热冷路径分离、PGO/LTO/BOLT、减少间接调用和缩小热路径代码体积入手。
