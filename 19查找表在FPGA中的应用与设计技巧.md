# 查找表在 FPGA 中的应用与设计技巧

## 什么是查找表

查找表（LUT, Look-Up Table）在 FPGA 语境中有**两重含义**：

### 含义一：FPGA 硬件基本单元

FPGA 芯片内部最底层的逻辑单元就是 **LUT**（通常是 6 输入 LUT），它本质上是一个 **64×1 的 SRAM**：

```
输入 A,B,C,D,E,F (6 bit) → 寻址 64 个存储单元 → 输出 1 bit
```

任何组合逻辑（AND、OR、MUX、加法器等）最终都会被 Vivado 综合成 LUT+FF。

### 含义二：Verilog 编码手法（本文重点）

用 `case`、`assign` 或 ROM 实现的**数据映射表**。综合工具会自动将其映射到硬件 LUT 或 Block RAM 中。

---

## 段码查找表（最经典实例）

动态数码管扫描中的段码译码是查找表最典型的应用：

```verilog
// 输入: 4 bit 十六进制数
// 输出: 7 bit 段码 (共阳极, 0=亮)
always @(*) begin
    case (hex)
        4'h0: seg = 7'b1000000;  // 0
        4'h1: seg = 7'b1111001;  // 1
        4'h2: seg = 7'b0100100;  // 2
        4'h3: seg = 7'b0110000;  // 3
        4'h4: seg = 7'b0011001;  // 4
        4'h5: seg = 7'b0010010;  // 5
        4'h6: seg = 7'b0000010;  // 6
        4'h7: seg = 7'b1111000;  // 7
        4'h8: seg = 7'b0000000;  // 8
        4'h9: seg = 7'b0010000;  // 9
        4'ha: seg = 7'b0001000;  // A
        4'hb: seg = 7'b0000011;  // b
        4'hc: seg = 7'b1000110;  // C
        4'hd: seg = 7'b0100001;  // d
        4'he: seg = 7'b0000110;  // E
        4'hf: seg = 7'b0001110;  // F
        default: seg = 7'b1000000;
    endcase
end
```

### 段码推导方法

7 段数码管布局：
```
  aaaa
 f    b
 f    b
  gggg
 e    c
 e    c
  dddd  dp
```

**段码 = {g,f,e,d,c,b,a}** (bit6~bit0)

以显示 `0` 为例：a,b,c,d,e,f 亮 → `{1,0,0,0,0,0,0}` = `7'b1000000`

技巧：在纸上画个 7 段图，要亮的段写 0（共阳极），不亮的写 1，反顺序写出即得段码。

### 共阳极 / 共阴极转换

```verilog
// 共阳极段码 → 共阴极段码 = 按位取反
seg_cathode = ~seg_anode;
// 例如: 0 的共阳极 1000000 → 共阴极 0111111
```

---

## 查找表的 4 种 Verilog 实现方式

### 方式 1：case 语句（最直观）

适合 16 个以内的条目，综合为 LUT + MUX：

```verilog
always @(*) begin
    case (addr)
        4'd0: out = 10;
        4'd1: out = 23;
        4'd2: out = 45;
        // ...
        default: out = 0;
    endcase
end
```

### 方式 2：数组 / 连续赋值（适合密集表）

适合 256 个以上条目，工具会自动推断为 Block RAM：

```verilog
// 8 位输入 → 8 位输出, 256 条目
reg [7:0] lut [0:255];

initial begin
    lut[0]   = 8'h3B;
    lut[1]   = 8'h0C;
    // ... 加载 256 个值
end

always @(posedge clk)
    dout <= lut[addr];
```

### 方式 3：组合逻辑公式（适合有规律的映射）

某些映射可以推导出公式，省 LUT：

```verilog
// 0→0, 1→1, 2→3, 3→7, 4→15 (2^n - 1)
// 用公式代替查表:
assign out = (1 << in) - 1;
```

### 方式 4：BRAM 初始化（大量数据）

通过 `.mem` / `.coe` 文件加载 ROM：

```verilog
// 正弦波表 1024 点
(* ramstyle = "block" *) reg [7:0] rom [0:1023];

initial begin
    $readmemh("sine_wave.mem", rom);
end
```

---

## 应用场景

| 场景 | 输入 | 输出 | 说明 |
|------|------|------|------|
| 段码译码 | 4 bit hex | 7/8 bit 段码 | 本节实例 |
| DDS 波形合成 | 相位累加器 | 正弦波幅值 | 1024×8 bit ROM |
| 二进制转 BCD | 8 bit 二进制 | 12 bit BCD | 双字节查找 |
| CRC 校验 | 8 bit 数据 | 8 bit CRC | 256 条目表 |
| 颜色映射 | RGB888 | RGB565 | 视频处理 |
| 状态机输出 | 当前状态 | 控制信号 | 用表代替 if-else |
| 字符点阵 | ASCII | 8×16 像素 | VGA/HDMI 字符显示 |

---

## LUT vs BRAM 的选择

| 条件 | 用 LUT | 用 BRAM |
|------|--------|---------|
| 条目数 < 64 | ✅ 合适 | ❌ 浪费 |
| 条目数 64~256 | 看资源余量 | 看资源余量 |
| 条目数 > 256 | ❌ 费 LUT | ✅ 合适 |
| 需要同时多路读取 | ❌ | ✅ (双端口) |
| 延迟要求极低 | ✅ (组合输出) | ❌ (至少 1 clk) |

**经验法则**：256 条以下用 LUT（`case` 或 `always @*`），以上用 Block RAM（`reg [7:0] mem [0:N]`）。

---

## 调试技巧

### 1. 用 Vivado 查看 LUT 映射

综合后打开 **Schematic**，双击 LUT6 原语可以看到内部的 INIT 值（64 bit 真值表）。

### 2. 推断 BRAM 的写法

必须满足以下条件才能被工具识别为 Block RAM：

```verilog
// ✅ 正确写法 (推断 BRAM)
reg [7:0] mem [0:255];
always @(posedge clk)
    dout <= mem[addr];

// ❌ 不会被推断为 BRAM 的写法
reg [7:0] mem [0:255];
always @(*)          // 组合逻辑读取 → 分布式 LUT RAM
    dout = mem[addr];
```

### 3. 用 `$readmemh` 加载数据

```bash
# sine_wave.mem (hex 格式)
00
02
05
08
...
```

提前在 PC 上用 Python/C 生成 `.mem` 文件，仿真和综合都能使用。

```verilog
reg [7:0] wave [0:1023];
initial $readmemh("sine_wave.mem", wave);
```

---

## 性能对比

| 实现方式 | 延迟 | LUT 消耗 | BRAM 消耗 | 适合场景 |
|---------|------|---------|-----------|---------|
| case 组合 | 0 clk | 4~16 LUT | 0 | 小型译码 (16 条内) |
| case 时序 | 1 clk | 4~16 LUT+FF | 0 | 需要同步输出 |
| reg 数组同步读 | 1 clk | 0 | 1 个 | 256 条以上 |
| reg 数组组合读 | 0 clk | 256+ LUT | 0 | 不推荐，极费资源 |

## 与 74HC595 笔记的关联

- 段码查找表是 [[12数码管74HC595动态扫描显示]] 中 Seg_Control 模块的核心
- 级联 HC595 的 16 bit 数据拼装 `{1'b0, seg, sel}` 也是查表的输出拼接
- 也可用查找表代替移位寄存器实现串并转换（用计数器查表输出每位）

## 参考

- Vivado 综合手册 UG901: 关于 RAM/ROM 推断
- 小梅哥 ch12 例程：段码查找表 + HC595 驱动
- [[13HC595详解_串并转换与Verilog实现]]
