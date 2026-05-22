# D触发器与Verilog实现

## 概述

D 触发器（D Flip-Flop）是数字电路中最基本的**时序单元**。它在时钟上升沿（或下降沿）将输入 D 的值**采样并保持**到输出 Q，直到下一个时钟沿。

## 符号

```
 D ──→ [DFF] ──→ Q
          ↑
         clk
```

## 时序图

```
clk  ─ ╱￣╲__╱￣╲__╱￣╲__╱￣╲__╱￣╲__
D    ────────[ 1 ]──────[ 0 ]─────────
Q    ──[之前的值]── 1 ─────── 0 ──────
                    ↑          ↑
              上升沿采样    上升沿采样
```

## Verilog 实现

### 1. 最简单的 D 触发器（无复位）

```verilog
always @(posedge clk) begin
    q <= d;
end
```

### 2. 带异步复位的 D 触发器

```verilog
always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        q <= 1'b0;    // 复位值
    else
        q <= d;       // 时钟沿采样
end
```

`rst_n` 出现在敏感列表里 → **异步复位**（不受时钟控制，立即复位）

### 3. 带异步复位+置位的 D 触发器

```verilog
always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        q <= 1'b0;
    else if (set)
        q <= 1'b1;
    else
        q <= d;
end
```

### 4. 多个 D 触发器串联（打拍同步器）

三个 DFF 首尾相连，用于同步异步输入信号：

```verilog
reg rx_sync1;
reg rx_sync2;
reg rx_sync3;

always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        rx_sync1 <= 1'b1;
        rx_sync2 <= 1'b1;
        rx_sync3 <= 1'b1;
    end else begin
        rx_sync1 <= rs232_rx;   // DFF1: 输入采样
        rx_sync2 <= rx_sync1;   // DFF2: 第一级输出
        rx_sync3 <= rx_sync2;   // DFF3: 第二级输出
    end
end
```

### 数据流

```
rs232_rx ──→ [DFF1] ──→ [DFF2] ──→ [DFF3] ──→
               ↑          ↑          ↑
             rx_sync1   rx_sync2   rx_sync3
```

每个时钟上升沿，数据向后传一级：

| 时钟沿 | rs232_rx | rx_sync1 | rx_sync2 | rx_sync3 |
|--------|----------|----------|----------|----------|
| 0 | 1 | 1 (reset) | 1 (reset) | 1 (reset) |
| 1 | 1 | 1 | 1 | 1 |
| 2 | 0 | **1→0** | 1 | 1 |
| 3 | 0 | 0 | 1→**0** | 1 |
| 4 | 0 | 0 | 0 | 1→**0** |

- `rx_sync2` = 同步后的稳定信号（用于采样判断）
- `rx_negedge = rx_sync2 & ~rx_sync3` = 下降沿检测脉冲

## 常见误区

### 误区 1：用移位寄存器代替打拍

```verilog
// ❌ 这是 4 位移位寄存器，不是 3 个独立 DFF
rx_sync1 <= {rx_sync1[2:0], rs232_rx};
```

这样 `rx_sync1` 是 4 位的，后续比较需要指定位，容易搞混。

```verilog
// ✓ 3 个独立 DFF，每级 1 位
rx_sync1 <= rs232_rx;
rx_sync2 <= rx_sync1;
rx_sync3 <= rx_sync2;
```

### 误区 2：阻塞赋值

```verilog
// ❌ 阻塞赋值，综合出 latch 或错误逻辑
always @(posedge clk) begin
    rx_sync1 = rs232_rx;
    rx_sync2 = rx_sync1;
    rx_sync3 = rx_sync2;
end
```

阻塞赋值会在同一时钟沿**立即更新**，导致三级同步实际退化为一级。

### 误区 3：用组合逻辑判断沿

```verilog
// ❌ 组合逻辑直接用原始信号
assign negedge = !rs232_rx && old_rx;

// ✓ 同步后再判断
wire rx_negedge = rx_sync2 & ~rx_sync3;
```

## 拼接运算符 `{}` 与移位寄存器

### 语法

```verilog
{信号A, 信号B, 信号C}  // 首尾拼接成一个新向量
```

### 示例

```verilog
{1'b0, 1'b1}            → 2'b01
{4'b1010, 4'b0101}      → 8'b1010_0101
{a[3:0], b[3:0]}        → 8 位: {a高位, b低位}
```

### UART 接收中的移位

```verilog
rx_shift <= {rx_sync2, rx_shift[7:1]};
```

#### 分步拆解

```
rx_shift = [b7, b6, b5, b4, b3, b2, b1, b0]
               ↑                            ↑
              [7]                         [0]

rx_shift[7:1] = [b7, b6, b5, b4, b3, b2, b1]
                └─── 丢掉 b0（右移挤出）───┘

{rx_sync2, rx_shift[7:1]}
    ↑            ↑
 新来的 bit    旧的 7 位右移
```

#### 效果：每采样一次，数据右移一位，新 bit 从最高位进入

```
复位: 0000_0000
bit0: {0, 0000_000} = 0000_0000
bit1: {1, 0000_000} = 1000_0000
bit2: {1, 1000_000} = 1100_0000
bit3: {0, 1100_000} = 0110_0000
bit4: {1, 0110_000} = 1011_0000
bit5: {0, 1011_000} = 0101_1000
bit6: {0, 0101_100} = 0010_1100
bit7: {1, 0010_110} = 1001_0110  ← 0x96
```

### LSB first 的数据方向

UART 发送顺序：bit0(LSB) → bit1 → ... → bit7(MSB)

| 采样 | 写入位置 | 到达 rx_shift 最终位置 |
|------|---------|----------------------|
| bit0(LSB) | [7] | 8 次右移后 → [0] |
| bit1 | [7] | 7 次右移后 → [1] |
| ... | ... | ... |
| bit7(MSB) | [7] | 停在第 [7] 位 |

### 等价写法

```verilog
// 右移 + 新 bit 放高位
rx_shift <= {rx_sync2, rx_shift[7:1]};

// 等价于左移 + 新 bit 放低位
rx_shift <= {rx_shift[6:0], rx_sync2};
// 此时 bit0(先到) → [0]，bit7(后到) → [7]，结果相同
```

## 状态机（FSM）设计思路

### 核心概念

**软件**：做完 A 自动执行 B — `while(条件)` → `delay()` → `for()` 顺序执行

**硬件**：每个时钟沿问自己一次"我在哪个状态？条件满足了吗？"— 没有顺序，只有轮询

### 基本结构

```verilog
// 状态定义
parameter IDLE = 0, START = 1, DATA = 2, STOP = 3;
reg [1:0] state;

always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        state <= IDLE;
    else begin
        case (state)
            IDLE:  if (条件) state <= NEXT_STATE;
            NEXT:  if (条件) state <= ANOTHER;
            // ...
        endcase
    end
end
```

**没有 `while`、没有 `delay()`、没有顺序执行。只有每个时钟沿的判断。**

### 两种写法

#### 方式 A：状态切换 + 状态内动作混写（简单，适合小状态机）

```verilog
always @(posedge clk) begin
    case (state)
        IDLE: begin
            counter <= 0;
            if (start) state <= WORK;
        end
        WORK: begin
            counter <= counter + 1;
            if (counter == MAX) state <= DONE;
        end
        DONE: begin
            done_flag <= 1;
            state <= IDLE;
        end
    endcase
end
```

#### 方式 B：状态切换 + 状态内动作分开写（推荐，逻辑更清晰）

```verilog
// always 1：只管"什么时候切状态"
always @(posedge clk) begin
    case (state)
        IDLE:  if (rx_negedge)        state <= START;
        START: if (div_cnt == 216)    state <= DATA;
        DATA:  if (bit_cnt == 8)      state <= STOP;
        STOP:  if (div_cnt == 216)    state <= IDLE;
    endcase
end

// always 2：只管"每个状态里做什么"
always @(posedge clk) begin
    case (state)
        IDLE:  div_cnt <= 0;
        START: div_cnt <= div_cnt + 1;
        DATA:  if (div_cnt == 216) begin
                   rx_shift[bit_cnt] <= rx_sync2;
                   bit_cnt <= bit_cnt + 1;
               end
        STOP:  if (div_cnt == 216) begin
                   data_byte <= rx_shift;
                   data_valid <= 1;
               end
    endcase
end
```

### 状态机执行流程

```
时钟沿来了：
  1. 检查当前 state
  2. 检查该状态的条件（div_cnt==216？bit_cnt==8？）
  3. 条件满足 → 做事 + 切到下一个 state
  4. 条件不满足 → 继续在当前 state 做事（比如计数器继续加）
  5. 等下一个时钟沿再来一次
```

### UART RX 对应

| 状态 | 切换条件 | 做的事 |
|------|---------|--------|
| IDLE | `rx_negedge`（下降沿） | 复位 div_cnt=0 |
| START | `div_cnt==216`（半比特到） | 验证 rx_sync2==0 |
| DATA | `bit_cnt==8`（采完 8 位） | div_cnt==216 时采样 |
| STOP | `div_cnt==216`（半比特到） | 输出 data_byte + data_valid 脉冲 |

### 和软件对比

| | 软件 | 硬件 |
|--|------|------|
| 控制流 | `while` `for` 顺序执行 | `case` 每拍轮询 |
| 等待 | `delay()` `sleep()` | 计数器到位触发 |
| 状态切换 | 代码自然流到下一行 | 条件满足才切 |
| 定时 | 系统调用阻塞 | div_cnt 自由跑，到值触发 |

## 记忆口诀

> **同步打两拍，沿检测再加一拍**
> - 第一拍：捕获异步输入
> - 第二拍：输出已稳定，**用来采样**
> - 第三拍：延迟一拍，**用来和前一拍一起做边沿检测**
