# UART串口接收模块设计与验证

## 概述

UART 接收端需要**异步检测起始位**，然后**在位的正中间采样**。不像发送端主动控制节奏，接收端是被动的——不知道数据何时到达。

### 和发送模块的核心区别

| | 发送 (TX) | 接收 (RX) |
|--|----------|----------|
| 时序控制 | 自己启动，知道节奏 | 被动等起始位，硬同步 |
| 计数器 | 自由运行或启动后计数 | 下降沿时复位对齐 |
| 波特率时钟 | `bps_clk` 直接可用 | 需用自己的 `div_cnt` 对齐相位 |

## 工程结构

```
Usart_top (顶层)
└── Uart_Byte_Rx → 字节接收控制器
```

无需波特率发生器，RX 内部自带 `div_cnt` 计数器。

## 模块端口

```verilog
module Usart_Byte_Rx(
    input           clk,
    input           rst_n,
    input           rs232_rx,       // 来自 PC 的串口线 (J21)
    output [7:0]    data_byte,
    output reg      data_valid      // 单周期脉冲
);
```

## 核心设计

### 1. 异步信号同步

`rs232_rx` 与 FPGA 时钟异步，需三级 DFF 打拍防亚稳态：

```verilog
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        rx_sync1 <= 1'b1;
        rx_sync2 <= 1'b1;
        rx_sync3 <= 1'b1;
    end else begin
        rx_sync1 <= rs232_rx;
        rx_sync2 <= rx_sync1;
        rx_sync3 <= rx_sync2;
    end
end
```

- `rx_sync2`：稳定的同步信号，用于采样判断
- `rx_negedge = ~rx_sync2 & rx_sync3`：下降沿检测

### 2. 位计数器（替代 bps_clk）

用 `bps_clk` 的问题：它是自由运行的，和数据不同步。

改用**自己的计数器**，在下降沿复位到 0，相位对齐：

```verilog
reg [8:0] div_cnt;  // 0~433
// 在每拍的 case 分支里：div_cnt <= (div_cnt == 433) ? 0 : div_cnt + 1;
// 下降沿时由 IDLE 状态：div_cnt <= 0;
```

### 3. 采样时机

关键洞察：**计数器自由循环，每次走到 216 即为正中间**

```
下降沿 (div_cnt=0)
  ↓
div_cnt=0 → 1 → ... → 216 → 217 → ... → 433 → 0 → 1 → ... → 216 → ...
                        ↑                                ↑
                   半比特到 (217拍)                    又过 434 拍
                  起始位正中                          bit0 正中
```

计数器不重置，自由跑，每次 `div_cnt == 216` 就是下一比特的中间。

### 4. 状态机

```
IDLE ─ nege ─→ START ─ div_cnt==216 ─→ DATA ─ bit_cnt==7 ─→ STOP ─ div_cnt==216 ─→ IDLE
                      └ rx_sync2==0 ┘                                 └ rx_sync2==1 ┘
                      (验证起始位有效)                                (验证停止位有效)
```

```verilog
case (state)
    IDLE: begin
        div_cnt <= 0;
        bit_cnt <= 0;
        data_valid <= 0;
        if (rx_negedge) state <= START;
    end
    START: begin
        div_cnt <= (div_cnt == 433) ? 0 : div_cnt + 1;
        if (div_cnt == 216) begin
            if (rx_sync2 == 0)  state <= DATA;
            else                state <= IDLE;  // 毛刺，丢弃
        end
    end
    DATA: begin
        div_cnt <= (div_cnt == 433) ? 0 : div_cnt + 1;
        if (div_cnt == 216) begin
            rx_shift <= {rx_sync2, rx_shift[7:1]};
            if (bit_cnt == 7) begin
                bit_cnt <= 0;
                state <= STOP;
            end else begin
                bit_cnt <= bit_cnt + 1;
            end
        end
    end
    STOP: begin
        div_cnt <= (div_cnt == 433) ? 0 : div_cnt + 1;
        if (div_cnt == 216) begin
            if (rx_sync2 == 1) begin
                data_byte <= rx_shift;
                data_valid <= 1;
            end
            state <= IDLE;
        end
    end
endcase
```

### 5. 移位寄存器（LSB first）

```verilog
rx_shift <= {rx_sync2, rx_shift[7:1]};
```

UART 先发 bit0（LSB），数据从最高位进入，右移 8 次后 LSB 到达最低位。

## 顶层模块

```verilog
module Usart_top(
    input           clk,
    input           rst_n,
    input           rs232_rx,
    output [7:0]    led
);

    wire [7:0] rx_data;
    wire rx_valid;

    Usart_Byte_Rx u_rx (
        .clk        (clk),
        .rst_n      (rst_n),
        .rs232_rx   (rs232_rx),
        .data_byte  (rx_data),
        .data_valid (rx_valid)
    );

    reg [7:0] led_reg;
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            led_reg <= 8'b0;
        else if (rx_valid)
            led_reg <= rx_data;
    end

    assign led = led_reg;
endmodule
```

接收到的字节显示在 LED 上，保持到下一字节到达。

## 管脚约束

| 信号 | 管脚 | 电平 |
|------|------|------|
| clk | Y18 | LVCMOS33 |
| rst_n | F15 | LVCMOS33 |
| rs232_rx | **J21** | LVCMOS33 |
| led[0] | M22 | LVCMOS33 |
| led[1] | N22 | LVCMOS33 |
| led[2] | L21 | LVCMOS33 |
| led[3] | K21 | LVCMOS33 |
| led[4] | K22 | LVCMOS33 |
| led[5] | J22 | LVCMOS33 |
| led[6] | H22 | LVCMOS33 |
| led[7] | M21 | LVCMOS33 |

## 验证结果

| 发送字符 | ASCII | LED 显示 | 结果 |
|---------|-------|---------|------|
| `1` | 0x31 (0011_0001) | 0011_0001 | ✓ |
| `2` | 0x32 (0011_0010) | 0011_0010 | ✓ |
| `A` | 0x41 (0100_0001) | 0100_0001 | ✓ |

- 波特率：115200
- 时钟：50MHz
- 分频系数：434（50_000_000 / 115200）
- 误差：0.006%（远小于 ±2% 容限）

## 关键经验

1. **起始位硬同步**：每个字节的下降沿重新对齐计数器，误差不累积
2. **不能依赖 bps_clk**：它是自由运行的，相位与数据无关
3. **边沿极性**：`~rx_sync2 & rx_sync3` 是下降沿，`rx_sync2 & ~rx_sync3` 是上升沿
4. **非阻塞赋值**：打拍必须用 `<=`，`=` 会退化为一级延迟
5. **状态机用 case**：比 if/else 链更清晰，不会漏 `end`
