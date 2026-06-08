---
tags:
  - FPGA
  - AXI4
  - AMBA
  - 原子操作
  - 排序模型
  - Xilinx
---

## 核心结论

- AXI4 支持两种原子操作：**Locked Access**（锁访问）和 **Exclusive Access**（独占访问）
- **Locked Access**：Master 锁定总线，独占访问直到解锁——简单但总线利用率低
- **Exclusive Access**：通过监视机制实现原子操作，不阻塞总线——现代 ARM 推荐方案
- 排序模型定义了事务间的**可见性**和**顺序约束**，影响多 Master 系统正确性
- ZYNQ PS 使用 Exclusive Access 实现 Linux 原子操作（`ldrex`/`strex`）

---

## 一、原子操作概述

多 Master 系统中，**读-改-写**序列需要原子性，防止中间状态被其他 Master 破坏：

```
非原子操作（有竞争）:
  Master A:  读  → 改  → 写
  Master B:           读  → 改  → 写
                      ↑ 这里读到旧值，导致 Master A 的修改丢失！

原子操作:
  Master A:  读-改-写 (不可分割)
  Master B:  ──等待──→ 读-改-写
```

AXI4 提供两种机制：

| 机制 | 协议开销 | 总线影响 | 适用场景 |
|:----:|:--------:|:--------:|----------|
| **Locked Access** | 简单 | 阻塞总线 | 简单系统、Legacy 兼容 |
| **Exclusive Access** | 复杂 | 不阻塞总线 | 多核、RTOS、Linux |

---

## 二、Locked Access（锁访问）

### 工作机制

```
Master 流程:
  1. 发出 ARLOCK=1 (或 AWLOCK=1) 标记锁事务
  2. 发出锁写/锁读
  3. 保持 ARLOCK=1 (或 AWLOCK=1) 直到所有锁操作完成
  4. 发出非锁事务 → 解锁总线

总线行为:
  ┌─── 锁定区域 ───┐
  │ R0 │ R1 │ W0 │ W1 │  ← 此期间其他 Master 被阻塞
  └─────────────────┘
           ↓
  解锁后其他 Master 才能访问
```

### ARLOCK / AWLOCK 信号

| AxLOCK[1:0] | 类型 | 说明 |
|:-----------:|:----:|------|
| 00 | **Normal** | 普通访问，无锁 |
| 01 | **Exclusive** | 独占访问（推荐） |
| 10 | **Locked** | 锁访问（传统方式） |
| 11 | 保留 | — |

### 锁访问的限制

```
不推荐使用 Locked 的原因:
  1. 总线利用率低——锁定期间其他 Master 被阻塞
  2. 可能导致死锁——锁持有者被更高优先级打断
  3. 协议要求 Slave 在锁期间保持仲裁优先权
  4. ARM 已弃用，推荐 Exclusive Access
```

---

## 三、Exclusive Access（独占访问）

### 工作机制

Exclusive Access 不锁定总线，通过 **监视器 (Monitor)** 实现原子操作：

```
┌──────────┐                    ┌──────────┐
│ Master A  │── Exclusive Read ──→│  Slave   │
│           │←── Monitor OK ────│  (带监视器)│
│           │                    │          │
│ (内部计算)│                    │ 标记地址  │
│           │                    │ 为独占监视│
│           │── Exclusive Write →│          │
│           │←── EXOKAY/FAIL ──│  检查标记  │
└──────────┘                    └──────────┘
```

### 典型用法：ARM `ldrex` / `strex`

```
ARM 汇编实现原子加 1:

try:
  ldrex r1, [r0]      // 独占读取 [r0] → r1, 监视地址 [r0]
  add   r1, r1, #1    // 内部计算
  strex r2, r1, [r0]  // 独占写入 [r0]
  cmp   r2, #0        // r2=0 成功, r2=1 失败
  bne   try            // 失败则重试
```

### Exclusive Monitor

AXI4 定义两种监视器：

```
Local Monitor（本地监视器）:
  - 位于 Master 内部
  - 监视 Master 自己的 Exclusive 访问
  - Master 粒度

Global Monitor（全局监视器）:
  - 位于 Slave 或 Interconnect 中
  - 监视所有 Master 对共享地址的访问
  - 地址粒度

写成功的条件:
  Local Monitor  OK  +  Global Monitor  OK  →  EXOKAY
  Local/Global 任一失败                      →  FAIL
```

### 监控状态转移

```
Exclusive Read 后:
  状态: Open  →  已标记该地址

同一 Master 再次 Exclusive Read 同一地址:
  状态: 已标记  →  重新标记（覆盖旧标记）

其他 Master 写入该地址:
  状态: 已标记  →  失效（清除标记）

Exclusive Write:
  如果标记仍有效 →  EXOKAY
  如果标记已失效 →  FAIL（需重试）
```

### Xilinx ZYNQ 中的 Exclusive Access

```
ZYNQ PS (Cortex-A9) 的 Exclusive 支持:
  - SCU (Snoop Control Unit) 实现 Local Monitor
  - DDR 控制器 (DDRC) 实现 Global Monitor
  - AXI Interconnect 透传 Exclusive 信号

注意事项:
  - PL 自定义 Slave 若需支持 Exclusive，必须实现 Global Monitor
  - 不支持 Exclusive 的 Slave 返回 OKAY（非 EXOKAY）
  - Master 收到 OKAY 而非 EXOKAY = 不支持独占，需回退到软件锁
```

### Exclusive Slave 实现框架

```verilog
module axi_exclusive_slave #(
    parameter C_S_AXI_DATA_WIDTH = 32
) (
    // AXI4 总线接口
    ...
);
    // Exclusive Monitor 状态
    reg        excl_mon_active;
    reg [31:0] excl_mon_addr;
    reg [3:0]  excl_mon_id;

    // 写事务处理
    always @(posedge S_AXI_ACLK or negedge S_AXI_ARESETN) begin
        if (!S_AXI_ARESETN) begin
            excl_mon_active <= 1'b0;
        end else begin
            // 其他 Master 写入标记地址 → 清除监视
            if (写事务完成 && waddr == excl_mon_addr && wid != excl_mon_id)
                excl_mon_active <= 1'b0;

            // Exclusive Write 检查
            if (AWLOCK == 2'b01 && 写事务完成) begin
                if (excl_mon_active && excl_mon_addr == AWADDR)
                    BRESP = 2'b01;  // EXOKAY
                else
                    BRESP = 2'b00;  // FAIL → OKAY
                excl_mon_active <= 1'b0;
            end
        end
    end

    // 读事务处理
    always @(posedge S_AXI_ACLK) begin
        if (ARLOCK == 2'b01 && ARVALID && ARREADY) begin
            excl_mon_active <= 1'b1;
            excl_mon_addr   <= ARADDR;
            excl_mon_id     <= ARID;
        end
    end
endmodule
```

---

## 四、事务间排序模型 (Ordering Model)

### 写观察 (Write Observation)

写观察定义了其他 Master 何时能看到当前 Master 的写入结果：

```
Master A 写 → Master B 读同一地址:

时间线:
  A: 写数据发出 ──→ Interconnect ──→ Slave 完成
                                     │
  B:                                 │ 何时 B 的读能见到新值？
                                     ↓
  B: 读所见值 = 旧值  │  过渡期  │  新值 (写观察点)
```

### 观察点与排序规则

```
规则 1: 同 ID 保序
  - AWID=0 事务1 → AWID=0 事务2
  - 事务2 的写观察不会早于事务1

规则 2: 不同 ID 无排序要求
  - AWID=0 与 AWID=1 之间无顺序保证
  - Master 必须通过其他机制同步（DMB/DSB 指令）

规则 3: 读写依赖
  - ARID=X  →  RID=X (读)
  - 同一 ID 的读必须在之前的写完成后才能观察到
```

### Barrier 与同步

ARM 提供内存屏障指令控制排序：

```
DMB (Data Memory Barrier):
  之前所有内存访问 → 所有之后内存访问
  确保观察顺序

DSB (Data Synchronization Barrier):
  之前所有内存访问完成 → 再执行后续指令
  更严格，包括对 CPU 的影响

ISB (Instruction Synchronization Barrier):
  刷新流水线，确保指令一致性
```

### AXI4 中的 Barrier 支持

```
AXI4 协议本身不提供 Barrier 事务类型。
Barrier 由 CPU 内部实现：
  - CPU 发出 DMB/DSB 后
  - 内部等待所有 outstanding 事务完成
  - 再发出新事务

跨 Master 的 Barrier:
  - 需通过软件协议 + 共享内存实现
  - 或使用专用的硬件 Barrier 模块
```

---

## 五、ID 分配策略与实践

### 基本策略

```
策略 1: 全保序（简单）
  ARID = 0, AWID = 0
  → 所有事务保序，适合控制类访问

策略 2: 读写分离（中等）
  ARID = 0, AWID = 1
  → 写不必等待读完成，提高效率

策略 3: 多通道乱序（高性能）
  ARID = {channel_id, seq_num}
  AWID = {channel_id, seq_num}
  → 最大化总线利用率
```

### Xilinx IP 的 ID 行为

| IP | 读 ID | 写 ID | 说明 |
|:---|:-----:|:-----:|------|
| **AXI DMA** | 固定/可配 | 固定/可配 | Simple模式固定 ID，Scatter-Gather 可设 |
| **AXI VDMA** | 固定 | 固定 | 读通道和写通道 ID 独立 |
| **MIG** | 可变内部 ID | 可变内部 ID | 内部使用多 ID 优化调度 |
| **AXI Interconnect** | 透传/加前缀 | 透传/加前缀 | 通过 ID_WIDTH 参数扩展 |
| **AXI SmartConnect** | 透传/加前缀 | 透传/加前缀 | 更高效的 ID 管理 |

### 设计建议

```
AXI4 排序设计 checklist:

[ ] 确定系统需要多少 outstanding 事务
[ ] 确定是否需要乱序（多通道/多 Master）
[ ] 分配 ID 时预留扩展位
[ ] Slave 实现需正确处理同 ID 保序
[ ] Exclusive 访问的 ID 策略需全局一致
[ ] Interconnect 配置合适的 ID_WIDTH
[ ] 多 Master 系统考虑 TZ 隔离对排序的影响
```

---

## 参考

- ARM AMBA AXI4 Protocol Specification (IHI0022E) — Chapter A5~A7
- ARM Cortex-A9 MPCore Technical Reference Manual — SCU 章节
- Xilinx UG1037: AXI Reference Guide — Atomic Access
- [[30AXI4总线协议详解]] — AXI4 基础与通道信号
- [[32AXI4_LowPower低功耗与性能优化]] — 性能优化相关
