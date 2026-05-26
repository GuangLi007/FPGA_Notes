# 74HC595 详解：串并转换原理与 Verilog 实现

## 一、74HC595 是什么

74HC595 是 **串行输入 → 并行输出** 的移位寄存器芯片，自带**存储寄存器**（可锁存）。核心功能：用少量引脚（3 根）控制大量输出（8 根），是 FPGA/单片机"引脚不够用"时的经典解决方案。

### 内部结构

```
                  ┌─────────────┐
SER (串行输入) ───→│ 8位移位寄存器 │──→ Q7' (级联输出)
                  │  (串入串出)  │
                  └──────┬──────┘
                         │ 并行传输
                    ┌────▼──────┐
RCLK (锁存时钟) ───→│ 8位存储寄存器 │──→ QA~QH (并行输出)
                    └────┬──────┘
                         │
                    OE (输出使能, 低有效)
```

**双寄存器结构的意义**：移位时可以不影响输出，全部移完后再一次性更新输出，避免中间态。

### 引脚功能

| 引脚 | 名称 | 说明 |
|------|------|------|
| 14 | SER | 串行数据输入 |
| 11 | SCK (SRCLK) | 移位时钟，上升沿移入 1 bit |
| 12 | RCK (RCLK) | 锁存时钟，上升沿更新输出 |
| 10 | SCLR (SRCLR) | 移位寄存器清空(低有效)，不用接 VCC |
| 13 | OE (G) | 输出使能(低有效)，不用接 GND |
| 9 | Q7' (QH') | 级联输出，接下一片 SER |
| 15~7 | QA~QH | 并行输出 |

## 二、时序图

### 单次 8 位传输

```
SCLK  __̅__̅__̅__̅__̅__̅__̅__̅__̅__̅__̅__̅__̅__̅__̅_
       ┆  ┆  ┆  ┆  ┆  ┆  ┆  ┆
SER   ---<b7><b6><b5><b4><b3><b2><b1><b0>---
       ┆  ┆  ┆  ┆  ┆  ┆  ┆  ┆
RCLK  __________________________________̅___
```

1. SCLK 每个上升沿，SER 当前电平移入移位寄存器（MSB first）
2. 8 个时钟后，SER 上的 8 位数据进入移位寄存器
3. RCLK 上升沿，移位寄存器内容**一次性**锁存到存储寄存器，QA~QH 更新

### 级联 16 位传输

```
SCLK  __̅____̅__ ... 16个上升沿 ... __̅____̅__
       ┆  ┆               ┆  ┆
SER   <b15><b14> ...     <b1><b0>
       ┆  ┆               ┆  ┆
RCLK  ________________________________̅___
```

- 两片 595 级联：Q7' 接下一片 SER
- 先移入的 8 位 → 第二片（后级）
- 后移入的 8 位 → 第一片（前级）
- 16 个 SCLK 后，RCLK 一起锁存

> **口诀**：先发的数据去得远（到后级），后发的数据留得近（在前级）。

## 三、Verilog 实现方式对比

### 方式 A：状态机驱动（推荐，可控性强）

```verilog
module hc595_driver_fsm (
    input  wire       clk,
    input  wire       rst_n,
    input  wire [15:0] data,
    input  wire       trig,        // 启动传输脉冲
    output reg        ds,          // SER
    output reg        sh_cp,       // SCLK
    output reg        st_cp,       // RCLK
    output reg        busy         // 忙标志
);

reg [15:0] r_data;
reg [4:0]  bit_cnt;  // 0~16 计数

localparam IDLE  = 2'b00;
localparam SHIFT = 2'b01;
localparam LATCH = 2'b10;

reg [1:0] state;

always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        state  <= IDLE;
        ds     <= 0;
        sh_cp  <= 0;
        st_cp  <= 0;
        busy   <= 0;
        bit_cnt <= 0;
    end else case (state)
        IDLE: begin
            st_cp <= 0;
            if (trig) begin
                r_data <= data;
                ds     <= data[15];      // 先发 MSB
                bit_cnt <= 1;
                busy   <= 1;
                state  <= SHIFT;
            end
        end

        SHIFT: begin
            if (bit_cnt[0]) begin        // 奇数拍: SCLK 上升沿
                sh_cp <= 1;
                bit_cnt <= bit_cnt + 1;
            end else begin               // 偶数拍: 准备下一位数据
                sh_cp <= 0;
                if (bit_cnt < 31) begin  // 31 对应第 16 位 (0~31 = 16 个周期)
                    ds        <= r_data[14];
                    r_data    <= {r_data[14:0], 1'b0};
                end
                bit_cnt <= bit_cnt + 1;
                if (bit_cnt == 31)       // 16 位移完
                    state <= LATCH;
            end
        end

        LATCH: begin
            st_cp <= 1;
            state <= IDLE;
            busy  <= 0;
        end
    endcase
end

endmodule
```

**时序细节**：
```
         ┌─┐  ┌─┐  ┌─┐        ┌─┐  ┌─┐
CLK      ┘ └──┘ └──┘ └── ... ──┘ └──┘ └──
bit_cnt   1   2   3           31  32  33
sh_cp     ─̅───̅───̅─ ... ────̅───────
ds       <b15><b14>          <b0>
st_cp    ────────────────────────̅────────
```

### 方式 B：计数器驱动（简洁，参考工程的写法）

已完成的 `Segment_Display` 工程即用此方式。核心思路：一个计数器 `cnt 0~33` 覆盖所有时序状态，用 `cnt[0]` 区分 SCLK 高低电平。

```verilog
module Hc595_Driver (
    input  wire       clk,
    input  wire       reset_n,
    input  wire [15:0] data,
    input  wire       s_en,      // 启动传输
    output reg        ds,        // SER 串行数据
    output reg        sh_cp,     // SCLK 移位时钟
    output reg        st_cp      // RCLK 锁存时钟
);

reg [15:0] r_data;
reg [5:0]  cnt;
reg        active;

always @(posedge clk or negedge reset_n) begin
    if (!reset_n) begin
        active <= 1'b0;
        cnt    <= 0;
        ds     <= 1'b0;
        sh_cp  <= 1'b0;
        st_cp  <= 1'b0;
        r_data <= 16'd0;
    end else if (!active) begin
        sh_cp <= 1'b0;
        st_cp <= 1'b0;
        if (s_en) begin          // 收到启动信号
            active <= 1'b1;      // 进入移位状态
            cnt    <= 1;         // 从 1 开始 (奇数拍拉高 SCLK)
            r_data <= data;      // 锁存待发数据
            ds     <= data[15];  // 先发 MSB
        end
    end else begin
        if (cnt == 33) begin     // 16 bit × 2 拍 + 1 拍锁存 = 33
            st_cp  <= 1'b1;      // 锁存脉冲上升沿
            active <= 1'b0;      // 传输完毕
            cnt    <= 0;
        end else if (cnt[0]) begin // 奇数: SCLK 上升沿 (移位)
            sh_cp <= 1'b1;
            cnt   <= cnt + 1;
        end else begin            // 偶数: SCLK 低电平, 准备下一位
            sh_cp <= 1'b0;
            if (cnt < 31) begin   // 最后一位不发新数据 (保留旧值)
                ds     <= r_data[14];
                r_data <= {r_data[14:0], 1'b0};
            end
            cnt <= cnt + 1;
        end
    end
end

endmodule
```

**时序流水表** (`cnt` 值及对应动作)：

| cnt | sh_cp | ds | 动作 |
|-----|-------|----|------|
| 0 | 0 | — | 空闲，等待 s_en |
| 1 | 1 | b15 | SCLK↑，移入 b15 |
| 2 | 0 | b14 | 准备下一位 |
| 3 | 1 | b14 | SCLK↑，移入 b14 |
| 4 | 0 | b13 | 准备 |
| ... | ... | ... | ... |
| 31 | 1 | b0 | SCLK↑，移入 b0 |
| 32 | 0 | — | 保持 (cnt<31 条件不满足，不改 ds) |
| 33 | — | — | st_cp=1，锁存输出，回到空闲 |

**工程路径**：`/home/kl/My_Project/FPGA/Segment_Display/Segment_Display.srcs/sources_1/new/Hc595_Driver.v`

### 方式 C：带时钟分频的通用驱动

若系统时钟远快于 SCLK 需求，可加分频：

```verilog
// 每 4 个 clk 产生一个 SCLK 周期
reg [1:0] clk_div;

always @(posedge clk) clk_div <= clk_div + 1;
wire sclk_en = (clk_div == 2'b11);  // 每 4 clk = 80ns → 12.5MHz SCLK
```

---

## 四、串并转换的通用思路

HC595 本质是**串行→并行**的典型代表。在 FPGA 中，串并转换有两种实现方式：

### 方法 1：移位寄存器（基本款）

```verilog
// 串行输入 → 并行输出
reg [7:0] shift_reg;

always @(posedge clk)
    shift_reg <= {shift_reg[6:0], serial_in};

assign parallel_out = shift_reg;
```

每来一个时钟，串行数据从低位移入，8 个时钟后得到 8 位并行数据。

### 方法 2：计数器 + 移位（可控款）

```verilog
reg [7:0] shift_reg;
reg [2:0] bit_cnt;

always @(posedge clk) begin
    if (bit_cnt == 3'b111)
        parallel_out <= {shift_reg[6:0], serial_in};  // 收集完毕
    shift_reg <= {shift_reg[6:0], serial_in};
    bit_cnt   <= bit_cnt + 1;
end
```

### 方法 3：并转串（反过来，如 HC595 的驱动端）

```verilog
reg [7:0] shift_reg;

always @(posedge clk) begin
    serial_out <= shift_reg[7];
    shift_reg  <= {shift_reg[6:0], 1'b0};
end
```

### 串并转换的应用场景

| 场景 | 串行协议 | 并行端 | 典型速率 |
|------|---------|--------|---------|
| 数码管驱动 | HC595 协议 | 8 位段码/位选 | ~10 MHz |
| SPI 从机接收 | SPI MISO/MOSI/SCLK | 命令/数据寄存器 | ~50 MHz |
| I2C 通信 | SDA/SCL 双线 | 地址/数据 | ~400 kHz |
| 摄像头数据 | DVP PCLK 串行像素 | 8/10/12 位像素 | ~100 MHz |
| UART 接收 | RX 串行位流 | 8 位数据 | ~115.2 kHz |

## 五、仿真 testbench

### 驱动模块仿真

```verilog
`timescale 1ns / 1ps

module tb_hc595_driver;

reg        clk;
reg        rst_n;
reg [15:0] data;
reg        trig;
wire       ds;
wire       sh_cp;
wire       st_cp;
wire       busy;

hc595_driver_fsm uut (.*);

initial begin
    clk = 0;
    forever #10 clk = ~clk;  // 50MHz
end

initial begin
    rst_n = 0;
    data  = 16'hA5B7;
    trig  = 0;
    #100 rst_n = 1;
    #20  trig = 1;
    #20  trig = 0;
    #2000;
    $finish;
end

// 显示移出的位序列
always @(posedge sh_cp)
    $write("%b", ds);

// 锁存时显示结果
always @(posedge st_cp)
    $display(" | LATCHED");

endmodule
```

### HC595 芯片级仿真（行为模型）

```verilog
`timescale 1ns / 1ps

module hc595_model (
    input  wire       ser,    // 串行输入
    input  wire       sclk,   // 移位时钟
    input  wire       rclk,   // 锁存时钟
    input  wire       oe_n,   // 输出使能(低有效)
    output wire       q7_h,   // 级联输出
    output reg  [7:0] q       // 并行输出
);

reg [7:0] shift_reg;

// 移位
always @(posedge sclk)
    shift_reg <= {shift_reg[6:0], ser};

assign q7_h = shift_reg[7];

// 锁存
always @(posedge rclk)
    q <= shift_reg;

// 输出使能 (三态, 简化版)
wire [7:0] q_out;
assign q_out = oe_n ? 8'bz : q;

endmodule
```

### 级联仿真

```verilog
module tb_cascade;

reg        ser, sclk, rclk;
wire       q7_h_1;
wire [7:0] q1, q2;

hc595_model u1 (.ser(ser), .sclk(sclk), .rclk(rclk), .oe_n(0), .q7_h(q7_h_1), .q(q1));
hc595_model u2 (.ser(q7_h_1), .sclk(sclk), .rclk(rclk), .oe_n(0), .q7_h(), .q(q2));

initial begin
    ser = 0; sclk = 0; rclk = 0;
    // 移入 16 位: b15~b0 = 1010_0101_1111_0000
    repeat (16) begin
        ser = $urandom_range(0,1);  // 实际应按顺序给
        #20 sclk = 1; #20 sclk = 0;
    end
    #20 rclk = 1; #20 rclk = 0;
    $display("u1 = %b, u2 = %b", q1, q2);
end

endmodule
```

## 六、工程中的实际应用

### ACX720 上的连接

参考 [[11数码管74HC595动态扫描显示]]

### 典型时序参数

| 参数 | 值 |
|------|-----|
| SCLK 周期 | 40ns (25MHz) |
| 16 位传输时间 | 16×40ns = 640ns |
| 锁存脉冲宽度 | ≥20ns |
| 帧刷新率 | 125Hz (8ms/帧) |

### 常见问题

**Q: 为何要 count 到 33 而不是 32？**
因为每个 bit 需要 2 个 clk 周期（1 个置高 SCLK，1 个置低 SCLK），16 bit × 2 = 32，再加 1 个锁存周期 = 33。

**Q: 先发段码还是先发位选？**
取决于级联顺序。ACX720 上 FIFO 先入段码（到后级 595），后入位选（留在前级 595），所以数据拼接为 `{段码, 位选}`。发的时候先发 MSB（段码高位），16 位里前 8 位是段码。

**Q: 为何 RCLK 要等移位完再拉高？**
避免移位过程中输出出现中间态毛刺。典型做法：全部移完后，单独给一个 RCLK 脉冲。

## 七、扩展阅读

- 74HC595 Datasheet：`/home/kl/资料/小梅哥FPGA/07_器件手册/74HC595.pdf`
- 小梅哥参考例程：`ch12_acx720_hex8_hc595.rar`
- 已有笔记：[[11数码管74HC595动态扫描显示]]
- 工程记录：[[Segment_Display_1_工程记录与思路]]
