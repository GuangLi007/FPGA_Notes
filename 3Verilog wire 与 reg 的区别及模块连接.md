---
tags:
  - FPGA
---

## 核心结论

- **wire**：物理连线，组合逻辑，不能存储值
- **reg**：寄存器，时序逻辑，可以存储值
- **模块端口连接**：子模块的 `output reg` 可以连接到顶层的 `wire`

---

## 一、wire 与 reg 的区别

| 特性 | wire | reg |
|------|------|-----|
| **本质** | 物理连线 | 寄存器（存储单元） |
| **赋值方式** | `assign` 语句 | `always` 块内 |
| **能否存储值** | ❌ 不能 | ✅ 能 |
| **综合结果** | 导线 | 触发器/锁存器 |
| **默认值** | 高阻 `z` | 未知 `x` |
| **过程块内赋值** | ❌ 不允许 | ✅ 允许 |

---

## 二、使用规则

### wire 的使用

```verilog
wire a, b;
wire [7:0] data;

// 只能用 assign 赋值
assign a = b & c;

// 不能在 always 块中赋值
always @(*) begin
    a = b;  // ❌ 错误！wire 不能在 always 中赋值
end
```

### reg 的使用

```verilog
reg c, d;
reg [7:0] cnt;

// 只能在 always 块中赋值
always @(posedge clk) begin
    c <= d;     // ✅ 正确
    cnt <= cnt + 1;
end

// 不能用 assign 赋值
assign c = d;   // ❌ 错误！reg 不能用 assign
```

---

## 三、模块端口声明规则

| 端口方向 | 模块内部使用 | 可声明类型 |
|---------|------------|-----------|
| `input` | 只能读取 | `wire`（默认） |
| `output` | 可以赋值 | `wire` 或 `reg` |
| `inout` | 双向 | `wire`（默认） |

```verilog
module example(
    input clk,           // 默认为 wire
    input [7:0] data,    // 默认为 wire
    output reg result,   // reg，可在 always 中赋值
    output trigger       // 默认为 wire
);
```

---

## 四、模块间连接规则

### 子模块的 `output reg` → 顶层的 `wire`

```verilog
// 子模块：蜂鸣器控制器
module Beep(
    input clk,
    output reg beep    // 子模块内部是 reg
);

// 顶层模块
module top(
    output beep        // 顶层端口默认是 wire
);

Beep u_beep(
    .clk(clk),
    .beep(beep)        // ✅ 合法！wire 连接 reg
);
```

### 连接兼容性表

| 子模块端口 | 顶层连接线 | 是否合法 |
|-----------|-----------|---------|
| `output reg` | `wire` | ✅ 合法 |
| `output wire` | `wire` | ✅ 合法 |
| `output reg` | `reg` | ⚠️ 不推荐 |
| `input wire` | `reg` | ✅ 合法 |
| `input wire` | `wire` | ✅ 合法 |

---

## 五、为什么可以这样连接？

**本质理解**：

- **子模块的 `output reg`**：子模块内部有一个寄存器，它的输出引脚连接到模块边界
- **顶层的 `wire`**：只是一根导线
- **连接的含义**：把这根导线的**一端**接到子模块的输出引脚上，**另一端**接到顶层端口

子模块的寄存器驱动导线，导线传递信号到顶层，**不冲突**。

```
[子模块寄存器] ---(驱 动)---> [导线 wire] ---> [顶层端口]
```

---

## 六、常见错误及避免方法

### 错误1：在 always 块中给 wire 赋值

```verilog
wire a;

always @(posedge clk) begin
    a <= b;  // ❌ 错误！
end
```

**修正**：改用 `reg`

```verilog
reg a;

always @(posedge clk) begin
    a <= b;  // ✅ 正确
end
```

### 错误2：用 assign 给 reg 赋值

```verilog
reg a;

assign a = b;  // ❌ 错误！
```

**修正**：改用 `wire`

```verilog
wire a;

assign a = b;  // ✅ 正确
```

### 错误3：多个驱动源连接到同一 wire

```verilog
wire a;

assign a = b;
assign a = c;  // ❌ 多个驱动冲突！
```

**修正**：使用三态门或合并逻辑

```verilog
wire a = b | c;  // ✅ 合并
```

---

## 七、经验总结

| 场景 | 推荐类型 | 原因 |
|------|---------|------|
| 模块间连线 | `wire` | 纯粹的信号传递 |
| 顶层输出端口 | `wire`（默认） | 由子模块驱动 |
| 计数器 | `reg` | 需要存储计数值 |
| 状态机状态 | `reg` | 需要记忆当前状态 |
| 组合逻辑输出 | `wire` | `assign` 语句 |
| 时序逻辑输出 | `reg` | `always` 块赋值 |

---

## 八、快速判断

| 情况              | 使用     |
| --------------- | ------ |
| 在 `always` 块中赋值 | `reg`  |
| 用 `assign` 语句赋值 | `wire` |
| 模块实例化连接         | `wire` |
| 存储值（计数、状态）      | `reg`  |
| 单纯传递信号          | `wire` |
```