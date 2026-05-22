---
title: "深度学习编译器中的算子调度与性能建模"
description: "从算子、调度源语、调度方案搜索和性能模型几个角度，梳理深度学习编译器如何把计算映射到目标硬件。"
date: 2026-05-22T00:00:00+08:00
slug: "deep-learning-compiler-operator-schedule-performance-model"
categories:
    - "随笔"
tags:
    - "深度学习编译"
    - "算子调度"
    - "性能建模"
    - "MLC"
draft: false
---

这篇随笔想梳理一个很容易被忽略的问题：深度学习编译器并不只是做模型格式转换，它更核心的工作是把模型里的计算映射到具体硬件上，并尽量生成高效的执行方式。

如果说算子定义了“算什么”，那么调度决定“怎么计算”，性能模型则帮助编译器判断“哪种计算方式更值得选”。

## 1. 背景：为什么需要算子调度与优化？

在深度学习模型部署中，模型通常由大量算子组成，例如：

- Conv2D
- MatMul / GEMM
- BatchNorm
- ReLU
- Softmax
- LayerNorm
- Reshape
- Transpose
- Add
- Mul

从数学角度看，算子定义的是“计算什么”；但从工程实现角度看，仅仅知道计算公式是不够的。真正影响部署性能的是：

> 这个算子在目标硬件上应该如何执行。

例如，同样是矩阵乘法：

```text
C = A × B
```

在不同硬件上，其高性能实现方式完全不同：

- 在 CPU 上，需要考虑 cache、SIMD、多线程；
- 在 GPU 上，需要考虑 thread block、warp、shared memory、tensor core；
- 在嵌入式 MCU 上，需要考虑 SRAM、Flash、DMA、int8 kernel；
- 在 NPU/AI 加速器上，需要考虑算子支持范围、片上 buffer、数据布局和算子融合。

因此，深度学习编译器的核心任务不只是把模型从一种格式转换成另一种格式，而是要将计算图映射到目标硬件上，并生成高效执行代码。

---

## 2. 什么是算子？

算子可以理解为神经网络计算图中的基本计算单元。

例如：

```text
Conv2D
MatMul
ReLU
Softmax
LayerNorm
Add
Reshape
```

它们回答的是：

> 模型要算什么？

例如 Conv2D 的数学含义是卷积，MatMul 的数学含义是矩阵乘法，ReLU 的数学含义是非线性激活。

但是，算子本身并不规定它应该如何高效运行。

例如一个矩阵乘法可以有很多种实现方式：

```c
for (int i = 0; i < M; i++) {
    for (int j = 0; j < N; j++) {
        for (int k = 0; k < K; k++) {
            C[i][j] += A[i][k] * B[k][j];
        }
    }
}
```

这段代码能算对，但不一定快。

真正的高性能实现需要考虑：

- 循环顺序
- 分块大小
- 数据复用
- cache 命中率
- SIMD 向量化
- 多线程并行
- 中间结果是否写回内存
- 数据布局是否适合硬件

这些都属于“调度”的范畴。

---

## 3. 什么是调度？

算子描述的是：

> 算什么？

调度描述的是：

> 怎么算才快？

例如，对于矩阵乘法：

```text
C[M, N] = A[M, K] × B[K, N]
```

原始计算可以写成三层循环：

```c
for i in range(M):
    for j in range(N):
        for k in range(K):
            C[i][j] += A[i][k] * B[k][j]
```

但是这个循环有很多种优化方式：

- 可以分块计算；
- 可以调整循环顺序；
- 可以把数据放入 cache；
- 可以使用 SIMD 一次计算多个元素；
- 可以展开循环；
- 可以使用多线程并行；
- 可以融合前后算子，减少中间结果写回。

这些优化方式共同构成了调度。

---

## 4. 什么是调度源语？

调度源语，也叫 scheduling primitive，是深度学习编译器用于改变算子执行方式的基础操作。

它不是模型中的算子，而是编译器优化算子实现时使用的基本变换工具。

常见调度源语包括：

| 调度源语 | 作用 | 直观理解 |
|---|---|---|
| split | 拆分循环 | 把一个大循环拆成外层循环和内层循环 |
| tile | 分块计算 | 把大矩阵或大特征图切成小块计算 |
| reorder | 调整循环顺序 | 改变 i、j、k 等循环的嵌套顺序 |
| fuse | 融合循环 | 把多个循环合并成一个循环 |
| vectorize | 向量化 | 使用 SIMD 指令一次处理多个数据 |
| unroll | 循环展开 | 减少循环控制开销，提高指令级并行 |
| parallel | 并行化 | 使用多线程或多核并行执行 |
| cache_read | 缓存读取 | 将输入数据放入更快的局部存储 |
| cache_write | 缓存写入 | 将输出或中间结果放入局部缓存 |
| compute_at | 控制计算位置 | 决定某个中间结果在哪一层循环中计算 |
| inline | 内联计算 | 将简单算子直接嵌入到后续计算中 |

这些源语可以组合成不同的调度方案。

---

## 5. 调度源语示例：矩阵乘法优化

### 5.1 原始矩阵乘法

假设有如下矩阵乘法：

```text
C = A × B
```

朴素实现如下：

```c
for (int i = 0; i < M; i++) {
    for (int j = 0; j < N; j++) {
        for (int k = 0; k < K; k++) {
            C[i][j] += A[i][k] * B[k][j];
        }
    }
}
```

这个实现的问题是：

- cache 利用率可能较低；
- 数据复用不充分；
- 没有 SIMD 向量化；
- 没有多线程；
- 循环控制开销较大。

---

### 5.2 split / tile：分块计算

分块的思想是：

> 不要一次处理整个大矩阵，而是把矩阵切成小块，一块一块计算。

例如把 `i` 和 `j` 方向按照 32 进行分块：

```c
for (int io = 0; io < M; io += 32) {
    for (int jo = 0; jo < N; jo += 32) {
        for (int i = io; i < io + 32; i++) {
            for (int j = jo; j < jo + 32; j++) {
                for (int k = 0; k < K; k++) {
                    C[i][j] += A[i][k] * B[k][j];
                }
            }
        }
    }
}
```

这样做的好处是：

- 小块数据更容易留在 cache 中；
- 减少对外部内存的重复访问；
- 提高数据复用率；
- 为 SIMD 和并行化创造条件。

这类优化在 CPU、GPU 和 NPU 编译器中都非常常见。

---

### 5.3 reorder：调整循环顺序

循环顺序会影响数据访问的连续性。

例如原始顺序是：

```c
for i:
    for j:
        for k:
```

可以调整为：

```c
for i:
    for k:
        for j:
```

在某些数据布局下，后一种方式可以让内存访问更加连续，更适合 cache 和 SIMD。

因此，reorder 的本质是：

> 通过调整循环嵌套顺序，让数据访问模式更适合目标硬件。

---

### 5.4 vectorize：向量化

普通标量计算一次只处理一个元素：

```c
C[i][j] += A[i][k] * B[k][j];
```

向量化后可以一次处理多个元素：

```c
C[i][j : j+8] += A[i][k] * B[k][j : j+8];
```

在 CPU 上，这通常对应 SIMD 指令，例如：

- ARM NEON
- x86 SSE / AVX / AVX2 / AVX-512
- RISC-V Vector Extension

向量化的目标是：

> 用一条指令处理多个数据，提高计算吞吐。

---

### 5.5 unroll：循环展开

循环展开是把循环体重复展开，减少循环控制开销。

例如原始代码：

```c
for (int k = 0; k < K; k++) {
    sum += A[k] * B[k];
}
```

展开后：

```c
for (int k = 0; k < K; k += 4) {
    sum += A[k]     * B[k];
    sum += A[k + 1] * B[k + 1];
    sum += A[k + 2] * B[k + 2];
    sum += A[k + 3] * B[k + 3];
}
```

这样做可以：

- 减少循环判断和跳转；
- 增加指令级并行；
- 帮助编译器生成更好的 SIMD 代码。

---

### 5.6 parallel：并行化

并行化是把不同数据块分配给多个线程或多个计算单元。

例如：

```c
parallel_for (int i = 0; i < M; i++) {
    for (int j = 0; j < N; j++) {
        for (int k = 0; k < K; k++) {
            C[i][j] += A[i][k] * B[k][j];
        }
    }
}
```

在 CPU 上，这可能对应多线程；

在 GPU 上，这可能对应大量 thread block；

在 NPU 上，这可能对应多个计算阵列或多个执行单元。

并行化的目标是：

> 让多个计算资源同时工作，提高整体吞吐。

---

## 6. 什么是调度方案？

一个调度方案通常是多个调度源语的组合。

例如，对于矩阵乘法，一个调度方案可能是：

```text
1. split i, j, k
2. reorder io, jo, ko, ii, jj, kk
3. cache_read A
4. cache_read B
5. vectorize jj
6. unroll kk
7. parallel io
```

不同调度方案的性能可能差异很大。

例如：

```text
方案 A：tile = 16, vectorize = 4, unroll = 2
方案 B：tile = 32, vectorize = 8, unroll = 4
方案 C：tile = 64, vectorize = 16, unroll = 8
```

哪个方案最快，取决于目标硬件。

因此，深度学习编译器需要搜索或选择合适的调度方案。

---

## 7. 为什么需要搜索调度方案？

调度空间通常非常大。

以矩阵乘法为例，可以选择的参数包括：

- tile size
- 循环顺序
- SIMD 宽度
- unroll 因子
- 是否 parallel
- 是否 cache_read
- 是否 cache_write
- 中间结果计算位置
- 数据布局

这些组合起来可能有成千上万种候选方案。

人工逐个尝试不现实，所以工业界通常使用自动调度或半自动调度方法。

相关系统和工具链包括：

- TVM AutoTVM / Ansor
- Halide Auto-scheduler
- XLA
- MLIR-based compiler
- TensorRT
- TFLite Delegate
- OpenVINO
- Triton autotune

这些系统并不都以同一种方式做自动调度，但它们都需要在不同层面回答类似问题：

> 在给定硬件上，为某个算子或计算图找到较优的执行方式。

---

## 8. 什么是性能建模？

性能建模是指：

> 估计某个算子在某个硬件、某种 shape、某种数据布局、某种调度方案下的执行代价。

它不是简单判断：

```text
Conv 快不快？
MatMul 快不快？
```

而是要判断：

```text
这个 Conv2D，输入 shape 是多少？
数据类型是什么？
数据布局是什么？
tile size 是多少？
是否能放进 cache？
是否能 SIMD？
是否需要额外 layout 转换？
是否需要中间结果写回？
```

也就是说，性能建模关注的是：

```text
算子 + shape + dtype + layout + schedule + hardware
```

共同决定的执行效率。

---

## 9. 性能建模不能只看计算量

一个常见误区是只看 FLOPs 或 MACs。

例如：

```text
算子执行时间 ≈ 计算量 / 硬件峰值算力
```

这个估计通常过于理想化。

真实执行时间还受到很多因素影响：

- 内存带宽
- cache 命中率
- 数据复用率
- SIMD 利用率
- 并行度
- kernel launch overhead
- 数据布局转换
- 算子融合情况
- 中间结果读写
- 硬件特殊指令支持

因此，一个算子不一定是计算瓶颈，也可能是访存瓶颈。

---

## 10. Roofline Model：计算受限与访存受限

工业界常用 Roofline Model 来理解算子性能。

其核心思想是分析算子的算术强度：

```text
Arithmetic Intensity = 计算量 / 数据搬运量
```

如果算术强度高，说明每搬运一次数据可以做很多计算，这类算子更可能是计算受限：

```text
compute-bound
```

如果算术强度低，说明数据搬运很多但计算很少，这类算子更可能是访存受限：

```text
memory-bound
```

---

## 11. 示例：MatMul 通常更容易吃满算力

矩阵乘法：

```text
C[M, N] = A[M, K] × B[K, N]
```

计算量约为：

```text
M × N × K 次 MAC
```

如果按 FLOPs 统计，通常会写成约 `2 × M × N × K`。

数据量约为：

```text
A: M × K
B: K × N
C: M × N
```

MatMul 的特点是：

- 计算量大；
- 数据可以高度复用；
- A 和 B 的元素会被多次使用；
- 适合分块、向量化和并行化。

因此，大规模 MatMul 通常更容易成为 compute-bound 算子，也更容易利用硬件的峰值算力。

这也是为什么 GPU、NPU 和各种深度学习加速器都重点优化 GEMM/MatMul。

---

## 12. 示例：ReLU / Add 通常是访存受限

ReLU 的计算非常简单：

```text
y = max(x, 0)
```

每个元素只需要一次比较，但需要：

- 读输入；
- 写输出。

它的计算量很少，但数据搬运不可避免。

因此，ReLU 通常是 memory-bound。

对于这类算子，单独优化计算指令意义不大，更常见的优化方式是算子融合。

例如：

```text
Conv + BatchNorm + ReLU
```

融合后可以避免中间结果反复写回内存、再从内存读出。

---

## 13. 算子融合为什么重要？

算子融合的目标是减少中间结果的读写。

例如原始计算图：

```text
Conv → BatchNorm → ReLU
```

如果不融合，执行过程可能是：

```text
Conv 输出写回内存
BatchNorm 从内存读 Conv 输出
BatchNorm 输出写回内存
ReLU 从内存读 BatchNorm 输出
ReLU 输出写回内存
```

如果融合为一个 kernel：

```text
Conv + BatchNorm + ReLU
```

则可以在一次计算过程中完成：

```text
Conv 计算
直接加 BN 参数
直接做 ReLU
最终结果写回一次
```

这样可以显著减少访存开销。

因此，对于 memory-bound 算子，算子融合通常比单独优化计算更重要。

---

## 14. 性能建模常见方法

### 14.1 手工规则模型

这是工程中最常见、最容易落地的方法。

编译器内置一些规则，例如：

```text
大规模 Conv2D 使用加速器 kernel
小规模算子走 CPU
ReLU/Add 尽量融合
Reshape/Transpose 尽量消除或下推
MatMul 根据 shape 选择不同 kernel
动态 shape 回退到通用实现
```

优点：

- 实现简单；
- 可控性强；
- 适合产品化部署；
- 便于针对特定硬件调优。

缺点：

- 泛化能力有限；
- 规则维护成本较高；
- 对新模型和新 shape 适应性较弱。

---

### 14.2 解析代价模型

解析代价模型会估计计算时间和访存时间。

常见形式：

```text
Compute_Time = Ops / Effective_Compute_Throughput
Memory_Time = Bytes / Effective_Memory_Bandwidth
```

整体时间可以粗略估计为：

```text
Time ≈ max(Compute_Time, Memory_Time) + Overhead
```

例如某个卷积算子：

```text
计算量 = 200M MACs
数据搬运量 = 20MB
有效算力 = 50GMAC/s
有效带宽 = 2GB/s
```

则：

```text
Compute_Time = 200M / 50G = 4ms
Memory_Time = 20MB / 2GB/s = 10ms
```

此时更可能是访存瓶颈，优化重点应该放在：

- 减少访存；
- 提高数据复用；
- 做算子融合；
- 改善数据布局；
- 减少中间结果写回。

---

### 14.3 实测 + 自动搜索

更复杂的系统会自动生成多个候选 kernel，然后在目标硬件上实际运行测试。

流程如下：

```text
算子定义
  ↓
生成候选调度方案
  ↓
编译成目标代码
  ↓
在目标硬件上运行
  ↓
记录执行时间
  ↓
更新代价模型
  ↓
继续搜索更优方案
```

这种方法的代表包括：

- TVM AutoTVM
- TVM Ansor
- Halide Auto-scheduler
- Triton autotune

优点：

- 可以找到比人工规则更优的实现；
- 能适应不同硬件；
- 对复杂调度空间更有效。

缺点：

- 搜索成本高；
- 系统实现复杂；
- 需要目标硬件反馈；
- 编译时间可能较长。

---

## 15. 编译器如何使用性能模型？

深度学习编译器通常会按照如下流程优化模型：

```text
读取模型计算图
  ↓
进行图优化
  ↓
分析算子类型、shape、dtype、layout
  ↓
判断目标硬件是否支持
  ↓
估计不同执行方案的代价
  ↓
选择算子实现或调度方案
  ↓
进行算子融合和 layout 转换
  ↓
生成目标代码
  ↓
部署运行
```

其中性能模型主要用于回答这些问题：

```text
这个算子放在哪里执行？
这个 shape 使用哪个 kernel？
这个算子是否值得融合？
这个 layout 是否需要转换？
这个调度方案是否可能更快？
这个算子是否应该回退到 CPU？
```

---

## 16. 工业界主流编译器关注的优化方向

工业界深度学习编译器通常关注以下几类优化。

### 16.1 计算图优化

包括：

- 常量折叠
- 死代码消除
- 算子融合
- 算子替换
- 公共子表达式消除
- 冗余 Transpose 消除
- layout 传播
- shape 推导

例如：

```text
Conv + BatchNorm → Conv
Conv + BatchNorm + ReLU → FusedConv
```

---

### 16.2 算子级优化

包括：

- 分块
- 向量化
- 循环展开
- 并行化
- cache 优化
- 数据预取
- kernel 选择
- 算子自动调度

这一层主要关注单个算子如何跑得更快。

---

### 16.3 内存优化

包括：

- 内存复用
- 中间 buffer 复用
- 原地计算
- 减少中间结果写回
- 减少 layout conversion
- 静态内存规划
- 动态内存峰值分析

对于端侧部署，内存优化尤其重要。

---

### 16.4 数据布局优化

常见数据布局包括：

```text
NCHW
NHWC
NC4HW4
NCHWc
Blocked Layout
```

不同硬件适合不同的数据布局。

例如：

- GPU 可能偏好某些 channel blocking；
- CPU SIMD 可能偏好通道按向量宽度对齐；
- 移动端推理框架常用 NHWC；
- 某些后端会使用内部 blocked layout。

layout 优化的目标是：

> 让数据排列方式更适合硬件访问和并行计算。

---

### 16.5 量化优化

在部署中，量化也是重要优化方向。

常见量化形式包括：

- FP32
- FP16
- BF16
- INT8
- INT4

对于 INT8 推理，常见计算路径是：

```text
INT8 × INT8 → INT32 accumulation → requant → INT8
```

量化优化需要考虑：

- scale 和 zero-point；
- per-tensor / per-channel 量化；
- 对称 / 非对称量化；
- bias 的 int32 表示；
- requant 是否高效；
- 量化算子是否能融合；
- 精度损失是否可接受。

---

## 17. 一个完整例子：Conv + BN + ReLU 的优化过程

假设模型中有如下结构：

```text
Conv2D → BatchNorm → ReLU
```

### 17.1 原始执行方式

原始情况下，三个算子分别执行：

```text
Conv2D 输出中间结果
BatchNorm 读取 Conv2D 输出并生成新结果
ReLU 读取 BatchNorm 输出并生成最终结果
```

问题是：

- 中间结果多次读写内存；
- kernel 启动次数多；
- cache 利用率低；
- ReLU 本身计算很轻，单独执行不划算。

---

### 17.2 图优化：BN Folding

在推理阶段，BatchNorm 可以折叠进 Conv 权重和 bias 中。

原始形式：

```text
y = BN(Conv(x, W, b))
```

可以转换为：

```text
y = Conv(x, W', b')
```

这样 BatchNorm 算子就被消除了。

---

### 17.3 算子融合：Conv + ReLU

折叠 BN 后，计算图变成：

```text
Conv2D → ReLU
```

进一步可以融合为：

```text
FusedConv2DRelu
```

这样 Conv 输出不需要先写回内存再给 ReLU 读取，而是在 Conv 计算完成后直接进行 ReLU。

---

### 17.4 调度优化

对于融合后的 Conv kernel，编译器还可以继续做：

- tile；
- reorder；
- vectorize；
- unroll；
- parallel；
- cache_read；
- cache_write；
- 数据布局变换。

最终目标是：

```text
减少访存
提高并行度
提高 SIMD 利用率
提高 cache 命中率
减少中间结果写回
```

---

## 18. 总结

深度学习编译器中的算子优化可以概括为三层问题：

### 第一层：算子语义

回答：

```text
算什么？
```

例如：

```text
Conv2D
MatMul
Softmax
LayerNorm
ReLU
```

---

### 第二层：调度优化

回答：

```text
怎么计算才快？
```

例如：

```text
split
tile
reorder
vectorize
unroll
parallel
cache_read
cache_write
compute_at
```

---

### 第三层：性能建模

回答：

```text
这个执行方案在目标硬件上是否高效？
```

需要考虑：

```text
计算量
访存量
数据复用
cache 命中
SIMD 利用率
并行度
数据布局
算子融合
中间结果写回
硬件支持范围
```

---

## 19. 核心结论

深度学习编译器不是简单地把模型从 ONNX、TensorFlow 或 PyTorch 转成目标代码，而是要完成：

```text
计算图优化
算子选择
调度搜索
性能建模
内存规划
layout 优化
代码生成
```

其中，调度源语是算子优化的基础工具，性能模型是选择优化方案的重要依据。

可以用一句话总结：

> 算子决定“算什么”，调度决定“怎么计算”，性能模型决定“哪种计算方式在目标硬件上更快”。
