# UART串口发送模块设计与验证

## 概述

UART（Universal Asynchronous Receiver/Transmitter）是一种**异步串行通信**协议。它不需要时钟线，通信双方通过约定相同的**波特率**来同步。

### 一帧数据格式（8N1，最常用）

```
空闲(高) ──→ 起始位(0) ──→ bit0 ──→ bit1 ──→ ... ──→ bit7 ──→ 停止位(1) ──→ 空闲(高)
           └──────────── 共 10 位 ────────────┘
```

| 字段 | 说明 |
|------|------|
| 起始位 | 拉低 1 位时间，告诉接收端开始 |
| 数据位 | 8 位，LSB first（低位先发） |
| 停止位 | 拉高 1 位时间 |
| 空闲 | 保持高电平 |
| 波特率 | 每秒传输的位数，如 115200 |

## 工程结构

```
Usart_top (顶层)
├── Clk_Div      → 波特率时钟发生器
└── Uart_Byte_Tx → 字节发送控制器
```

### 模块划分

| 模块 | 功能 | 输入 | 输出 |
|------|------|------|------|
| `Clk_Div` | 50MHz 分频产生 115200Hz 脉冲 | clk, rst_n | bps_clk (脉冲) |
| `Uart_Byte_Tx` | 按 UART 协议发送一个字节 | clk, rst_n, bps_clk, data_byte[7:0], send_en | uart_tx, tx_done |
| `Usart_top` | 顶层，实例化子模块 + 状态机控制流程 | clk, rst_n | uart_tx |

## 代码原理

### Clk_Div — 波特率发生器

50MHz / 115200 ≈ 434 个时钟周期。计数器从 0 到 433，每到 433 输出一个高脉冲。

```verilog
// 关键逻辑
always @(posedge Clk_In or negedge rst_n) begin
    if (!rst_n) begin
        Count <= 0;
        Clk_Out <= 0;
    end else if (Count == 433) begin
        Count <= 0;
        Clk_Out <= 1;          // 输出一个周期的高脉冲
    end else begin
        Count <= Count + 1;
        Clk_Out <= 0;
    end
end
```

> **注意**：433 = 434 - 1，因为计数从 0 开始。`434 * 115200 ≈ 50MHz`

### Uart_Byte_Tx — 字节发送

使用位计数器 `bit_cnt`（0~9）配合 `bps_clk` 逐位发送。

```verilog
// 两个关键寄存器
reg [3:0] bit_cnt;   // 当前发送到的位位置
reg tx_busy;         // 发送中标志

// 发送流程
send_en && !tx_busy  → 拉低 uart_tx（起始位）, tx_busy=1
bps_clk 到来时:
  bit_cnt=0~7: 输出 data_byte[bit_cnt]
  bit_cnt=8:   输出 1（停止位）
  bit_cnt=9:   tx_done=1, tx_busy=0
```

> **理解非阻塞赋值 `<=`**：`bit_cnt <= bit_cnt + 1` 和 `case(bit_cnt)` 同时执行，case 读到的是加 1 **前**的值。

### Usart_top — 顶层控制状态机

上电复位后自动发送一次 `0x55`（二进制 0101_0101）：

```verilog
状态0: send_en=1, data_byte=8'h55 → 状态1
状态1: send_en=0（脉宽）, 等待 tx_done → 状态2
状态2: 永久空闲
```

## 引脚分配（ACX720 开发板）

| 端口 | FPGA 引脚 | 说明 |
|------|-----------|------|
| clk | Y18 | 50MHz 有源晶振 |
| rst_n | F15 | 按键 S0，按下为低 |
| uart_tx | M15 | 连到 CH340（USB 转串口） |

约束文件写法：
```tcl
create_clock -name clk -period 20.000 [get_ports clk]
set_property PACKAGE_PIN Y18 [get_ports clk]
set_property PACKAGE_PIN F15 [get_ports rst_n]
set_property PACKAGE_PIN M15 [get_ports uart_tx]
set_property IOSTANDARD LVCMOS33 [get_ports {clk rst_n uart_tx}]
```

## 调试心得

### 问题1：send_en 脉宽过长导致重复发送
- **现象**：复位一次收到 `UU`（两帧 0x55）
- **原因**：send_en 在 tx_done 后才拉低，中间多了一个周期的重叠
- **解决**：进入状态1后立刻拉低 send_en

### 问题2：VIO IP 核使用
- 需要先在 IP Catalog 中生成 IP，再写实例化代码
- probe_in 数量必须与 IP 配置一致
- 驱动同一信号的两个源（状态机 + VIO）会产生冲突

## 板级验证

1. 烧录 .bit 文件到 FPGA
2. 用 USB 线连接开发板的 CH340 口到电脑
3. `screen /dev/ttyUSB0 115200`
4. 按 S0 复位，应收到 `U`（0x55 的 ASCII）

## 扩展方向

- 添加 VIO IP 核 → 实时修改 data_byte，不用重新烧录
- 添加 UART_RX 模块 → 实现收发回环
- 多字节发送 → 用状态机发送字符串
- 自定义帧协议 → 帧头 + 数据 + 校验
