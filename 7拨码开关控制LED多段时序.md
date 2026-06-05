# 拨码开关控制 LED 多段时序

## 概述

8位拨码开关 SW[7:0] 分别控制 1 秒内 8 个 0.125s 时段的 LED 亮灭。核心机制：计数器 + 整数除法 `cnt / T` 生成段索引。

### 需求
- 1s 循环 = 8 段 × 0.125s
- 第 n 段：`led = SW[n]`
- 拨码开关拨上（1）→ LED 在该段亮，拨下（0）→ 灭

---

## 一、代码

```verilog
module Top(
    input clk,
    input rst_n,
    input [7:0] SW,
    output reg led
);

parameter T = 6_250_000;     // 0.125s = 6,250,000 cycles @50MHz

integer cnt;
reg [2:0] Control;

always @(posedge clk or negedge rst_n) begin
    if (rst_n == 0) begin
        cnt <= 0;
        led <= 0;
        Control <= 0;
    end else begin
        cnt <= cnt + 1;
        Control <= cnt / T;              // 每 T 周期 +1

        if (cnt == 50_000_000 - 1) begin // 1s 到，重置
            cnt <= 0;
            Control <= 0;
        end

        case (Control)
            0: led <= SW[0];
            1: led <= SW[1];
            2: led <= SW[2];
            3: led <= SW[3];
            4: led <= SW[4];
            5: led <= SW[5];
            6: led <= SW[6];
            7: led <= SW[7];
            default: led <= led;
        endcase
    end
end

endmodule
```

---

## 二、时序分段表

| 时段 | cnt 范围 | Control | 持续周期 | 时间 | led |
|------|---------|---------|---------|------|-----|
| 1 | 0 ~ 6,249,999 | 0 | 6,250,000 | 0.125s | SW[0] |
| 2 | 6,250,000 ~ 12,499,999 | 1 | 6,250,000 | 0.125s | SW[1] |
| 3 | 12,500,000 ~ 18,749,999 | 2 | 6,250,000 | 0.125s | SW[2] |
| 4 | 18,750,000 ~ 24,999,999 | 3 | 6,250,000 | 0.125s | SW[3] |
| 5 | 25,000,000 ~ 31,249,999 | 4 | 6,250,000 | 0.125s | SW[4] |
| 6 | 31,250,000 ~ 37,499,999 | 5 | 6,250,000 | 0.125s | SW[5] |
| 7 | 37,500,000 ~ 43,749,999 | 6 | 6,250,000 | 0.125s | SW[6] |
| 8 | 43,750,000 ~ 49,999,999 | 7 | 6,250,000 | 0.125s | SW[7] |
| 重置 | 50,000,000 → 0 | 0 | (1 cycle) | — | — |

---

## 三、关键机制：cnt / T 除法分段

```verilog
Control <= cnt / T;
```

- `T = 6_250_000`，`cnt` 从 0 计数到 49,999,999
- `cnt / T` 整数除法结果：0, 0, ..., 0（T 次）→ 1, 1, ..., 1（T 次）→ ... → 7
- 即 **每 T 个周期 Control 自动 +1**，天然分段

### 与 if-else if 方案的对比

| 方案 | 优点 | 缺点 |
|------|------|------|
| `cnt / T` 除法 | 段数变化只需改 T，代码简短 | 除法器资源消耗大 |
| if-else if 链 | 零除法，资源省 | 段数多时代码冗长 |

> 本工程用除法方案，适合段数固定、资源不紧张的场合。

---

## 四、错误指出

### ① 拼写错误：Countrol → Control

```verilog
// ❌ 原代码
if (cnt == 50*meta) begin
    cnt <= 0;
    Countrol <= 0;      // 未声明，综合报错
end

// ✓ 修正
if (cnt == 50_000_000 - 1) begin
    cnt <= 0;
    Control <= 0;
end
```

后果：`Countrol` 是未声明标识符，综合报 `module not found` 类错误。

---

### ② 时间常数错误：T 算错

```verilog
// ❌ 原代码：T = 125 * 1000 = 125,000
// 125,000 / 50MHz = 2.5ms，远小于 0.125s
parameter kilo = 1000;
parameter T = 125 * kilo;

// ✓ 修正：直接写 6,250,000
parameter T = 6_250_000;  // 0.125s × 50MHz
```

50MHz 下 0.125s 的计算：
```
0.125 × 50,000,000 = 6,250,000 cycles
```

---

### ③ 复位位置：复位应在 always 块顶部

```verilog
// ❌ 原代码：复位在中间，容易被忽略
cnt <= cnt + 1;
Control <= cnt / T;
// ... 一段逻辑 ...
if (rst_n == 0) begin ... end

// ✓ 修正：复位放最前面
if (rst_n == 0) begin
    // 复位所有
end else begin
    // 正常逻辑
end
```

虽然非阻塞赋值 `<=` 特性使"后赋值覆盖前赋值"生效，但复位放底部**可读性差**，容易误以为复位没处理。

---

### ④ 1s 周期边界：多了一个时钟

```verilog
// ❌ 原代码
if (cnt == 50*meta) begin  // cnt == 50,000,000 时触发
    cnt <= 0;
end
cnt <= cnt + 1;            // 每个周期都加
```

当 `cnt == 50,000,000` 时，同时执行 `cnt <= 0` 和 `cnt <= cnt + 1`，后者覆盖前者 → `cnt = 1`。有效周期 = **50,000,001 cycles**，多了 20ns。

```verilog
// ✓ 修正
if (cnt == 50_000_000 - 1) begin
    cnt <= 0;
end else begin
    cnt <= cnt + 1;
end
```

> 用 `else` 保证互斥，避免覆盖问题。

---

## 五、引脚分配（ACX720）

| 端口 | FPGA 引脚 | 说明 |
|------|-----------|------|
| clk | Y18 | 50MHz 系统时钟 |
| rst_n | F15 | 按键 S0，低电平复位 |
| SW[0] | G22 | 拨码开关 0 |
| SW[1] | D22 | 拨码开关 1 |
| SW[2] | E22 | 拨码开关 2 |
| SW[3] | G21 | 拨码开关 3 |
| SW[4] | E21 | 拨码开关 4 |
| SW[5] | D21 | 拨码开关 5 |
| SW[6] | C22 | 拨码开关 6 |
| SW[7] | B22 | 拨码开关 7 |
| led | M22 | LED0，高电平点亮 |

---

## 六、调试记录

| 步骤 | 操作 | 现象 |
|------|------|------|
| 1 | 编译烧录 | 0 errors, Bitstream OK |
| 2 | 拨 SW0 到 ON | 每 1s 第 1 段 LED 亮 |
| 3 | 拨 SW0/SW1 到 ON | 第 1、2 段 LED 亮 |
| 4 | 全拨 OFF | LED 全灭 |
| 5 | 全拨 ON | LED 全程亮 |
