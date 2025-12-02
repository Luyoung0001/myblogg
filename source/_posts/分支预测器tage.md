---
title: riscv-boom's tage
date: 2025-12-02 15:03:15
tags: 分支预测
categories: 计算机体系结构
mermaid: true
---

TAGE is the most advanced branch predictor in BOOM. It uses multiple prediction tables indexed by different history lengths to achieve high accuracy.

Key Concepts:
  - Multiple tables with geometrically increasing history lengths
    (e.g., 2, 4, 8, 16, 32, 64 bits of history)
  - Each table has: tag, 3-bit counter (ctr), 2-bit usefulness (u)
  - Provider selection: longest matching history table wins
  - Alternate prediction: second-longest matching table
  - When provider counter is weak (3 or 4), use alternate prediction

Update Policy:
  - On correct prediction: increment counter, maybe increment u
  - On misprediction: decrement counter, try to allocate new entry
  - Periodically decay u-bits to allow entry replacement

## TAGE (Tagged Geometric History Length)

## 功能定位
- **最准确的方向预测器**
- 使用全局历史
- 多个表，不同历史长度
- BOOM的核心预测器

## 核心思想

**问题**：不同分支需要不同长度的历史

```
例子1：if (i % 4 == 0) ...
       需要2-bit历史

例子2：if (pattern_in_100_iterations) ...
       需要100-bit历史
```

**TAGE的解决方案**：维护多个表，每个表使用不同长度的历史

## 表结构

**默认6个表**（tage.scala:198-203）：

| 表# | 行数 | 历史长度 | 标签位数 |
|-----|------|----------|----------|
| T0 | 128 | 2 bits | 7 bits |
| T1 | 128 | 4 bits | 7 bits |
| T2 | 256 | 8 bits | 8 bits |
| T3 | 256 | 16 bits | 8 bits |
| T4 | 128 | 32 bits | 9 bits |
| T5 | 128 | 64 bits | 9 bits |

**每个表项**（tage.scala:78-82）：

```scala
class TageEntry extends Bundle {
  val valid = Bool()           // 有效位
  val tag   = UInt(tagSz.W)   // 标签
  val ctr   = UInt(3.W)       // 3-bit预测计数器（0-7）
  val u     = UInt(2.W)       // 2-bit useful计数器
}
```

## 索引和标签计算

**Folded History**（tage.scala:50-56）：

```scala
// 将长历史折叠到短宽度
def compute_folded_hist(hist: UInt, l: Int) = {
  val nChunks = (histLength + l - 1) / l
  val chunks = (0 until nChunks).map { i =>
    hist(min((i+1)*l, histLength)-1, i*l)
  }
  chunks.reduce(_^_)  // XOR所有块
}
```

**例子**：
```
历史 = 64 bits: [H63..H0]
目标长度 = 8 bits

Chunk 0: H[7:0]
Chunk 1: H[15:8]
Chunk 2: H[23:16]
...
Chunk 7: H[63:56]

Folded = H[7:0] ^ H[15:8] ^ ... ^ H[63:56]
```

**索引计算**（tage.scala:58-63）：

```scala
val idx_history = compute_folded_hist(hist, log2Ceil(nRows))
val idx = (PC ^ idx_history)(log2Ceil(nRows)-1, 0)

val tag_history = compute_folded_hist(hist, tagSz)
val tag = ((PC >> log2Ceil(nRows)) ^ tag_history)(tagSz-1, 0)
```

## 预测过程

**Step 1: 查询所有表**（tage.scala:245）

```scala
val f2_resps = VecInit(tables.map(_.io.f2_resp))
```

**Step 2: 找到Provider和Alternate**（tage.scala:266-282）

```scala
var s2_provided = false.B      // 是否有表命中
var s2_provider = 0.U          // 命中的表（最长历史）
var s2_alt_provided = false.B  // 是否有次长表命中
var s2_alt_provider = 0.U      // 次长表索引

for (i <- 0 until nTables) {
  val hit = f2_resps(i)(w).valid

  // 如果有表命中且当前已有provider，那当前provider变成alternate
  s2_alt_provided = s2_alt_provided || (s2_provided && hit)
  s2_alt_provider = Mux(hit, s2_provider, s2_alt_provider)

  // 更新provider（总是选最长历史的命中表）
  s2_provided = s2_provided || hit
  s2_provider = Mux(hit, i.U, s2_provider)
}
```

**Step 3: 生成预测**（tage.scala:287-292）

```scala
io.resp.f3(w).taken := Mux(s3_provided,
  // 有provider
  Mux(prov.ctr === 3.U || prov.ctr === 4.U,  // 弱预测？
    // 弱预测：使用alternate或base predictor
    Mux(s3_alt_provided, alt.ctr(2), base_pred),
    // 强预测：使用provider的ctr[2]
    prov.ctr(2)
  ),
  // 无provider：使用base predictor
  base_pred
)
```

**预测逻辑图解**：

```
       是否有Provider命中？
         /          \
       Yes          No
        |            |
     ctr=3或4?   使用Base预测
     (弱预测)
      /    \
    Yes    No
     |      |
  有Alt?  使用ctr[2]
   /  \
 Yes  No
  |    |
 Alt  Base
```

## 更新和分配

**正常更新**（tage.scala:319-335）：

```scala
when (分支指令 && commit更新) {
  when (provider存在) {
    // 1. 更新预测计数器
    update_ctr := inc_ctr(old_ctr, actual_taken)

    // 2. 更新useful位
    val new_u = inc_u(old_u, alt_differs, mispredicted)
    update_u := new_u
  }
}
```

**Useful位的更新规则**（tage.scala:227-230）：

```scala
def inc_u(u: UInt, alt_differs: Bool, mispredict: Bool): UInt = {
  Mux(!alt_differs, u,  // alt和provider预测相同，保持u
    Mux(mispredict,     // 预测错误
      Mux(u === 0.U, 0.U, u - 1.U),  // u--（最小0）
      Mux(u === 3.U, 3.U, u + 1.U)   // u++（最大3）
    )
  )
}
```

U-bit 不直接衡量一个专家“预测得有多准”。它衡量的是：“当这位专家的预测，与它的备用专家意见不合时，他的预测到底有多可靠？”

**分配策略**（预测错误时）（tage.scala:301-314）：

```scala
// 1. 找到可分配的表
val allocatable = VecInit(f3_resps.map { r =>
  !r.valid &&          // 未命中
  r.u === 0.U &&       // useful为0
  (i > provider)       // 历史长度比provider长
})

// 2. 随机化选择（避免总是选同一个）
val lfsr = random.LFSR(nTables)
val masked_entry = PriorityEncoder(allocatable & lfsr)
val first_entry = PriorityEncoder(allocatable)

val alloc_entry = Mux(allocatable(masked_entry),
  masked_entry,  // 优先随机选择
  first_entry    // 否则选第一个
)
```

**分配时的操作**（tage.scala:340-347）：

```scala
when (allocate.valid) {
  // 1. 写入新entry
  update_mask(allocate)(idx) := true.B
  update_alloc(allocate)(idx) := true.B

  // 2. 设置初始ctr
  update_ctr := Mux(taken, 4.U, 3.U)  // 弱taken或弱not-taken

  // 3. useful设为0
  update_u := 0.U
} .otherwise {
  // 分配失败：降低其他表的useful
  for (i <- 未命中的表) {
    update_u(i) := 0.U
  }
}
```

## Useful位的周期性清零

**问题**：useful位会逐渐饱和，阻止新分配

**解决**：周期性清零（tage.scala:109-116）

```scala
val clear_u_ctr = RegInit(0.U)
clear_u_ctr := clear_u_ctr + 1.U

val doing_clear_u = clear_u_ctr(log2Ceil(uBitPeriod)-1, 0) === 0.U
val clear_u_idx = clear_u_ctr >> log2Ceil(uBitPeriod)

// 每2048次更新，清零一个set的useful位
when (doing_clear_u) {
  for (w <- 0 until bankWidth) {
    u_mem.write(clear_u_idx, 0.U)
  }
}
```

1. 对预测的影响

预测决策: 最终的预测采纳的是 provider (专家A) 的结果。
备用专家的角色: 只有当 provider 的 ctr 值不够强 (<4) 时，备用专家的意见才会被考虑。

当意见相同时:

即使 provider 的 ctr 不够强，备用专家的 ctr 也不够强，或者它们的值一样，采纳任何一个结果都是一样的。alt_differs 将始终为 false。

结论: 在这种情况下，专家B 对于最终的预测结果，完全没有贡献。它就像一个只会附和的“应声虫”。
2. 对 U-bit 的影响

inc_u 函数的逻辑是： Mux(!alt_differs, u, ...)由于 alt_differs 始终为 false，两位专家的 U-bit 将永远不会被更新。

结论: 两个专家的 U-bit 会停滞在它们最初的值。它们既不会因为预测正确而被奖励，也不会因为“独特贡献”而提升可信度。

3. 对替换的影响

全局老化: 由于 U-bit 不会被 +1，在全局老化的大潮下，它们的 U-bit 最终都会被周期性地“清零”。
动态替换: 一旦任何一个专家的 U-bit 变为 0，它就立刻进入了 “可被随意解雇” 的候选人名单。
结论: 系统会自然而然地、优先地将这些“没有增量价值”的专家替换掉，把宝贵的存储空间留给那些可能在其他分支上表现更好的新条目。


## 3-bit计数器更新

```scala
def inc_ctr(ctr: UInt, taken: Bool): UInt = {
  Mux(!taken,
    Mux(ctr === 0.U, 0.U, ctr - 1.U),  // Not taken: --
    Mux(ctr === 7.U, 7.U, ctr + 1.U)   // Taken: ++
  )
}
```

## TAGE (Tagged Geometric History Length) 分支预测器工作流程详解

## 1. 整体架构概览
```txt

  ┌──────────────────────────────────────────────────────────────────────────────────────────┐
  │                              TAGE Branch Predictor                                        │
  │                                                                                           │
  │  ┌─────────────────────────────────────────────────────────────────────────────────────┐ │
  │  │                         多表结构 (6 个 TAGE 表)                                      │ │
  │  │                                                                                      │ │
  │  │   Table 0          Table 1          Table 2          Table 3          Table 4/5     │ │
  │  │  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐   │ │
  │  │  │ 128 rows │     │ 128 rows │     │ 256 rows │     │ 256 rows │     │ 128 rows │   │ │
  │  │  │ hist=2   │     │ hist=4   │     │ hist=8   │     │ hist=16  │     │ hist=32/64│  │ │
  │  │  │ tag=7bit │     │ tag=7bit │     │ tag=8bit │     │ tag=8bit │     │ tag=9bit │   │ │
  │  │  └──────────┘     └──────────┘     └──────────┘     └──────────┘     └──────────┘   │ │
  │  │       │                │                │                │                │         │ │
  │  │       └────────────────┴────────────────┴────────────────┴────────────────┘         │ │
  │  │                                         │                                            │ │
  │  │                                         ▼                                            │ │
  │  │                              ┌─────────────────────┐                                 │ │
  │  │                              │   Provider 选择逻辑  │                                 │ │
  │  │                              │  (最长历史命中优先)   │                                 │ │
  │  │                              └─────────────────────┘                                 │ │
  │  │                                         │                                            │ │
  │  │                                         ▼                                            │ │
  │  │                              ┌─────────────────────┐                                 │ │
  │  │                              │   最终预测输出       │                                 │ │
  │  │                              │   io.resp.f3.taken  │                                 │ │
  │  │                              └─────────────────────┘                                 │ │
  │  └─────────────────────────────────────────────────────────────────────────────────────┘ │
  │                                                                                           │
  │  基础预测器 (Base Predictor): 由 io.resp_in(0) 提供 (通常是 BIM)                           │
  └──────────────────────────────────────────────────────────────────────────────────────────┘

```
### 2. TAGE 表条目结构

```txt

  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                           TageEntry (主表条目)                                   │
  └─────────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                    tageEntrySz = 1 + tagSz + 3 bits                             │
  ├───────────────┬─────────────────────────────────┬───────────────────────────────┤
  │  valid (1b)   │         tag (7-9 bits)          │        ctr (3 bits)           │
  │  条目有效位    │         地址标签                 │       3-bit 饱和计数器         │
  └───────────────┴─────────────────────────────────┴───────────────────────────────┘

  TageResp (响应结构):
  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                         TageResp Bundle                                          │
  ├─────────────────────────────────┬───────────────────────────────────────────────┤
  │         ctr (3 bits)            │              u (2 bits)                        │
  │       预测计数器值               │            有用性计数器                         │
  └─────────────────────────────────┴───────────────────────────────────────────────┘

                            3-bit 计数器状态
      ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
      │  0  │  1  │  2  │  3  │  4  │  5  │  6  │  7  │
      │ SNT │ NT  │ NT  │ WNT │ WT  │  T  │  T  │ ST  │
      └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
            ◄─── Not Taken ───►◄──── Taken ────►

      预测: ctr[2] (最高位) = 1 表示 Taken

      特殊情况: ctr = 3 或 4 时为弱预测，可能使用备选预测

```
### 3. 有用性计数器 (Usefulness Counter) 结构

```txt
  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                    Usefulness (u) 计数器存储                                     │
  └─────────────────────────────────────────────────────────────────────────────────┘

  存储结构:
      us = SyncReadMem(nRows, Vec(bankWidth*2, Bool()))

      每行存储 bankWidth*2 个 bit (每个 bank 位置 2 bits)

      ┌─────────────────────────────────────────────────────────────────────────┐
      │  Row 0: │u_lo[0]│u_hi[0]│u_lo[1]│u_hi[1]│ ... │u_lo[N-1]│u_hi[N-1]│    │
      ├─────────────────────────────────────────────────────────────────────────┤
      │  Row 1: │u_lo[0]│u_hi[0]│u_lo[1]│u_hi[1]│ ... │u_lo[N-1]│u_hi[N-1]│    │
      ├─────────────────────────────────────────────────────────────────────────┤
      │  ...                                                                    │
      └─────────────────────────────────────────────────────────────────────────┘

      u = Cat(u_hi, u_lo) = 2-bit usefulness

  u 计数器含义:
      ┌───────┬──────────────────────────────────────────────────────────────┐
      │ u = 0 │ 无用条目，可被替换分配                                        │
      │ u = 1 │ 低有用性                                                     │
      │ u = 2 │ 中有用性                                                     │
      │ u = 3 │ 高有用性，很难被替换                                          │
      └───────┴──────────────────────────────────────────────────────────────┘
```
### 4. 索引和标签的折叠历史计算

```txt
  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                        折叠历史 (Folded History) 计算                            │
  └─────────────────────────────────────────────────────────────────────────────────┘

  目的: 将长全局历史压缩为较短的索引/标签

  compute_folded_hist(hist, l):

      示例: histLength = 16, l = 7 (目标长度)

      hist (16 bits):
      ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
      │h15│h14│h13│h12│h11│h10│h9 │h8 │h7 │h6 │h5 │h4 │h3 │h2 │h1 │h0 │
      └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘

      分成 chunks (每个 7 bits):
      ┌─────────────────────────┐  ┌─────────────────────────┐  ┌─────────┐
      │ Chunk 0: h[6:0]         │  │ Chunk 1: h[13:7]        │  │Chunk 2: │
      │ h6 h5 h4 h3 h2 h1 h0    │  │ h13 h12 h11 h10 h9 h8 h7│  │ h15 h14 │
      └─────────────────────────┘  └─────────────────────────┘  └─────────┘
                │                            │                       │
                └────────────────┬───────────┴───────────────────────┘
                                 │
                                 ▼
                            XOR reduce
                                 │
                                 ▼
                      ┌─────────────────────┐
                      │  folded_hist (7b)   │
                      │  用于索引或标签      │
                      └─────────────────────┘

  compute_tag_and_hash(unhashed_idx, hist):

      ┌─────────────────────────────────────────────────────────────────────┐
      │                                                                      │
      │  idx_history = compute_folded_hist(hist, log2(nRows))               │
      │  idx = (unhashed_idx ^ idx_history)[log2(nRows)-1:0]                │
      │                                                                      │
      │  tag_history = compute_folded_hist(hist, tagSz)                     │
      │  tag = ((unhashed_idx >> log2(nRows)) ^ tag_history)[tagSz-1:0]     │
      │                                                                      │
      └─────────────────────────────────────────────────────────────────────┘

      索引: PC 低位 XOR 折叠历史
      标签: PC 高位 XOR 折叠历史 (不同折叠方式)
```
### 5. 几何历史长度分布

```txt
  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                        几何历史长度 (Geometric History)                          │
  └─────────────────────────────────────────────────────────────────────────────────┘

  默认配置 (tableInfo):

      Table    nSets    histLen    tagSz    特点
      ─────────────────────────────────────────────────────────
        0       128        2         7      最短历史，快速响应
        1       128        4         7
        2       256        8         8
        3       256       16         8
        4       128       32         9
        5       128       64         9      最长历史，高精度
      ─────────────────────────────────────────────────────────

  历史长度分布 (几何级数):

      histLen
        │
     64 ┤                                              ┌───┐
        │                                              │ T5│
     32 ┤                                    ┌───┐     └───┘
        │                                    │ T4│
     16 ┤                          ┌───┐     └───┘
        │                          │ T3│
      8 ┤                ┌───┐     └───┘
        │                │ T2│
      4 ┤      ┌───┐     └───┘
        │      │ T1│
      2 ┤ ┌───┐└───┘
        │ │ T0│
      0 ┼─┴───┴─────────────────────────────────────────────►
            0    1    2    3    4    5               Table

  设计理念:
    - 短历史表: 捕捉局部分支行为，训练快
    - 长历史表: 捕捉全局相关性，精度高
    - 几何分布: 覆盖不同时间尺度的模式
```
### 6. 预测流水线时序

```txt
  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                            预测流水线时序                                        │
  └─────────────────────────────────────────────────────────────────────────────────┘

    周期     F0 (请求)        F1 (访问)         F2 (响应)         F3 (输出)
     │          │                │                 │                  │
     ▼          ▼                ▼                 ▼                  ▼
   ─────────────────────────────────────────────────────────────────────────────────
     │                           │                 │                  │
     │  ┌──────────────────┐     │                 │                  │
     │  │ io.f0_valid      │     │                 │                  │
     │  │ io.f0_pc         │─────│                 │                  │
     │  └──────────────────┘     │                 │                  │
     │                           │                 │                  │
     │                    ┌──────▼──────┐          │                  │
     │                    │ f1_req_valid│          │                  │
     │                    │ f1_req_pc   │          │                  │
     │                    │ f1_req_ghist│          │                  │
     │                    │             │          │                  │
     │                    │ 计算 idx,tag│          │                  │
     │                    │ SRAM read   │──────────│                  │
     │                    └─────────────┘ SyncRead │                  │
     │                                             │                  │
     │                                     ┌───────▼───────┐          │
     │                                     │ f2_resps      │          │
     │                                     │ (所有表响应)   │          │
     │                                     │               │          │
     │                                     │ 标签比较       │          │
     │                                     │ s2_req_rhits  │          │
     │                                     │               │          │
     │                                     │ Provider 选择 │──────────│
     │                                     │ s2_provider   │  RegNext │
     │                                     └───────────────┘          │
     │                                                                │
     │                                                        ┌───────▼───────┐
     │                                                        │ f3_resps      │
     │                                                        │ s3_provider   │
     │                                                        │               │
     │                                                        │ 最终预测      │
     │                                                        │ io.resp.f3    │
     │                                                        │               │
     │                                                        │ io.f3_meta    │
     │                                                        └───────────────┘
   ─────────────────────────────────────────────────────────────────────────────────
```
关键点:
- 所有表并行访问 (F1 发起读请求)
- F2 阶段完成标签匹配和 Provider 选择
- F3 阶段输出最终预测结果

### 7. Provider 选择逻辑

```txt


  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                        Provider 选择算法                                         │
  └─────────────────────────────────────────────────────────────────────────────────┘

  核心原则: 最长历史命中优先 (Longest Matching History)

  对于每个 bank 位置 w:

      ┌─────────────────────────────────────────────────────────────────────────┐
      │                     遍历所有表 (从短历史到长历史)                         │
      │                                                                          │
      │   for i in 0..tageNTables:                                              │
      │       hit = f2_resps(i)(w).valid                                        │
      │                                                                          │
      │       // 更新备选 Provider (之前的 Provider)                             │
      │       if (already_provided && hit):                                     │
      │           alt_provided = true                                           │
      │           alt_provider = previous_provider                              │
      │                                                                          │
      │       // 更新主 Provider                                                 │
      │       if (hit):                                                         │
      │           provided = true                                               │
      │           provider = i                                                  │
      └─────────────────────────────────────────────────────────────────────────┘

  示例:

      Table 0 (hist=2):  ✗ Miss
      Table 1 (hist=4):  ✓ Hit    ← 成为 alt_provider
      Table 2 (hist=8):  ✗ Miss
      Table 3 (hist=16): ✓ Hit    ← 成为 provider (最终选择)
      Table 4 (hist=32): ✗ Miss
      Table 5 (hist=64): ✗ Miss

      结果:
        provider = 3 (Table 3, hist=16)
        alt_provider = 1 (Table 1, hist=4)

  选择过程可视化:

      Table 0    Table 1    Table 2    Table 3    Table 4    Table 5
         │          │          │          │          │          │
         ▼          ▼          ▼          ▼          ▼          ▼
      ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐
      │Miss │    │ Hit │    │Miss │    │ Hit │    │Miss │    │Miss │
      └─────┘    └──┬──┘    └─────┘    └──┬──┘    └─────┘    └─────┘
                    │                     │
                    │    ┌────────────────┘
                    │    │
                    ▼    ▼
                ┌───────────────┐
                │ alt = Table 1 │
                │ prov = Table 3│
                └───────────────┘
```
### 8. 最终预测决策逻辑

```txt

  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                         最终预测决策                                             │
  └─────────────────────────────────────────────────────────────────────────────────┘

  决策树:

                           ┌──────────────────────────┐
                           │   s3_provided?           │
                           │   (是否有表命中)          │
                           └────────────┬─────────────┘
                                        │
                      ┌─────────────────┴─────────────────┐
                      │                                   │
                      ▼                                   ▼
                provided = true                    provided = false
                      │                                   │
                      ▼                                   ▼
      ┌───────────────────────────────┐      ┌───────────────────────────┐
      │ prov.ctr == 3 或 4 ?          │      │ 使用基础预测器             │
      │ (弱预测，置信度低)             │      │ io.resp_in(0).f3(w).taken │
      └───────────────┬───────────────┘      └───────────────────────────┘
                      │
         ┌────────────┴────────────┐
         │                         │
         ▼                         ▼
      ctr = 3/4                ctr ≠ 3/4
      (弱预测)                  (强预测)
         │                         │
         ▼                         ▼
      ┌─────────────────┐    ┌─────────────────┐
      │ alt_provided?   │    │ 使用 provider   │
      └────────┬────────┘    │ prov.ctr[2]     │
               │              └─────────────────┘
      ┌────────┴────────┐
      │                 │
      ▼                 ▼
    有备选            无备选
      │                 │
      ▼                 ▼
  ┌──────────┐    ┌──────────────────┐
  │alt.ctr[2]│    │io.resp_in(0).f3  │
  │使用备选   │    │使用基础预测器     │
  └──────────┘    └──────────────────┘


  代码对应 (tage.scala:287-292):

      io.resp.f3(w).taken := Mux(s3_provided,
        Mux(prov.ctr === 3.U || prov.ctr === 4.U,           // 弱预测?
          Mux(s3_alt_provided, alt.ctr(2),                   // 用备选
              io.resp_in(0).f3(w).taken),                    // 用基础
          prov.ctr(2)),                                      // 用 provider
        io.resp_in(0).f3(w).taken                            // 无命中，用基础
      )
```
### 9. 元数据 (Meta) 结构

```txt
  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                          TageMeta 结构                                           │
  └─────────────────────────────────────────────────────────────────────────────────┘

  class TageMeta:
  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                                                                                  │
  │  provider[bankWidth]     : Valid[log2(tageNTables)]                             │
  │      记录每个 bank 位置的 Provider 表编号                                         │
  │                                                                                  │
  │  alt_differs[bankWidth]  : Bool                                                 │
  │      备选预测是否与最终预测不同                                                   │
  │                                                                                  │
  │  provider_u[bankWidth]   : UInt(2.W)                                            │
  │      Provider 的 usefulness 值                                                  │
  │                                                                                  │
  │  provider_ctr[bankWidth] : UInt(3.W)                                            │
  │      Provider 的计数器值                                                         │
  │                                                                                  │
  │  allocate[bankWidth]     : Valid[log2(tageNTables)]                             │
  │      误预测时可分配的表编号                                                       │
  │                                                                                  │
  └─────────────────────────────────────────────────────────────────────────────────┘

  alt_differs 计算:

      alt_differs = s3_alt_provided && (alt.ctr[2] ≠ final_taken)

      含义: 备选预测与最终预测不同
      用途: 更新 usefulness 时判断 provider 是否真正有用

```
### 10. 分配槽位选择 (Allocatable Slots)
```txt
  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                        分配槽位选择算法                                          │
  └─────────────────────────────────────────────────────────────────────────────────┘

  目标: 误预测时找到一个可分配的表条目

  条件:
    1. 该表未命中 (!r(w).valid)
    2. 该条目 usefulness = 0 (可替换)
    3. 使用更长历史的表 (比 provider 历史更长)

      allocatable_slots = (
          VecInit(f3_resps.map(r => !r(w).valid && r(w).bits.u === 0.U)).asUInt &
          ~(MaskLower(UIntToOH(provider.bits)) & Fill(tageNTables, provider.valid))
      )

  可视化:

      假设 provider = Table 2

      Table:     0      1      2      3      4      5
                 │      │      │      │      │      │
      Miss?      ✓      ✓      ─      ✓      ✗      ✓
      u==0?      ✓      ✗      ─      ✓      ─      ✓
                 │      │      │      │      │      │
                 ▼      ▼      ▼      ▼      ▼      ▼
      可分配?    ✗      ✗      ✗      ✓      ✗      ✓
                (比provider短) (prov) (可选)  (命中) (可选)

      allocatable_slots = 0b101000 (Table 3 和 Table 5)

  选择策略 (带随机性):

      first_entry  = PriorityEncoder(allocatable_slots)  // 第一个可分配
      masked_entry = PriorityEncoder(allocatable_slots & LFSR)  // 随机掩码后第一个

      alloc_entry = Mux(allocatable_slots(masked_entry),
                        masked_entry,    // 随机选择有效
                        first_entry)     // 否则选第一个

      这种策略在保证分配的同时引入随机性，避免某些表被过度使用
```
### 11. 更新逻辑

```txt

  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                              更新流程                                            │
  └─────────────────────────────────────────────────────────────────────────────────┘

                           s1_update (来自后端提交)
                                     │
                                     ▼
          ┌──────────────────────────────────────────────────────────────────────┐
          │                     计算误预测掩码                                    │
          │  s1_update_mispredict_mask = UIntToOH(cfi_idx) & cfi_mispredicted   │
          └──────────────────────────────────────────────────────────────────────┘
                                     │
                  ┌──────────────────┴──────────────────┐
                  │                                     │
                  ▼                                     ▼
      ┌─────────────────────────────┐      ┌─────────────────────────────┐
      │  正常更新 (每个分支位置)      │      │  误预测时的额外处理          │
      │                             │      │  (仅对 cfi_idx 位置)        │
      │  if (br_mask[w] && commit): │      │                             │
      │    if (provider.valid):     │      │  if (mispredicted):         │
      │      更新 provider 的 ctr   │      │    if (allocate.valid):     │
      │      更新 provider 的 u     │      │      分配新条目             │
      │                             │      │    else:                    │
      │                             │      │      降低更长表的 u         │
      └─────────────────────────────┘      └─────────────────────────────┘


  Provider 更新详情:

      ┌─────────────────────────────────────────────────────────────────────────┐
      │  new_u = inc_u(provider_u, alt_differs, mispredict)                    │
      │                                                                         │
      │  inc_u 逻辑:                                                            │
      │    if (!alt_differs):                                                  │
      │        u 不变 (备选预测相同，无法判断 provider 是否有用)                  │
      │    elif (mispredict):                                                  │
      │        u-- (provider 预测错误，降低有用性)                               │
      │    else:                                                               │
      │        u++ (provider 预测正确且与备选不同，证明有用)                      │
      │                                                                         │
      │  ctr = inc_ctr(old_ctr, was_taken)                                     │
      │    正常的 3-bit 饱和计数器更新                                           │
      └─────────────────────────────────────────────────────────────────────────┘


  误预测时的分配:

      ┌─────────────────────────────────────────────────────────────────────────┐
      │  if (allocate.valid):                                                  │
      │      // 有可分配的槽位                                                  │
      │      s1_update_mask[allocate.bits][idx] = true                         │
      │      s1_update_alloc[allocate.bits][idx] = true                        │
      │      s1_update_taken = cfi_taken                                       │
      │      s1_update_u = 0  (新条目 usefulness 从 0 开始)                     │
      │                                                                         │
      │  else:                                                                 │
      │      // 无可分配槽位，降低更长历史表的 u                                  │
      │      decr_mask = ~MaskLower(UIntToOH(provider.bits))                   │
      │      对所有 decr_mask 中的表: u = 0                                     │
      │                                                                         │
      │      这样下次就有更多 u=0 的条目可供分配                                  │
      └─────────────────────────────────────────────────────────────────────────┘

```
### 12. Usefulness 周期性清除

```txt

  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                     Usefulness 周期性清除机制                                    │
  └─────────────────────────────────────────────────────────────────────────────────┘

  目的: 防止 u 计数器饱和后永远无法分配新条目

  参数: uBitPeriod = 2048 (默认)

  机制:

      clear_u_ctr: 全局计数器 (log2(uBitPeriod) + log2(nRows) + 1 bits)

      ┌─────────────────────────────────────────────────────────────────────────┐
      │                                                                          │
      │  doing_clear_u = (clear_u_ctr 低 log2(uBitPeriod) 位 == 0)              │
      │                                                                          │
      │  每 uBitPeriod (2048) 个周期清除一行                                     │
      │                                                                          │
      │  clear_u_idx = clear_u_ctr >> log2(uBitPeriod)                          │
      │                                                                          │
      │  交替清除高/低位:                                                         │
      │    clear_u_hi = clear_u_ctr 最高位 == 1                                  │
      │    clear_u_lo = clear_u_ctr 最高位 == 0                                  │
      │                                                                          │
      │  clear_u_mask: 偶数位置清 lo，奇数位置清 hi                               │
      │                                                                          │
      └─────────────────────────────────────────────────────────────────────────┘

  时序图:

      周期:      0      2048     4096     6144     8192    ...
                │        │        │        │        │
                ▼        ▼        ▼        ▼        ▼
      Row 0:  [清lo]           [清hi]              [清lo]
      Row 1:           [清lo]           [清hi]
      Row 2:                   [清lo]           [清hi]
      ...

      一个完整周期: 2048 * nRows * 2 个时钟周期

      效果: u[1:0] 中的每一位都会被周期性清零
            使得长期未命中的条目逐渐变为可分配状态

```
### 13. 写旁路机制

```txt
  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                         写旁路 (Write Bypass)                                    │
  └─────────────────────────────────────────────────────────────────────────────────┘

  每个 TageTable 有独立的写旁路缓冲:

      nWrBypassEntries = 2

      ┌──────────────────────────────────────────────────────────────────────────┐
      │  wrbypass_tags[0..1]  : 标签                                             │
      │  wrbypass_idxs[0..1]  : 索引                                             │
      │  wrbypass[0..1]       : Vec(bankWidth, ctr)  计数器值                    │
      │  wrbypass_enq_idx     : 循环入队指针                                     │
      └──────────────────────────────────────────────────────────────────────────┘

  工作流程:

      更新请求
          │
          ▼
      ┌────────────────────────────────────────┐
      │ 检查 update_tag, update_idx 是否命中   │
      │ wrbypass_hits = (tag match && idx match)│
      └────────────────────┬───────────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
           命中                       未命中
              │                         │
              ▼                         ▼
      ┌───────────────────┐    ┌───────────────────┐
      │ 使用 wrbypass[hit]│    │ 使用 io.update_   │
      │ 的 ctr 作为旧值   │    │ old_ctr 作为旧值  │
      └───────────────────┘    └───────────────────┘
              │                         │
              └────────────┬────────────┘
                           │
                           ▼
                 ┌───────────────────────────┐
                 │  计算 new_ctr             │
                 │  inc_ctr(old_ctr, taken)  │
                 └────────────┬──────────────┘
                              │
                 ┌────────────┴────────────┐
                 │                         │
                 ▼                         ▼
              命中                       未命中
                 │                         │
                 ▼                         ▼
      ┌───────────────────┐    ┌───────────────────────┐
      │ 更新 wrbypass[hit]│    │ 分配 wrbypass[enq]    │
      │ 的 ctr            │    │ 更新 tag, idx, ctr    │
      └───────────────────┘    │ enq_idx++             │
                               └───────────────────────┘

```
### 14. 完整数据流图

```txt

  ┌──────────────────────────────────────────────────────────────────────────────────────────┐
  │                              TAGE 完整数据流                                              │
  └──────────────────────────────────────────────────────────────────────────────────────────┘

                                ┌─────────────────┐
                                │  F0: 取指请求    │
                                │  io.f0_pc       │
                                │  io.f0_valid    │
                                └────────┬────────┘
                                         │ RegNext
                                         ▼
      ┌─────────────────────────────────────────────────────────────────────────────────────┐
      │                              F1: 并行表访问                                          │
      │                                                                                      │
      │   ┌─────────────────────────────────────────────────────────────────────────────┐   │
      │   │                     对每个 Table (0..5):                                     │   │
      │   │     (s1_hashed_idx, s1_tag) = compute_tag_and_hash(pc, ghist)               │   │
      │   │     table.read(s1_hashed_idx)                                               │   │
      │   │     us.read(s1_hashed_idx)                                                  │   │
      │   └─────────────────────────────────────────────────────────────────────────────┘   │
      │                                                                                      │
      └───────────────────────────────────────────────┬──────────────────────────────────────┘
                                                      │ SyncReadMem
                                                      ▼
      ┌─────────────────────────────────────────────────────────────────────────────────────┐
      │                              F2: 标签比较 & Provider 选择                            │
      │                                                                                      │
      │   ┌────────────────────────────────────────────────────────────────────────────┐    │
      │   │  对每个 Table:                                                              │    │
      │   │    s2_req_rhits = (entry.valid && entry.tag == s2_tag && !doing_reset)     │    │
      │   │    f2_resp.valid = s2_req_rhits                                            │    │
      │   │    f2_resp.bits.ctr = entry.ctr                                            │    │
      │   │    f2_resp.bits.u = Cat(u_hi, u_lo)                                        │    │
      │   └────────────────────────────────────────────────────────────────────────────┘    │
      │                                                                                      │
      │   ┌────────────────────────────────────────────────────────────────────────────┐    │
      │   │  Provider 选择 (对每个 bank w):                                             │    │
      │   │    遍历所有表，选择最长历史命中的表作为 provider                              │    │
      │   │    记录前一个命中作为 alt_provider                                          │    │
      │   └────────────────────────────────────────────────────────────────────────────┘    │
      │                                                                                      │
      └───────────────────────────────────────────────┬──────────────────────────────────────┘
                                                      │ RegNext
                                                      ▼
      ┌─────────────────────────────────────────────────────────────────────────────────────┐
      │                              F3: 最终预测输出                                        │
      │                                                                                      │
      │   ┌────────────────────────────────────────────────────────────────────────────┐    │
      │   │  io.resp.f3(w).taken =                                                      │    │
      │   │    Mux(s3_provided,                                                         │    │
      │   │      Mux(weak_prediction,                                                   │    │
      │   │        Mux(s3_alt_provided, alt.ctr[2], base_pred),                        │    │
      │   │        prov.ctr[2]),                                                        │    │
      │   │      base_pred)                                                             │    │
      │   └────────────────────────────────────────────────────────────────────────────┘    │
      │                                                                                      │
      │   ┌────────────────────────────────────────────────────────────────────────────┐    │
      │   │  io.f3_meta =                                                               │    │
      │   │    provider, alt_differs, provider_u, provider_ctr, allocate               │    │
      │   └────────────────────────────────────────────────────────────────────────────┘    │
      │                                                                                      │
      └──────────────────────────────────────────────────────────────────────────────────────┘

                                ┌─────────────────┐
                                │  更新路径        │
                                │  s1_update      │
                                └────────┬────────┘
                                         │
                                         ▼
      ┌─────────────────────────────────────────────────────────────────────────────────────┐
      │                              S1: 更新处理                                            │
      │                                                                                      │
      │   ┌────────────────────────────────────────────────────────────────────────────┐    │
      │   │  正常更新:                                                                  │    │
      │   │    更新 provider 的 ctr 和 u                                               │    │
      │   │                                                                             │    │
      │   │  误预测处理:                                                                │    │
      │   │    if (allocate.valid): 分配新条目                                         │    │
      │   │    else: 降低更长历史表的 u                                                │    │
      │   └────────────────────────────────────────────────────────────────────────────┘    │
      │                                                                                      │
      └───────────────────────────────────────────────┬──────────────────────────────────────┘
                                                      │ RegNext
                                                      ▼
      ┌─────────────────────────────────────────────────────────────────────────────────────┐
      │                              S2: 写入 SRAM                                           │
      │                                                                                      │
      │   tables(i).io.update_* := RegNext(s1_update_*)                                     │
      │   table.write(update_idx, update_wdata, update_mask)                                │
      │   us.write(update_idx, update_u, update_u_mask)                                     │
      │                                                                                      │
      └──────────────────────────────────────────────────────────────────────────────────────┘

```
### 15. TAGE 与其他预测器的协作

```txt
  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │                       TAGE + BIM 协作关系                                        │
  └─────────────────────────────────────────────────────────────────────────────────┘

                                ┌─────────────────┐
                                │    取指 PC       │
                                └────────┬────────┘
                                         │
                      ┌──────────────────┴──────────────────┐
                      │                                     │
                      ▼                                     ▼
              ┌───────────────┐                     ┌───────────────┐
              │     TAGE      │                     │     BIM       │
              │  (复杂预测器)  │                     │  (基础预测器)  │
              │               │                     │               │
              │ 6个历史表      │                     │ 2-bit 计数器   │
              │ 标签匹配       │                     │ 简单快速       │
              └───────┬───────┘                     └───────┬───────┘
                      │                                     │
                      │                                     │
                      ▼                                     │
              ┌───────────────────────────────────────────┐ │
              │           TAGE 命中?                      │ │
              └─────────────────┬─────────────────────────┘ │
                                │                           │
                   ┌────────────┴────────────┐              │
                   │                         │              │
                   ▼                         ▼              │
                命中                       未命中           │
                   │                         │              │
                   ▼                         │              │
      ┌──────────────────────┐               │              │
      │ 弱预测 (ctr=3/4)?    │               │              │
      └──────────┬───────────┘               │              │
                 │                           │              │
        ┌────────┴────────┐                  │              │
        │                 │                  │              │
        ▼                 ▼                  │              │
      是                 否                  │              │
        │                 │                  │              │
        ▼                 ▼                  │              │
    ┌─────────┐    ┌─────────┐              │              │
    │ 有备选? │    │使用TAGE │              │              │
    └────┬────┘    │provider │              │              │
         │         └─────────┘              │              │
    ┌────┴────┐                             │              │
    │         │                             │              │
    ▼         ▼                             ▼              │
  有备选    无备选                          │              │
    │         │                             │              │
    ▼         │                             │              │
  ┌─────┐     │                             │              │
  │ alt │     └──────────────┬──────────────┘              │
  └─────┘                    │                             │
                             │◄────────────────────────────┘
                             │
                             ▼
                     ┌───────────────┐
                     │  BIM 预测     │
                     │ (基础预测器)   │
                     └───────────────┘

```
层次关系:
- TAGE 是主预测器，提供高精度预测
- BIM 是后备预测器，在 TAGE 未命中或弱预测无备选时使用
- BTB 提供目标地址 (与 taken/not-taken 预测分离)

### 16. 关键参数总结

  | 参数               | 默认值       | 说明                         |
  |------------------|-----------|----------------------------|
  | tableInfo        | 6个表       | (nSets, histLen, tagSz) 配置 |
  | tageNTables      | 6         | TAGE 表数量                   |
  | uBitPeriod       | 2048      | usefulness 清除周期            |
  | singlePorted     | false     | 是否单端口 SRAM                 |
  | nWrBypassEntries | 2         | 每表写旁路条目数                   |
  | tageEntrySz      | 1+tagSz+3 | 表条目位宽                      |

  | 表编号 | nRows | histLen | tagSz | 用途     |
  |-----|-------|---------|-------|--------|
  | 0   | 128   | 2       | 7     | 极短历史模式 |
  | 1   | 128   | 4       | 7     | 短历史模式  |
  | 2   | 256   | 8       | 8     | 中短历史模式 |
  | 3   | 256   | 16      | 8     | 中等历史模式 |
  | 4   | 128   | 32      | 9     | 长历史模式  |
  | 5   | 128   | 64      | 9     | 极长历史模式 |

TAGE 是一种高精度分支预测器，通过多个使用不同历史长度的表来捕捉不同时间尺度的分支相关性，是现代高性能处理器中最常用的分支预测算法之一。

