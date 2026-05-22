# 计数器控制 LED 多阶段时序

## 概述

用一个计数器 + `if-else if` 级联实现 LED 的多阶段时序控制。核心思想：**计数器是尺子，在不同刻度做不同事情**。

### 需求

- LED 亮 0.1s → 灭 0.1s → 亮 0.4s → 灭 0.4s → 循环
- 50MHz 时钟，总周期 1s = 50,000,000 个时钟

---

## 一、parameter 常数技巧

```verilog
parameter kilo = 1000;
parameter mega = 1000 * kilo;

parameter on_time = 5 * mega;         // 0.1s
parameter off_time = 5 * mega;        // 0.1s
parameter on_time_long = 20 * mega;   // 0.4s
parameter off_time_long = 20 * mega;  // 0.4s
```

### 好处

| 优点 | 说明 |
|------|------|
| 代码清晰 | `on_time` 比 `5_000_000` 更直观 |
| 方便修改 | 改一处，全局生效 |
| 仿真加速 | 改小常数即可快速验证 |
| 零资源消耗 | 编译时被替换成具体数字，不占 FPGA 资源 |

### 技巧：取公因数

```
0.1s = 5_000_000 = 5 * mega
0.4s = 20_000_000 = 20 * mega
                    ↑
              公因数是 mega
```

用 `mega` 做基准单位，所有时间都是它的整数倍，代码更整洁。

---

## 二、完整代码

```verilog
module Top(
    input clk,
    input rest_n,
    output reg led
);

parameter kilo = 1000;
parameter mega = 1000 * kilo;

integer count;

parameter on_time = 5 * mega;
parameter off_time = 5 * mega;
parameter on_time_long = 20 * mega;
parameter off_time_long = 20 * mega;

always @(posedge clk or negedge rest_n) begin
    if (rest_n == 0) begin
        count <= 0;
        led <= 0;
    end

    if (count < on_time) begin
        led <= 1;
    end else if (count < on_time + off_time) begin
        led <= 0;
    end else if (count < on_time + off_time + on_time_long) begin
        led <= 1;
    end else begin
        led <= 0;
    end

    if (count == 50_000_000) begin
        count <= 0;
    end
    count <= count + 1;
end

endmodule
```

---

## 三、计数器分段表

| 阶段 | Count 范围 | 持续周期 | 时间 | led |
|------|-----------|---------|------|-----|
| 亮(短) | 0 ~ 4,999,999 | 5,000,000 | 0.1s | 1 |
| 灭(短) | 5,000,000 ~ 9,999,999 | 5,000,000 | 0.1s | 0 |
| 亮(长) | 10,000,000 ~ 29,999,999 | 20,000,000 | 0.4s | 1 |
| 灭(长) | 30,000,000 ~ 49,999,999 | 20,000,000 | 0.4s | 0 |
| 重置 | 50,000,000 → 0 | 1 | - | - |

### 时序图

```
count:  0 ──→ 5M ──→ 10M ──→ 30M ──→ 50M → 0
led:    1     0      1       0       1 (循环)
        ├0.1s─┤0.1s─┤── 0.4s ──┤── 0.4s ──┤
```

---

## 四、if-else if 链的工作原理

```verilog
if (count < on_time)                    // 0 ~ 5M-1
    led <= 1;
else if (count < on_time + off_time)    // 5M ~ 10M-1
    led <= 0;
else if (count < on_time + off_time + on_time_long)  // 10M ~ 30M-1
    led <= 1;
else                                    // 30M ~ 49M
    led <= 0;
```

### 关键理解

- **互斥**：`if-else if` 保证同一时刻只有一个分支生效
- **顺序优先**：从上到下匹配，第一个满足的执行，后面的跳过
- **条件是累加的**：`on_time + off_time` 表示前两段的总长度

---

## 五、count 重置逻辑

```verilog
if (count == 50_000_000) begin
    count <= 0;
end
count <= count + 1;
```

### 执行顺序

```
时钟沿到来：
  1. 检查 count == 50_000_000？→ 是则 count <= 0
  2. count <= count + 1  ← 始终执行
```

> **注意**：非阻塞赋值 `<=` 在时钟沿结束后才更新。所以当 `count == 50_000_000` 时，先执行 `count <= 0`，再执行 `count <= count + 1`，最终 `count = 1`。
>
> 下一个时钟沿 `count = 1`，从阶段1重新开始。**LED 会多亮一个周期**。

### 改进写法

```verilog
// 方式1: 用 else 避免冲突
if (count == 50_000_000)
    count <= 0;
else
    count <= count + 1;

// 方式2: 用 if-else if 链统一管理
else if (count < 50_000_000)
    count <= count + 1;
else
    count <= 0;
```

---

## 六、常见问题

### 问题 1：integer vs reg

```verilog
integer count;       // 32位有符号，适合仿真和简单计数
reg [25:0] count;    // 26位无符号，更精确控制位宽
```

| 类型 | 位宽 | 符号 | 适用场景 |
|------|------|------|---------|
| `integer` | 32 | 有符号 | 仿真、简单工程 |
| `reg [N:0]` | N+1 | 无符号 | 精确控制资源 |

### 问题 2：count 没有单独加 1

```verilog
// ❌ 错误：count 永远不增加
if (count < on_time) begin
    led <= 1;
    count <= count + 1;  // 只在阶段1加
end

// ✓ 正确：count 在 always 块末尾统一加
if (count < on_time)
    led <= 1;
// ...
count <= count + 1;  // 每拍都加
```

### 问题 3：复位后 led 不亮

```verilog
// ❌ 复位值和第一阶段都是 0
if (!rst_n) led <= 0;
// 第一阶段: led <= 0;

// ✓ 第一阶段应该是 1
if (count < on_time) led <= 1;
```

---

## 七、引脚分配（ACX720）

| 端口 | FPGA 引脚 | 说明 |
|------|-----------|------|
| clk | Y18 | 50MHz 时钟 |
| rest_n | F15 | 按键 S0，按下复位 |
| led | M22 | LED0，高电平点亮 |

---

## 八、扩展思路

### 多 LED 流水灯

```verilog
if (count < T)       led <= 4'b0001;
else if (count < 2*T) led <= 4'b0010;
else if (count < 3*T) led <= 4'b0100;
else if (count < 4*T) led <= 4'b1000;
else                  count <= 0;
```

### 呼吸灯（PWM）

```verilog
// 用另一个快速计数器做 PWM
// 慢计数器控制占空比从 0% 渐变到 100%
if (pwm_cnt < duty)
    led <= 1;
else
    led <= 0;
```

---

## 记忆口诀

> **parameter 是刻度尺，if-else if 是标尺**
> - 常数相乘零消耗，编译替换是魔法
> - count 统一加一拍，重置记得用 else
> - 取个公因数更清晰，改仿真时缩比例
