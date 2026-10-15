# GPU MSS L2 Cache Bank Design Specification

| Item | Value |
| --- | --- |
| Title | L2 Bank Design Specification |
| Authors | liusm, zhujt, luoml |
| Version | 1.6 |
| Date | 2025-05-20 |

> **Conversion note.** This Markdown is reconstructed from `GPU_MSS_L2_Cache_Bank_TSpec.txt` (Word → TXT). Encoding artifacts (smart quotes, Verilog `'` bits), broken tables, and Word auto-numbering collisions were repaired. Figures existed only as placeholders in the source and are called out as such. Original section-number collisions (for example two `3.6` headings, persistence subsections restarting at `3.1`, WDC missing from the TOC) are corrected; the old number is noted where a later section referred to it.

---

## Revision History

| Date | Revision | Author | Description |
| --- | --- | --- | --- |
| 2024.12.26 | V1.0 | liusm | Pass the review |
| 2025.2.5 | V1.1 | liusm | Add description of lookup pipeline and touch mask |
| 2025.2.26 | V1.2 | zhujt | Update atomic special value result |
| 2025.3.11 | V1.3 | zhujt | RCTRL implement update |
| 2025.3.14 | V1.3 | liusm | Add ISCHED–RCTRL wdata interface to improve performance |
| 2025.3.25 | V1.4 | luoml | Adjust the bank group from 4 banks to 2 banks and the memory scheduler description |
| 2025.4.11 | V1.5 | zhujt | Add FP32 atomic CAS; FP32 min/max special value update |
| 2025.5.20 | V1.6 | liusm | Add DDR hash module; update memory scheduler priority |

---

## Table of Contents

- [1 Overview](#1-overview)
- [2 Features](#2-features)
- [3 Functional Description](#3-functional-description)
  - [3.1 AXI Transactions](#31-axi-transactions)
  - [3.2 Cache Organization](#32-cache-organization)
  - [3.3 Persistence and PLRU Replacement](#33-persistence-and-plru-replacement)
  - [3.4 Sector Cacheline and Prefetch](#34-sector-cacheline-and-prefetch)
  - [3.5 Atomic Transaction](#35-atomic-transaction)
  - [3.6 Buffer Mode](#36-buffer-mode)
  - [3.7 Cache Policy](#37-cache-policy)
  - [3.8 Coarse Flush](#38-coarse-flush)
  - [3.9 Data Flow](#39-data-flow)
  - [3.10 Memory Scheduler](#310-memory-scheduler)
  - [3.11 CRM](#311-crm)
  - [3.12 Clear Partial](#312-clear-partial)
  - [3.13 Redundancy](#313-redundancy)
- [4 State and Configuration](#4-state-and-configuration)
- [5 Design Constraints](#5-design-constraints)
- [6 Interface](#6-interface)
- [7 Open Issues](#7-open-issues)
- [8 Hardware Architecture Description](#8-hardware-architecture-description)
  - [8.1 ALIGN](#81-align)
  - [8.2 TagCtrl](#82-tagctrl)
  - [8.3 RCTRL](#83-rctrl)
  - [8.4 ISCHED](#84-isched)
  - [8.5 WDC](#85-wdc)
  - [8.6 Memory Scheduler](#86-memory-scheduler)
  - [8.7 Mem_If](#87-mem_if)
- [9 Area](#9-area)
- [10 Performance](#10-performance)

---

## 1 Overview

L2 cache is the last-level cache. All transactions that access DDR must pass through L2 cache. L2 cache must be able to serve multiple masters.

L2 cache is based on the AXI5 protocol. The function of L2 cache is a subset of AXI5.

L2 cache is a distributed cache: the cache can be duplicated to form a huge-capacity cache. Each duplicated cache is independent. L2 cache is easy to scale up.

L2 cache is a PIPT architecture. The input address is a physical address. This architecture connects L2 cache to DDR directly and reduces the chance of congestion in the physical layout.

L2 cache bank is the main component of L2 cache. All read / write / atomic transactions are processed in the L2 cache bank, as shown in Figure 1.

> **Figure 1 — L2 Architecture**  
> *(Figure image not present in the TXT source.)*

Transactions are routed to L2 groups. Each group has 8 cache banks and is connected to DDR directly. The L2 groups are all independent.

> **Figure 2 — L2 bank location in system**  
> *(Figure image not present in the TXT source.)*

> **Figure 3 — L2 bank architecture**  
> *(Figure image not present in the TXT source.)*

> **Note on bank count.** The overview states 8 banks per group. Later register maps and the buffer-mode table enumerate **80 banks** (`L2B0`–`L2B79`) across **8 groups** (`L2G0`–`L2G7`). Both statements are preserved as written.

---

## 2 Features

1. PIPT cache
2. 256 KB per cache bank
3. 256 B per cacheline
   - 2 sectors
   - Configurable prefetch size: 128 B, 256 B
4. Support cache / bypass read / write transactions
5. Highest linefill priority
6. Support Atomic Operation
   - Support FP32 operation
7. Support persistence level
8. Support buffer mode
   - Buffer size is configurable: 25%, 50%
9. Support NSP protocol

---

## 3 Functional Description

### 3.1 AXI Transactions

Bank supports load / store operations with different memory attributes. Based on the AXI specification, transactions can be either of type Device or Normal.

- A read or write transaction where the request has `AxCache[1]` **deasserted** is a **Device** transaction. Device transactions can be used to access a Peripheral region or Memory locations.
- A read or write transaction where the request has `AxCache[1]` **asserted** is a **Normal** transaction. Normal transactions are used to access Memory locations and are not expected to access Peripheral regions.

Device transactions shall **not** perform lookup and cacheline allocation. They are required to pass through L2 cache.

Normal transactions will perform lookup and trigger further actions based on the memory attributes `AxCache[3:0]`.

As the last-level cache in the GP3 architecture, all memory transactions pass through L2 cache. L2 cache can be regarded as the “final destination” when processing Write-through transactions. These transactions are observed by other masters when they perform lookup. All Normal memory write transactions can return a write response **before** receiving the DDR write response.

GP3 memory defines several memory-transaction behaviours. The behaviour is identified by a policy defined in SPC. AXI uses the `AxCache` signal to identify the policy:

| AxCache meaning | Policy |
| --- | --- |
| Write-back | `policy == 0` |
| Write-Through | `policy == 1` |

#### 3.1.1 Transaction request

##### 3.1.1.1 Size attribute

Refer to the AXI5 Spec.

Size indicates the maximum number of bytes in each data transfer. Size must not exceed the data width of an interface.

In the GP3 architecture, GPC, CE and FW are assumed to issue transactions with `arsize = 6` (64 bytes). PCIE and VPU may issue transactions with `arsize < 6` (64 bytes). Therefore L2 cache shall support transactions with `axsize <= 6` (64 bytes).

For a narrow transfer (`AxSize < 64` bytes), `blen` is fixed to 0.

##### 3.1.1.2 Length attribute

The Length defines the number of data transfers in a transaction.

The total number of transfers in a transaction is Size × Length. A Manager must issue the number of write data transfers according to Length. A subordinate must issue the number of read data transfers according to Length.

In GP3, the number of transfers in every transaction issued to L2 cache is limited to 2 Kbit. That means:

```text
Size × Length <= 256 bytes
```

L2 cacheline size is 256 bytes, so **no transaction accesses across a cacheline**, even for a narrow transfer.

The following rules apply to transaction length:

- For wrapping burst, Length can be 2, 4, 8 or 16.
- For fixed burst, Length can be up to 16.
- A transaction must not cross a 4 KB address boundary.

For GPC, CE and FW transactions, Length is always less than 4. For PCIE, Length could be any value, but a transaction that fetches more than 2 Kbit of data is split in the NOC. L2 cache only sees transactions with `blen <= 3`.

##### 3.1.1.3 Burst attribute

The Burst attribute describes how the address increments between transfers in a transaction. L2 cache supports **INCR** burst types.

**INCR**

All transactions must not cross a 2 Kbit address boundary:

```text
((Addr[7:0] >> AxSize) << AxSize) + AxSize × (AxLength + 1) <= 256
```

**WRAP**

The address wraps around to a lower address if an upper address limit is reached. Constraint:

```text
AxSize × Length <= 256
```

**Narrow transfer constraint**

```text
AxSize < 64 bytes  →  AxLength == 0
```

#### 3.1.2 Write and read data

When a Manager generates a transfer that is narrower than its data channel, the address and control information determine the byte lane that the transfer uses.

- When Burst is INCR or WRAP, different byte lanes are used for each data transfer in the transaction.
- When Burst is FIXED, the same byte lanes are used for each data transfer in the transaction.

> **Figure 4 — Narrow transfer**  
> *(Figure image not present in the TXT source.)*

#### 3.1.3 Request attributes

`AxCache` is overridden as cache policy. For more detail, see [3.7 Cache Policy](#37-cache-policy) (originally referenced as section 3.6).

The combination of `AxCACHE` signals indicates a memory type. The memory-type encodings follow the AXI specification.

---

### 3.2 Cache Organization

The cacheline is organized as **128 sets × 8 ways**. Each set has PLRU bits that point to the least-recently-used cacheline. Every cacheline has a tag field and a data field. The sectors in a cacheline share `addr_tag` and `persistence`. Each sector has independent `valid`, `beat_dirty_mask` and `partial` state.

The address on the AXI request interface is always aligned to 512 bits. A cacheline is 2048 bits wide.

**Table 1 — Cacheline Field**

| Signal Name | Scope | Description |
| --- | --- | --- |
| `addr_tag` | cacheline | Used to perform lookup |
| `persistence` | cacheline | Influences replacement |
| `valid` | sector | Indicates the sector is valid |
| `beat_dirty_mask` | beat | Each bit is associated with 512 bits. If some bytes in those 512 bits are dirty, the corresponding `beat_dirty_mask` bit shall be asserted. |
| `partial` | sector | Each bit is associated with 1024 bits. If some bytes in those 1024 bits are invalid, `partial` shall be asserted. |
| `access counter` | cacheline | Number of in-flight instructions on RCTRL |
| `initial mask` | sector | Indicates the cacheline is being accessed for the first time. If `initial_mask == 0`, the request shall initialize the total cacheline mask even if the request only accesses part of the cacheline. |
| `touched_mask` | cacheline | A control register specifies a time window. In that window, if a cacheline is not referenced, `touched_mask` is deasserted. |
| `Data[2047:0]` | cacheline | Cacheline data |
| `Mask[255:0]` | byte | Byte-level mask. A bit is associated with a byte. If a bit is set, the associated byte is newer than DDR and must be flushed to DDR when evicted. |

> **Figure 5 — SRAM distribution**  
> *(Figure image not present in the TXT source.)*

---

### 3.3 Persistence and PLRU Replacement

Each cacheline has a field named `persistence[1:0]`. Persistence describes how long the cacheline will stay in L2 cache.

| `persistence[1:0]` | Description |
| --- | --- |
| `2'b00` | Highest priority when selecting a cacheline to evict |
| `2'b01` | — |
| `2'b10` | — |
| `2'b11` | Lowest priority when selecting a cacheline to evict |

The persistence field is written into tag RAM when a cacheline is allocated. The field only influences replacement priority.

On lookup, TagCtrl knows the cacheline states of the specific set (valid, partial, or just dirty). TagCtrl also knows which cachelines are in-flight. These cachelines are also named **active** cachelines. Active cachelines cannot be replaced. Inactive cachelines may be invalid, dirty, or clean. Invalid cachelines always take priority to be replaced. Then the cacheline with `persistence == 0` is replaced.

```verilog
replace_way[2:0] = (&active_mask == 1'b0)       ? empty_replace_way :
                   (&persistence0_mask == 1'b0) ? persistence0_way  :
                   (&persistence1_mask == 1'b0) ? persistence1_way  :
                   (&persistence2_mask == 1'b0) ? persistence2_way  :
                                                  persistence3_way;
```

Persistence for an address region should remain stable. If any AXI request hits a cacheline, the persistence field in tag RAM and the request shall be the same. An assertion is considered to check this behaviour in system test.

A persistent L2 cacheline from a previous kernel may remain in L2 long after it has been used. L2 bank provides two methods to reduce the persistence value.

#### 3.3.1 Reduce the unreferenced cacheline persistence

A flag named `touched_mask` tracks whether a cacheline is referenced in a specific time window. Bank generates a persistence-reduce command to unroll all sets:

- If the `touched` flag of a cacheline is set, the persistence value remains stable.
- Otherwise, the persistence value decreases by 1.

**Note.** If a cacheline is only used once, the persistence value should also decrease. Therefore `touched_mask` is set only when the cacheline is **hit**. `touched_mask` is still zero when the cacheline is first allocated.

- `touched_mask` is set to 1 only when a request **hits** the cacheline.
- `touched_mask` is cleared when a reduce-persistence command or a clear-persistence command is received.

#### 3.3.2 Reset all cacheline persistence

A persistence-reset command is issued via the socif interface. Bank unrolls all sets and resets persistence to zero. The command does not change other cacheline status.

#### 3.3.3 Programming Guide

Reset all cacheline persistence:

```c
Write32(GC_CR_L2_CMD, 0x1);
```

Reduce unreferenced cacheline persistence every `0x1f_ffff_fff` cycles:

```c
Write32(GC_CR_L2_PERSIST_REDUCE, 0x3f_ffff);
```

It is possible to trigger a clear-persistence operation by `Write64(GC_CR_L2_CMD)`. To improve system robustness, L2 will ignore the second write notify.

---

### 3.4 Sector Cacheline and Prefetch

Each cacheline has two sectors. Each sector has independent state and flags. A request may access a single sector or two sectors. The prefetch size carried on `AXI.user` also has an effect.

When a bypass-read request arrives with `prefetch == 1`, prefetch is overridden to 0: prefetch size has no effect on a bypass-read request.

Each sector has INVALID, DIRTY, PARTIAL and CLEAN states:

| State | Encoding |
| --- | --- |
| INVALID | `valid = 0` |
| DIRTY | `valid = 1`; `beat_dirty_mask != 0`; `partial = 0` |
| PARTIAL | `valid = 1`; `beat_dirty_mask != 0`; `partial != 0` |
| CLEAN | `valid = 1`; `beat_dirty_mask == 0`; `partial == 0` |

Sector fields: `valid`, `beat_dirty_mask[1:0]`, `partial`.

**Linefill:** read data from DDR and write to the cacheline.

Any write transaction will **not** trigger linefill on miss. Therefore the `partial` bit sits alongside the sector.

Prefetch size is only applied to cacheable read and atomic requests. Prefetch size on write transactions is ignored.

#### 3.4.1 Access One Sector

**Table 2 — Behaviour When Accessing One Sector**

| Operation | Policy | State | Description |
| --- | --- | --- | --- |
| AXI_RD | 0 | miss | Allocate cacheline and trigger linefill. Return data after linefill completion. |
| AXI_RD | 0 | INVALID | Trigger linefill. Return data after linefill completion. |
| AXI_RD | 0 | DIRTY / CLEAN | Return data. |
| AXI_RD | 0 | PARTIAL | Trigger linefill. Merge linefill data with cacheline data. Return data after linefill completion. |
| AXI_RD | 1 | miss | Do not allocate cacheline. Bypass the request to DDR. |
| AXI_RD | 1 | INVALID / DIRTY / CLEAN / PARTIAL | Same as policy == 0. |
| AXI_ST | 0 | miss | Allocate cacheline. Write data to cacheline directly without linefill. |
| AXI_ST | 0 | INVALID | Write data to cacheline directly without linefill. |
| AXI_ST | 0 | DIRTY / CLEAN | Write data to cacheline. |
| AXI_ST | 0 | PARTIAL | Write data to cacheline. |
| AXI_ST | 1 | miss | Bypass request to DDR; do not allocate cacheline. |
| AXI_ST | 1 | INVALID / DIRTY / CLEAN / PARTIAL | Refer to [3.4.2](#342-access-two-sectors). Bypass write always invalidates a cacheline. Behaviour must consider the other sector’s status. |
| AXI_AA | — | miss | Allocate cacheline and trigger linefill. Perform AA after linefill completion. |
| AXI_AA | — | INVALID | Trigger linefill. Perform AA after linefill completion. |
| AXI_AA | — | DIRTY / CLEAN | Perform AA operation. |
| AXI_AA | — | PARTIAL | Trigger linefill. Perform AA after linefill completion. |

#### 3.4.2 Access Two Sectors

Prefetch size is carried on `axi.user`. Prefetch size is **128 B by default**; **256 B is optional**. Prefetch size is applied for read transactions because write transactions do not trigger linefill on miss. When a transaction accesses two sectors, or accesses one sector with prefetch size 256 B, both sector states must be considered.

**Table 3 — Behaviour When Accessing Two Sectors**

| Operation | Policy | Sector State | Description |
| --- | --- | --- | --- |
| AXI_RD | 0 | `{INVALID, X}` or `{X, INVALID}` | Trigger linefill. Return data after linefill completion. |
| AXI_RD | 0 | `{CLEAN/DIRTY, CLEAN/DIRTY}` | Return data. |
| AXI_RD | 0 | `{PARTIAL, CLEAN/DIRTY}` or `{CLEAN/DIRTY, PARTIAL}` | Trigger linefill. Return data after linefill completion. |
| AXI_RD | 1 | `{INVALID, INVALID}` | Do not allocate cacheline; bypass request to DDR. |
| AXI_RD | 1 | `{CLEAN/DIRTY, CLEAN/DIRTY}` | Same as policy == 0. |
| AXI_RD | 1 | `{PARTIAL, CLEAN/DIRTY}` or `{CLEAN/DIRTY, PARTIAL}` | Same as policy == 0. |
| AXI_WR | 0 | `{INVALID, INVALID}` | Allocate cacheline. Write data to cacheline without linefill. |
| AXI_WR | 0 | `{PARTIAL/CLEAN/DIRTY/INVALID, PARTIAL/CLEAN/DIRTY/INVALID}` | Write data to cacheline. |
| AXI_WR | 1 | `{INVALID, INVALID}` | Do not allocate cacheline. |
| AXI_WR | 1 | `{CLEAN, CLEAN}` | Invalidate cacheline and bypass request. |
| AXI_WR | 1 | `{PARTIAL/DIRTY, XXX}` or `{XXX, PARTIAL/DIRTY}` | Merge the cacheline with write data and flush to DDR. Invalidate cacheline. |
| AXI_AA | — | `{INVALID, INVALID}` | Allocate cacheline. Perform AA after linefill completion. |
| AXI_AA | — | `{PARTIAL/INVALID, XXX}` or `{XXX, PARTIAL/INVALID}` | Trigger linefill. Perform AA after linefill completion. |
| AXI_AA | — | `{CLEAN/DIRTY, CLEAN/DIRTY}` | Perform AA operation. |

---

### 3.5 Atomic Transaction

Exclusive access is not supported. `axlock` is always asserted to 0.

#### 3.5.1 AXI Atomic Transaction

AXI atomic transactions are **not** supported.

#### 3.5.2 AXI Extended Atomic Transaction

GPU defines two atomic operations: **atomic64** and **atomic32**. MCU L1 is able to merge atomic operations and issue them to L2. The majority of atomic operations use a single operand in combination with the destination address, with the exception of **CMPXCHG**, which has two operands.

PCIE and VPU atomic transactions are transferred to GPC-specific atomic transactions.

The AXI extended atomic transaction is defined in:

```text
01.Documents/GP3/02.Tech/02.Architecture/03.Spec/GPU_Ordering_System_Design_CHS_V0PXX.docx
```

---

### 3.6 Buffer Mode

Buffer mode is also named **scratch memory**. The storage for buffer mode is configurable. The address range is specified by register. Once the address is located in the buffer-mode address range, the request is called a **scratch request**.

Unlike a normal request, a scratch request does **not** perform lookup and therefore does not need to replace or allocate a cacheline. A fixed percentage of L2 cache capacity is used for buffer mode. If buffer mode is enabled, the address that accesses L2 cache is derived from the PA.

#### 3.6.1 SRAM Distribution

`GC_CR_L2_BUFFER_PERCENTAGE` has three configurations:

| Value | Meaning |
| --- | --- |
| `0x0` | Disable buffer mode |
| `0x1` | 25% of L2 cache |
| `0x2` | 50% of L2 cache |
| `0x3` | reserved |

The SRAM address is derived from `OffsetAddr`:

```text
set[6:0]  = Addr[14:8]
way[2:0]  = Addr[16:15] + 6    // when 25%
way[2:0]  = Addr[16:15] + 4    // when 50%
beat[1:0] = Addr[7:6]
SramAddr  = {way, set, beat}
```

#### 3.6.2 Register Configuration

| Register Name | Description |
| --- | --- |
| `GC_CR_L2_BUFFER_BASE[47:0]` | Buffer-mode base address |
| `GC_CR_L2_BUFFER_PERCENTAGE` | Buffer-mode storage: `00` = 0%, `01` = 25%, `10` = 50%, `11` = reserved |

| `BUFFER_PERCENTAGE` | Cache Size | Buffer Size | Buffer Size of each L2 Bank (80 banks) |
| --- | --- | --- | --- |
| `2'b00`, 0%, disable | 20 MB | 0 MB | 0 KB |
| `2'b01`, 25% | 15 MB | 5 MB | 64 KB |
| `2'b10`, 50% | 10 MB | 10 MB | 128 KB |
| `2'b11` | NA | NA | NA |

#### 3.6.3 Redundancy

If one bank is broken, the original request to that bank is routed to another bank. Buffer-mode storage decreases.

Example: bank 0 is broken but banks 1, 2, 3 work. The request to bank 0 is routed to bank 3. The original buffer-mode address space for bank 3 disappears. A hash algorithm modifies the buffer-mode request address to avoid accessing the broken bank 0.

> **Figure 6 — Bank redundancy for buffer mode**  
> *(Figure image not present in the TXT source.)*

#### 3.6.4 Programming Guide

Buffer mode can be configured after reset and **before** NOC issues transactions. Until the next reset, the configuration cannot change.

---

### 3.7 Cache Policy

> Originally numbered **3.6 Cache Policy** (Word auto-number collision with Buffer Mode). Internal references to “section 3.6 Cache Policy” point here.

Cache policy is derived from `axcache[0]`:

| `axcache[0]` | Cache Policy |
| --- | --- |
| 1 | `policy == 0` (cache / write-back) |
| 0 | `policy == 1` (bypass / write-through) |

**Table 4 — Behaviour of different operations**

| Operation | Cache Policy | Hit / Miss | Description |
| --- | --- | --- | --- |
| AXI_RD | 0 | — | 1. Allocate cacheline. 2. Perform linefill when miss, or hit partial / invalid sector. 3. Return data from L2C. |
| AXI_RD | 1 | Hit | 1. Issue linefill to DDR when hit partial or invalid sector.\[1\] 2. Return data from L2C. |
| AXI_RD | 1 | Miss | 1. Do not allocate cacheline. 2. Bypass request to DDR. 3. Return data to NOC. |
| AXI_WR | 0 | — | 1. Return early write response. 2. Allocate cacheline. 3. Write data to cacheline. |
| AXI_WR | 1 | Hit | 1. Return early write response. 2. Invalidate cacheline. 3. Merge cacheline data with write data if hit partial or dirty cacheline. 4. Bypass request to DDR. |
| AXI_WR | 1 | Miss | 1. Return early write response. 2. Bypass request to DDR. |
| AXI_AA | — | — | 1. Return early write response. 2. Allocate cacheline. 3. Trigger linefill when hit partial or invalid sector. 4. Perform atomic. 5. Return original data. |

\[1\] `prefetch_size` does not affect the burst length of a linefill triggered by a bypass read.

From the NOC perspective, the two policies behave as follows.

**Bypass read / write**

- A cache lookup is required for read and write transactions.
- The write response can be obtained from L2 cache.
- Read data can be obtained from L2 cache or DDR.
- Write transactions are required to be made visible at DDR.
- Allocation of write transactions is not required.
- Allocation of read transactions is not required.

**Cache read / write**

- A cache lookup is required for read and write transactions.
- The write response can be obtained from L2 cache.
- Read data can be obtained from L2 cache.
- Write transactions are **not** required to be made visible at DDR.
- Allocation of write transactions is required.
- Allocation of read transactions is required.

---

### 3.8 Coarse Flush

> Originally numbered **3.7**.

Flush is issued through the socif interface. Flush transactions and NOC transactions are issued in parallel. L2 cache has **no** ordering constraint between Flush and NOC transactions.

Flush unrolls all cachelines and writebacks all dirty cachelines. If some cacheline is still in progress, the unroll progress shall be stalled. All writeback requests are issued on the NSP interface. If the unroll process is not finished, NOC transactions shall stall.

- **Flush** and **FlushInval** are generated by register.
- **Invalidate** is generated on reset by L2 cache. The instruction is used to initialize tag RAM.

**Table 5 — CFI operation behaviour**

| Operation | Encoding | Description |
| --- | --- | --- |
| Invalid | `0x0` | Invalidation does not care about cacheline status. Recommended when all cachelines are idle. Invalid cannot affect in-flight requests in L2. If L2 receives INVALID, in-flight requests shall return correct data to NOC as normal. On reset, an invalid operation is generated to initialize tag RAM. |
| Flush | `0x1` | Flush dirty cachelines. The cacheline still persists in L2 cache. Partial cachelines are evicted to DDR. |
| FlushInval | `0x2` | Flush and invalidate cachelines. All dirty / partial cachelines shall be written back to DDR. |

#### 3.8.1 Programming Guide

It is recommended to call `Write32` to trigger a CFI operation, because `Write64` will trigger two CFI operations. If software wants to perform any operation after CFI is finished, poll CFI status until all bits are set. When CFI is finished, the function can move forward.

```c
Write32(GC_CR_L2_CFI_OPERATION, 0x1);
POLL(GC_CR_L2_CFI_STATUSI,  0xffffffff_fffffff);
POLL(GC_CR_L2_CFI_STATUSII, 0xffff);
```

It is possible to trigger CFI operations by `Write64(GC_CR_L2_CFI_OPERATION)`. To improve system robustness, L2 will ignore the second write notify.

---

### 3.9 Data Flow

> Originally numbered **3.8**.

#### 3.9.1 Cache read

> **Figure 7 — Cache read miss data flow**  
> *(Figure image not present in the TXT source.)*

Cache-read miss:

1. NOC issues a read transaction to the bank.
2. ALIGN issues cache-read transactions to TagCtrl to perform hit test.
3. Replace a clean cacheline; issue RD instruction to ISCHED to wait for linefill completion; issue initial-mask clear pulse to RCTRL to clear the cacheline initial mask; trigger linefill to MEM_IF.
4. Issue PA to Mem Scheduler.
5. Mem Scheduler returns PA and trans id.
6. Allocate ROB index and NSP ID; issue read transaction to DDR.
7. Receive DDR read data response.
8. Perform linefill.
9. Linefill complete; issue pulse to fetch instructions from ISCHED.
10. Issue RD instruction to read data.
11. Return cacheline data to ALIGN.

Cache-read hit on a **partial** cacheline uses the same data flow as cache-read miss, **without** clearing the initial mask.

> **Figure 8 — Cache read hit data flow**  
> *(Figure image not present in the TXT source.)*

Cache-read hit:

1. NOC issues a read transaction.
2. ALIGN issues a cache-read transaction and performs hit test.
3. Lookup hit. TagCtrl issues RD instruction to ISCHED.
4. ISCHED issues RD instruction to RCTRL to read data.
5. RCTRL returns data to ALIGN.

#### 3.9.2 Bypass read

> **Figure 9 — Bypass read miss data flow**  
> *(Figure image not present in the TXT source.)*

1. NOC issues a read transaction.
2. Issue bypass-read transaction and perform hit test.
3. Lookup miss; do not allocate cacheline.
4. Issue PA to Mem Scheduler.
5. Mem Scheduler returns PA and trans id.
6. Allocate NSP ID and ROB index; then issue read transaction to DDR.
7. DDR returns data. TagCtrl holds the data in ROB.
8. When the last-beat data is received, return the data through the bypass-read return channel.

#### 3.9.3 Cache write

> **Figure 10 — Cache write miss data flow**  
> *(Figure image not present in the TXT source.)*

1. Receive request on AW channel and WDATA channel.
2. Write transactions are issued to TagCtrl to perform lookup. ALIGN allocates an IWDB index to store the write data.
3. After syncing AW and WDATA and arbitrating with the read request, return an early write response. As long as there is enough space in IWDB to hold the pending write data, the request can be issued to TagCtrl. Bank returns the early write response after the request is arbitrated and all pending write data has arrived.
4. Lookup miss; allocate a cacheline. TagCtrl issues WR instruction to ISCHED. At the same time, clear the initial mask.
5. ISCHED issues IWDB request to WDC.
6. WDC fetches data from IWDB.
7. IWDB returns data and frees the IWDB index.
8. WDC returns IWDB data to ISCHED.
9. ISCHED issues WR instruction with write data to RCTRL. Return response to TagCtrl to decrease the cacheline access counter.

#### 3.9.4 Bypass write

##### 3.9.4.1 Bypass write miss

> **Figure 11 — Bypass write miss data flow**  
> *(Figure image not present in the TXT source.)*

1. Receive request on AW channel and WDATA channel.
2. Write transactions are issued to TagCtrl to perform lookup. ALIGN allocates an IWDB index to store the write data.
3. After syncing AW and WDATA and arbitrating with the read request, return an early write response. As long as there is enough space in IWDB to hold the pending write data, the request can be issued to TagCtrl. Bank returns the early write response after the request is arbitrated and all pending write data has arrived.
4. Issue WT instruction to MEMIF.
5. Issue PA to Mem Scheduler.
6. Mem Scheduler returns PA and trans id.
7. Issue WT instruction to WDC to fetch write data.
8. WDC issues IWDB request.
9. IWDB returns data and the IWDB index is freed.
10. IWDB returns data to WDC.
11. WDC returns data to TagCtrl to sync.
12. MEMIF issues the write request with write data to the scheduler.

##### 3.9.4.2 Bypass write hit dirty cacheline

> **Figure 12 — Bypass write hit on dirty cacheline data flow**  
> *(Figure image not present in the TXT source.)*

1. Receive request on AW channel and WDATA channel.
2. Write transactions are issued to TagCtrl to perform lookup. ALIGN allocates an IWDB index to store the write data.
3. After syncing AW and WDATA and arbitrating with the read request, return an early write response. As long as there is enough space in IWDB to hold the pending write data, the request can be issued to TagCtrl. Bank returns the early write response after the request is arbitrated and all pending write data has arrived.
4. Issue WB_WR instruction to MEMIF.
5. Issue PA to Mem Scheduler.
6. Mem Scheduler returns PA and trans id.
7. MEMIF issues WB_WR to WDC to fetch write data.
8. WDC issues WB_WR to ISCHED to split the request.
9. ISCHED fetches bypass-write data through WDC.
10. WDC issues IWDB request.
11. IWDB returns data and frees the IWDB index.
12. WDC returns data to ISCHED.
13. ISCHED issues WB_WR to RCTRL.
14. RCTRL merges the bypass-write data with cacheline data and returns the data to WDC.
15. WDC returns the data to MEMIF.
16. MEMIF syncs write command and write data; issues write transaction to DDR.
17. DDR returns write response.

#### 3.9.5 Writeback

> **Figure 13 — Cache read miss on dirty cacheline**  
> *(Figure image not present in the TXT source.)*

1. NOC issues a cache-read transaction to perform lookup.
2. Lookup miss and replace a dirty cacheline.
3. Issue RD instruction to ISCHED first, then issue WB instruction and linefill instruction to MEMIF.
4. Issue write-back PA to Mem Scheduler.
5. Issue linefill PA to Mem Scheduler.
6. Mem Scheduler returns PA and trans id.
7. Issue WB instruction to WDC to fetch write data.
8. WDC issues WB instruction to ISCHED to split.
9. ISCHED issues WB to RCTRL.
10. RCTRL reads data and returns it to WDC.
11. WDC returns the data to MEMIF.
12. MEMIF syncs WDATA and write request, then issues write transaction to DDR via NSP.
13. Issue read transaction to DDR via NSP.
14. Receive read response and perform linefill.
15. Linefill complete and notify ISCHED.
16. ISCHED issues RD instruction to RCTRL.
17. RCTRL returns data to ALIGN.

It is possible that a bypass write only accesses some beats, but all beats are dirty. RCTRL needs to read out all beats but only merge some beats with the bypass-write data. In ISCHED, WB_WR can be transferred to WB when the beat does not need to be merged with bypass-write data.

---

### 3.10 Memory Scheduler

> Originally numbered **3.9**.

**DDR background.** LPDDR supports up to 8 GB capacity. Each LPDDR has 16 banks. These banks are divided into 4 bank groups. Unlike GDDR, bank groups inside LPDDR are independent. Banks inside a bank group are processed in order. To utilize LPDDR bandwidth, it is better to distribute requests to different LPDDR bank groups to hide the latency of activating an LPDDR bank.

The memory scheduler interleaves requests to different LPDDR bank groups and reorders requests in a time window to select different bank groups. It is possible that a burst request accesses multiple DDR bank groups. If so, the memory scheduler should be **disabled**, because it is meaningless to schedule the request. The memory scheduler only works when a request accesses a **single** DDR bank group.

A hash function interleaves requests to different LPDDR bank groups. Figure 14 shows how PA (physical address) flows from BIF2AXI to DDR. L2 bank also instantiates a DDR Hash module to generate a new DDR bank.

Use the hash function to distribute requests to different DDR banks.

> **Figure 14 — Memory scheduler scheme**  
> *(Figure image not present in the TXT source.)*

The DDR controller remaps address to bank and row index based on configure registers `gc_l2_bank_rowid_offset[16:0][4:0]` and `gc_l2_bank_bankid_offset[3:0][4:0]`. In DDR, the LSB two bits are abandoned. Therefore the offset starts from bit 2:

```text
real_offset = gc_*_offset + 2
```

```c
hash_bank_id[3:0] = DDR_Hash(hash_pa0[32:0], gc_l2_bank_rowid_offset, gc_l2_bank_bankid_offset);
hash_pa1 = hash_pa0;

for (int i = 0; i < 4; i++) {
    real_bankid_offset[i] = gc_l2_bank_bankid_offset[i] + 2;
    hash_pa1[real_bankid_offset[i]] = hash_bank_id[i];
    bank_id[i] = hash_bank_id[i];
    assert(!(real_bankid_offset[i] inside {0,1,2,3,4,5,6,7,33}));
}

for (int i = 0; i < 17; i++) {
    real_rowid_offset[i] = gc_l2_bank_rowid_offset[i] + 2;
    rowid[i] = hash_pa0[real_rowid_offset[i]]; // ignore MSB 2 bits when configure 0.5 GB/CH
    assert(!(real_rowid_offset[i] inside {0,1,2,3,4,5,6,7,33}));
}

for (int i = 0; i < 6; i++) {
    real_columnid_offset[i] = gc_l2_bank_column_offset[i] + 2;
    columnid[i] = hash_pa0[real_columnid_offset[i]];
    assert(!(real_columnid_offset[i] inside {0,1,2,3,4,33}));
}

hash_pa1[5:0] = 0;
```

Notes:

- `hash_pa0` is 34 bits and can access a 16 GB address space. The MSB indicates which DDR channel `hash_pa0` is going to. It cannot be hashed.
- Memory scheduler cannot reorder requests that access the same cacheline, so the scheduler does not care about `hash_pa0[7:0]`.
- When configuring `gc_l2_bank_rowid_offset` and `gc_l2_bank_bankid_offset`, they cannot point to bits 7 and 6, to ensure that data in a cacheline locate in the same DDR row and DDR bank.
- It is **not recommended** to enable L2 memory scheduler and DDR hash at the same time. When two DDR hashes are both enabled, the DDR scheduler will get the raw DDR bank id. The DDR hash function is disabled.
- When the L2 memory scheduler is disabled, the address and order issued to DDR are the original address and order.

> **Figure 15 — GDDR SGRAM addressing**  
> *(Figure image not present in the TXT source. The source text notes that LPDDR5 has fixed bank width and column bits; row width has several configurations.)*

**Table 6 — DDR Row Width Configuration**

| Row Bits | DDR Density | Register Configuration |
| --- | --- | --- |
| 13 | 2 Gb | `0x0` |
| 14 | 3 Gb, 4 Gb | `0x1` |
| 15 | 6 Gb, 8 Gb | `0x2` |
| 16 | 12 Gb, 16 Gb | `0x3` |
| 17 | 24 Gb, 32 Gb | `0x4` |

Define 4 bits in `GC_CR_L2_FUNCTION_ENABLE[7:4]` to note the DDR density.

**Table 7 — Address used for scheduler**

| | `0x0` | `0x1` | `0x2` | `0x3` | `0x4` |
| --- | --- | --- | --- | --- | --- |
| RowID\[16\] | 0 | 0 | 0 | 0 | `hash_pa0[real_rowid_offset[16]]` |
| RowID\[15\] | 0 | 0 | 0 | `hash_pa0[real_rowid_offset[15]]` | `hash_pa0[real_rowid_offset[15]]` |
| RowID\[14\] | 0 | 0 | `hash_pa0[real_rowid_offset[14]]` | `hash_pa0[real_rowid_offset[14]]` | `hash_pa0[real_rowid_offset[14]]` |
| RowID\[13\] | 0 | `hash_pa0[real_rowid_offset[13]]` | `hash_pa0[real_rowid_offset[13]]` | `hash_pa0[real_rowid_offset[13]]` | `hash_pa0[real_rowid_offset[13]]` |
| RowID\[12\] | `hash_pa0[real_rowid_offset[12]]` | `hash_pa0[real_rowid_offset[12]]` | `hash_pa0[real_rowid_offset[12]]` | `hash_pa0[real_rowid_offset[12]]` | `hash_pa0[real_rowid_offset[12]]` |

> **Figure 16 — Memory scheduler integration**  
> *(Figure image not present in the TXT source.)*

When a bank needs to read or write DDR, a PA is issued to the mem scheduler and stored in a PA queue. The PA queue records the original order using a drop queue. Each PA queue selects a PA. When a PA is issued back to its own bank, the row id is sent so that its own queues output the same-row-id request; otherwise the queue needs to output different DDR-bank requests. If the queue does not have a same-row-id request or a different-bank request, it outputs the oldest request. The output request is issued to the corresponding bank with the PA.

Two banks share a memory scheduler.

- If both banks work, bank0 requests are routed to PA queue0, bank1 requests to PA queue1.
- If bank0 is broken, `PA[33]` of bank1 can be 0 or 1. When MSB is 1, the request is routed to PA queue1. When MSB is 0, the request is routed to PA queue0.
- PA queue0 always sees `PA[32] == 0`. PA queue1 always sees `PA[32] == 1`.

Scheduler priority (highest first):

1. Out-of-time oldest request
2. Request with same row id and same bank id
3. Request with different DDR bank id (based on LRU)

The **oldest request** is a request whose subsequent N−1 requests have all been outputted. N is the depth of the PA queue. Example, PA queue depth = 4, input sequence:

```text
req0 → req1 → req2 → req3 → req4 → req5 → req6
```

- Req0 becomes the oldest request when req1, 2, 3 get higher priority and are all outputted.
- Req1 becomes the oldest request when req2, 3, 4 get higher priority and are all outputted.
- If multiple requests in the PA queue satisfy the condition at the same time, the oldest request is the first request. When req2, 3, 4 get higher priority and are outputted, req0 and req1 both satisfy the definition; only req0 is treated as the oldest request.

#### 3.10.1 LRU algorithm

For timing, DDR banks are divided into 8 bank groups.

- On reset, banks 0–15 have priority from high to low.
- When the memory scheduler selects bank 1, priority from high to low is banks 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 2, 3, 0, 1.
- When the memory scheduler selects bank 6, priority from high to low is banks 4, 5, 6, 7, 0, 1, 3, 2.
- When the memory scheduler selects bank group 0, priority from high to low is banks 4, 5, 6, 7, 3, 2, 1, 0.
- When the PA queue has multiple requests that access the same bank group, the memory scheduler selects the oldest one.

> **Figure 17 — LRU algorithm update process**  
> *(Figure image not present in the TXT source.)*

---

### 3.11 CRM

> **Figure 18 — L2 CRM integration**  
> *(Figure image not present in the TXT source.)*

To reduce power consumption, CRM turns off the bank clock when no request is processed in the bank. CRM serves two banks and their memory scheduler. If one of the banks receives a request from NOC, CRM generates a clock to drive bank0, bank1 and the memory scheduler.

Each bank is connected to DDRC directly. When one bank is broken, the request is routed to the other bank. A crossbar sits between bank and DDRC. The crossbar routes the bank request to 2 DDRCs. It is not possible that two banks access two DDRCs at the same time.

The softreset port on CRM is tied to zero because GPU cannot soft-reset L2 when other remote applications access L2.

The clock and reset sequential diagram of an L2 group is shown in the source figure (image not present).

---

### 3.12 Clear Partial

> Not present in the original TOC. Appears in the body after CRM.

It is possible that an AXI master writes a whole sector or cacheline. The `partial` flag alongside the cacheline shall be cleared to avoid fetching unnecessary data from DDR when later hit by a read. Clear-partial saves DDR bandwidth and reduces pressure on DDR. If clear-partial is not supported, the bank still works; it is a performance optimization.

Due to bank microarchitecture constraints, command and data are processed independently. Clear-partial may be triggered **only** when a write is going to write a whole sector or cacheline.

**On miss:** if a cache write is going to write a whole sector or cacheline, issue `check_partial[1:0]` to RamCtrl and check whether the bytes of the corresponding sector are all written. If they are, return clear-partial to TagCtrl.

Examples:

- `start_beat = 2'b00`, `blen = 2'b01` → `check_partial[1:0] = 2'b01`
- `start_beat = 2'b00`, `blen = 2'b10` → `check_partial[1:0] = 2'b01`

**On hit:** only a partial or invalid sector is considered for clear-partial. When a cache write is going to write the whole sector, issue `check_partial[1:0]` to RamCtrl.

Examples:

- Hit status `valid[1:0] = 2'b11`, `partial[1:0] = 2'b11`, `beat_dirty_mask[3:0] = 4'b1111`; `start_beat = 2'b00`, `blen = 2'b01` → `check_partial = 2'b01`
- Hit status `valid[1:0] = 2'b11`, `partial[1:0] = 2'b11`, `beat_dirty_mask[3:0] = 4'b0111`; `start_beat = 2'b00`, `blen = 2'b10` → `check_partial = 2'b01`
- Hit status `valid[1:0] = 2'b11`, `partial[1:0] = 2'b10`, `beat_dirty_mask[3:0] = 4'b0111`; `start_beat = 2'b01`, `blen = 2'b01` → `check_partial = 2'b00`

Pseudo code:

```verilog
n_dirty_mask[3:0] = 4'd0;
case (blen)
    2'b00: n_dirty_mask = 4'b0001;
    2'b01: n_dirty_mask = 4'b0011;
    2'b10: n_dirty_mask = 4'b0111;
    2'b11: n_dirty_mask = 4'b1111;
endcase

if (hit)
    n_dirty_mask = hit_cacheline_dirty_mask | (n_dirty_mask << start_beat[1:0]);
else
    n_dirty_mask = n_dirty_mask << start_beat[1:0];

end_beat[1:0]    = start_beat[1:0] + blen[1:0];
access_mask[1:0] = {end_beat[1], start_beat[0]};

if (hit) begin
    if (access_mask[0]) // access the first sector
        if ((hit_cacheline_valid[0] && hit_cacheline_partial[0]) || ~hit_cacheline_valid[0])
            check_partial[0] = &n_dirty_mask[1:0];
    if (access_mask[1]) // access the second sector
        if ((hit_cacheline_valid[1] && hit_cacheline_partial[1]) || ~hit_cacheline_valid[1])
            check_partial[1] = &n_dirty_mask[3:2];
end else begin
    check_partial[0] = &n_dirty_mask[1:0];
    check_partial[1] = &n_dirty_mask[3:2];
end

for (i = start_beat; i <= end_beat; i++)
    mask_data[i*64 +: 64] = mask_data[i*64 +: 64] | wstrb[63:0];

clear_partial[0] = check_partial[0] ? &mask_data[127:0]   : 1'b0;
clear_partial[1] = check_partial[1] ? &mask_data[255:128] : 1'b0;
partial[1:0]     = partial[1:0] & ~clear_partial[1:0];
```

> **Figure 19 — Clear partial process flow**  
> *(Figure image not present in the TXT source.)*

`check_partial` is issued by TagCtrl and bypasses ISCHED. RamCtrl checks mask-RAM data when `check_partial` is set. If the corresponding mask RAM is all written, it returns `clr_partial` to TagCtrl to clear the partial flag.

Before clear-partial, TagCtrl can also issue a request with `check_partial` to ISCHED again.

---

### 3.13 Redundancy

> Not present in the original TOC.

To reduce tape-out cost, the design provides a method to walk around a broken bank.

#### From DFT perspective

| Module | Area (8-bank version) | Scope |
| --- | --- | --- |
| Bank | 19.2w × 8 = 153.6w | Bank |
| Memory Scheduler | 7.4w | Bank group |
| NSP Crossbar | — | — |
| CRM | — | — |
| Int_pipe | — | — |
| Regbank | — | — |
| ADB | — | — |
| Misc | — | — |

Banks are hardened independently. DFT will know if some bank is broken. For common modules such as CRM and regbank, DFT only knows if some logic in those modules is broken. If some logic in those modules is broken, the corresponding bank group is abandoned.

#### From NOC perspective

If just a single bank is broken, the request is routed to another bank. For example:

- Bank 0 is broken → the request is routed to bank 1. Bank 1 can access both DDRCs.
- Bank 0 and bank 1 are broken → the request is routed to other banks and the corresponding DDRCs are disabled.

If other modules such as CRM are broken, the request is routed to other bank groups. For example:

- If CRM is broken, the L2G is abandoned, even if all the banks work.

---

## 4 State and Configuration

### CLK_CTRL3

| | |
| --- | --- |
| Address | `0x00a0` |
| Type | RW |

| Bits | Field |
| --- | --- |
| 15:14 | L2G7 |
| 13:12 | L2G6 |
| 11:10 | L2G5 |
| 9:8 | L2G4 |
| 7:6 | L2G3 |
| 5:4 | L2G2 |
| 3:2 | L2G1 |
| 1:0 | L2G0 |

| Value | Meaning |
| --- | --- |
| `2'b00` | OFF — the domain clock is forced off |
| `2'b01` | ON — the domain clock is forced on |
| `2'b10` | AUTO — automatic clock gating is active (**default**) |

If this field is set to `2'b01`, all L2 bank clocks are forced on.

### CLK_STATUS3

| | |
| --- | --- |
| Address | `0x00a8` |
| Type | RO |

| Bits | Field |
| --- | --- |
| 7 | L2G7 |
| 6 | L2G6 |
| 5 | L2G5 |
| 4 | L2G4 |
| 3 | L2G3 |
| 2 | L2G2 |
| 1 | L2G1 |
| 0 | L2G0 |

If this field is set, the clock of one of the 80 banks is running.

### GC_CR_PWR_MAN_HYSTERESIS

| Bits | Field | Description |
| --- | --- | --- |
| 63:5 | Reserved | — |
| 4:0 | value | Number of hysteresis clock cycles for the clock-management module. This value cannot be less than `0xf`. |

### GC_CR_SOFT_RESET

| | |
| --- | --- |
| Address | `0x0100` |
| Type | RW |

| Bits | Field |
| --- | --- |
| 63:8 | Others |
| 7 | L2G |
| 6:0 | Others |

This register is **removed** because L2 is accessed by other data masters such as VPU and PCIE.

### GC_CR_CRM_RESET_CLK

| | |
| --- | --- |
| Address | `0xf278` |

| Bits | Field |
| --- | --- |
| 63:5 | Reserved |
| 1:0 | CTRL |

| Value | Name | Description |
| --- | --- | --- |
| `0x0` | OFF | Clock always OFF during reset |
| `0x1` | ON_16 | Clock ON for 16 clock cycles during reset |
| `0x2` | ON_64 | Clock ON for 64 clock cycles during reset |
| `0x3` | ON | Clock always ON during reset |

This register does **not** work because soft reset is disabled.

### GC_CR_L2_CFI_OPERATION

| | |
| --- | --- |
| Address | `0xe000` |
| Type | RW, write notify |

| Bits | Field |
| --- | --- |
| 63:2 | Reserved |
| 1:0 | CFI_OPERATION |

CFI operation encoding:

| Encoding | Operation |
| --- | --- |
| `2'b00` | INVALID |
| `2'b01` | FLUSH |
| `2'b10` | FLUSH_INVALID |

Any CFI operation clears `GC_CR_L2_CFI_STATUSI` and `GC_CR_L2_CFI_STATUSII`.

### GC_CR_L2_CFI_STATUSI

| | |
| --- | --- |
| Address | `0xe008` |
| Type | RO |

One status bit per bank. Banks whose index `mod 10` is 8 or 9 are **not** in this register; they live in STATUSII.

| Bits | Banks |
| --- | --- |
| 63:60 | L2B77, L2B76, L2B75, L2B74 |
| 59:55 | L2B73, L2B72, L2B71, L2B70, L2B67 |
| 54:50 | L2B66, L2B65, L2B64, L2B63, L2B62 |
| 49:45 | L2B61, L2B60, L2B57, L2B56, L2B55 |
| 44:40 | L2B54, L2B53, L2B52, L2B51, L2B50 |
| 39:35 | L2B47, L2B46, L2B45, L2B44, L2B43 |
| 34:30 | L2B42, L2B41, L2B40, L2B37, L2B36 |
| 29:25 | L2B35, L2B34, L2B33, L2B32, L2B31 |
| 24:20 | L2B30, L2B27, L2B26, L2B25, L2B24 |
| 19:15 | L2B23, L2B22, L2B21, L2B20, L2B17 |
| 14:10 | L2B16, L2B15, L2B14, L2B13, L2B12 |
| 9:5 | L2B11, L2B10, L2B7, L2B6, L2B5 |
| 4:0 | L2B4, L2B3, L2B2, L2B1, L2B0 |

### GC_CR_L2_CFI_STATUSII

| | |
| --- | --- |
| Address | `0xe0a0` |
| Type | RO |

| Bits | Bank |
| --- | --- |
| 15 | L2B79 |
| 14 | L2B78 |
| 13 | L2B69 |
| 12 | L2B68 |
| 11 | L2B59 |
| 10 | L2B58 |
| 9 | L2B49 |
| 8 | L2B48 |
| 7 | L2B39 |
| 6 | L2B38 |
| 5 | L2B29 |
| 4 | L2B28 |
| 3 | L2B19 |
| 2 | L2B18 |
| 1 | L2B9 |
| 0 | L2B8 |

### GC_CR_L2_BUFFER_MODE

| | |
| --- | --- |
| Address | `0xe018` |
| Type | RW |

| Bits | Field |
| --- | --- |
| 63:2 | Reserved |
| 1:0 | BUFFER_MODE_PERCENTAGE |

### GC_CR_L2_PERSIST_REDUCE

| | |
| --- | --- |
| Address | `0xe020` |
| Type | RW |

| Bits | Field |
| --- | --- |
| 63:32 | Reserved |
| 21:1 | PERSIST_REDUCE_THRESHOLD |
| 0 | PERSIST_REDUCE_ENABLE |

A 16-bit counter sits in L2_bank. This counter can reach about 2<sup>16</sup> cycles. The threshold specifies the multiplier of the maximum counter value.

### GC_CR_L2_CMD

| | |
| --- | --- |
| Address | `0xe028` |
| Type | RW, write notify |

| Bits | Field |
| --- | --- |
| 63:2 | Reserved |
| 1 | L2_CFI_CLR |
| 0 | PERSISTENT_RST_ALL |

### GC_CR_L2G_IDLEI

| | |
| --- | --- |
| Address | `0xe030` |
| Type | RO |

Bit mapping is the same as `GC_CR_L2_CFI_STATUSI` (one idle bit per bank, same bank list).

### GC_CR_L2G_IDLEII

| | |
| --- | --- |
| Address | `0xe098` |
| Type | RO |

Bit mapping is the same as `GC_CR_L2_CFI_STATUSII`.

### GC_CR_L2G_ERROR3

| | |
| --- | --- |
| Address | `0xe050` |
| Type | RO |

| Bits | Field |
| --- | --- |
| 63:3 | reserved |
| 2:0 | L2B63_ERR_CODE |

### GC_CR_L2G_ERROR2

| | |
| --- | --- |
| Address | `0xe048` |
| Type | RO |

| Bits | Field |
| --- | --- |
| 62:60 | L2B62_ERR_CODE |
| 59:57 | L2B61_ERR_CODE |
| 56:54 | L2B60_ERR_CODE |
| 53:51 | L2B59_ERR_CODE |
| 50:48 | L2B58_ERR_CODE |
| 47:45 | L2B57_ERR_CODE |
| 44:42 | L2B56_ERR_CODE |
| 41:39 | L2B55_ERR_CODE |
| 38:36 | L2B54_ERR_CODE |
| 35:33 | L2B53_ERR_CODE |
| 32:30 | L2B52_ERR_CODE |
| 29:27 | L2B51_ERR_CODE |
| 26:24 | L2B50_ERR_CODE |
| 23:21 | L2B49_ERR_CODE |
| 20:18 | L2B48_ERR_CODE |
| 17:15 | L2B47_ERR_CODE |
| 14:12 | L2B46_ERR_CODE |
| 11:9 | L2B45_ERR_CODE |
| 8:6 | L2B44_ERR_CODE |
| 5:3 | L2B43_ERR_CODE |
| 2:0 | L2B42_ERR_CODE |

### GC_CR_L2G_ERROR1

| | |
| --- | --- |
| Address | `0xe040` |
| Type | RO |

| Bits | Field |
| --- | --- |
| 62:60 | L2B41_ERR_CODE |
| 59:57 | L2B40_ERR_CODE |
| 56:54 | L2B39_ERR_CODE |
| 53:51 | L2B38_ERR_CODE |
| 50:48 | L2B37_ERR_CODE |
| 47:45 | L2B36_ERR_CODE |
| 44:42 | L2B35_ERR_CODE |
| 41:39 | L2B34_ERR_CODE |
| 38:36 | L2B33_ERR_CODE |
| 35:33 | L2B32_ERR_CODE |
| 32:30 | L2B31_ERR_CODE |
| 29:27 | L2B30_ERR_CODE |
| 26:24 | L2B29_ERR_CODE |
| 23:21 | L2B28_ERR_CODE |
| 20:18 | L2B27_ERR_CODE |
| 17:15 | L2B26_ERR_CODE |
| 14:12 | L2B25_ERR_CODE |
| 11:9 | L2B24_ERR_CODE |
| 8:6 | L2B23_ERR_CODE |
| 5:3 | L2B22_ERR_CODE |
| 2:0 | L2B21_ERR_CODE |

### GC_CR_L2G_ERROR0

| | |
| --- | --- |
| Address | `0xe038` |
| Type | RO |

| Bits | Field |
| --- | --- |
| 62:60 | L2B20_ERR_CODE |
| 59:57 | L2B19_ERR_CODE |
| 56:54 | L2B18_ERR_CODE |
| 53:51 | L2B17_ERR_CODE |
| 50:48 | L2B16_ERR_CODE |
| 47:45 | L2B15_ERR_CODE |
| 44:42 | L2B14_ERR_CODE |
| 41:39 | L2B13_ERR_CODE |
| 38:36 | L2B12_ERR_CODE |
| 35:33 | L2B11_ERR_CODE |
| 32:30 | L2B10_ERR_CODE |
| 29:27 | L2B9_ERR_CODE |
| 26:24 | L2B8_ERR_CODE |
| 23:21 | L2B7_ERR_CODE |
| 20:18 | L2B6_ERR_CODE |
| 17:15 | L2B5_ERR_CODE |
| 14:12 | L2B4_ERR_CODE |
| 11:9 | L2B3_ERR_CODE |
| 8:6 | L2B2_ERR_CODE |
| 5:3 | L2B1_ERR_CODE |
| 2:0 | L2B0_ERR_CODE |

### GC_CR_L2_FUNCTION_ENABLE

| | |
| --- | --- |
| Address | `0xe058` |
| Type | RW |

| Bits | Field |
| --- | --- |
| 63:8 | Reserved |
| 7:4 | `ddr_capacity` |
| 3:2 | `hash_mode` |
| 1 | `bypass_mode` |
| 0 | `MEM_SCHEDULER` |

`ddr_capacity`:

| Value | Meaning |
| --- | --- |
| `4'b0000` | 2 Gb, row id is 13 bits |
| `4'b0001` | 3 Gb / 4 Gb, row id is 14 bits |
| `4'b0010` | 6 Gb / 8 Gb, row id is 15 bits |
| `4'b0011` | 12 Gb / 16 Gb, row id is 16 bits |
| `4'b0100` | 24 Gb / 32 Gb, row id is 17 bits |
| others | illegal configuration |

`bypass_mode`:

| Value | Meaning |
| --- | --- |
| 1 | Override the cache policy to 1 |
| 0 | Cache policy is derived from `axcache` |

`mem_scheduler`:

| Value | Meaning |
| --- | --- |
| 1 | Enable memory scheduler |
| 0 | Disable memory scheduler. If disabled, bank does not issue any request to the memory scheduler. |

`hash_mode`:

| Value | Meaning |
| --- | --- |
| `0x0` | No hash |
| `0x1` | PVR hash |
| `0x2` | Configurable hash |
| `0x3` | PVR BG hash |

### GC_CR_L2_MEM_SCHED_COL_OFFSET

| | |
| --- | --- |
| Address | `0xe060` |
| Type | RW |

| Bits | Field |
| --- | --- |
| 63:61 | reserved |
| 60:56 | column_id_bit7 |
| 52:48 | column_id_bit6 |
| 44:40 | column_id_bit5 |
| 36:32 | column_id_bit4 |
| 28:24 | column_id_bit3 |
| 20:16 | column_id_bit2 |
| 12:8 | column_id_bit1 |
| 4:0 | column_id_bit0 |

### GC_CR_L2_MEM_SCHED_BANK_OFFSET

| | |
| --- | --- |
| Address | `0xe068` |
| Type | RW |

| Bits | Field |
| --- | --- |
| 36:32 | bank_id_bit4 |
| 28:24 | bank_id_bit3 |
| 20:16 | bank_id_bit2 |
| 12:8 | bank_id_bit1 |
| 4:0 | bank_id_bit0 |

```text
ddr_bank[0] = pa[bank_id_bit0];
```

Any bit of PA can only be pointed to once by either a row offset or a bank offset. It is illegal to assign multiple bank offsets or multiple row offsets to the same position of PA.

### GC_CR_L2_MEM_SCHED_ROW_PARTI

| | |
| --- | --- |
| Address | `0xe070` |
| Type | RW |

| Bits | Field |
| --- | --- |
| 60:56 | row_id_bit7 |
| 52:48 | row_id_bit6 |
| 44:40 | row_id_bit5 |
| 36:32 | row_id_bit4 |
| 28:24 | row_id_bit3 |
| 20:16 | row_id_bit2 |
| 12:8 | row_id_bit1 |
| 4:0 | row_id_bit0 |

Any bit of PA can only be pointed to once by either a row offset or a bank offset.

### GC_CR_L2_MEM_SCHED_ROW_PARTII

| | |
| --- | --- |
| Address | `0xe078` |
| Type | RW |

| Bits | Field |
| --- | --- |
| 60:56 | row_id_bit15 |
| 52:48 | row_id_bit14 |
| 44:40 | row_id_bit13 |
| 36:32 | row_id_bit12 |
| 28:24 | row_id_bit11 |
| 20:16 | row_id_bit10 |
| 12:8 | row_id_bit9 |
| 4:0 | row_id_bit8 |

Any bit of PA can only be pointed to once by either a row offset or a bank offset.

### GC_CR_L2_MEM_SCHED_ROW_PARTIII

| | |
| --- | --- |
| Address | `0xe080` |
| Type | RW |

| Bits | Field |
| --- | --- |
| 63:13 | reserved |
| 12:8 | row_id_bit17 |
| 4:0 | row_id_bit16 |

Any bit of PA can only be pointed to once by either a row offset or a bank offset.

### GC_CR_DDR_HASH_CONFIG_REG0

| | |
| --- | --- |
| Address | `0xe088` |
| Type | RW |

| Bits | Field |
| --- | --- |
| 63:48 | reserved |
| 47:44 | Row id bit 11 |
| 43:40 | Row id bit 10 |
| 39:36 | Row id bit 9 |
| 35:32 | Row id bit 8 |
| 31:28 | Row id bit 7 |
| 27:24 | Row id bit 6 |
| 23:20 | Row id bit 5 |
| 19:16 | Row id bit 4 |
| 15:12 | Row id bit 3 |
| 11:8 | Row id bit 2 |
| 7:4 | Row id bit 1 |
| 3:0 | Row id bit 0 |

### GC_CR_DDR_HASH_CONFIG_REG1

| | |
| --- | --- |
| Address | `0xe090` |
| Type | RW |

| Bits | Field |
| --- | --- |
| 63:48 | reserved |
| 47:44 | Row id bit 23 |
| 43:40 | Row id bit 22 |
| 39:36 | Row id bit 21 |
| 35:32 | Row id bit 20 |
| 31:28 | Row id bit 19 |
| 27:24 | Row id bit 18 |
| 23:20 | Row id bit 17 |
| 19:16 | Row id bit 16 |
| 15:12 | Row id bit 15 |
| 11:8 | Row id bit 14 |
| 7:4 | Row id bit 13 |
| 3:0 | Row id bit 12 |

Constraint: `0 <= Row id bit < 9`.

> The original Word TOC also lists `GC_CR_L2G_PERF_EVENT_SEL`. No field description for that register appears in the TXT body.

---

## 5 Design Constraints

5.1 The `arid` / `awid` shall be unique before returning a response.

5.2 All transactions cannot cross a 2 Kbit address boundary.

5.3 All `blen` of narrow transfers are fixed to 0.

5.4 NOC cannot issue exclusive transactions.

5.5 All atomic transactions must be compatible with AXI Extended Atomic transactions.

5.6 NOC must issue all the write data to a slave, then switch to another slave.

5.7 Atomic store is not supported. L2 supports atomic load, atomic swap, and atomic compare.

5.8 NOC must support read-return interleave at the L2 side.

---

## 6 Interface

### Parameters

### AXI instruction encoding

| Instruction | Description |
| --- | --- |
| AXI_RD | Read cacheline data or DDR data and return to NOC |
| AXI_WR | Write data to cacheline or DDR |
| AXI_AA | Perform atomic operation on cacheline |
| CFG_INVALID | Invalidate all cachelines without flushing dirty cachelines to DDR. After invalidation, L2 bank is empty. |
| CFG_FLUSH | Flush all dirty cachelines to DDR. Partial cachelines are flushed and invalidated. After flush, all valid cachelines in L2 are clean. |
| CFG_FLUSH_INVALID | Flush all dirty / partial cachelines and invalidate all clean cachelines. |
| CFG_REDUCE_PERSIST | Reduce all unreferenced cacheline persistence in a time window |
| CFG_RST_PERSIST | Reset all cacheline persistence |

### Cacheline instruction encoding

| Name | Encoding | Description |
| --- | --- | --- |
| RD | `0x0` | Read cacheline and return data to AXI5. The initial mask **must** be asserted when the instruction is decoded in RCTRL. |
| WR | `0x1` | Write cacheline. The initial mask can be 0 when the instruction is decoded in RCTRL. When the initial mask is 0, the instruction shall set the initial mask to 1 and initialize mask RAM. |
| AA | `0x2` | Read cacheline data, modify data, then write to cacheline. The initial mask **must** be asserted when the instruction is decoded in RCTRL. |
| FLUSH | `0x4` | Read out cacheline data to the NSP bus. All byte masks shall be cleared. The initial mask keeps unchanged. The initial mask **must** be asserted when the instruction is decoded in RCTRL. |
| WB | `0x5` | Read out cacheline data to the NSP bus. The initial mask **must** be asserted when the instruction is decoded in RCTRL. The initial mask **must** be cleared when the instruction is decoded. |
| WB_WR | `0x6` | Fetch data from IWDB, merge the write data with cacheline data, and flush to DDR. If sector initial mask is 0, do not read mask RAM and data RAM. If initial mask is 1, read mask RAM and data RAM. |
| PREFETCH | `0x7` | Special instruction that tells ISCHED to wait for linefill completion. If any request wants to access the data, it must wait in ISCHED. After linefill completion, the instruction is cleared. Used for linefill. |

---

## 7 Open Issues

7.1 Bank only supports narrow transfer with `blen == 0`.

7.2 IWDB depth is limited to parameter `IWDB_DEPTH`. Write transactions with `blen` greater than `IWDB_DEPTH` are not accepted by the bank if narrow transfer with `blen >= IWDB_DEPTH` is supported.

7.3 How does software know that CFI is finished? Track CFI status in a configure register, or read idle status. If idle status is used, idle shall be accurate. The idle signal cannot be registered due to a timing issue.

---

## 8 Hardware Architecture Description

### 8.1 ALIGN

#### 8.1.1 Function description

1. Split AXI requests into cache access and buffer-mode access; generate independent interfaces.
2. Assign an IWDB index for write / atomic operations and send the index downstream with the request interface. Store the assigned index in a FIFO; pop the FIFO after receiving data from the AXI-W channel; then send write data to IWDB using the assigned index as the IWDB address.
3. Process CFI commands: receive a CFI request from a configure register and insert it onto the TagCtrl interface; do not block subsequent requests.
4. Convert all requests to bus-width alignment. Output burst size is a fixed 64 bytes; output burst length depends on the request burst length and supports up to 4 bursts.
5. Generate a write response after receiving the AW request and the corresponding data.
6. Generate a persistence-update instruction according to the configure register.

##### Buffer-mode process

Buffer Mode has a dedicated physical address space for GP3. The input request address range of L2 is buffer-mode space plus DDR space. Data in Buffer Space is not written to DDR, not invalidated, and not flushed.

If the request address is in the range `{buffer_mode_base_addr, buffer_mode_base_addr + buffer_size}`, it is mapped into the buffer-mode area before L2. All L2 banks use the same buffer-mode space and do not care about Bank ID. The buffer-mode address received by L2 is an **offset address** (the buffer-mode base address has already been subtracted). The `is_buffer_mode` flag on the `AXI.axuser` interface indicates a buffer-mode request.

Software needs to **read buffer mode after writing it**; otherwise the read-return data is invalid.

If some banks are broken, reduce the buffer-mode space and skip the broken bank. The remaining buffer-mode space should be evenly distributed to different banks. The upstream needs to guarantee this; see [3.13 Redundancy](#313-redundancy).

##### Write Response

ALIGN can return a write response directly after receiving the write request and write data, then send the request to TagCtrl on the `align_tagctrl` interface. It does not need to wait for the write data to complete to the cacheline or DDR, and does not care whether the operation is a cacheable write or a bypass (uncacheable) write. TagCtrl must handle the data hazard after ALIGN returns the write response upstream. ALIGN returns write responses **in order**.

##### Read Data Arbitration

ISCHED must make sure that data within one cacheline is distributed to the same RCTRL, so for one request from ALIGN to TagCtrl, it always accesses the same RCTRL. RCTRL sets the `last` flag when it returns all required data.

For an AXI request that does not cross a cacheline, ALIGN selects returned data from multiple RCTRLs with RR arbitration, but returns the data of the same transaction as soon as possible.

For a bypass-read miss, the read-return data comes from ROB directly, not through RCTRL. ALIGN arbitrates between read-return data from multiple RCTRLs and ROB with an RR strategy following the rule above.

##### Atomic Data Format

ALIGN writes data and STRB to IWDB for write and atomic requests. The data format in IWDB is shown below. L2 needs to support atomic CAS 16-byte for PCIE.

| Operation | Data format |
| --- | --- |
| write | `{strb[512+:32], data[0+:512]}` |
| atomic | `{atomic_swap[535+:1], atomic_size[533+:2], atomic_op[528+:5], dword_active[512+:16], data[0+:512]}` |

`atomic_swap` is taken from `AXI.awuser`:

- `atomic_swap = 0` — compare data in the low-bit position, swap data in the high-bit position.
- `atomic_swap = 1` — compare data in the high-bit position, swap data in the low-bit position.

Wrap boundary is double the atomic size for compare-and-swap:

- 8-byte wrap boundary for atomic 32
- 16-byte wrap boundary for atomic 64
- 32-byte wrap boundary for atomic 128

Ignore `atomic_swap` for non-CAS operations. AXI write strobe indicates the valid data position.

| Atomic information | Operand data |
| --- | --- |
| atomic 32 CAS, `atomic_swap=0`, `wstrb='hFF` | compare = `wdata[0+:32]`; swap = `wdata[32+:32]` |
| atomic 32 CAS, `atomic_swap=1`, `wstrb='hFF` | compare = `wdata[32+:32]`; swap = `wdata[0+:32]` |
| atomic 64 CAS, `atomic_swap=1`, `wstrb='hFFFF_0000` | compare = `wdata[192+:64]`; swap = `wdata[128+:64]` |

`atomic_size` from `AXI.awuser`:

| Atomic size | Encode |
| --- | --- |
| atomic 32 | `2'b00` |
| atomic 64 | `2'b01` |
| atomic 128 (PCIE & CAS atomic only) | `2'b10` |

`atomic_op` is assembled from `AXI.awatop`, `AXI.awuser.extended_atomic_opcode`, and `AXI.awuser.is_fp32`:

| Atomic operation | Opcode | Description |
| --- | --- | --- |
| AA_OP_ADD | `'h0_0` | Unsigned Addition: `memory(addr) = memory(addr) + operand_a`. `dword_active = 1` indicates `operand_a`. |
| AA_OP_SUB | `'h0_1` | Unsigned Subtraction: `memory(addr) = memory(addr) - operand_a` |
| AA_OP_XCHG | `'h0_2` | Exchange: `memory(addr) = operand_a` (AXI atomic swap) |
| AA_OP_CMPXCHG | `'h0_3` | Compare-Exchange: if `memory(addr) == operand_a` then `memory(addr) = operand_b` |
| AA_OP_UMIN | `'h0_4` | Unsigned Minimum: if `memory(addr) > operand_a` then `memory(addr) = operand_a` |
| AA_OP_IMIN | `'h0_5` | Signed Minimum (AXI SMIN): if `memory(addr) > operand_a` then `memory(addr) = operand_a` |
| AA_OP_UMAX | `'h0_6` | Unsigned Maximum: if `memory(addr) < operand_a` then `memory(addr) = operand_a` |
| AA_OP_IMAX | `'h0_7` | Signed Maximum (AXI SMAX): if `memory(addr) < operand_a` then `memory(addr) = operand_a` |
| AA_OP_AND | `'h0_8` | Bitwise AND: `memory(addr) = memory(addr) AND operand_a` |
| AA_OP_OR | `'h0_9` | Bitwise OR: `memory(addr) = memory(addr) OR operand_a` |
| AA_OP_XOR | `'h0_A` | Bitwise XOR (AXI EOR): `memory(addr) = memory(addr) XOR operand_a` |
| AA_OP_FADD | `'h1_0` | FP32 Addition: `memory(addr) = memory(addr) + operand_a` |
| AA_OP_FSUB | `'h1_1` | FP32 Subtraction: `memory(addr) = memory(addr) - operand_a` |
| AA_OP_FMIN | `'h1_5` | FP32 Minimum: if `memory(addr) > operand_a` then `memory(addr) = operand_a` |
| AA_OP_FMAX | `'h1_7` | FP32 Maximum: if `memory(addr) > operand_a` then `memory(addr) = operand_a` |
| AA_OP_FCMPXCHG | `'h1_3` | FP32 compare-and-swap: if `memory(addr) == operand_a` then `memory(addr) = operand_b` |

Compress each 4-bit AXI `WSTRB` to 1-bit `dword_active`:

```text
dword_active[i] = &wstrb[4*i +: 4],   (0 <= i < 16)
```

This avoids increasing IWDB width for atomic operations.

For atomic operands, the corresponding write strobe must be a **continuous sequence of 1**. Other write strobes are illegal.

| Atomic operation | Illegal write strobe |
| --- | --- |
| atomic 32, not CAS | `(&wstrb[i*4+:4] != 1) && (|wstrb[i*4+:4] == 1)` — not a continuous 4-bit 1 |
| atomic 64, not CAS | `(&wstrb[i*8+:8] != 1) && (|wstrb[i*8+:8] == 1)` — not a continuous 8-bit 1 |
| atomic 32, CAS | `(&wstrb[i*8+:8] != 1) && (|wstrb[i*8+:8] == 1)` — not a continuous 8-bit 1 |
| atomic 64, CAS | `(&wstrb[i*16+:16] != 1) && (|wstrb[i*16+:16] == 1)` — not a continuous 16-bit 1 |
| atomic 128, CAS | `(&wstrb[i*32+:32] != 1) && (|wstrb[i*32+:32] == 1)` — not a continuous 32-bit 1 |

##### CFI Process

ALIGN generates a CFI command to TagCtrl after receiving a CFI command from software through a configure register.

ALIGN inserts a CFI instruction on the `align_cmu` interface after receiving the CFI request, then ALIGN can process subsequent requests as normal.

CFI operation flow:

1. Set `GC_CR_L2_CFI_OPERATION` to indicate invalidate-only, flush-only, or flush-and-invalidate.
2. Set `GC_CR_L2_CFI_TRIGGER` so hardware starts CFI.
3. Hardware sets `GC_CR_L2_CFI_COMPLETE` after CFI completes. Driver polls `GC_CR_L2_CFI_COMPLETE` until CFI completes.
4. Set `GC_CR_L2_CFI_COMPLETE_CLR` to clear the CFI-complete flag. CFI operation is done.

##### Persistence Update

L2 supports two modes to update the persistence value in TAG RAM via configure register:

- Configure `GC_CR_L2_PERSIST_REDUCE_ENABLE`: reduce the persistence value if the cacheline is not accessed within the timing window; generate `CFG_REDUCE_PERSIST` on the `align_cmu` interface.
- Configure `GC_CR_L2_PERSIST_CLR_ALL`: reset all persistence values in TAG RAM; generate `CFG_RST_PERSIST` on the `align_cmu` interface.

**Note.** All cachelines share the same timing window. The counter increases after system reset is released, rolls back, and reduces the persistence value if the counter equals the threshold. Persistence decreases once after twice the timing window in the worst case.

ALIGN cannot reduce the persistence value exactly. The real timing window is `{threshold, 2*threshold − 1}`.

If the clock is disabled, this timing-window counter remains unchanged, and the timing-window reduce-persistence value is indeterminate in that case.

Insert-command priority:

```text
CFG_CFI > CFG_RST_PERSIST > CFG_REDUCE_PERSIST > RR(AXI RD/WR request)
```

#### 8.1.2 Hardware description

> **Figure 20 — ALIGN microarchitecture**  
> *(Figure image not present in the TXT source.)*

##### IWDB Index Assignment

ALIGN allocates an IWDB index for write and atomic requests. After receiving a write command, ALIGN allocates an IWDB index for this request according to the write burst length. It pushes the allocated IWDB index, burst length and index offset to the index FIFO; pops the index FIFO after receiving write data; and writes data to IWDB according to the information in the index FIFO. The index FIFO is popped once for each write-last transaction. IWDB will make sure the data written to SRAM can then be read.

ALIGN compresses the original AXI write strobe, then sends the compressed `dword_active` and `dword_partial` to IWDB.

**Index FIFO data encode**

| Data | Description |
| --- | --- |
| `[0 +: IWDB_DEPTH_BITS]`, IWDB index | Data index within IWDB |
| `[ +: AXI_AWLEN_WIDTH]`, burst length | `0`: burst 1; `1`: burst 2. Remove `awlen`, increase IWDB index until write last |
| `[ +: 2]` | `atomic_size` |
| `[ +: 5]` | `atomic_op` |
| `[ +: 1]` | `atomic_wrap` |
| `[ +: 1]` | `is_atomic_req` |

##### Write Response

ALIGN generates and returns a write response to upstream directly after receiving the write command and write data. It is always an early write response.

##### Buffer Mode

There is a 1-bit flag on the `AXI.axuser` interface that indicates whether this is a buffer-mode request.

```text
Physical address = {BankId[5:0], WayId[1:0], SetId[6:0], [7:0]}
```

| Buffer-mode percentage | Cacheline address | Offset Address | Comment |
| --- | --- | --- | --- |
| 25% | set | `request_addr[14:8]` | 0–127 |
| 25% | way | `request_addr[16:15] + 6` | L2 adds offset 6 to get the final `way_id`. `request_addr[16]` is always 0. |
| 25% | beat | `request_addr[7:6]` | 512-bit alignment |
| 50% | set | `request_addr[14:8]` | 0–127 |
| 50% | way | `request_addr[16:15] + 4` | L2 adds offset 4 to get the final `way_id` |
| 50% | beat | `request_addr[7:6]` | 512-bit alignment |

---

### 8.2 TagCtrl

#### 8.2.1 Function Description

> **Figure 21 — TagCtrl Microarchitecture**  
> *(Figure image not present in the TXT source.)*

##### 8.2.1.1 CFI / Persistence unroll

L2 cache supports coarse CFI operations: Flush, Flush Invalidate and Invalidate. Invalidate is generated on reset or by register configuration. This instruction unrolls all sets and invalidates all cachelines regardless of cacheline status. It is used to clear the random value in tag RAM on reset.

For Flush and FlushInvalidate (generated by register configuration), the instruction is issued to perform lookup. Only valid and dirty cachelines are evicted. CFI clears the `valid` and `beat_dirty_mask` fields.

**Table 8 — Cacheline update under CFI and persistence operation**

| Field | FLUSH | FLUSH_INVALID | INVALID | REDUCE_PERSIST | RST_PERSIST |
| --- | --- | --- | --- | --- | --- |
| valid | no\[1\] | clear | clear | no | no |
| beat_dirty_mask | clear | clear | clear | no | no |
| partial | clear\[1\] | clear | clear | no | no |
| persistence | no | no | clear | reduce\[4\] | clear |
| addr | no | no | clear | no | no |
| access_counter | no | no | no | no | no |
| initial_mask | no | no\[2\] | no\[3\] | no | no |
| touched_mask | no | no | clear | clear | clear |
| byte_mask | yes | no | clear | no | no |
| data | no | no | clear | no | no |

\[1\] A partial cacheline is invalidated when flushed. These cachelines are written back to memory rather than flushed, because there is no “partial but clean” cacheline status.

\[2\] Flush-invalid does not issue a clear-initial-mask pulse to RCTRL when hitting a clean cacheline. For a dirty cacheline, the initial mask is cleared because flush-invalid issues a WB instruction to ISCHED. WB always clears the initial mask.

\[3\] Invalid does not clear the initial mask. After invalidation, all cachelines are invalid. Any request that allocates a cacheline will issue clear-initial-mask to RCTRL.

\[4\] Reduce persistence only when `touched_mask` is 0.

When a dirty cacheline is flushed, it becomes a clean cacheline. An entry is allocated in the in-flight writeback buffer. If any request hits the cacheline, a cacheline instruction is generated and issued to ISCHED. It is impossible to be replaced while flush is in-flight.

While CFI is ongoing, other instructions stall on the ALIGN–TagCtrl interface.

CFI cannot be executed when the lookup stages, overflow FIFO and hazard table are not empty.

- `touched_mask` is set to 0 when a cacheline is allocated. This assumes the cacheline will be referenced in the current time window. If it is not referenced, the persistence value decreases and `touched_mask` is cleared when `REDUCE_PERSIST` executes.
- A counter tracks pending write requests before the CFI operation and is triggered by CFI operations. Any write response from NSP decreases the counter. Only when the counter is zero can CFI complete be issued to regbank.
- The next CFI operation can be issued only when the previous CFI is finished.

##### 8.2.1.2 Forwarding Pipeline

**Table 9 — Cacheline Tag Field**

| Name | Scope | Description |
| --- | --- | --- |
| `plru_bits[6:0]` | 8 cachelines | Points to the least-recently-used cacheline. Updated when a cacheline is hit or replaced. A bypass request that does not hit will not update PLRU. |
| `valid` | sector | `valid[1:0] == 00` means the cacheline is invalid. `\|valid[1:0] == 1` means the cacheline is valid. `valid[0] == 1` means sector 0 is valid. `valid[1] == 1` means sector 1 is valid. |
| `beat_dirty_mask` | beat | Each bit is associated with 512 bits in data SRAM. It means there are dirty bytes in those 512 bits. When evicted, the dirty bytes shall be flushed to DDR. |
| `partial` | beat | Indicates whether the sector has invalid bytes. When a cache write allocates a cacheline, the partial bit is set. Only when a read / atomic hits the partial cacheline can the partial bit be cleared. |
| `addr` | cacheline | Address tag field |
| `persistence` | cacheline | Persistence level. Indicates how long the cacheline will stay in L2. It only influences replace. A cacheline with higher persistence has lower priority to be replaced. |
| `access_counter` | cacheline | Number of in-flight instructions on the cacheline |

- `access_counter` decreases when instructions complete and increases when a cacheline instruction is issued to ISCHED. It is implemented in flops.
- The pipeline is stalled when `access_counter` has reached its maximum and the request wants to increase the counter.

In TagCtrl, stage 2 needs to know the 8 cacheline statuses shown in Table 9.

Request sources in TagCtrl: ALIGN, hazard table, overflow FIFO.

Source priority:

1. **Overflow FIFO** — when the overflow FIFO has a request to process, the ALIGN interface shall be stalled until the FIFO is empty. A request in the overflow FIFO can be pulled out when the hazard table is cleared.
2. **Hazard table** — hazard-table request has priority when the overflow FIFO also has a request to handle. Hazard table always has higher priority than ALIGN.
3. **ALIGN** — lowest priority.

The first action in TagCtrl is to read TagRam. TagRam is a dual-port RAM and can provide a read and a write per cycle. In theory the pipeline cannot be stalled by a read/write RAM conflict. Dual-port RAM cannot be written and read on the same address, so a forwarding path is provided to avoid same-address read/write and achieve full bandwidth.

> **Figure 22 — Forwarding path in TagRam**  
> *(Figure image not present in the TXT source.)*

The full pipeline has four stages.

**Stage 0.** Address is generated to access tag SRAM. Forward stage-3 data to stage 1 when the address is equal. When stage 0 is stalled, stage 0 cannot read TagRam and the forwarding path does not work.

**Stage 1.** SRAM data is read out. When the pipeline is stalled, the SRAM output data shall be registered so that the RAM output does not change. Forwarding data from stage 2 or stage 3 shall also be registered when stage 1 is stalled.

Data registered by stage 2 has 4 sources (highest priority first):

1. Forwarding data from stage 2
2. Forwarding data from stage 3
3. Stalled data
4. Stage 1 data

**Stage 2.** The readout data from the pipeline is always the newest; otherwise the function does not work. The readout data is the cacheline tag field in Table 9.

An AXI request performs lookup and allocates a cacheline. The lookup result is issued to MEM_IF. The lookup result may trigger two transactions to DDR. For example, a cache-read miss that replaces a dirty cacheline: bank issues a write request to DDR first, then a read transaction to DDR.

Dependency: if a request needs to be issued to ISCHED and MEMIF at the same time, TagCtrl must guarantee the request is issued to ISCHED first, then to MEMIF, or issued to both at the same time.

Example: a read miss should issue to ISCHED and MEMIF. If ISCHED is full, ready is deasserted and the request cannot be issued to MEMIF. If ISCHED is ready, the request can be issued to MEMIF.

**Stage 3.** The lookup result changes cacheline status and must be written to TagRam. Pending write data may be needed by stage 1 or stage 2.

**Table 10 — Forwarding data between different stages**

| From \ To | Stage 0 | Stage 1 | Stage 2 | n_stage3 | Stage 3 |
| --- | --- | --- | --- | --- | --- |
| Stage 0 | — | Ignore | Ignore | Ignore | Forward |
| Stage 1 | — | — | Ignore | Forward | Forward |
| Stage 2 | — | — | — | — | — |
| Stage 3 | — | — | — | — | — |

The first row is the data that needs to be forwarded in stage 0 to stage 3. The first column is the data that needs forwarding data from stage 0 to stage 3.

- **Stage 0 vs stage 3:** when stage 3 holds the newest data, stage-3 data is forwarded to stage 0 and registered.
- **Stage 0 vs stage 2:** don’t care. If newest data will be pushed to stage 3, the request is going to stage 2. It is solved by the stage-1 vs stage-3 hazard.
- **Stage 1 vs stage 3:** when stage 3 holds the newest data and stage 1 holds the data from TagRam, forward stage-3 data to stage 1. While stage 1 is stalled, the data is registered.
- **Stage 1 vs stage 2:** when stage 2 generates the newest data and stage 1 needs it, forward stage-2 data to stage 1.
- **Stage 2 vs stage 3:** when stage 2 and stage 3 access the same address, they should have the same value.

##### 8.2.1.3 Hit Test

**Hazard check**

To improve performance, some special cases are treated as hazards and stall the lookup pipeline.

- **Bypass / cache read hit in-flight partial cacheline (`access_counter != 0`)**  
  The partial bit in an in-flight partial cacheline may be inaccurate. Any read that hits the cacheline may trigger an unnecessary linefill. Push the read into the hazard table or stall the pipeline when hitting an in-flight partial cacheline. When all in-flight requests on the cacheline have executed, the read in the hazard table can be popped.  
  **This hazard is removed** because RCTRL can solve it. After removal, it is possible that linefill and write occur on the RCTRL input port at the same time. If linefill comes, RCTRL just writes data to the cacheline. If write comes first, linefill should read mask RAM first, then write data on invalid bytes.

- **Bypass write hit in-flight dirty / partial cacheline**  
  Bypass write is always visible to DDR. Once a bypass write hits a dirty cacheline, bank should merge bypass-write data with cacheline data. If there are pending requests on the cacheline, bank should wait for the previous request to complete and then merge. This scene is treated as a hazard because bypass write is asynchronous with other requests. Without this hazard, the bypass write might merge older data.

**Update cacheline — access one sector**

**Table 11 — Update Tag Field When Accessing One Sector**

| operation | policy | state | memif_instr | cacheline instr | linefill_waiting | wb_valid | plru_bits | beat_dirty_mask | partial | valid | access_counter | wb_entry | initial mask |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| AXI_RD | 0 | miss on dirty/partial | LF | RD | 1 | 1 | update | clr | clr | 1 | +1 | allocate | — |
| AXI_RD | 0 | miss on clean/invalid | LF | RD | 1 | 0 | update | clr | clr | 1 | +1 | unchanged | clr |
| AXI_RD | 0 | clean | — | RD | 0 | 0 | update | unchanged | clr | 1 | +1 | unchanged | — |
| AXI_RD | 0 | partial | LF | RD | 1 | 0 | update | unchanged | clr | 1 | +1 | unchanged | — |
| AXI_RD | 0 | dirty | — | RD | 0 | 0 | update | unchanged | clr | 1 | +1 | unchanged | — |
| AXI_RD | 1 | miss | BRD | — | 0 | 0 | unchanged | — | — | 0 | — | unchanged | — |
| AXI_RD | 1 | clean | — | RD | 0 | 0 | update | unchanged | clr | 1 | +1 | unchanged | — |
| AXI_RD | 1 | partial | LF | RD | 1 | 0 | update | unchanged | clr | 1 | +1 | unchanged | — |
| AXI_RD | 1 | dirty | — | RD | 0 | 0 | update | unchanged | clr | 1 | +1 | unchanged | — |
| AXI_WR | 0 | miss on dirty/partial | NOPE | WR | 0 | 1 | update | update | 1 | 1 | +1 | allocate | — |
| AXI_WR | 0 | miss on clean/invalid | — | WR | 0 | 0 | update | update | 1 | 1 | +1 | unchanged | clr |
| AXI_WR | 0 | clean | — | WR | 0 | 0 | update | update | 0 | 1 | +1 | unchanged | — |
| AXI_WR | 0 | partial | — | WR | 0 | 0 | update | update | 1 | 1 | +1 | unchanged | — |
| AXI_WR | 0 | dirty | — | WR | 0 | 0 | update | update | 0 | 1 | +1 | unchanged | — |
| AXI_WR | 1 | miss | ST | WT | 0 | 0 | unchanged | — | — | 0 | — | unchanged | — |
| AXI_WR | 1 | clean | ST | WT | 0 | 0 | update | clr | clr | 0 | 0 | unchanged | — |
| AXI_WR | 1 | partial | ST | WB_WR | 0 | 0 | update | clr | clr | 0 | 0 | allocate | — |
| AXI_WR | 1 | dirty | ST | WB_WR | 0 | 0 | update | clr | clr | 0 | 0 | allocate | — |
| AXI_AA | — | miss on dirty/partial | LF | AA | 1 | 1 | update | update | clr | 1 | +1 | allocate | — |
| AXI_AA | — | miss on clean/invalid | LF | AA | 1 | 0 | update | update | clr | 1 | +1 | unchanged | clr |
| AXI_AA | — | clean | — | AA | 0 | 0 | update | update | clr | 1 | +1 | unchanged | — |
| AXI_AA | — | partial | LF | AA | 1 | 0 | update | update | clr | 1 | +1 | unchanged | — |
| AXI_AA | — | dirty | — | AA | 0 | 0 | update | update | clr | 1 | +1 | unchanged | — |
| flush | — | clean | — | — | — | 0 | unchanged | clr | clr | 1 | — | unchanged | — |
| flush | — | partial | ST | WB | 0 | 0 | unchanged | clr | clr | 0 | — | allocate | — |
| flush | — | dirty | ST | FLUSH | 0 | 0 | unchanged | clr | clr | 1 | — | allocate | — |
| inval | — | clean | — | — | — | 0 | clr | clr | clr | 0 | — | unchanged | — |
| inval | — | partial | — | — | — | 0 | clr | clr | clr | 0 | — | unchanged | — |
| inval | — | dirty | — | — | — | 0 | clr | clr | clr | 0 | — | unchanged | — |
| flushinval | — | clean | — | — | — | 0 | unchanged | clr | clr | 0 | — | unchanged | — |
| flushinval | — | partial | ST | WB | — | 0 | unchanged | clr | clr | 0 | — | allocate | — |
| flushinval | — | dirty | ST | WB | — | 0 | unchanged | clr | clr | 0 | — | allocate | — |

When the AXI request accesses only one sector but `prefetch size == 256B`, the instruction issued to ISCHED depends on the other sector’s state.

In some special cases, two cacheline instructions may be issued to ISCHED. When a read / atomic hits a clean / dirty sector and prefetches the other sector while the other sector is invalid or partial, a PREFETCH instruction is generated and issued to ISCHED to wait for linefill completion. Then an RD instruction is issued to ISCHED to read out SRAM.

**Update cacheline — access two sectors**

**Table 12 — Update Tag Field When Accessing Two Sectors**

Read, policy 0 (common columns unless noted: `memif_instr` / `cacheline instr` / `plru` / `beat_dirty_mask` / `partial` / `valid` / `Access_cnt` / `wb_entry` follow the first row; only differing fields are shown):

| Sector 1 | Sector 0 | memif_instr | cacheline instr | linefill_waiting | plru_bits | beat_dirty_mask | partial | valid | Access_cnt | wb_entry | memif | isched |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| invalid | clean | LF | RD | `2'b10` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| invalid | partial | LF | RD | `2'b11` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| invalid | dirty | LF | RD | `2'b10` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| clean | invalid | LF | RD | `2'b01` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| clean | clean | — | RD | `2'b00` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | No | Yes |
| clean | partial | LF | RD | `2'b01` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| clean | dirty | — | RD | `2'b00` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | No | Yes |
| partial | invalid | LF | RD | `2'b11` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| partial | clean | LF | RD | `2'b10` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| partial | partial | LF | RD | `2'b11` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| partial | dirty | LF | RD | `2'b10` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| dirty | invalid | LF | RD | `2'b01` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| dirty | clean | — | RD | `2'b00` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | No | Yes |
| dirty | partial | LF | RD | `2'b01` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| dirty | dirty | — | RD | `2'b00` | Update | Unchanged | `2'b00` | `2'b11` | +1 | Unchanged | No | Yes |

Read, policy 1:

- `{invalid, invalid}` is treated as miss. Behaviour refers to Table 11.
- All other sector combinations refer to policy 0, because a bypass-read hit on a cacheline is treated as a cache-read hit.

Write, policy 0:

| Sector 1 | Sector 0 | memif_instr | cacheline instr | linefill_waiting | plru | beat_dirty_mask | partial | valid | Access_cnt | wb_entry | memif | isched |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| invalid | invalid\[1\] | — | WR | — | Update | Update | Update | `2'b11` | +1 | Unchanged | No | Yes |
| clean / partial / dirty | — | — | WR | — | Update | Update | Update | `2'b11` | +1 | Unchanged | No | Yes |

Write, policy 1\[2\]:

| Sector 1 | Sector 0 | memif_instr | cacheline instr | plru | beat_dirty_mask | partial | valid | Access_cnt | wb_entry | memif | isched |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| invalid | clean | ST | WT | Update | `2'b00` | `2'b00` | `2'b00` | 0 | Unchanged | Yes | No |
| invalid | partial | ST | WB_WR | Update | `2'b00` | `2'b00` | `2'b00` | 0 | Allocate | Yes | Yes |
| invalid | dirty | ST | WB_WR | Update | `2'b00` | `2'b00` | `2'b00` | 0 | Allocate | Yes | Yes |
| clean | invalid | ST | WT | Update | `2'b00` | `2'b00` | `2'b00` | 0 | Unchanged | Yes | No |
| clean | clean | ST | WT | Update | `2'b00` | `2'b00` | `2'b00` | 0 | Unchanged | Yes | No |
| clean | partial | ST | WB_WR | Update | `2'b00` | `2'b00` | `2'b00` | 0 | Allocate | Yes | Yes |
| clean | dirty | ST | WB_WR | Update | `2'b00` | `2'b00` | `2'b00` | 0 | Allocate | Yes | Yes |
| partial | invalid / clean / partial / dirty | ST | WB_WR | Update | `2'b00` | `2'b00` | `2'b00` | 0 | Allocate | Yes | Yes |
| dirty | invalid / clean / partial / dirty | ST | WB_WR | Update | `2'b00` | `2'b00` | `2'b00` | 0 | Allocate | Yes | Yes |

Atomic, policy don’t-care\[2\]:

| Sector 1 | Sector 0 | memif_instr | cacheline instr | linefill_waiting | plru | beat_dirty_mask | partial | valid | Access_cnt | wb_entry | memif | isched |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| invalid | clean | LF | AA | `2'b10` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| invalid | partial | LF | AA | `2'b11` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| invalid | dirty | LF | AA | `2'b10` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| clean | invalid | LF | AA | `2'b01` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| clean | clean | — | AA | `2'b00` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | No | Yes |
| clean | partial | LF | AA | `2'b01` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| clean | dirty | — | AA | `2'b00` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | No | Yes |
| partial | invalid | LF | AA | `2'b11` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| partial | clean | LF | AA | `2'b10` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| partial | partial | LF | AA | `2'b11` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| partial | dirty | LF | AA | `2'b10` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| dirty | invalid | LF | AA | `2'b01` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| dirty | clean | — | AA | `2'b00` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | No | Yes |
| dirty | partial | LF | AA | `2'b01` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | Yes | Yes |
| dirty | dirty | — | AA | `2'b00` | Update | Update | `2'b00` | `2'b11` | +1 | Unchanged | No | Yes |

\[1\] For write operations, if miss, refer to Table 11.  
\[2\] Atomic operation does not care about cache policy. Abstract behaviour refers to Table 3.

##### 8.2.1.4 Cacheline replacement

PLRU is applied to select a cacheline to replace. PLRU bits are stored in tag RAM. Generally the PLRU bit points to the least-recently-used cacheline. Selection is also influenced by persistence and `access_counter`. Replacement priority (highest first):

1. Invalid cacheline
2. `persistence == 0` cachelines with `access_counter == 0`
3. `persistence == 1` cachelines with `access_counter == 0`
4. `persistence == 2` cachelines with `access_counter == 0`
5. `persistence == 3` cachelines with `access_counter == 0`

Cachelines with `access_counter != 0` cannot be replaced.

- Replace a dirty / partial cacheline: `initial_mask` is cleared by the WB instruction.
- Replace a clean cacheline: `initial_mask` is not cleared until the line is allocated to another request.

---

### 8.3 RCTRL

#### Function Description

| Resource | Size |
| --- | --- |
| Data RAM | 2 RCTRL × 512 bit × 4096 depth |
| Dirty Mask RAM | 2 RCTRL × 128 bit × 2048 depth; 1 entry per sector; data dirty mask, active high, byte granularity |
| Initial Mask | 2 RCTRL × 2048 bit; 1 bit per sector; total 4096 bit |

RCTRL processes instructions from ISCHED and linefill requests from ROB. It accesses the cacheline according to the request, including updating data, dirty mask and initial mask.

| Instruction | Dirty Mask RAM | Data RAM | Initial Mask |
| --- | --- | --- | --- |
| RD | — | Read RAM and return to ALIGN (NOC) | —. Illegal if initial mask is 0. |
| WR | Write original data mask if initial mask is 0. Write 1, ignore 0 if initial mask is 1. | Write data according to data mask | Set to 1 if initial mask is 0 |
| AA | Write 1, ignore 0 according to `dword_active`, if initial mask is 1. | Read RAM and return to NOC (ALIGN). Then write atomic result according to `dword_active`. | —. Illegal if initial mask is 0. |
| WB | Read RAM and send to DDR | Read RAM and send to DDR if `initial_mask` is 1 | Clear |
| FLUSH | Read RAM and send to DDR, then write all 0 | Read RAM and send to DDR | —. Illegal if initial mask is 0. |
| WB_WR | Read RAM, merge with the request write mask, then send to DDR | If `initial_mask` is 1: read RAM, merge with the request write data (request data overwrites dirty data), then send to DDR. If `initial_mask` is 0: do not read data RAM and mask RAM; send write data to WDC directly. | Clear |
| Linefill | Write all 0 if initial mask is 0. Read RAM but no update if initial mask is 1. | If initial mask is 0: write data directly. If initial mask is 1: write data if dirty-byte mask is 0; ignore data if dirty-byte mask is 1; ignore all data if dirty-byte mask is all 1. | Set to 1 if initial mask is 0 |

For a valid cacheline, data status is:

| TAG.Partial | Byte Dirty Mask | Data Status |
| --- | --- | --- |
| 0 | 0 | Clean, data valid |
| 0 | 1 | Dirty, data valid |
| 1 | 0 | Clean, data invalid. TagCtrl must not RD a partial cacheline; it must be read after linefill and after partial is cleared. |
| 1 | 1 | Dirty, data valid |

#### Hardware Description

RCTRL uses a multi-stage pipeline to control RAM access.

| Stage | Instruction | Operation |
| --- | --- | --- |
| stage 0 | RD | Read data RAM |
| stage 0 | WR | Read dirty-mask RAM if need check-partial |
| stage 0 | AA | Read data RAM |
| stage 0 | WB | Read dirty-mask and data RAM if `initial_mask` is 1 |
| stage 0 | Flush | Read dirty-mask and data RAM |
| stage 0 | WB_WR | Read dirty-mask and data RAM if `initial_mask` is 1 |
| stage 0 | Linefill | Read dirty-mask RAM if initial mask is 1 |
| stage 1 | RD / AA / WB / Flush / WB_WR | Hold RAM output data |
| stage 1 | WR / Linefill | Hold RAM output data if stage 0 read it |
| stage 2 | RD | Return data to ALIGN |
| stage 2 | WR | Sync write data from ISCHED |
| stage 2 | AA | Get request data from ISCHED; execute atomic operation.\[1\] |
| stage 2 | WB | Sync write data from ISCHED; return data to WDC |
| stage 2 | Flush | Return data to WDC |
| stage 2 | WB_WR | Sync write data from ISCHED; return data to WDC |
| stage 2 | Linefill | Update dirty mask if initial mask is 1 |
| stage 2 | ALL | Generate instruction-done at this cycle for all operations |
| stage 3\[2\] | WR | Write data RAM according to write data and write strobe. If initial mask is 1: write 1 to dirty-mask RAM if request write strobe is 1; do not update dirty-mask RAM if request write strobe is 0. If initial mask is 0: write 1 / 0 to dirty-mask RAM matching the request write strobe. |
| stage 3 | AA | Write data RAM according to atomic result. Write 1 to dirty-mask RAM if the atomic operation updates data. |
| stage 3 | Flush | Clear dirty mask to 0 |
| stage 3 | Linefill | If initial mask is 1: write linefill data if dirty mask is 0; do not update mask. If initial mask is 0: update dirty mask to 0 and write all linefill data to data RAM. |

Notes:

\[1\] Generate the atomic result at the current cycle for integer atomic requests; generate the atomic result at the next cycle for FP atomic requests. There is a bubble if the current RD / AA needs the FP32 atomic result of the previous cycle.

\[2\] Actually, write data and mask RAM at stage 4 inside the `spram_pipeline` module.

RCTRL uses forwarding to handle data dependence and improve performance. There are 5 forward data paths covering different read / write sequences.

| Direction | stage1 → stage0 | Meaning |
| --- | --- | --- |
| read | stage1 → stage0 | Forward data from stage 3 to stage 2 |
| read | stage2 → stage0 | Forward data from stage 3 to stage 1 |
| write | stage3 → stage0 | Do not read RAM; forward data from stage 4 to stage 1. If read and write access different addresses, stall the read. |
| write | stage4 → stage1 | Forward data from the write-data buffer to stage 1 |
| write | stage4 → stage0 | Actually forward data from stage 5 (previous write-data buffer out) to stage 1 |

**Forward matrix**

| From \ To | stage0 | stage1 | stage2 | stage3 | stage4 |
| --- | --- | --- | --- | --- | --- |
| stage0 | — | ignore\[1\] | ignore\[2\] | ignore\[4\] | forward\[7\] |
| stage1 | — | — | ignore\[3\] | forward (read)\[5\] | forward\[8\] |
| stage2 | — | — | — | forward (not fp32)\[6\] | forward (fp32)\[9\] |
| stage3 | — | — | — | — | — |
| stage4 | — | — | — | — | — |

Notes:

1. Stage1 → stage0 is the same as stage2 → stage1.
2. Stage2 → stage0 is the same as stage3 → stage1.
3. Stage2 → stage1 is the same as stage3 → stage2.
4. Stage3 → stage0 is the same as stage4 → stage1.
5. Stage3 → stage1 forward is read-to-read only.
6. If reading a previous integer atomic result, forward from write data; else forward from stage-3 write data. Clear dirty mask on FLUSH; do not forward write data in that case. This case does not exist after adding RAW hazard.
7. Actually forward data from stage 5 to stage 1 (previous write).
8. Stage4 → stage1 is write-to-read forward only, from the write-data buffer.
9. Includes previous FP32 atomic result or stage-4 write data forwarded to stage 2. This case does not exist after adding RAW hazard.

##### Write Buffer

Stage 4 writes SRAM and stage 0 reads SRAM. In the current design, write always has high priority and executes first if read and write SRAM in the same cycle. This stalls the pipeline when read and write interleave frequently and reduces SRAM utilization.

RCTRL adds a write-data buffer and adjusts read / write priority to optimize performance. Write-data caching behaves like a FIFO: push the write request into the data buffer and pop it when the SRAM write can execute.

Special cases:

1. **Write merge.** If a request in the write-data buffer has the same address as the new write request, merge them. The new write overwrites the old one. Do not update the write pointer of the buffer in this case.
2. **Pop priority.**
   - If the write-data buffer is full, always pop it and write data to SRAM; stall the read request.
   - Else execute the read request and push the new write request into the write-data buffer.
   - If there is no read request and the write-data buffer is not empty, pop the write-data buffer and write data to SRAM.
3. **Forwarding.** Keep the forwarding data path unchanged. For forward from stage 4 to stage 1, forward data from the write buffer to the readout. For forward from stage 4 to stage 0, save write data and forward to the readout at the next cycle. Forward data from stage 4 to stage 0 when all data bytes are written.

##### FP32 Atomic ALU

`dword_active` indicates the position of operand data. For a non-CAS atomic, the cacheline data at that position may be updated after the atomic completes. For CAS, after the atomic completes, the cacheline data at the compare-data position may be updated. For an atomic request, the returned data is always the **original** data in the cacheline.

```text
final_dword_active = dword_active                              // not CAS
final_dword_active = dword_active & (dword_active >> N)        // CAS and atomic_swap = 0
final_dword_active = dword_active & (dword_active << N)        // CAS and atomic_swap = 1
```

`N = 1` for CAS 32, `N = 2` for CAS 64, `N = 4` for CAS 128.

| Atomic information | Original `dword_active` | Final `dword_active` |
| --- | --- | --- |
| CAS 32, `atomic_swap=0` | `'b0011` | `'b0011 & 'b0001 = 'b0001` |
| CAS 32, `atomic_swap=1` | `'b0011` | `'b0011 & 'b0110 = 'b0010` |
| CAS 64, `atomic_swap=0` | `'b1111_0000` | `'b1111_0000 & 'b0011_1100 = 'b0011_0000` |
| CAS 128, `atomic_swap=1` | `'b1111_1111` | `'b1111_1111 & 'b1111_0000 = 'b1111_0000` |

`dword_active` encode:

| Atomic operation | `dword_active` | Example |
| --- | --- | --- |
| atomic 32, not CAS (ignore `atomic_swap`) | Active high; execute atomic on the data if `dword_active` is 1 | `'b1101_1001` — execute atomic on dword 0, 3, 4, 6, 7 |
| atomic 64, not CAS (ignore `atomic_swap`) | Must be two consecutive `dword_active` bits | `'b1100_0011` — execute atomic on `{dword1, 0}`, `{dword7, 6}` |
| atomic 32 CAS, `atomic_swap=0` | Must be 2 consecutive bits. Compare in LSB, swap in MSB | `'b0011`: compare = dword0, swap = dword1 |
| atomic 32 CAS, `atomic_swap=1` | Must be 2 consecutive bits. Compare in MSB, swap in LSB | `'b0011`: compare = dword1, swap = dword0 |
| atomic 64 CAS, `atomic_swap=0` | Must be 4 consecutive bits. Compare in LSB, swap in MSB | `'b1111_0000`: compare = `{dword5, 4}`, swap = `{dword7, 6}` |
| atomic 128 CAS, `atomic_swap=1` | Must be 8 consecutive bits. Compare in LSB, swap in MSB | `'b1111_1111`: compare = `{dword7..4}`, swap = `{dword3..0}` |

See `dword_active` description at:

```text
01.Documents\GP3\02.Tech\02.Architecture\03.Spec\GPU_Ordering_System_Design_CHS_xx.docx
```

Atomic operations execute inside the `aa_alu_wrap` module. Supported: atomic 32, atomic 64, atomic FP32, atomic compare 128.

Get `operand_b` by shifting `operand_a` according to `dword_active` within the CAS boundary.

- For F32 atomic, rounding mode is **RNE** (Round to Nearest Even).
- For atomic MIN / MAX, if the atomic result is the same as the data in the cacheline, do **not** update the byte dirty mask.
- For atomic CMPXCHG, update the byte dirty mask if compare data matches, even if swap data is the same as the original data in the cacheline.

##### Atomic add special number

| Class | Encoding |
| --- | --- |
| ±infinity | Sign = 0 or 1; Exponent = 2<sup>e</sup>−1 (all 1s); Fraction = 0 |
| NAN | Exponent = 2<sup>e</sup>−1 (all 1s); Fraction ≠ 0 |
| Denormal | Exponent = 0; Fraction ≠ 0 |

**FP32 ADD special-value result.** For FP32 SUB, translate to `[addr] + (−operand_a)` and follow the same table.

| `[addr]` | Operand A | Result | Note |
| --- | --- | --- | --- |
| normal | NAN | NAN (`0x7FC00000`) | `[addr] + operand_a` has the same result as `operand_a + [addr]` |
| +inf | −inf | NAN | |
| +inf | +inf | +inf (`0x7F800000`) | |
| −inf | −inf | −inf (`0xFF800000`) | |
| normal / denormal | +inf | +inf | |
| normal / denormal | −inf | −inf | |
| NAN | ±inf / ±denormal / NAN | NAN | If any operand is NAN, result is NAN |
| −inf | NAN | NAN | |
| denormal | normal | normal; treat denormal as zero | Difference of denormal and normal is less than 23. FOP ISA has a bit to flush denormal to zero or not; atomic (DMA) always flushes denormal to zero (same as CUDA). |
| denormal | denormal | `+0.0f` | If any one of `[addr]` and `operand_a` is positive, ignore the values of `[addr]` and `operand_a`. |
| denormal | denormal | `−0.0f` | If and only if `[addr]` and `operand_a` are both negative. |
| normal | normal | +inf | Result Exp = `0xFF`, mantissa = 0; both operands positive |
| normal | normal | −inf | Result Exp = `0xFF`, mantissa = 0; both operands negative |
| normal | normal | `+0.0f` | Different sign bits and `|positive| > |negative|`. Result Exp = `0x0`, mantissa ≠ 0 (denormal). Flush denormal to zero. |
| normal | normal | `−0.0f` | Different sign bits and `|negative| > |positive|`. Result Exp = `0x0`, mantissa ≠ 0 (denormal). Flush denormal to zero. |

**FP32 MIN special-value result**

| `[addr]` | Operand A | Result | Update |
| --- | --- | --- | --- |
| NAN | NAN / normal / denormal / ±inf | NAN (`0x7FC00000`) | yes\[1\] |
| normal / denormal / ±inf | NAN | NAN (`0x7FC00000`) | yes |
| +inf | +inf | +inf (`0x7F800000`) | no\[1\] |
| −inf | −inf | −inf (`0xFF800000`) | no |
| +inf | −inf | −inf | yes |
| −inf | +inf | −inf | no |
| normal / denormal | +inf | normal / denormal | no |
| normal / denormal | −inf | −inf | yes |
| +inf | normal / denormal | normal / denormal | yes |
| −inf | normal / denormal | −inf | no |
| normal / denormal | normal / denormal | Compare sign / exp / mantissa as normal | yes / no |

**FP32 MAX special-value result**

| `[addr]` | Operand A | Result | Update |
| --- | --- | --- | --- |
| NAN | NAN / normal / denormal / ±inf | NAN (`0x7FC00000`) | yes\[1\] |
| normal / denormal / ±inf | NAN | NAN (`0x7FC00000`) | yes |
| +inf | +inf | +inf (`0x7F800000`) | no\[1\] |
| −inf | −inf | −inf (`0xFF800000`) | no |
| +inf | −inf | +inf | no |
| −inf | +inf | +inf | yes |
| normal / denormal | +inf | +inf | yes |
| normal / denormal | −inf | normal / denormal | no |
| +inf | normal / denormal | +inf | no |
| −inf | normal / denormal | normal (is Operand A) | yes |
| normal / denormal | normal / denormal | Compare sign / exp / mantissa as normal | yes / no |

\[1\] If the result is True, update Operand A to the memory address; else keep the original data unchanged. The returned data is always the data in memory.

**FP32 CMPXCHG special-value result**

When `rgx_cr_usc_lms_fp32_denormal_en = 1`:

| `[addr]` | Operand A | Result | Update |
| --- | --- | --- | --- |
| NAN | NAN / normal / denormal / ±inf | not equal | no |
| normal / ±inf | NAN | not equal | no |
| denormal | NAN | not equal, but flush `[addr]` to zero | yes |
| denormal | normal / denormal / ±inf | Flush input denormal to zero then bitwise comparison. If not equal, result = flushed(`[addr]`); if equal, result = flushed(`operand_b`). | yes / no |
| normal / denormal / ±inf | denormal | Flush input denormal to zero then bitwise comparison. If not equal, result = flushed(`[addr]`); if equal, result = flushed(`operand_b`). | yes / no |
| normal / ±inf | normal / ±inf | Bitwise comparison. If not equal, result = `[addr]`; if equal, result = `operand_b`. | yes / no |

When `rgx_cr_usc_lms_fp32_denormal_en = 0`:

| `[addr]` | Operand A | Result | Update |
| --- | --- | --- | --- |
| NAN | NAN / normal / denormal / ±inf | not equal | no |
| normal / denormal / ±inf | NAN | not equal | no |
| normal / denormal / ±inf | normal / denormal / ±inf | Bitwise comparison. If not equal, result = `[addr]`; if equal, result = `operand_b`. | yes / no |

- For FP32 ADD / SUB, denormal data is **always** flushed to zero.
- For FP32 MIN / MAX / CMPXCHG, input-denormal flushing is controlled by `rgx_cr_usc_lms_fp32_denormal_en`:
  - `0`: do not flush input denormal
  - `1`: flush input denormal to zero

`rgx_cr_usc_lms_fp32_denormal_en` only flushes **input** denormal data for FP32 MIN / MAX; the atomic result always comes from the operand data.

Example: FP32 MIN, request data is denormal `0x0000_1234`, original data is `0x1234_5678`, `rgx_cr_usc_lms_fp32_denormal_en = 1`. Flush request denormal to `0x0000_0000`. `0x0000_0000 < 0x1234_5678`, so the atomic result is `0x0000_1234`, **not** `0x0000_0000`.

```text
if (rgx_cr_usc_lms_fp32_denormal_en)
    if (op == FP32_MIN)
        if (flush(op_a) < flush(op_b))
            result = op_a;
else
    if (op == FP32_MIN)
        if (op_a < op_b)
            result = op_a;
```

| Op | LMS | L2 |
| --- | --- | --- |
| ADD / SUB F32 | CR\[1\] | always flush |
| MIN / MAX F32 | CR | CR (don’t flush result) |
| CAS F32 (CMPXCHG) | CR | CR (flush result) |

\[1\] CR = denormal flush controlled by configure register.

---

### 8.4 ISCHED

> **Figure 23 — ISCHED Microarchitecture**  
> *(Figure image not present in the TXT source.)*

#### Function Description

ISCHED provides a unified buffer to manage the order of accesses to the same cacheline or sector. All requests from TagCtrl are pushed into a buffer in order.

##### Maintain the order of accessing the same sector

Four hazard types — RAR, RAW, WAW, WAR — are all processed in order. Examples:

- TagCtrl issues R0, R1, R2, all accessing the same sector. ISCHED outputs R0, R1, R2 in order.
- TagCtrl issues R0, W1, A2, R3, all accessing the same sector. ISCHED also outputs R0, W1, A2, R3 in order.
- If TagCtrl issues R0, R1, R3 accessing different sectors, these requests may be output out of order.

A read request that hits on a miss shall wait in ISCHED until the linefill response returns from RCTRL.

##### Linefill vs Writeback Hazard

A read miss on a dirty cacheline triggers linefill and writeback at the same time. Bank must guarantee writeback is always executed first, then linefill. TagCtrl issues the read request to ISCHED with `linefill_waiting[1:0]` and `wb_waiting`.

- `linefill_waiting` — the read request shall wait for the linefill response from RCTRL.
- `wb_waiting` — the request shall wait for the WB response from RCTRL.

RCTRL returns either a linefill response or a WB response in a cycle. The order of linefill response and WB response is guaranteed by MEMIF. From ISCHED’s perspective, the WB response always returns first, then the linefill response.

##### Prefetch

A request from TagCtrl accesses sector 0, sector 1, or both. The information is derived from `blen` and `addr`. A prefetch request influences the calculation.

- A read with `blen == 0`, `start_beat = 0`, `prefetch = 1` that misses in TagCtrl is regarded as accessing **two** sectors, because TagCtrl issues a read to DDR and linefills two sectors.
- A read with `blen == 0`, `start_beat = 0`, `prefetch = 0` that misses or hits is regarded as accessing sector 0.
- A request with `blen == 1`, `start_beat == 2`, `prefetch == 1` that hits sector 1 and misses sector 0 (sector 0 invalid) is still regarded as accessing two sectors.

A read with prefetch potentially accesses two sectors. Prefetch does not by itself decide whether the request accesses two sectors. If the read only accesses one sector and `prefetch == 1`, the request is regarded as accessing two sectors when the other sector is invalid or partial.

##### Split request

The request stored in the Ring buffer is a burst request. Before accessing cacheline data, ISCHED splits the request. Each request issued to RCTRL is aligned to 512 bits.

Special case: when receiving a WB_WR request, ISCHED sometimes converts it to a WB request. Example:

- A bypass write with `blen == 1`, `start_beat == 1`, hits a dirty cacheline with `valid[1:0] = 2'b10`, `beat_dirty_mask[3:0] = 4'b1100`. Bank shall merge data in beat 1 and beat 2 with the bypass-write data and write to DDR. For beat 3, bank just reads out the data and writes to DDR. The WB_WR request is split into WB_WR beat 1, WB_WR beat 2, WB beat 3.

##### Fetch write data

WR / AA / WB_WR instructions write data to SRAM. They need to fetch data from IWDB using `iwdb_index`. The request in the sorting queue carries a start address. When the request is split, `iwdb_index` should be recalculated.

#### Hardware Description

ISCHED is responsible for:

- In-order output of requests sent by TagCtrl in the Ring buffer
- Arbitrating requests from TagCtrl, WDC and ALIGN, where WDC requests such as WB / WB_WR / FLUSH have the highest priority
- Splitting requests aligned to sectors or cachelines into 512-bit aligned requests in SPLIT
- Fetching write data from IWDB, without needing to synchronize write commands with write data by a lock buffer

##### Ring Buffer

The ring buffer uses registers to save the state bit and SRAM to save the sideband signal.

Request order is ensured using `Wr_ptr` and `Rd_ptr`, combined with a round-robin mechanism.

- **Write pointer:** always points to the next entry to be written.
- **Read pointer:** always points to the previous oldest valid entry.

The ring buffer can output two requests in one cycle, sent to different RCTRLs.

**Wr_ptr and Rd_ptr**

- `Wr_ptr`: when a request from TagCtrl comes in, `wr_ptr` increases by 1 until it collides with `rd_ptr`. That indicates the ring buffer is full and cannot receive more requests.
- `Rd_ptr`: `rd_ptr` can move forward until `status_valid` of the next entry is pulled low. Requests that do not need to wait for linefill / writeback can go forward first. The valid status bit of such an entry is pulled low before the valid bit of the oldest entry pointed to by `rd_ptr`. After processing the oldest entry (the next entry of `rd_ptr`), `rd_ptr` jumps one entry in one cycle and stops at the entry whose own `status_valid` is 0 and whose next entry’s `status_valid` is 1. If the read pointer equals the write pointer, it is considered full. In fact the ring buffer keeps one unused entry.

##### Entry issue

Related to the position of `rd_ptr`.

- If the entry that is the next entry of `rd_ptr` is in `waiting_linefill` or `waiting_wb`, `rd_ptr` cannot move.
- An entry that is not in a waiting state can be issued directly to RCTRL. Its valid status bit is deasserted.
- Until the waiting state of the next-of-`rd_ptr` entry is released, that entry has the highest priority to issue to RCTRL.

##### Waiting-state release

**Linefill done.** When `rctrl_isched_port` sends `linefill_done`, the ring buffer looks up all entries and ties the corresponding `waiting_linefill` bit to 0 according to the returned set, way and `sector_mask` from RCTRL (`linefill_waiting` is sector granularity). Also look up the port request from TagCtrl. If that port request is in the same case, overwrite its `waiting_linefill` status to 0.

**waiting_WB release.** `waiting_wb` is cacheline granularity, so each entry uses 1 bit. When the WB completion signal comes back, look up the set and way of all entries; also check the port request. If the port request is also in `waiting_wb`, overwrite it to 0.

##### Hazard method

When waiting status is 0 (lf waiting, wb waiting), the request is issued to RCTRL in TagCtrl order.

ISCHED must ensure order between requests that access the same sector, regardless of linefill or the request itself.

When TagCtrl allocates a new cacheline for a request, that causes a linefill. Because of prefetch size, the mask of the linefill request issued by TagCtrl may not be the same as the request’s sector mask.

A subsequent request that hits the same cacheline is a **secondary hit** and is **not** signed with linefill waiting.

By the ISCHED mechanism, a request that is not waiting is not blocked by ISCHED and is issued to RCTRL in order. That means a secondary-hit request can overtake the miss request, which is incorrect. So ISCHED must, based on the situation, merge the linefill waiting or wb waiting of the miss request onto the secondary-hit request. That preserves order between requests that access the same sector.

**Situation 1.** Request B and previous request A both access the same sector. A still has linefill-waiting status. Because B is a secondary hit, TagCtrl issues B without linefill waiting. ISCHED must merge A’s linefill status onto B.

| | Sector_mask | linefill_waiting |
| --- | --- | --- |
| A | `1X` | `11` |
| B | `1X` | `XX` → `11` (merged by A’s lf status) |

**Situation 2.** B and A access the same cacheline but not the same sector. If B’s `sector_mask` and A’s `linefill_waiting` have **no overlap**, B can issue before A.

| | Sector_mask | linefill_waiting |
| --- | --- | --- |
| A | `10` | `10` |
| B | `01` | `0X` |

**Situation 3.** B and A access the same cacheline but not the same sector. If B’s `sector_mask` and A’s `linefill_waiting` **overlap**, B cannot issue before A. A’s linefill status is merged onto B.

| | Sector_mask | linefill_waiting |
| --- | --- | --- |
| A | `10` | `01` |
| B | `01` | `X0` → `X1` (merged by A’s lf status) |

`X` means 0 or 1.

When a linefill-done flag from RCTRL arrives, all corresponding lf status bits (same set, way, sector mask) of entries in the Ring buffer are cleared. After linefill waiting is cleared, requests are issued to RCTRL in TagCtrl order.

WB waiting is similar to linefill waiting. WB-waiting granularity is cacheline, so if two requests access the same cacheline, merge the first request’s wb-waiting status onto the second request.

##### Arbiter & Split

**Arbiter.** Three inputs: ROB, buffer mode, WB channel.

- WB channel has the highest priority.
- ROB and buffer-mode requests are polled by round robin.

**Split**

1. **Request splitting**
   - Incoming requests are aligned to 1024-bit or 2048-bit, but RCTRL only supports 512-bit alignment. Each request must be split into multiple 512-bit requests.
   - Number of splits is determined by `blen` and `beat_dirty_mask`.
   - After all splits are processed, the `last` signal is asserted to indicate the final request.

2. **WB_WR split.** If the instruction is WB_WR, in addition to `blen` and burst type, SPLIT splits based on `tagctrl_isched_rec.beat_dirty_mask`.
   - When the bit of `cmu_isched_evict_beat_mask` is 0, this beat’s instruction is WB.
   - When the bit of `tagctrl_isched_rec.beat_dirty_mask` is 1, this beat’s instruction is WB_WR.

   A write-through request that hits a cacheline or sector does not always visit the whole cacheline. Beats that are not accessed do not need a merge.

   Except this instruction, other requests keep the same instruction after splitting.

3. **Counter.** Counts the number of 512-bit requests sent.
   - For every 512-bit request sent to RCTRL, the counter increments by 1.
   - When `counter == blen`, the last request is marked and the request is released.

##### Write data fetch

Handles commands that fetch data from IWDB: WR, WB_WR, and AA. Other requests (RD, WB, FLUSH) can bypass the latency FIFO and do not require this mechanism.

##### Credit Control

**Background.** Both the WB_WR / AA / WR path and the WT channel need to fetch data from IWDB. Without credit control for write-data requests, downstream backpressure from RCTRL can busy the IWDB read path. That can block a WT request from getting data from IWDB. If at that point the WDC order FIFO has a WT request in the first position, write commands may be prevented from being released from the FIFO. If WB and related requests from RCTRL are not processed, the entire write path can deadlock.

**Credit.** ISCHED manages a credit counter whose initial value equals the depth of its data FIFO. Each time a write-data request is issued, the credit counter decreases by one, because the request pulls data from IWDB into the data FIFO. When the credit counter reaches zero, all available space in the data FIFO has been allocated and no more write-data requests can be sent. When data in the FIFO is consumed, the credit counter is incremented.

---

### 8.5 WDC

> Not present in the original TOC. Inserted between ISCHED and Memory Scheduler.

#### Function description

##### Routing Write Commands and Write Data

**Routing Write Commands.** Write commands access the IWDB module to retrieve write data. Two types of commands need data from IWDB:

- From ISCHED: WR, WB_WR, AA
- From TagCtrl: WT, WB, WB_WR, Flush

IWDB is pipeline-based and only allows single-channel access, so these two types of commands must be arbitrated before entering the IWDB pipeline. This arbitration is one of WDC’s primary functions.

**Routing Write Data.** The destination for write data is the MEM_IF module in TagCtrl, which sends write data to DDR. Two types of write data need to be routed to MEM_IF:

- From RCTRL: WB and WB_WR write data
- From IWDB: WT write data

To keep write commands and write data aligned, WDC uses an order FIFO to store WT, WB and WB_WR instructions so their sequence stays consistent.

Similar to ISCHED, if WT commands are not credit-controlled, the system may deadlock. To allow WT instructions to access IWDB, there must be enough credit counters, matching the FIFO depth.

WDC also needs to split WT requests into 512-bit aligned requests.

#### Hardware description

The two `order_fifo`s are the keys to routing the write command and the write data.

---

### 8.6 Memory Scheduler

> Originally numbered **8.5**.

> **Figure — Mem scheduler scheme**  
> *(Figure image not present in the TXT source.)*

#### 8.6.1 Function Description

- Select request based on DDR-friendly priority.
- Maintain the original order of requests accessing the same cacheline.

Bank issues many PAs to the memory scheduler. The PA queue records the original order. The memory scheduler tracks the PA of the last request issued to DDR. If any request matches the row id and bank id of the last request PA, that matched request has the highest priority to output in the PA queue.

Each PA carries a unique buffer index. The buffer index points to the corresponding command location in BANK.

To avoid a request staying in the queue for a long time, the oldest request is selected at the highest priority.

#### 8.6.2 Hardware Description

A depth-32 buffer is implemented in the PA queue.

The following describes push / pop when queue depth is 4:

- Cycle 0: req0 is pushed into the PA queue.
- Cycle 1: req1 is pushed into the PA queue.
- Cycle 2: req2 is pushed; at the same time req1 is popped.
- Cycles 3 and 4: req3 and req4 are pushed.

The oldest request drops down when a request is pushed.

- An input request always flows into entry 3 when there is an empty entry in the PA queue.
- An input request is stalled when there is no space in the PA queue.
- The request in entry 0 always has the highest priority.

When entry 3 and entry 0 both have valid requests while there is space in the PA queue (as on cycle 4), req2 and req3 still drop. On cycle 5 the PA queue is full and the input request is stalled.

---

### 8.7 Mem_If

> Originally numbered **8.6**.

#### 8.7.1 Function Description

**Memory scheduler.** Memif issues addresses to the memory scheduler when the scheduler is enabled, via the `memif_mem_sched` interface. This interface carries a hashed address and a request index. The request index is called the **buffer index**: a unique id indicating where the corresponding sideband information is stored. In the memory scheduler, the hashed address decides priority based on the LRU algorithm. After selection, the memory scheduler returns the buffer index to memif. The PA on `mem_sched_memif` is ignored. For the detailed hash function on PA, see [3.10 Memory Scheduler](#310-memory-scheduler).

**NSP allocation.** Memif provides a unified ID pool for read and write requests. Write and read can allocate up to 128 IDs. A read can only be issued when ROB has enough empty space. Even if there is enough ID, if there is not enough empty space in ROB, the read is stalled.

**Write data and request synchronization.** Based on the NSP protocol, write request and write data shall be issued together. A write request shall wait for write data from IWDB or RCTRL.

**ROB management.** ROB has two functions:

- All bypass-read request data is returned to NOC directly without passing through RCTRL.
- Avoid backpressure on the NSP read-response interface.

Any read request should allocate **continuous** ROB space first, then allocate an NSP id. Sometimes there is enough space in ROB but it is not continuous; the read still waits until continuous space is freed.

A read request allocates ROB space at least at 512-bit granularity. NSP returns 256 bits per cycle, so at least 2 cycles are needed before read-request data is ready. ROB provides a 256-bit write port and a 512-bit read port. If all data has returned from NSP, read requests are arbitrated to access ROB by round robin.

- For a bypass-read request, readout data from ROB is issued to ALIGN directly.
- For linefill data, readout data from ROB is issued to RCTRL.

**Fetch write data from IWDB or RCTRL.** After the memory scheduler:

- A bypass-write request must fetch data from IWDB.
- WB / FLUSH / WB_WR must fetch data from RCTRL.

**Resolve linefill and writeback hazard.**

- If the memory scheduler is **disabled**, writeback is always issued to NSP first, so the linefill / writeback hazard does not exist.
- If the memory scheduler is **enabled**, linefill may be issued to NSP before writeback. It is possible that writeback has not been issued but linefill data has all returned. If a linefill request depends on writeback, it is marked `wb_check`. When allocating an NSP ID, MEMIF checks whether the corresponding cacheline writeback is finished by looking up the pending-writeback buffer in TagCtrl. If finished, TagCtrl issues a finished flag back to MEMIF and `wb_check` is cleared. If not finished, `wb_check` remains in the NSP entry. When writeback finishes, RCTRL broadcasts to TagCtrl and Memif, and `wb_check` in the NSP entry is cleared. The linefill request cannot be issued to RCTRL until `wb_check` is cleared.

#### 8.7.2 Hardware Description

This figure (not present in the TXT source) shows the data-path flow in MEMIF.

**Deadlock concern.** Allocate ROB index **before** allocating write-data sync. If ROB-index allocation is later than wdata sync, or they happen in the same cycle, deadlock occurs. Root cause: linefill and writeback share a pipeline in RCTRL. If writeback is stalled, linefill is also stalled. ROB space cannot be freed. The request at the head of the latency FIFO is stalled because there is not enough ROB space.

**wb_check update.** When a linefill request allocates an NSP ID and finds the dependent writeback is not finished, but RCTRL broadcasts writeback-done at the same time, `wb_check` is cleared. TagCtrl needs to return a wb-finish response to MEMIF.

**Address Hazard.** Address-overlap hazard between read and write does not exist because MEMIF does not reorder requests from TagCtrl unless the memory scheduler is enabled. When the memory scheduler is enabled, it keeps the order of the same cacheline: if two requests access the same cacheline, the scheduler outputs them in order. Therefore MEMIF does not need to handle Address Hazard.

---

## 9 Area

| Module | SRAM Area | Register Area | Reserved Area | Total |
| --- | --- | --- | --- | --- |
| TagCtrl | 1.6w | 0.4w | 0.4w | 2.4w |
| MEMIF | 0.5w | 0.2w | 0.2w | 0.9w |
| ISCHED | — | 0.5w | 0.3w | 0.8w |
| RCTRL | 25.5w | — | 1.5w | 27w |
| WDC, ALIGN, IWDB | — | — | 1w | 1w |
| **Sum** | | | | **32.7w** |

---

## 10 Performance

| Case | Latency |
| --- | --- |
| Read hit | 11 cycles |
| Write | 1 cycle after sync of write data and write command |
