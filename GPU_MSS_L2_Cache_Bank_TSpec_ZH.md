# GPU MSS L2 Cache Bank 设计说明

| 项目 | 内容 |
| --- | --- |
| 标题 | L2 Bank 设计说明 |
| 作者 | liusm、zhujt、luoml |
| 版本 | 1.6 |
| 日期 | 2025-05-20 |
| 说明 | 由英文 Spec 翻译并适度精简。模块名、寄存器名、信号名保持原文，便于对照代码。 |

---

## 修订记录

| 日期 | 版本 | 作者 | 内容 |
| --- | --- | --- | --- |
| 2024.12.26 | V1.0 | liusm | 通过评审 |
| 2025.2.5 | V1.1 | liusm | 补充 lookup 流水和 touch mask |
| 2025.2.26 | V1.2 | zhujt | 更新原子操作特殊值结果 |
| 2025.3.11 | V1.3 | zhujt | 更新 RCTRL 实现 |
| 2025.3.14 | V1.3 | liusm | 增加 ISCHED–RCTRL 写数据接口 |
| 2025.3.25 | V1.4 | luoml | Bank 组由 4 个改为 2 个，更新 Memory Scheduler |
| 2025.4.11 | V1.5 | zhujt | 增加 FP32 原子 CAS；更新 FP32 min/max 特殊值 |
| 2025.5.20 | V1.6 | liusm | 增加 DDR Hash；更新 Scheduler 优先级 |

---

## 目录

- [1 概述](#1-概述)
- [2 功能特性](#2-功能特性)
- [3 功能说明](#3-功能说明)
  - [3.1 AXI 事务](#31-axi-事务)
  - [3.2 Cache 怎么组织](#32-cache-怎么组织)
  - [3.3 持久度和替换](#33-持久度和替换)
  - [3.4 扇区和预取](#34-扇区和预取)
  - [3.5 原子操作](#35-原子操作)
  - [3.6 Buffer Mode](#36-buffer-mode)
  - [3.7 Cache 策略](#37-cache-策略)
  - [3.8 粗粒度刷 Cache](#38-粗粒度刷-cache)
  - [3.9 数据通路](#39-数据通路)
  - [3.10 Memory Scheduler](#310-memory-scheduler)
  - [3.11 CRM](#311-crm)
  - [3.12 清除 Partial](#312-清除-partial)
  - [3.13 冗余](#313-冗余)
- [4 状态和配置寄存器](#4-状态和配置寄存器)
- [5 设计约束](#5-设计约束)
- [6 接口和指令](#6-接口和指令)
- [7 遗留问题](#7-遗留问题)
- [8 硬件模块](#8-硬件模块)
- [9 面积](#9-面积)
- [10 性能](#10-性能)

---

## 1 概述

L2 Cache 是最后一级 Cache。所有要去 DDR 的访问都必须经过它，并且要同时服务多个主人（GPC、CE、FW、PCIE、VPU 等）。

它基于 **AXI5**（只用了 AXI5 的一部分功能），输入是 **物理地址（PIPT）**，所以可以直接连 DDR，布线也更不容易堵。

L2 是分布式的：可以复制很多份，每份独立，方便做大容量。真正处理读、写、原子操作的单元叫 **L2 Bank**。请求先被分到各个 **L2 Group**，每组连自己的 DDR。组与组之间互不影响。

> 原文概述写每组 8 个 Bank；后面寄存器和 Buffer Mode 表按 **80 个 Bank**（`L2B0`～`L2B79`）、**8 组**（`L2G0`～`L2G7`）来编。两处都按原文保留。

> **图 1** L2 整体架构；**图 2** Bank 在系统中的位置；**图 3** Bank 内部架构（原 TXT 无图）。

---

## 2 功能特性

1. PIPT Cache，每个 Bank **256 KB**
2. 一行 **256 B**，分成 **2 个扇区**；预取可选 128 B / 256 B
3. 支持进 Cache 的读写，也支持 Bypass 读写
4. Linefill（从 DDR 填行）优先级最高
5. 支持原子操作，包含 FP32
6. 支持 persistence（行能“赖”多久）
7. 支持 Buffer Mode（便签内存），容量可选 25% / 50%
8. 下游走 NSP 协议连 DDR

---

## 3 功能说明

### 3.1 AXI 事务

Bank 按 AXI 的内存属性处理访问。关键看 `AxCache`：

| 信号 | 含义 |
| --- | --- |
| `AxCache[1] = 0` | Device：不查 Cache、不分配行，直接穿过 |
| `AxCache[1] = 1` | Normal：查 Cache，再按策略做事 |
| Write-back | `policy = 0`，进 Cache |
| Write-through | `policy = 1`，最终要落到 DDR |

L2 是最后一级，Write-through 也可以把它看成“终点”：别的主人再查时能看到。所有 Normal 写都可以 **先回写响应，不用等 DDR 写完**。

#### 3.1.1 请求属性

**Size（每次搬多少字节）**

- 不能超过接口数据宽度
- GPC / CE / FW 一般 `arsize = 6`（64 B）
- PCIE / VPU 可能更窄，所以 L2 必须支持 `axsize <= 6`
- 窄传输（小于 64 B）时，`blen` 固定为 0

**Length（搬几次）**

- 总数据量 = Size × Length
- GP3 限制一笔不超过 2 Kbit，即 **`Size × Length <= 256 B`**
- 一行正好 256 B，所以 **任何请求都不会跨 cacheline**
- WRAP：Length 只能是 2 / 4 / 8 / 16
- FIXED：Length 最多 16
- 不能跨 4 KB 边界
- GPC / CE / FW 的 Length 小于 4；更长的 PCIE 请求会在 NOC 被切开，L2 只看到 `blen <= 3`

**Burst（地址怎么涨）**

- 主要支持 **INCR**，且不能跨 256 B 边界
- WRAP 时：`AxSize × Length <= 256`
- 窄传输：`AxSize < 64 B` 则 `AxLength == 0`

**数据通道**

比数据总线更窄的传输，由地址和 Burst 决定用哪几条字节通道：INCR / WRAP 每次可能换通道，FIXED 一直用同一组通道。

---

### 3.2 Cache 怎么组织

组织方式：**128 个 set × 8 个 way**。每个 set 有一套 PLRU 位，指向最该被换掉的行。

- AXI 请求地址按 512 bit 对齐
- 一行宽 2048 bit（256 B）
- 一行里的两个扇区 **共用** `addr_tag` 和 `persistence`
- 每个扇区自己的 `valid`、`beat_dirty_mask`、`partial`

| 字段 | 范围 | 作用 |
| --- | --- | --- |
| `addr_tag` | 整行 | 用来查是不是这一行 |
| `persistence` | 整行 | 影响被换掉的先后 |
| `valid` | 扇区 | 这半行有没有效 |
| `beat_dirty_mask` | beat | 对应 512 bit。里面有脏字节就拉高 |
| `partial` | 扇区 | 对应 1024 bit。里面还有无效字节就拉高 |
| `access_counter` | 整行 | 这行还有几条指令没做完 |
| `initial_mask` | 扇区 | 是不是第一次用。为 0 时即使只写一部分，也要初始化整行 mask |
| `touched_mask` | 整行 | 最近一个时间窗里有没有被碰到 |
| `Data[2047:0]` | 整行 | 数据 |
| `Mask[255:0]` | 字节 | 哪一字节比 DDR 新，换出时要写回 |

---

### 3.3 持久度和替换

`persistence[1:0]` 表示这行能“赖”多久：

| 值 | 含义 |
| --- | --- |
| `2'b00` | 最先被换掉 |
| `2'b01` / `2'b10` | 中间 |
| `2'b11` | 最不容易被换掉 |

分配新行时写入 Tag RAM，只影响替换，不影响对错。

查目录时，TagCtrl 知道这一 set 里谁有效、谁残缺、谁脏、谁还在飞（in-flight，也叫 **active**）。**正在干活的行不能换。** 能换的里面：空行优先，然后按 persistence 从低到高。

```verilog
replace_way = 没有空闲行 ? persistence0 : persistence1 : persistence2 : persistence3;
```

同一地址区域的 persistence 应当稳定：请求带来的值和 Tag 里已有的值必须一样。

上一轮 kernel 留下的“金贵”行可能一直占着。Bank 提供两种降级办法。

#### 3.3.1 降低一段时间没人用的行

用 `touched_mask` 记录这个时间窗里有没有被碰到。扫完全部 set：

- 被碰到过：persistence 不动
- 没被碰到：persistence 减 1

只使用一次的行也应该降级，所以 **只有 Hit 才把 `touched_mask` 置 1**。刚分配出来时它仍是 0。收到“降 persistence”或“清 persistence”命令时清掉。

#### 3.3.2 全部清零

软件经 socif 发复位命令，扫完全部 set，只把 persistence 清零，其它状态不动。

#### 3.3.3 软件怎么配

```c
Write32(GC_CR_L2_CMD, 0x1);                 // 全部清零
Write32(GC_CR_L2_PERSIST_REDUCE, 0x3f_ffff); // 大约每 0x1f_ffff_fff 周期降一次
```

`Write64` 可能连写两次。为了稳健，L2 **会忽略第二次写通知**。

---

### 3.4 扇区和预取

一行两半，状态独立。请求可能只碰一半，也可能两半都碰。`AXI.user` 上的 prefetch 也会影响。

Bypass 读即使带着 `prefetch = 1`，也会被改成 0（预取对 Bypass 读无效）。

| 状态 | 怎么判断 |
| --- | --- |
| INVALID | `valid = 0` |
| DIRTY | 有效，且有脏 beat，且没有洞 |
| PARTIAL | 有效，且有脏 beat，且有洞 |
| CLEAN | 有效，没脏、没洞 |

**Linefill**：从 DDR 读数据填进这一行。

**写 Miss 不会 linefill**，所以 `partial` 必须跟在扇区上。预取只对 **可缓存读和原子操作** 生效，写请求上的预取直接忽略。

#### 3.4.1 只访问一个扇区

| 操作 | 策略 | 状态 | 做什么 |
| --- | --- | --- | --- |
| 读 | 0 | Miss / INVALID | 分配行，linefill，填完再返回 |
| 读 | 0 | DIRTY / CLEAN | 直接返回 |
| 读 | 0 | PARTIAL | linefill，和已有数据合并后再返回 |
| 读 | 1 | Miss | 不分配行，直打 DDR |
| 读 | 1 | 其它 | 和策略 0 一样 |
| 写 | 0 | Miss / INVALID | 分配行，直接写入，不去 DDR 先读 |
| 写 | 0 | 已有行 | 直接写入 |
| 写 | 1 | Miss | 直打 DDR，不分配 |
| 写 | 1 | 已有行 | 见下一节。Bypass 写会作废整行，要看另一半状态 |
| 原子 | — | Miss / INVALID / PARTIAL | 分配或 linefill，填完再算 |
| 原子 | — | DIRTY / CLEAN | 直接算 |

#### 3.4.2 访问两个扇区

预取默认 128 B，可选 256 B。写 Miss 不 linefill，所以预取主要影响读。访问两扇区，或只访问一扇区但预取 256 B 时，两半状态都要看。

| 操作 | 策略 | 两扇区状态 | 做什么 |
| --- | --- | --- | --- |
| 读 | 0 | 任一侧 INVALID 或 PARTIAL | linefill，填完再返回 |
| 读 | 0 | 两侧都是 CLEAN / DIRTY | 直接返回 |
| 读 | 1 | 两侧都 INVALID | 不分配，直打 DDR |
| 读 | 1 | 其它 | 和策略 0 一样 |
| 写 | 0 | 两侧都 INVALID | 分配行，直接写入 |
| 写 | 0 | 其它组合 | 直接写入 |
| 写 | 1 | 两侧都 INVALID | 不分配 |
| 写 | 1 | 两侧都 CLEAN | 作废行，直打 DDR |
| 写 | 1 | 任一侧 PARTIAL / DIRTY | 和写入合并后刷 DDR，再作废行 |
| 原子 | — | 有 INVALID / PARTIAL | 分配或 linefill，再算 |
| 原子 | — | 两侧都 CLEAN / DIRTY | 直接算 |

---

### 3.5 原子操作

不支持独占访问，`axlock` 恒为 0。标准 AXI Atomic **不做**。

GPU 自己定义了 atomic32 / atomic64。MCU L1 可以合并后再发给 L2。多数原子只有 1 个操作数，**CMPXCHG（比较交换）有 2 个**。PCIE / VPU 的原子会先转成 GPC 这种格式。

详细编码见：

```text
GPU_Ordering_System_Design_CHS_V0PXX.docx
```

---

### 3.6 Buffer Mode

也叫便签内存（scratch）。地址范围由寄存器指定。落在这个范围内的请求叫 **scratch 请求**：

- 不查 Cache
- 不替换、不分配行
- 数据不写回 DDR，也不做 Invalidate / Flush

容量按百分比从 L2 里划出来。打开后，访问地址由物理地址推出来。

#### 3.6.1 SRAM 怎么映射

`GC_CR_L2_BUFFER_PERCENTAGE`：

| 值 | 含义 |
| --- | --- |
| `0x0` | 关掉 |
| `0x1` | 占 25% |
| `0x2` | 占 50% |
| `0x3` | 保留 |

```text
set[6:0]  = Addr[14:8]
way[2:0]  = Addr[16:15] + 6    // 25%
way[2:0]  = Addr[16:15] + 4    // 50%
beat[1:0] = Addr[7:6]
SramAddr  = {way, set, beat}
```

#### 3.6.2 寄存器和容量

| 寄存器 | 作用 |
| --- | --- |
| `GC_CR_L2_BUFFER_BASE[47:0]` | 便签区基地址 |
| `GC_CR_L2_BUFFER_PERCENTAGE` | 0% / 25% / 50% |

按 80 个 Bank 估算：

| 配置 | Cache | 便签 | 每个 Bank |
| --- | --- | --- | --- |
| 关掉 | 20 MB | 0 | 0 |
| 25% | 15 MB | 5 MB | 64 KB |
| 50% | 10 MB | 10 MB | 128 KB |

#### 3.6.3 坏 Bank 时

例如 Bank0 坏了，请求改走 Bank3，Bank3 原来的便签空间就没了。Hash 会改地址，避开坏 Bank。便签总容量会变小。

#### 3.6.4 软件注意

复位之后、NOC 发请求之前配好。**下次复位前不能再改。** 写完便签必须再读一次，否则读回来的数据无效。

---

### 3.7 Cache 策略

由 `axcache[0]` 决定（和不少人的直觉相反）：

| `axcache[0]` | Policy | 白话 |
| --- | --- | --- |
| 1 | 0 | 进 Cache / 写回 |
| 0 | 1 | Bypass / 写穿 |

| 操作 | 策略 | Hit/Miss | 做什么 |
| --- | --- | --- | --- |
| 读 | 0 | 任意 | 分配行；Miss 或碰到残缺/无效扇区就 linefill；从 L2 返回 |
| 读 | 1 | Hit | 残缺/无效仍去 DDR 补；从 L2 返回 |
| 读 | 1 | Miss | 不分配，直打 DDR，数据回 NOC |
| 写 | 0 | 任意 | 先回响应；分配行；写入 Cache |
| 写 | 1 | Hit | 先回响应；作废行；脏/残缺则先合并再写 DDR |
| 写 | 1 | Miss | 先回响应；直打 DDR |
| 原子 | — | 任意 | 先回响应；分配行；残缺/无效先 linefill；做原子；返回旧值 |

Bypass 读触发的 linefill，长度不受 prefetch 影响。

从 NOC 角度看：

| | 进 Cache | Bypass |
| --- | --- | --- |
| 要不要查 | 要 | 要 |
| 写响应从哪来 | L2 | L2 |
| 读数据从哪来 | 一定从 L2 | L2 或 DDR |
| 写要不要马上对 DDR 可见 | 不必 | 必须 |
| 读/写要不要占一行 | 必须 | 不必 |

---

### 3.8 粗粒度刷 Cache

软件经 socif 发 Flush。Flush 和 NOC 请求可以同时进，L2 **不保证二者顺序**。

刷的时候把所有行扫一遍，脏的写回 DDR（走 NSP）。某行还在处理就停下来等。扫不完，NOC 新请求也要停。

- **Flush / FlushInval**：寄存器触发
- **Invalidate**：复位时硬件自己发，用来把 Tag RAM 随机值清干净

| 操作 | 编码 | 做什么 |
| --- | --- | --- |
| Invalid | `0x0` | 不管脏不脏一律作废。建议在全空闲时用。不影响已经在飞的请求，它们仍要正确返回。复位时也会发一次 |
| Flush | `0x1` | 脏行写回，行还留着。残缺行会被逐出到 DDR |
| FlushInval | `0x2` | 写回后作废。脏/残缺都必须写回 |

软件建议用 `Write32`（`Write64` 会触发两次）。做完后要等：

```c
Write32(GC_CR_L2_CFI_OPERATION, 0x1);
POLL(GC_CR_L2_CFI_STATUSI,  0xffffffff_fffffff);
POLL(GC_CR_L2_CFI_STATUSII, 0xffff);
```

第二次写通知会被忽略。

---

### 3.9 数据通路

#### 3.9.1 Cache 读

**Miss**

1. NOC 发读
2. ALIGN 对齐后让 TagCtrl 查
3. 换掉一行干净行；给 ISCHED 发 RD 等 linefill；清 initial mask；让 MEM_IF 去 DDR
4. 地址进 Memory Scheduler，拿回 PA 和事务号
5. 分配 ROB、NSP ID，读 DDR
6. 数据回来做 linefill
7. 完成后 ISCHED 放行 RD，RCTRL 读出，ALIGN 还给 NOC

Hit 在 **partial** 行上：流程和 Miss 一样，只是不清 initial mask。

**Hit（干净/脏）**

1. 查中
2. TagCtrl 给 ISCHED 发 RD
3. ISCHED 转给 RCTRL
4. 数据经 ALIGN 返回

#### 3.9.2 Bypass 读 Miss

查不中、不占行。Scheduler → DDR → 数据进 ROB。最后一拍到齐后，走 Bypass 返回通道给 ALIGN，**不经过 RCTRL**。

#### 3.9.3 Cache 写

1. AW 和写数据到达，ALIGN 在 IWDB 占位
2. AW 和数据对齐、并和读请求仲裁后，**立刻回写响应**（IWDB 还放得下就可以往 TagCtrl 发）
3. Miss 则占一行，发 WR 给 ISCHED，同时清 initial mask
4. ISCHED 经 WDC 从 IWDB 取数
5. 带着数据把 WR 发给 RCTRL；回来后 TagCtrl 把访问计数减一

#### 3.9.4 Bypass 写

**Miss**：查完发 WT 给 MEM_IF，Scheduler 排完后，WDC 从 IWDB 取数，MEM_IF 写 DDR。

**Hit 脏行**：发 WB_WR。ISCHED 拆拍，RCTRL 把 Cache 旧数据和这次写入合并，再经 WDC / MEM_IF 写 DDR。

如果 Bypass 写只碰到几拍，但整行都脏：没被碰到的拍会从 WB_WR 改成 WB（只读出写回，不合并）。

#### 3.9.5 读 Miss 打到脏行（要先写回）

1. 先给 ISCHED 发 RD，再给 MEM_IF 发 WB 和 linefill
2. 写回地址、linefill 地址分别进 Scheduler
3. WDC → ISCHED → RCTRL 读出脏数据，写回 DDR
4. 再读 DDR、linefill
5. 完成后 RD 才真正读 SRAM，返回 ALIGN

---

### 3.10 Memory Scheduler

LPDDR 最多 8 GB，16 个 bank，分成 4 个 bank group。组与组独立，组内按序。想吃满带宽，最好把请求打散到不同组，把“激活 bank”的等待藏起来。

Scheduler 做两件事：按 bank group 交错，并在一个时间窗里重排。如果一笔突发已经跨了多个 DDR bank group，再排就没意义，**应当关掉**。只访问一个 bank group 时它才工作。

L2 Bank 里也有一份 DDR Hash，用来打散 bank。DDR 控制器按寄存器把地址映射成 bank / row。DDR 会丢掉最低 2 bit，所以：

```text
真正偏移 = 寄存器偏移 + 2
```

注意：

- `hash_pa0` 是 34 bit，最高位表示哪个 DDR 通道，**不能 hash**
- 同一 cacheline 不能乱序，所以不管 `hash_pa0[7:0]`
- row / bank 偏移不能指到 bit 7、6，保证一行落在同一个 DDR row 和 bank
- **不建议 L2 Scheduler 和 DDR Hash 同时开**。两个 Hash 都开时，Scheduler 拿到的是原始 bank 号，L2 Hash 失效
- 关掉 Scheduler 时，发往 DDR 的地址和顺序都保持原样

**DDR 行宽**（`GC_CR_L2_FUNCTION_ENABLE[7:4]`）：

| 配置 | 行位数 | 密度 |
| --- | --- | --- |
| `0x0` | 13 | 2 Gb |
| `0x1` | 14 | 3 / 4 Gb |
| `0x2` | 15 | 6 / 8 Gb |
| `0x3` | 16 | 12 / 16 Gb |
| `0x4` | 17 | 24 / 32 Gb |

行宽不够的高位 RowID 补 0。

**两个 Bank 共用一个 Scheduler**

- 都正常：Bank0 → 队列 0，Bank1 → 队列 1
- Bank0 坏了：看 Bank1 的 `PA[33]`，1 走队列 1，0 走队列 0
- 队列 0 眼里 `PA[32]` 永远是 0，队列 1 永远是 1

出队优先级（高到低）：

1. 等太久的“最老请求”
2. 和上一笔同 row、同 bank
3. 不同 DDR bank（按 LRU）

最老请求的定义：它后面连续 N−1 笔都已经出去了（N 是队列深度）。同时满足时，取更早的那笔。

**LRU 大意**：DDR bank 分成 8 个组。复位时 bank 0～15 优先级从高到低。每次选出一个 bank 后，优先级轮转，让刚用过的沉到底。同一 bank group 有多笔时，选最老的。

---

### 3.11 CRM

为了省电：Bank 里没活就关时钟。CRM 管两个 Bank 和它们的 Scheduler。任一边从 NOC 来请求，三家一起叫醒。

每个 Bank 直连 DDRC。一个 Bank 坏了，请求改走另一个。中间有 crossbar，接到 2 个 DDRC。**两个 Bank 不会同时分别打两个 DDRC。**

GPU 不能随便软复位 L2（PCIE / VPU 可能还在用），所以 CRM 的 softreset 脚被绑死为 0。

---

### 3.12 清除 Partial

主人可能一次写满半行或整行。这时应清掉 `partial`，免得以后读还去 DDR 补洞，浪费带宽。这是性能优化，不做也能工作。

命令和数据是分开处理的，**只有确认会写满整个扇区时** 才可能清。

- **Miss**：写满扇区则给 RamCtrl 发 `check_partial`，字节都写过了就回报 TagCtrl 去清
- **Hit**：只考虑残缺或无效的扇区。写满才检查

RamCtrl 看 mask RAM：对应字节全是 1，就回 `clr_partial`。`check_partial` 由 TagCtrl 发出，绕过 ISCHED。

---

### 3.13 冗余

为了降低流片报废成本，单个 Bank 坏了可以绕开。

**DFT 视角**

Bank 单独硬化。DFT 能知道哪个 Bank 坏了。CRM、regbank 这类公共模块只能知道“里面有逻辑坏了”，一坏整组作废。

8 Bank 版本面积量级：Bank 约 19.2w × 8 = 153.6w，Scheduler 约 7.4w。

**NOC 视角**

- 只坏一个 Bank：请求改走配对的另一个，那个 Bank 能访问两边 DDRC
- 一对都坏：请求改走别的 Bank，对应 DDRC 关掉
- CRM 等公共模块坏了：整组 L2G 作废，即使 Bank 都是好的

---

## 4 状态和配置寄存器

### CLK_CTRL3

- 地址 `0x00a0`，读写
- 每组 2 bit：`L2G0`～`L2G7` 在 bit `[1:0]`～`[15:14]`
- `00` 强制关钟；`01` 强制开钟；`10` 自动门控（默认）
- 写成 `01` 时，所有 L2 Bank 时钟强制打开

### CLK_STATUS3

- 地址 `0x00a8`，只读
- `L2G0`～`L2G7` 各 1 bit。置位表示该组里有 Bank 时钟在跑

### GC_CR_PWR_MAN_HYSTERESIS

- `[4:0]` 是时钟管理回滞周期，不能小于 `0xf`

### GC_CR_SOFT_RESET

- 地址 `0x0100`。**已删除**：L2 还被 VPU、PCIE 访问，不能随便软复位

### GC_CR_CRM_RESET_CLK

- 地址 `0xf278`，`[1:0]`：复位期间时钟关 / 开 16 拍 / 开 64 拍 / 一直开
- 软复位已禁用，这个寄存器实际上不起作用

### GC_CR_L2_CFI_OPERATION

- 地址 `0xe000`，写通知
- `[1:0]`：`00` Invalid，`01` Flush，`10` FlushInval
- 任一 CFI 都会清掉后面两个 STATUS 寄存器

### GC_CR_L2_CFI_STATUSI / STATUSII

- STATUSI 地址 `0xe008`，STATUSII 地址 `0xe0a0`，只读
- 每个 Bank 1 bit，表示该 Bank 的 CFI 是否完成
- STATUSI：大多数 Bank
- STATUSII：编号个位为 8 或 9 的 Bank（`L2B8/9、18/19、…、78/79`）

### GC_CR_L2_BUFFER_MODE

- 地址 `0xe018`，`[1:0]` 为便签容量百分比

### GC_CR_L2_PERSIST_REDUCE

- 地址 `0xe020`
- bit 0：使能；`[21:1]`：阈值
- Bank 里有一个约 2<sup>16</sup> 周期的计数器，阈值是它的倍数

### GC_CR_L2_CMD

- 地址 `0xe028`，写通知
- bit 0：全部 persistence 清零；bit 1：清 CFI 完成标志

### GC_CR_L2G_IDLEI / IDLEII

- 地址 `0xe030` / `0xe098`，只读
- 位图和 CFI STATUS 一样，表示该 Bank 是否空闲

### GC_CR_L2G_ERROR0～3

- 每个 Bank 3 bit 错误码
- ERROR0（`0xe038`）：B0～B20
- ERROR1（`0xe040`）：B21～B41
- ERROR2（`0xe048`）：B42～B62
- ERROR3（`0xe050`）：B63（原文只列到这里）

### GC_CR_L2_FUNCTION_ENABLE

- 地址 `0xe058`

| 位 | 字段 | 含义 |
| --- | --- | --- |
| `[7:4]` | `ddr_capacity` | `0000` 2Gb / 13-bit 行；`0001` 3/4Gb；`0010` 6/8Gb；`0011` 12/16Gb；`0100` 24/32Gb |
| `[3:2]` | `hash_mode` | `0` 无；`1` PVR；`2` 可配；`3` PVR BG |
| `[1]` | `bypass_mode` | `1` 强制策略变成 1 |
| `[0]` | `MEM_SCHEDULER` | `1` 打开调度；关掉则 Bank 不再给 Scheduler 发请求 |

### Memory Scheduler 地址偏移

这些寄存器告诉硬件：物理地址的哪几位是 column / bank / row。

| 寄存器 | 地址 | 内容 |
| --- | --- | --- |
| `GC_CR_L2_MEM_SCHED_COL_OFFSET` | `0xe060` | column 的 bit0～7 各占 5 bit |
| `GC_CR_L2_MEM_SCHED_BANK_OFFSET` | `0xe068` | bank 的 bit0～4 |
| `GC_CR_L2_MEM_SCHED_ROW_PARTI` | `0xe070` | row bit0～7 |
| `GC_CR_L2_MEM_SCHED_ROW_PARTII` | `0xe078` | row bit8～15 |
| `GC_CR_L2_MEM_SCHED_ROW_PARTIII` | `0xe080` | row bit16～17 |

**同一位物理地址不能被 row 和 bank 同时指到，也不能被指两次。**

```text
ddr_bank[0] = pa[bank_id_bit0];
```

### DDR Hash 配置

- `GC_CR_DDR_HASH_CONFIG_REG0`（`0xe088`）：row bit 0～11
- `GC_CR_DDR_HASH_CONFIG_REG1`（`0xe090`）：row bit 12～23
- 约束：`0 <= 每个 row id bit < 9`

> 原文目录里还有 `GC_CR_L2G_PERF_EVENT_SEL`，正文没有字段说明。

---

## 5 设计约束

1. 回响应之前，`arid` / `awid` 必须唯一
2. 事务不能跨 256 B 边界
3. 窄传输的 `blen` 固定为 0
4. NOC 不能发独占事务
5. 原子操作必须符合 AXI 扩展原子格式
6. NOC 必须把一笔写的全部数据发给同一个 slave，再换下一个
7. 不做 atomic store；只做 atomic load / swap / compare
8. NOC 必须支持 L2 侧读返回交织

---

## 6 接口和指令

对外三张嘴：

| 接口 | 连谁 | 干什么 |
| --- | --- | --- |
| AXI5 | NOC | 读、写、原子进来；读数据和写响应回去 |
| NSP | DDR | linefill、写回、Bypass 读写 |
| socif / 寄存器 | 软件 | 刷 Cache、配便签、开关调度、看状态 |

### AXI 侧指令

| 指令 | 作用 |
| --- | --- |
| `AXI_RD` | 读 Cache 或 DDR，返回 NOC |
| `AXI_WR` | 写 Cache 或 DDR |
| `AXI_AA` | 在 Cache 行上做原子操作 |
| `CFG_INVALID` | 全部作废，不写回，Bank 变空 |
| `CFG_FLUSH` | 脏行写回；残缺行写回并作废；剩下的有效行都变干净 |
| `CFG_FLUSH_INVALID` | 脏/残缺写回，干净行也作废 |
| `CFG_REDUCE_PERSIST` | 降低时间窗内没人用的行的 persistence |
| `CFG_RST_PERSIST` | 全部 persistence 清零 |

### Cacheline 内部指令（TagCtrl → ISCHED / RCTRL）

| 名字 | 编码 | 作用 |
| --- | --- | --- |
| RD | `0x0` | 读行，返回 AXI。进 RCTRL 时 initial mask 必须已是 1 |
| WR | `0x1` | 写行。initial mask 可以是 0，为 0 时先初始化 mask RAM 再置 1 |
| AA | `0x2` | 读-改-写。initial mask 必须是 1 |
| FLUSH | `0x4` | 读出送到 NSP，清掉字节 mask，initial mask 不动 |
| WB | `0x5` | 读出送到 NSP，并清掉 initial mask |
| WB_WR | `0x6` | 从 IWDB 取这次写入，和行内数据合并后刷 DDR。initial mask 为 0 则不读 RAM |
| PREFETCH | `0x7` | 让 ISCHED 等到 linefill 完成，别人要用这数据必须在 ISCHED 里等 |

---

## 7 遗留问题

1. Bank 目前只支持窄传输且 `blen == 0`
2. IWDB 深度是参数 `IWDB_DEPTH`。如果以后支持更长的窄写，超过这个深度的写进不来
3. 软件怎么知道 CFI 做完：看 CFI STATUS，或看 idle。若看 idle，idle 必须准确，而且因时序问题 **不能打一拍寄存器**

---

## 8 硬件模块

一个 Bank 的主通路：

```text
NOC/AXI → ALIGN → TagCtrl ┬→ ISCHED → RCTRL → SRAM
                          └→ MEM_IF  → (Memory Scheduler) → DDR/NSP
写数据：ALIGN → IWDB → WDC → ISCHED / MEM_IF
```

两 Bank 共用 Memory Scheduler 和 CRM。

---

### 8.1 ALIGN（门口接待）

职责：

1. 把请求分成普通 Cache 访问和 Buffer Mode，走不同接口
2. 写 / 原子先在 IWDB 占格子，数据从 AXI-W 来了再按 index 写入
3. 把软件的 CFI 插到 TagCtrl 口上，不挡住后面的普通请求
4. 全部对齐成 64 B beat，一次最多 4 beat
5. 收齐写命令和写数据后立刻回写响应
6. 按寄存器产生降 persistence / 清 persistence 指令

**便签内存**

L2 看到的地址范围 = 便签区 + DDR 区。便签里的数据不写 DDR、不作废、不 Flush。上游已经减掉基地址，L2 收到的是偏移；`AXI.axuser` 里的 `is_buffer_mode` 标明这是便签请求。所有 Bank 共用同一片便签空间，不管 Bank ID。

软件写完必须再读，否则读数据无效。有 Bank 坏了，上游要把剩下的便签空间均匀分到好 Bank 上。

物理地址拼法：`{BankId[5:0], WayId[1:0], SetId[6:0], [7:0]}`。25% 时 way = `addr[16:15]+6`（`addr[16]` 恒 0）；50% 时 `+4`。

**写响应**

ALIGN 回完响应就把请求丢给 TagCtrl，不管后面有没有真正写入。数据冒险由 TagCtrl 接着管。写响应按顺序回。

**读数据仲裁**

同一 cacheline 的数据会进同一个 RCTRL。RCTRL 送完置 `last`。ALIGN 用轮询从多个 RCTRL 和 ROB（Bypass 读 Miss）里挑数据，但同一笔事务会尽快凑齐。

**原子数据在 IWDB 里的格式**

| 类型 | 格式 |
| --- | --- |
| 写 | `{strb[512+:32], data[0+:512]}` |
| 原子 | `{atomic_swap, atomic_size, atomic_op, dword_active[15:0], data[511:0]}` |

`atomic_swap = 0`：比较值在低位、交换值在高位；`= 1` 则反过来。比较交换的回绕边界是原子宽度的 2 倍（32 位是 8 B，64 位是 16 B，128 位是 32 B）。非 CAS 忽略这个位，有效位置看写选通。

原子宽度：`00` = 32，`01` = 64，`10` = 128（仅 PCIE CAS）。L2 需要支持 PCIE 的 16 字节 CAS。

| 操作 | 编码 | 含义 |
| --- | --- | --- |
| ADD / SUB | `0x00` / `0x01` | 无符号加减 |
| XCHG | `0x02` | 交换 |
| CMPXCHG | `0x03` | 相等才换成 operand_b |
| UMIN / IMIN | `0x04` / `0x05` | 无符号 / 有符号最小 |
| UMAX / IMAX | `0x06` / `0x07` | 无符号 / 有符号最大 |
| AND / OR / XOR | `0x08` / `0x09` / `0x0A` | 位运算 |
| FADD / FSUB | `0x10` / `0x11` | FP32 加减 |
| FMIN / FMAX | `0x15` / `0x17` | FP32 最小 / 最大 |
| FCMPXCHG | `0x13` | FP32 比较交换 |

每 4 bit `WSTRB` 压成 1 bit `dword_active`。原子操作数对应的选通必须是连续的 1，否则非法。

**插命令优先级**

```text
CFI > 清 persistence > 降 persistence > 普通读写轮询
```

降 persistence 的真实时间窗是 `{threshold, 2×threshold−1}`，ALIGN 做不到绝对准时。时钟关掉时计数器不动，这个窗口变得不确定。

---

### 8.2 TagCtrl（目录管理员）

只改 Tag，不搬货物。请求来源和优先级：

1. overflow FIFO（它不空时卡住 ALIGN）
2. hazard table（永远高于 ALIGN）
3. ALIGN

#### 8.2.1 CFI / persistence 扫全表

Invalidate 把所有行作废，不管状态，用来清复位后 Tag RAM 的随机值。Flush / FlushInval 只逐出有效且脏的行。

| 字段 | Flush | FlushInval | Invalid | 降 persist | 清 persist |
| --- | --- | --- | --- | --- | --- |
| valid | 残缺行会清 | 清 | 清 | 不动 | 不动 |
| dirty / partial | 清 | 清 | 清 | 不动 | 不动 |
| persistence | 不动 | 不动 | 清 | 仅 untouched 减 1 | 清 |
| touched_mask | 不动 | 不动 | 清 | 清 | 清 |
| initial_mask | 不动 | 脏行因发 WB 会清 | 不清 | 不动 | 不动 |
| 字节 mask | 会处理 | 不动 | 清 | 不动 | 不动 |

CFI 进行中，ALIGN→TagCtrl 的其它指令停住。lookup 各级、overflow FIFO、hazard table 不空时，不能开下一次 CFI。还有一个计数器盯着刷之前发出去的写，NSP 写响应把它减到 0，才能报 CFI 完成。

刚分配的行 `touched_mask = 0`，意思是：先假定本窗口会被用到；若没有，降 persistence 时就会减。

#### 8.2.2 查目录流水

Tag RAM 是双口，理论上每拍可读可写，但同一地址不能同时读写，所以做了转发。四级流水：

- **第 0 级**：产生 Tag 地址。和第 3 级同地址则转发。卡住时不读 RAM
- **第 1 级**：读出数据。卡住时把 RAM 输出和转发数据锁住
- **第 2 级**：用最新的目录做命中、分配、决定发给谁。可能同时触发写回和 linefill（先写 DDR，再读 DDR）
- **第 3 级**：把新状态写回 Tag

若 ISCHED 和 MEM_IF 都要发：**必须先发给 ISCHED（或同时发）**。ISCHED 满了就不能先去 DDR。

`access_counter` 用触发器做：发给 ISCHED 加一，做完减一。到上限还要再加时，流水暂停。

**两种 hazard**

- 读打中“还在飞的残缺行”：partial 可能不准。这个 hazard **后来删了**，改由 RCTRL 处理。删了之后，linefill 和写可能同时到 RCTRL：先到的写会让 linefill 先读 mask，再只填无效字节
- Bypass 写打中还在飞的脏/残缺行：必须等前面做完再合并，否则可能合到旧数据。这个仍当 hazard

#### 8.2.3 命中后怎么改目录（单扇区，精简）

读策略 0：Miss 发 LF+RD，等 linefill；干净/脏只发 RD；残缺发 LF+RD。  
读策略 1：Miss 发 BRD，不占行；Hit 和策略 0 一样。

写策略 0：一律发 WR，更新 dirty；Miss 到脏/残缺行时还要占写回表项。  
写策略 1：Miss 发 ST+WT；Hit 干净发 WT 并作废；Hit 脏/残缺发 WB_WR 并作废。

原子：和读类似，指令换成 AA，不管策略。

Flush：干净行只清 dirty；残缺发 WB 并作废；脏发 FLUSH 但行还留着。  
Inval：一律清掉。  
FlushInval：干净直接作废；脏/残缺发 WB 再作废。

只访问一半但 prefetch=256 B 时，另一半若无效/残缺，会先发一条 PREFETCH 等 linefill，再发 RD。

双扇区时规律相同：任一侧需要补洞就 linefill，`linefill_waiting` 用 2 bit 分别标记两侧。写策略 0 直接 WR；写策略 1 两侧都干净就 WT 作废，否则 WB_WR。原子不管策略。

#### 8.2.4 换哪一行

顺序：空行 → persistence 0/1/2/3（且 `access_counter == 0`）。正在干活的行不能换。

- 换脏/残缺行：WB 会清 initial mask
- 换干净行：initial mask 留到下次被别人占用再清

---

### 8.3 RCTRL（搬货物）

资源：2 个 RCTRL × 512 bit × 4096 深的数据 RAM；脏 mask 是 2 × 128 bit × 2048（每个扇区一项，字节粒度）；initial mask 每扇区 1 bit。

| 指令 | 脏 mask | 数据 RAM | initial mask |
| --- | --- | --- | --- |
| RD | 不改 | 读出还给 ALIGN | 必须已是 1 |
| WR | 第一次用：按选通写成新 mask；以后只把 1 写进去 | 按 mask 写 | 为 0 则置 1 |
| AA | 按 `dword_active` 把 1 写进去 | 先读出返回，再写回结果 | 必须已是 1 |
| WB | 读出给 DDR | initial=1 才读数据给 DDR | 清掉 |
| FLUSH | 读出给 DDR，再写成全 0 | 读出给 DDR | 必须已是 1 |
| WB_WR | 和这次写入的 mask 合并后给 DDR | initial=1：读出合并；=0：不读 RAM，直接把 IWDB 数据给 WDC | 清掉 |
| Linefill | initial=0：写成全 0；=1：只读不改 | =0：整行写入；=1：只填脏 mask 为 0 的字节 | 为 0 则置 1 |

**关键：已经脏的字节，不能被 DDR 旧数据盖掉。**

有效行的数据含义：

| Partial | 字节脏 | 含义 |
| --- | --- | --- |
| 0 | 0 | 干净且有效 |
| 0 | 1 | 脏且有效 |
| 1 | 0 | 干净但有洞，不能直接 RD，必须先 linefill |
| 1 | 1 | 脏且有效 |

流水大致四级：第 0 级按需读 RAM，第 2 级出结果/做原子，第 3～4 级写回。整数原子当拍出结果，FP32 下一拍，必要时插气泡。

写缓冲像 FIFO：满了优先把缓冲写进 SRAM、卡住读；否则先做读、新写推进缓冲。同地址的写会合并。

**CAS 时 `dword_active` 还要再收窄**：按 `atomic_swap` 左移或右移后相与，只留下真正要改的那一半（比较值那一侧）。返回值永远是改之前的旧数据。

FP32 舍入是 RNE。MIN/MAX 如果结果和原值一样，不改脏标记。CMPXCHG 只要比较命中就更新脏标记，哪怕交换值和原值相同。

加减遇到 NaN / 无穷 / 非规格数：有 NaN 就出 NaN（`0x7FC00000`）；非规格数一律刷成 0（和 CUDA 一样）。MIN/MAX/CAS 的输入是否刷 0，由 `rgx_cr_usc_lms_fp32_denormal_en` 控制。注意 MIN/MAX **比较用刷过的值，写回仍用原始操作数**。例如请求是非规格数 `0x00001234`、原值更大、使能刷 0，比较按 0 算，但写回去的是 `0x00001234`。

---

### 8.4 ISCHED（同一行的交警）

所有 TagCtrl 请求按序推进环形缓冲，保证 **同一扇区** 上 RAR / RAW / WAW / WAR 都不乱。不同扇区可以乱序。

读 Miss 要在这里等到 RCTRL 的 linefill 完成。读 Miss 打到脏行时，TagCtrl 会带上 `linefill_waiting` 和 `wb_waiting`。MEM_IF 保证写回响应先回来，所以 ISCHED 眼里永远是先 WB、后 linefill。

**预取怎么算扇区**

- `blen=0, start_beat=0, prefetch=1` 且 Miss：当成访问两扇区（会 linefill 两半）
- 同样条件但 prefetch=0：只算扇区 0
- 只访问一半且 prefetch=1，另一半无效/残缺：也当成两扇区

**拆请求**

进缓冲的是突发，发给 RCTRL 前拆成 512 bit。WB_WR 时：`beat_dirty_mask` 为 1 的拍保持 WB_WR，为 0 的拍改成 WB。

WR / AA / WB_WR 还要用 `iwdb_index` 取数；拆开后 index 要重算。

**环形缓冲**

状态存在寄存器，旁带信息存在 SRAM。可一拍给两个 RCTRL 各吐一笔。写指针碰到读指针就满（实际留 1 个空项不用）。读指针后面那条如果在等 linefill / WB，指针不能往前走；不在等待的条目可以先发出去。

**二次命中**

后面的请求打中同一行时，TagCtrl 不会再标 linefill waiting。若不处理，它会超过前面的 Miss，顺序就错了。所以 ISCHED 要把前面的 waiting **合并**到后面这条上：

- 两笔碰同一扇区：后面继承前面的 linefill waiting
- 碰同一行但不同扇区，且和前面的 waiting **没重叠**：后面可以先走
- 有重叠：同样要合并

WB waiting 按整行算，同一行直接合并。

**仲裁**：WB 通道最高；ROB 和便签请求轮询。

**Credit**：ISCHED 的写数据 FIFO 有额度。每向 IWDB 取一次减 1，消费完加 1。额度为 0 就不再取。这是为了避免和 WT 抢 IWDB 把整条写通路卡死。

---

### 8.5 WDC（写通道对账）

IWDB 只能单通道流水，所以两拨命令要先仲裁：

- 来自 ISCHED：WR、WB_WR、AA
- 来自 TagCtrl：WT、WB、WB_WR、Flush

写数据也有两路要汇到 MEM_IF：RCTRL 的 WB / WB_WR，以及 IWDB 的 WT。用 order FIFO 保证命令顺序和数据顺序一致。WT 同样要 credit，并拆成 512 bit。

---

### 8.6 Memory Scheduler

- 按对 DDR 友好的优先级出队
- 同一 cacheline 保持原顺序
- 每笔 PA 带一个 buffer index，指回 Bank 里的命令位置
- 队列深 32。输入往队尾挤，队头优先级最高；满了就挡住输入
- 等太久的请求会被提到最高优先，避免饿死

---

### 8.7 Mem_If（出门海关）

- **Scheduler**：打开时把 hash 过的地址和 buffer index 发出去；回来只认 index，地址忽略
- **NSP ID**：读写共用，最多 128 个。读还要求 ROB 有足够空位
- **写同步**：NSP 要求命令和数据一起发，所以要等 IWDB 或 RCTRL 的数据
- **ROB**：Bypass 读直接从这里回 ALIGN；同时也避免 DDR 读返回把接口堵住。必须占 **连续** 空间，粒度至少 512 bit。NSP 每拍回 256 bit，所以至少 2 拍才齐。写口 256 bit，读口 512 bit，多笔齐了之后轮询读出
- **取写数据**：Bypass 写走 IWDB；WB / FLUSH / WB_WR 走 RCTRL
- **linefill 和写回**：Scheduler 关掉时写回一定先发，没有这个问题。打开后 linefill 可能先发出去，数据先回来。依赖写回的 linefill 会标 `wb_check`，直到 TagCtrl / RCTRL 确认写回完成才能进 RCTRL

**死锁注意**：必须先占 ROB，再去做写数据同步。反了（或同一拍一起做），RCTRL 里 linefill 和写回又共一条流水，写回一堵、linefill 也堵、ROB 释放不了，队头就永远走不掉。

地址重叠：MEM_IF 自己不重排（除非 Scheduler 打开）。Scheduler 打开时也会保持同一行的顺序，所以 MEM_IF 不用再做地址 hazard。

---

## 9 面积

| 模块 | SRAM | 寄存器 | 预留 | 合计 |
| --- | --- | --- | --- | --- |
| TagCtrl | 1.6w | 0.4w | 0.4w | 2.4w |
| MEMIF | 0.5w | 0.2w | 0.2w | 0.9w |
| ISCHED | — | 0.5w | 0.3w | 0.8w |
| RCTRL | 25.5w | — | 1.5w | 27w |
| WDC + ALIGN + IWDB | — | — | 1w | 1w |
| **合计** | | | | **32.7w** |

绝大部分面积在 RCTRL 的数据 RAM 上。

---

## 10 性能

| 场景 | 延迟 |
| --- | --- |
| 读命中 | 11 拍 |
| 写 | 写命令和写数据对齐之后 1 拍（提前回响应） |
