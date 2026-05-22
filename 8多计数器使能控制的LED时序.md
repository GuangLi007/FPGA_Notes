# 多计数器使能控制的 LED 时序（另一种思路）

## 概述

任务 4（2s 动态 + 1s 静态）的**另一种实现方式**：用多个计数器 + 使能信号替代单计数器 + 除法 `cnt/T`。

### 对比两种思路

| 方案 | 核心机制 | 优势 | 劣势 |
|------|---------|------|------|
| **方案 A**（任务4） | 1 个 counter + `cnt / T` 除法 + seg_cnt | 代码简洁，变量少 | 除法器占资源 |
| **方案 B**（本文） | 多 counter + 使能信号级联 | 零除法，资源省 | 代码长，变量多 |

---

## 一、思路方法

### 核心问题：如何把时间拆成"段"？

方案 A：counter 一直数，`cnt / T` 把连续计数分成段。
方案 B：**counter 只数到 T 就停**，然后产生使能脉冲，启动下一个 counter。

### 分层计时思想

```
clk (50MHz)  ──→  cnt_125ms  ──使能──→  cnt_1s  ──使能──→  seg_cnt
   0.125s timer        ↑        1s timer       ↑       状态计数器
                  en_125ms               en_1s
```

- **cnt_125ms**：计数 0→6,250,000-1，每满一次发出 `en_125ms` 脉冲
- `en_125ms` **使能** Control +1（代替 `cnt/T`）
- **cnt_1s**：对 `en_125ms` 计数，每 8 次（8×0.125s=1s）发出 `en_1s` 脉冲
- `en_1s` **使能** seg_cnt 加 1 和状态切换
- **seg_cnt**：记录当前状态持续的秒数

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

// ====== 计时器参数 ======
parameter T_125MS = 6_250_000;   // 0.125s 计数值
parameter T_1S    = 8;           // 8 × 0.125s = 1s

// ====== 寄存器声明 ======
reg [22:0] cnt_125ms;           // 0.125s 计数器
reg        en_125ms;            // 0.125s 到，单周期脉冲
reg        en_125ms_dly;        // 延迟一拍做边沿检测

reg [2:0]  cnt_1s;              // 1s 计数器（对 en_125ms 计数）
reg        en_1s;               // 1s 到，单周期脉冲

reg [1:0]  seg_cnt;             // 状态段计数（0→1 或 0→?）
reg [2:0]  Control;             // 0.125s 段索引
reg [1:0]  state;               // DYNAMIC / STATIC

// ====== cnt_125ms: 0.125s 定时器 ======
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        cnt_125ms <= 0;
        en_125ms  <= 0;
    end else if (cnt_125ms == T_125MS - 1) begin
        cnt_125ms <= 0;
        en_125ms  <= 1;
    end else begin
        cnt_125ms <= cnt_125ms + 1;
        en_125ms  <= 0;
    end
end

// ====== en_125ms 边沿延迟（用于后续取上升沿） ======
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        en_125ms_dly <= 0;
    end else begin
        en_125ms_dly <= en_125ms;
    end
end

// ====== Control: 每 0.125s +1，由 en_125ms 脉冲使能 ======
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        Control <= 0;
    end else if (en_125ms && !en_125ms_dly) begin  // en_125ms 上升沿
        if (Control == 7)
            Control <= 0;
        else
            Control <= Control + 1;
    end
end

// ====== cnt_1s: 对 en_125ms 计数，每 8 次发 en_1s ======
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        cnt_1s <= 0;
        en_1s  <= 0;
    end else if (en_125ms && !en_125ms_dly) begin
        if (cnt_1s == T_1S - 1) begin
            cnt_1s <= 0;
            en_1s  <= 1;
        end else begin
            cnt_1s <= cnt_1s + 1;
            en_1s  <= 0;
        end
    end else begin
        en_1s  <= 0;
    end
end

// ====== 状态机 + seg_cnt: 由 en_1s 使能 ======
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        state   <= DYNAMIC;
        seg_cnt <= 0;
        led     <= 0;
    end else if (en_1s) begin
        if (state == DYNAMIC) begin
            if (seg_cnt == 1) begin          // 2s 到 → 切静态
                state   <= STATIC;
                seg_cnt <= 0;
            end else begin
                seg_cnt <= seg_cnt + 1;      // 第 1s 结束
            end
        end else begin                        // STATIC → 切动态
            state   <= DYNAMIC;
            seg_cnt <= 0;
        end
    end
end

// ====== LED 输出: DYNAMIC 时跟随 Control ======
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        led <= 0;
    end else if (state == DYNAMIC) begin
        case (Control)
            0: led <= SW[0];
            1: led <= SW[1];
            2: led <= SW[2];
            3: led <= SW[3];
            4: led <= SW[4];
            5: led <= SW[5];
            6: led <= SW[6];
            7: led <= SW[7];
        endcase
    end
    // STATIC: led 保持
end

endmodule
```

---

## 三、使能链工作流程

```
时钟:          ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
cnt_125ms:  0 → 1 → 2 → ... → T-1 → 0 → 1 → ...
en_125ms:   0   0   0         1     0   0
                                 │
                                 ▼
Control:           ... → n → n+1 → ...  (en_125ms 使能)
cnt_1s:                 0 → 1 → ... → 7 → 0
en_1s:                                   1
                                          │
                                          ▼
seg_cnt:                              0 → 1 (或状态切换)
```

---

## 四、使能信号的作用

### en_125ms：定时使能
```
cnt_125ms == T_125MS - 1 时拉高一个周期
→ 使能 Control 加 1
→ 使能 cnt_1s 加 1
```

```verilog
// 使能信号的标准写法
always @(posedge clk) begin
    if (rst_n) begin
        cnt <= 0;
        en  <= 0;
    end else begin
        if (cnt == T - 1) begin
            cnt <= 0;
            en  <= 1;        // 使能脉冲
        end else begin
            cnt <= cnt + 1;
            en  <= 0;        // 默认拉低
        end
    end
end
```

### 使能 vs 时钟分频

| 方式 | 做法 | 问题 |
|------|------|------|
| ❌ 时钟分频 | 用 counter 产生慢时钟 | 产生时钟毛刺，时序不推荐 |
| ✅ 使能脉冲 | 生成单周期脉冲信号 | 所有逻辑仍用原时钟，时序干净 |

### 技巧：用 `en && !en_dly` 取上升沿

```verilog
always @(posedge clk) en_dly <= en;
// en && !en_dly 等价于 en 的上升沿
```

> 相比 `en` 本身可能会被多个模块同时使用，**取上升沿**确保 Control 和 cnt_1s 不会因 `en` 宽度变化而重复触发。

---

## 五、两种方案完整对比

| 对比维度 | 方案A (cnt / T) | 方案B (多计数器使能) |
|---------|----------------|-------------------|
| 计数器数量 | 1 个 | 3 个 (cnt_125ms, cnt_1s, seg_cnt) |
| 除法器 | 使用 `cnt / T` | 无除法，纯比较 |
| LUT 资源 | 较多（除法器） | 较少（比较器） |
| 代码行数 | ~50 行 | ~120 行 |
| 可读性 | 简洁直观 | 模块化，但变量多 |
| 可扩展性 | 改 T 即可调时间 | 分层清晰，每层可独立改 |
| 时序分析 | 除法器路径长 | 纯比较，路径短 |
| 适用场景 | 小工程、学习验证 | 资源敏感、需要精确分层 |

### 选择建议

```
简单工程、学习阶段  →  方案 A (cnt / T)
复杂工程、资源受限  →  方案 B (使能链)
需要独立调整各段    →  方案 B
```

---

## 六、使能链模式的应用扩展

使能链（enable chain）是一种通用设计模式，可以推广到任意分层时序：

```
clk → timer_us → timer_ms → timer_s → timer_min
      en_us      en_ms      en_s      en_min
```

每一层的使能脉冲驱动下一层计数，形成一个**定时器金字塔**：

```verilog
// 秒 → 分钟 的使能链示例
always @(posedge clk) begin
    if (cnt_s == 50_000_000 - 1) begin
        cnt_s <= 0;
        en_s  <= 1;       // 每秒一个脉冲
    end
end

// en_s 使能分钟计数器
always @(posedge clk) begin
    if (en_s) begin
        if (cnt_min == 59)
            cnt_min <= 0;
        else
            cnt_min <= cnt_min + 1;
    end
end
```

### 优点
- **每层独立**：改秒层不影响分层
- **时序干净**：全部用原时钟
- **资源可控**：每层计数器位宽小

---

## 七、常见错误

### 错误 1：使能信号没拉低
```verilog
// ❌ en_125ms 只升不降
if (cnt == T-1) en_125ms <= 1;

// ✓ 要显式拉低
if (cnt == T-1) begin
    cnt <= 0; en_125ms <= 1;
end else begin
    cnt <= cnt + 1; en_125ms <= 0;
end
```

### 错误 2：使能作为时钟用
```verilog
// ❌ 禁止：用使能信号做时钟
always @(posedge en_125ms)  // 会产生毛刺时钟

// ✓ 正确：使能作为条件
always @(posedge clk)
    if (en_125ms)
        Control <= Control + 1;
```

### 错误 3：使能边沿没处理好
```verilog
// ❌ en 可能有多个周期宽
if (en) Control <= Control + 1;  // 可能被加多次

// ✓ 取上升沿
if (en && !en_dly) Control <= Control + 1;
```

---

## 八、记忆口诀

> **使能链，分层传，每层自带上限值**
> **一个脉冲往下递，不改时钟也干净**
> **cnt/T 简单写，使能链资源省**
