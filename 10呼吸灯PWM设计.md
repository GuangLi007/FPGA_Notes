# 呼吸灯 PWM 设计

## 概述

呼吸灯：LED 亮度由暗→亮→暗循环变化，像呼吸一样。

### 原理：人眼视觉暂留 + PWM

- **PWM**（脉冲宽度调制）：快速亮灭，人眼感觉到的亮度 = 亮的时间比例
- **占空比** = 亮的时间 / 总周期时间
- 占空比 0% → LED 全灭，100% → 全亮，50% → 半亮

```
PWM 波形:
100% ┌──────┐     ┌──┐          ┌──────┐
 75% │      │     │  │     ↑    │      │
 50% │      │  ┌──┘  │  duty    │      │
 25% │      └──┘     │     ↓    │      │
  0% └───────────────┘     └────┘      └──
     占空比小←──── 时间 ────→占空比大
```

### 整体架构

```
                    ┌─────────────────────────┐
clk ──→ 1ms定时器 ──→ en_1ms ──→ duty 增减器 ──→ duty × 50 ──┐
        cnt_1ms               (0→999→0)                    │
                                                            ▼
clk ──→ PWM计数器 ──→ pwm_cnt ──→ pwm_cnt < 阈值? ──→ LED
        cnt_pwm
```

三层分工：
1. **cnt_1ms**：产生 1ms 基准时间脉冲 `en_1ms`
2. **duty**：每 1ms 变化一次，决定当前占空比（0~999）
3. **pwm_cnt**：在 1ms 内快速计数，与 `duty×50` 比较产生 PWM

---

## 一、第一层：1ms 定时器（时间基准）

```verilog
parameter T_1MS = 50_000;       // 50MHz ÷ 50,000 = 1ms

reg [15:0] cnt_1ms;
wire       en_1ms;

always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        cnt_1ms <= 0;
    else if (cnt_1ms == T_1MS - 1)
        cnt_1ms <= 0;
    else
        cnt_1ms <= cnt_1ms + 1;
end

assign en_1ms = (cnt_1ms == T_1MS - 1);
```

### 关键思想：使能脉冲

`en_1ms` 是一个**单周期脉冲**（每 1ms 拉高一个时钟周期）。

- 每次 `cnt_1ms` 数到 49,999，`en_1ms = 1`
- 下一周期 `cnt_1ms` 归零，`en_1ms` 自动变回 0

> 使能脉冲替代了任务 3 中的 `cnt / T` 除法，是更高效的定时方式。

---

## 二、第二层：duty 增减器（亮度变化）

```verilog
reg [9:0] duty;    // 0~999, 占空比 * 1000
reg       up;      // 方向: 1=增亮, 0=减暗

always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        duty <= 0;
        up   <= 1;
    end else if (en_1ms) begin   // 每 1ms 变化一次
        if (up) begin
            if (duty == T_1S - 1) begin  // 到顶 → 转向
                duty <= duty - 1;
                up   <= 0;
            end else begin
                duty <= duty + 1;
            end
        end else begin
            if (duty == 0) begin         // 到底 → 转向
                duty <= duty + 1;
                up   <= 1;
            end else begin
                duty <= duty - 1;
            end
        end
    end
end
```

### 增亮/减暗的三角波

```
duty: 0 → 1 → 2 → ... → 999 → 998 → ... → 1 → 0 → 1 → ...
       ↑                                    ↑
    增亮 (up=1)                        减暗 (up=0)
       ├────────── 1s ──────────┤├────────── 1s ──────────┤
       └─────────────── 完整呼吸周期 2s ──────────────────┘
```

### 关键思想：方向标志位

用 1 位寄存器 `up` 记录当前方向，取代了复杂的边界判断：

| duty | up | 下一拍 | 含义 |
|------|-----|--------|------|
| 0 | 1 | duty=1, up=1 | 最暗，开始增亮 |
| 999 | 1 | duty=998, up=0 | 最亮，转向减暗 |
| 0 | 0 | duty=1, up=1 | 最暗，转向增亮 |
| 500 | 1 | duty=501 | 增亮中 |
| 500 | 0 | duty=499 | 减暗中 |

---

## 三、第三层：PWM 比较器（亮度控制）

```verilog
reg [15:0] pwm_cnt;

always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        pwm_cnt <= 0;
    else if (pwm_cnt == T_1MS - 1)
        pwm_cnt <= 0;
    else
        pwm_cnt <= pwm_cnt + 1;
end

always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        led <= 0;
    else
        led <= (pwm_cnt < duty * 50);
end
```

### PWM 对比过程

每 1ms（50,000 个时钟周期）内，`pwm_cnt` 从 0 数到 49,999。

LED 亮的时间取决于 `duty × 50`：

| duty | 阈值 | 亮的时间 | 占空比 |
|------|------|---------|--------|
| 0 | 0 | 0 | 0%（全灭） |
| 250 | 12,500 | 12,500/50,000 | 25% |
| 500 | 25,000 | 25,000/50,000 | 50% |
| 999 | 49,950 | 49,950/50,000 | 99.9%（全亮） |

### 为什么阈值 = duty × 50？

```
PWM 周期 = 1ms = 50,000 个时钟
duty 范围 = 0~999 (1000 级)
每级对应的时钟数 = 50,000 / 1000 = 50
→ 阈值 = duty × 50
```

---

## 四、整体波形时序图

```
1ms:  |←──── 50,000 clk ────→|

en_1ms:  __┌─┐_______________┌─┐______
           │                    │
duty:  N ──┘                    └── N+1

pwm_cnt: 0 → 1 → ... → 49,999 → 0 → ...

led:  ┌──────────────┐               ┌──
      │ 亮 (pwm<阈值)  │  灭           │
      └──────────────┘               └──
      ├── duty × 50 ─┤
      ├── 50,000 ─────┤
```

---

## 五、错误指出

以下是你原代码中的错误和背后的思想误区：

### 错误 1：一个信号不能在两个 always 块赋值

```verilog
// ❌ 你的代码
always @(posedge clk or negedge rst_n) begin  // 块A
    if(!rst_n) cnt3 <= 0;
    ...
end

always @(posedge clk) begin                    // 块B
    cnt3 <= cnt2 + 1;                          // 又写 cnt3
    ...
end
```

**报错：** `ambiguous clock in event control`

**思想误区：** 以为两块各管各的，综合时会合并。**实际上每个 reg 只能在一个 always 块里被赋值。**

```verilog
// ✓ 正确：一个寄存器只在一个 always 块内赋值
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) cnt3 <= 0;
    else        cnt3 <= cnt2 + 1;
end
```

> **铁律：** 一个 reg 只能在一个 always 块中赋值，多个 always 块驱动同一 reg → 综合报错。

---

### 错误 2：非阻塞赋值后写覆盖先写

```verilog
// ❌ 你的代码
if (cnt1 == 50_000_000 - 1) begin
    cnt1 <= 0;           // ① 想归零
end
cnt1 <= cnt1 + 1;        // ② 每拍加1

// 实际：② 覆盖 ① → cnt1 永远不归零！
```

**思想误区：** 以为 `if` 条件满足时 `cnt1 <= 0` 生效，`cnt1 <= cnt1+1` 不执行。

**非阻塞赋值真相：**

```
时钟沿到来：
  1. 计算所有 RHS（右边表达式）：cnt1+1, cnt1==50M-1, ...
  2. 同时赋值所有 LHS（左边变量）
  3. 同⼀变量多次赋值 → 最后一个赢
```

所以 `cnt1 <= cnt1+1`（第②句后的值）永远覆盖 `cnt1 <= 0`。

```verilog
// ✓ 正确：用 else 保证互斥
if (cnt1 == 50_000_000 - 1)
    cnt1 <= 0;
else
    cnt1 <= cnt1 + 1;
```

---

### 错误 3：寄存器不计数而是直接赋值

```verilog
// ❌ 你的代码
cnt3 <= cnt2 + 1;       // 每拍都 = cnt2+1
```

这导致 `cnt3` 不是计数器，而是 `cnt2` 的"影子"——`cnt3` 永远等于 `cnt2+1`。

**思想误区：** 把 `<=` 当成了数学等号。实际上 `cnt3 <= cnt2 + 1` 是"每时钟周期把 cnt2+1 的值赋给 cnt3"，不是"定义 cnt3 为 cnt2+1"。

```verilog
// ✓ 正确：计数器需要累加
cnt3 <= cnt3 + 1;
```

> **口诀：** `cnt <= cnt + 1` 才是计数，`cnt <= xxx` 是赋值。

---

### 错误 4：PWM 比较条件恒成立

```verilog
// ❌ 你的代码
if (cnt3 < cnt2 * 50)  // cnt3 ≈ cnt2+1, 所以 cnt2+1 < cnt2*50
```

对于 `cnt2 ≥ 1`：`cnt2+1 < cnt2×50` → 几乎**永远为真**。

```
cnt2=0: cnt3=1,   1 < 0   → 假, LED灭 (只有第1ms)
cnt2=1: cnt3=2,   2 < 50  → 真, LED亮
cnt2=2: cnt3=3,   3 < 100 → 真, LED亮
...
```

LED 在 1ms 后一直亮，没有呼吸效果。

```verilog
// ✓ 正确：独立计数器 pwm_cnt 与 duty*50 比较
if (pwm_cnt < duty * 50)  // pwm_cnt 独立计数，与 duty 无关
```

> **思想：** PWM 比较的两个输入必须**独立变化**——一个快速（pwm_cnt），一个慢速（duty）。

---

### 错误 5：XDC 引用不存在的端口

```xdc
set_property PACKAGE_PIN G22 [get_ports {SW[0]}]  // Top 没有 SW 端口
```

Vivado 会警告但没有语法错误。模块端口定义必须和 XDC ——对应。

---

## 六、三层的设计思想

### 为什么分三层？

```
需求：LED 亮度在 1s 内从 0% 变到 100%
```

| 方式 | 做法 | 问题 |
|------|------|------|
| ❌ 一层 | 一个大计数器 + `cnt/T` | 除法器资源大 |
| ✅ 三层 | 1ms 定时器 → duty 增减 → PWM | 每层只做一件事 |

**分层的本质：把"时间"和"动作"解耦**

| 层 | 职责 | 时间粒度 | 变化速度 |
|---|------|---------|---------|
| 1ms 定时器 | 产生基准时间 | 1ms | 固定 |
| duty 增减器 | 决定当前亮度 | 1ms | 每 1ms 变化 1 级 |
| PWM 比较器 | 输出占空比 | 20μs (1/50MHz) | 每时钟周期变化 |

每一层只关心自己的事，层与层之间通过**使能信号**通信。

### 使能信号：层间桥梁

```
cnt_1ms ──→ en_1ms ──→ duty 加减
pwm_cnt ──→ pwm_cnt < duty×50  ──→ led
```

`en_1ms` 是一个**单周期脉冲**，作用类似"时钟的时钟"：

- 对 duty 层来说，`en_1ms` 就是它的"时钟"
- 但 duty 层仍用系统时钟 `clk`，只在 `en_1ms` 有效时才变化

这保证了：
- 全部逻辑同步于同一个主时钟 `clk`
- 每层有独立的"节奏"

---

## 七、parameter 计算技巧

```verilog
parameter T_1MS = 50_000;      // 50MHz ÷ 1ms
parameter T_1S  = 1000;        // 1000 × 1ms
```

| 参数 | 值 | 计算 |
|------|-----|------|
| T_1MS | 50,000 | 50,000,000 Hz × 0.001s |
| T_1S | 1,000 | 1s ÷ 1ms |

**阈值推导：**

```
阈值 = duty × (T_1MS / T_1S) = duty × 50
```

`T_1MS / T_1S = 50,000 / 1,000 = 50` — 用 **parameter 计算**代替硬编码，方便修改。

想改变呼吸速度，只需改 T_1S：

| 改 T_1S | 效果 |
|---------|------|
| 500 | 呼吸变快 1 倍（1s 周期） |
| 2000 | 呼吸变慢 1 倍（4s 周期） |

---

## 八、引脚分配（ACX720）

| 端口 | FPGA 引脚 | 说明 |
|------|-----------|------|
| clk | Y18 | 50MHz |
| rst_n | F15 | 按键 S0 |
| led | M22 | LED0 |

---

## 记忆口诀

> **定时器生脉冲，duty 增减慢慢变**
> **PWM 快比较，阈值乘出占空比**
> **三层各做一件事，使能信号桥上走**
> **一个 reg 只在单块写，后赋覆盖前赋输**
