# 状态机控制动态 / 静态 LED 切换

## 概述

类似任务 3（拨码开关控制 LED 多段时序），但增加了状态切换：**2s 动态变化 → 1s 静态保持 → 重复**。

### 需求
- **动态 2s**：LED 按拨码开关 SW[7:0] 每 0.125s 切换一段，轮流显示
- **静态 1s**：LED **保持**动态结束时最后一刻的亮/灭状态，不变化
- 循环：动态 2s → 静态 1s → 动态 2s → ...

---

## 一、思路方法

### 核心问题
任务 3 只有一个 1s 循环，control 每 0.125s +1，**没有"停"的概念**。现在需要在 2s 后"冻结"LED 1s，然后再动。

### 解决方案：状态机 + seg_cnt
```
                     ┌─────────────┐
        ┌───────────│  DYNAMIC    │◄──────────┐
        │           │  (2s 动态)   │           │
        │           └──────┬──────┘           │
        │              seg_cnt==1             │
        │              (2s 到)                │
        │                  ▼                  │
        │           ┌─────────────┐           │
        └───────────│   STATIC    │───────────┘
                    │  (1s 静态)   │
                    └─────────────┘
```

### 关键设计
1. **1 个计数器 counter**：从 0 → 50M-1，正好 **1s**
2. **seg_cnt** 记录当前状态持续了多少个 1s 段
   - DYNAMIC 下 seg_cnt 从 0 → 1（2s 到），切 STATIC
   - STATIC 下 seg_cnt 归零，1s 后切回 DYNAMIC
3. **Control = counter / T**：每 T=6,250,000 周期 +1，即每 0.125s 切一段

---

## 二、代码

```verilog
module Top(
    input clk,
    input rst_n,
    input [7:0] SW,
    output reg led
);

parameter DYNAMIC = 1, STATIC = 2;
parameter T = 6_250_000;         // 0.125s

reg [1:0] state;
reg [2:0] Control;
reg [31:0] counter;
reg [1:0] seg_cnt;              // 记录当前state持续了几个1s

always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        counter <= 0;
        Control <= 0;
        state <= DYNAMIC;
        seg_cnt <= 0;
        led <= 0;
    end else begin
        counter <= counter + 1;
        Control <= counter / T;

        // 每 1s (50M cycles) 检查状态切换
        if (counter == 50_000_000 - 1) begin
            counter <= 0;
            if (state == DYNAMIC) begin
                if (seg_cnt == 1) begin        // 第2次 → 切静态
                    state <= STATIC;
                    seg_cnt <= 0;
                end else begin
                    seg_cnt <= seg_cnt + 1;    // 第1次 → 继续动态
                end
            end else begin                      // STATIC
                state <= DYNAMIC;
                seg_cnt <= 0;
            end
        end

        // DYNAMIC: LED 随 Control 切换
        if (state == DYNAMIC) begin
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
        // STATIC: led 不赋值 → 保持
    end
end

endmodule
```

---

## 三、时序表

| 时间段 | 状态 | counter | Control | led |
|--------|------|---------|---------|-----|
| 0s ~ 0.125s | DYNAMIC | 0 ~ 6.25M-1 | 0 | SW[0] |
| 0.125s ~ 0.25s | DYNAMIC | 6.25M ~ 12.5M-1 | 1 | SW[1] |
| ... | DYNAMIC | ... | ... | ... |
| 0.875s ~ 1s | DYNAMIC | 43.75M ~ 50M-1 | 7 | SW[7] |
| 1s ~ 2s | DYNAMIC (重来) | 0 ~ 50M-1 | 0→7 | SW[0]→SW[7] |
| ↓ seg_cnt==1 | ✅ 切 STATIC ||||
| 2s ~ 3s | STATIC | 0 ~ 50M-1 | 0→7 | **保持**不变 |
| ↓ | ✅ 切回 DYNAMIC ||||
| 3s ~ 5s | DYNAMIC | 0 ~ 50M-1 ×2 | 0→7 | SW[0]→SW[7] |

---

## 四、seg_cnt 技巧

```verilog
reg [1:0] seg_cnt;  // 只需2位 (0~2)
```

seg_cnt 是**计数器溢出次数计数器**，用来实现"持续 N 个 1s"：

| 场景 | 实现方式 | seg_cnt 取值 |
|------|---------|-------------|
| 持续 1s | state 每次 rollover 切换 | 不需要 seg_cnt |
| 持续 **2s** | seg_cnt 从 0 数到 1 再切 | 0→1→切 |
| 持续 **3s** | seg_cnt 从 0 数到 2 再切 | 0→1→2→切 |
| 持续 **Ns** | 条件改为 `seg_cnt == N-2` | 0→...→N-2→切 |

> **优点**：不浪费额外计数器，只加一个小寄存器
> **局限**：seg_cnt 只在 counter rollover 时变化，粒度是 1s

---

## 五、非阻塞赋值的执行顺序

```verilog
always @(posedge clk) begin
    counter <= counter + 1;       // A
    Control <= counter / T;       // B
    if (counter == 50M-1) begin
        counter <= 0;             // C (覆盖 A)
        state <= STATIC;          // D
    end
end
```

### 执行顺序（非阻塞赋值特性）

```
时钟沿到来:
  1. 计算所有 RHS (右边表达式): counter+1, counter/T, counter==50M-1 ...
     → 都用旧的 counter 值
  2. 同时赋值 LHS (左边):
     counter = 结果A  → 被 C 覆盖 → counter = 0
     Control = 旧counter / T
     state   = STATIC
```

### 关键理解

| 现象 | 原因 |
|------|------|
| `counter <= counter+1` 和 `counter <= 0` 同时写 | **最后一个赋值赢**，counter=0 |
| `Control` 在重置后用旧值 | 因为 RHS 用的是旧 counter |
| `if (counter == 50M-1)` 用旧值判断 | `==` 是立即求值的，不是非阻塞 |

---

## 六、状态机复位技巧

```verilog
// ✅ 推荐：复位在外，逻辑在 else 内
if (!rst_n) begin
    // 所有寄存器复位
end else begin
    // 所有正常逻辑
end

// ❌ 避免：复位混在逻辑中间
counter <= counter + 1;
// ... 各种逻辑 ...
if (!rst_n) begin    // 复位在这里可能被覆盖
    ...
end
```

> **原则**：复位要在 always 块**最前面**，使代码可读性最好、综合结果最可靠。

---

## 七、seg_cnt 状态迁移表

| 当前 state | seg_cnt | counter=50M-1 事件 | 下一 state |
|-----------|---------|-------------------|-----------|
| DYNAMIC | 0 | seg_cnt → 1 | DYNAMIC (第 1s 结束) |
| DYNAMIC | 1 | seg_cnt → 0, state→STATIC | STATIC (第 2s 结束) |
| STATIC | 0 | seg_cnt → 0, state→DYNAMIC | DYNAMIC (静态结束) |

---

## 八、引脚分配（ACX720）

同任务 3：

| 端口 | FPGA 引脚 |
|------|-----------|
| clk | Y18 |
| rst_n | F15 |
| SW[7:0] | G22/D22/E22/G21/E21/D21/C22/B22 |
| led | M22 |

---

## 记忆口诀

> **状态机分动和静，seg_cnt 数秒停**
> **counter 到顶滚一圈，状态切换看 seg**
> **动态 case 切八个，静态 led 不赋值**
