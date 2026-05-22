# UART 回环 (Loopback) 设计与验证

## 概念

把 UART RX 收到的数据直接发给 UART TX，实现"发什么回什么"。

```
PC 发 A  →  CH340  →  FPGA RX  →  UART_Byte_RX  →  UART_Byte_TX  →  FPGA TX  →  CH340  →  PC 收 A
```

## 工程结构

```
USART_Loopback/
├── build.tcl          ← 项目构建脚本
├── output/
│   └── Usart_top.bit  ← 比特流
└── src/
    ├── constrs/
    │   └── cons.xdc   ← 引脚约束
    └── hdl/
        ├── Clk_Div.v      ← 分频模块（TX 用）
        ├── Uart_Byte_Tx.v ← 串口发送模块
        ├── Uart_Byte_Rx.v ← 串口接收模块（模块名 Usart_Byte_Rx）
        └── Usart_top.v    ← 顶层（连接 RX → TX）
```

## 顶层连接

```verilog
Clk_Div      → bps_clk  → TX 波特率时钟
Usart_Byte_Rx → data_byte → TX.data_byte
Usart_Byte_Rx → data_valid → TX.send_en
```

- `data_valid`（单周期脉冲）直接连到 `send_en`
- `data_byte` 直接连到 `data_byte`
- RX 内部自带 `div_cnt` 计数器采样，不需要外部波特率时钟

## 关键点

- TX 用 `bps_clk`（Clk_Div 每 434 周期产生 1 个脉冲）驱动位计数器
- RX 用自己独立的 `div_cnt`（0~433）采样，在 216 处采样位中心
- 波特率计算：50MHz / 434 ≈ 115200
- `data_valid` 连 `send_en`，收到即发回

## 验证

用 `screen /dev/ttyUSB0 115200` 测试，发任意字符应收到相同字符回显。
