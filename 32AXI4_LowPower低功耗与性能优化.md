---
tags:
  - FPGA
  - AXI4
  - AMBA
  - 低功耗
  - 性能优化
  - Xilinx
---

## 核心结论

- AXI4 低功耗接口 (LPI) 通过 **CSYSREQ / CACTIVE / CLKSTOP** 实现时钟门控
- 低功耗控制是可选信号，由 Power Controller 统一管理时钟生成
- 吞吐量 = 数据位宽 × 时钟频率 × 每拍传输效率（握手占比）
- 乱序深度、数据位宽、突发长度是影响性能的三个核心参数
- AXI SmartConnect 时序优于 AXI Interconnect，优先选用

---

## 一、AXI4 Low-Power 低功耗接口

### 概述

AXI4 低功耗接口 (LPI, Low-Power Interface) 允许系统在总线空闲时关闭时钟以降低动态功耗。LPI 信号独立于 AXI4 数据通道，由 **Power Controller** 统一管理。

### LPI 信号

| 信号 | 方向 | 说明 |
|:----:|:----:|------|
| **CSYSREQ** | Power Controller → Clock Controller | 请求退出低功耗状态 |
| **CACTIVE** | Clock Controller → Power Controller | 指示时钟已恢复 / 请求进入低功耗 |
| **CLKSTOP** | Clock Controller → 各模块 | 时钟门控信号（内部使用） |

### 低功耗状态机

```
                ┌─────────────┐
                │  Normal     │
                │  (Active)   │
                └──────┬──────┘
                       │ CACTIVE=1, 总线空闲
                       │ Power Controller 判断可睡眠
                       ↓
                ┌─────────────┐
                │  Sleep Req  │
                │  (CSYSREQ=1)│
                └──────┬──────┘
                       │ 所有 AXI Master 完成当前事务
                       │ Clock Controller 停止时钟
                       ↓
                ┌─────────────┐
                │  Low-Power  │
                │  (Clk Stop) │  ← 无时钟，静态功耗
                └──────┬──────┘
                       │ CSYSREQ=0 或 AXI 新事务请求
                       │ Clock Controller 恢复时钟
                       ↓
                ┌─────────────┐
                │  Wake Up    │
                │  (CACTIVE=1)│
                └──────┬──────┘
                       │ 时钟稳定后回到 Normal
                       ↓
                ┌─────────────┐
                │  Normal     │
                │  (Active)   │
                └─────────────┘
```

### LPI 时序

```
CLK    ┌┐┌┐┌┐┌┐┌┐┌┐               ┌┐┌┐┌┐┌┐┌┐
正常时钟 ┘ └ ┘ └ ┘ └ ┘ └ ┘ ...  └ ┘ └ ┘ └ ┘ └ ┘
          │                      │
CACTIVE   ███████████_____________██████████████
                     │           │
CSYSREQ   ███████████_____________██████████████
                     │           │
             进入LP   │           │  退出LP
               ↓     │           │   ↓
             时钟停止  │           │  时钟恢复
```

### Xilinx FPGA 中的 LPI

```
ZYNQ PS 低功耗控制:
  - PS 内部 Power Controller 自动管理
  - PL 侧不直接处理 LPI 信号
  - PS-PL 接口 (AXI_HP/GP) 在 PS 进入睡眠时自动门控

PL 自定义低功耗设计:
  - Vivado IPI 不支持 AXI LPI 信号（需要手动实例化）
  - 更常见：使用 Clock Gating Cell（BUFGCE）门控 AXI 时钟
  - 或：通过 AXI4-Lite 控制寄存器关闭 IP 内部时钟

简单 FPGA 设计:
  无低功耗要求 → 忽略 LPI 信号，CACTIVE 恒为 1
```

---

## 二、性能模型与计算

### 吞吐量公式

```
理论峰值吞吐量:
  BW_peak = DataWidth × Freq
  例: 64bit × 200 MHz = 12.8 Gbps = 1.6 GB/s

实际吞吐量:
  BW_actual = BW_peak × Efficiency

  Efficiency = BurstData_Clocks / Total_Clocks
             = 突发传输时钟数 / (地址+数据+间隙总时钟数)
```

### 典型效率场景

```
场景 1: 长突发，无间隙 (最佳)
  AW: █
  W:  ████████ (8 拍, WLAST=1)
  B:           █
  Efficiency ≈ 8/10 = 80%（地址和响应开销）

场景 2: 长突发，有间隙 (常见)
  AW: █
  W:  ██░░████░░██ (8 拍, 中间被 READY 拉低暂停)
  B:              █
  Efficiency ≈ 8/15 = 53%

场景 3: 单拍传输 (最差)
  AW: █
  W:  █
  B:   █
  Efficiency ≈ 1/3 = 33%
```

### 不同配置的实测参考

| 配置 | 位宽 | 频率 | 理论峰值 | 典型实际 | 场景 |
|:---:|:----:|:----:|:--------:|:--------:|------|
| AXI_GP (ZYNQ) | 32bit | 150 MHz | 600 MB/s | ~400 MB/s | PS-PL 通用接口 |
| AXI_HP (ZYNQ) | 64bit | 150 MHz | 1.2 GB/s | ~900 MB/s | PS-PL 高速接口 |
| MIG DDR3 (ACX720) | 16bit | 400 MHz DDR | 1.6 GB/s | ~1.2 GB/s | 板载 DDR3 |
| AXI_Stream (PL) | 32bit | 200 MHz | 800 MB/s | ~750 MB/s | 流式数据管道 |

### 延迟构成

```
一次 AXI4 读事务的总延迟:

  T_total = T_addr_out + T_slave_latency + T_data_back

  其中:
    T_addr_out    = 地址通道握手延迟（1~N 周期）
    T_slave_latency = Slave 内部延迟（DDR: CAS+行列选通）
    T_data_back   = 数据返回握手延迟（1~N 周期）

  DDR3 读示例 (ACX720, 1066 MT/s):
    T_addr_out   ≈ 2~5  clk
    T_slave_lat  ≈ 10~15 clk (150 MHz AXI 时钟域)
    T_data_back  ≈ 2~5  clk
    Total        ≈ 14~25 clk ≈ 93~167 ns
```

---

## 三、性能优化策略

### 1. 增大突发长度

```
短突发 (AWLEN=3, 4 拍):
  地址开销占比大 → 效率低
  
长突发 (AWLEN=63, 64 拍):
  地址开销占比小 → 效率高

建议:
  - DDR 访问: 突发长度 ≥ 16（AXLEN ≥ 15）
  - 在数据位宽 × 突发长度接近 Cache Line 大小 (64B) 时最优
  - 64bit × 8 拍 = 64B = 1 Cache Line
```

### 2. 增加乱序深度

```
乱序深度对吞吐量的影响（模拟数据）:

  ┌────────┬──────────┐
  │ 乱序深度 │ 效率     │
  ├────────┼──────────┤
  │ 1      │ ~40%     │
  │ 2      │ ~55%     │
  │ 4      │ ~75%     │
  │ 8      │ ~90%     │
  │ 16     │ ~95%     │
  └────────┴──────────┘

实现方法:
  - Master 端: 维护多个 outstanding 事务队列
  - Slave 端: 增大内部 FIFO / 多 Bank 交错
  - Interconnect 端: 支持多 ID 交错传输
```

### 3. 匹配数据位宽

```
Master ↔ Slave 位宽不匹配会引入额外周期:

Master 64bit, Slave 32bit:
  AXI Data Width Converter 将 64bit 拆为 2 个 32bit 传输
  吞吐量减半！

建议:
  - 系统关键路径上位宽保持一致
  - 使用 AXI Data Width Converter 时注意位宽比
  - DMA 数据位宽与 DDR 数据位宽匹配
```

### 4. 减少握手停顿

```
Master 端优化:
  - 数据准备好再置 VALID（避免 VALID 空等待）
  - 提前准备好地址（地址通道不成为瓶颈）
  - 支持 W 通道数据早于 AW 通道发出

Slave 端优化:
  - 提前置 READY（看到 VALID 立即握手）
  - 内部流水线深度 ≥ 2（能不暂停就不暂停）
  - AW 和 W 通道同时准备好（无需等待对方）
```

### 5. 使用正确的 Interconnect

```
AXI Interconnect (PG059):
  - 共享总线/交叉开关
  - 全功能但逻辑多 → 时序压力大
  - 适合低速/少端口场景

AXI SmartConnect (PG201):
  - 仅交叉开关
  - 精简逻辑 → 时序好
  - 支持多时钟域、多接口协议
  - 7 系列+ 推荐选用

Register Slice:
  - 打断长路径
  - 增加 1 周期延迟
  - 显著提升 Fmax
  - 长路径上必须加！
```

### 6. AXI4-Stream 优化

```
AXI4-Stream 的特殊优化点:

取消批处理 (取消数据使能抖动):
  - Master: TVALID 保持为 1，直到突发结束
  - Slave: TREADY 保持为 1，除非 FIFO 满
  → 效率接近 100%

TKEEP 优化:
  - 数据对齐时 TKEEP 全 1，不产生气泡
  - 非对齐场景尽量让 Master 处理对齐

TLAST 尽早置位:
  - 最后一拍与数据同时有效
  - 避免额外等待周期
```

### 7. 实践 check list

```
性能优化 checklist:

[ ] 确定系统带宽瓶颈是 AXI 总线还是 Slave 内部
[ ] 使用大突发长度（≥16 拍）
[ ] 打开乱序传输（≥4 个 outstanding）
[ ] 匹配关键路径位宽（DMA ↔ DDR 位宽一致）
[ ] 优先选用 SmartConnect 而非 Interconnect
[ ] 长路径插入 Register Slice
[ ] 检查 WSTRB 使用（每字节独立写使能）
[ ] Monitor 实际效率，分析握手停顿原因
```

---

## 四、Vivado 性能分析工具

### AXI Performance Monitor

```
Vivado 中实例化 AXI Performance Monitor IP:
  - 监控 AXI 通道的 VALID/READY 握手
  - 统计吞吐量、延迟、突发长度分布
  - 导出为 CSV 分析

关键统计指标:
  - 写通道效率 = WVALID & WREADY 周期占比
  - 读通道效率 = RVALID & RREADY 周期占比
  - 平均突发长度
  - 地址到数据延迟
```

### ILA 手动测量

```verilog
ILA 触发条件示例:

// 测量写吞吐量: 计数 WVALID & WREADY
// 计算: 计数值 × 数据位宽 / 时间

// 测量读延迟:
// 从 ARVALID & ARREADY 到 RVALID & RREADY
// 使用触发器测量周期数
```

---

## 参考

- ARM AMBA AXI4 Protocol Specification (IHI0022E) — Appendix E (LPI)
- Xilinx UG1037: AXI Reference Guide — Performance
- Xilinx PG201: AXI SmartConnect
- Xilinx PG059: AXI Interconnect
- [[30AXI4总线协议详解]] — AXI4 基础
- [[31AXI4_Atomic原子操作与排序模型]] — 乱序与排序
