---
tags:
  - FPGA
  - ROM
  - BRAM
  - 存储
---

## 核心结论

- **ROM** = 只读存储器，FPGA 中无专用硬件，由 BRAM 或 LUT 配置为只读模式实现
- **BROM** = 用 Block RAM 配置成的 ROM（习惯叫法，Vivado IP 中不存在此名称）
- **分布式 ROM** = 用 LUT 实现的 ROM（小容量）
- **ROM 上电数据固定，运行时不可写**，更新数据需重新综合烧录

---

## 一、ROM vs BROM vs 分布式 ROM

| 类型 | 硬件载体 | 容量范围 | 读取方式 | 适用场景 |
|------|---------|---------|---------|---------|
| **通用 ROM** | BRAM 或 LUT | 任意 | 同步/异步 | 通称 |
| **BROM (Block ROM)** | **BRAM** (RAMB18E1/RAMB36E1) | ≥4Kb | 同步 (推荐) | 大容量查表 |
| **分布式 ROM (DRM)** | **LUT** (逻辑资源) | ≤256b | 同步/异步 | 小常量表 |

### 核心区别

| 对比项 | BROM | 分布式 ROM |
|--------|------|-----------|
| 资源 | BRAM（专用块内存，不影响逻辑） | LUT+FF（消耗逻辑资源） |
| 每块容量 | 18Kb / 36Kb | ~64 bit / LUT |
| 速度 | 高（专用布线） | 中等 |
| 级联 | 可自动级联多块 BRAM | 需手动用 MUXF7/F8 拼接 |
| 延迟 | 1 clk（同步读） | 0 clk（组合读）/1 clk（同步读） |

---

## 二、Vivado 中 ROM 的 3 种实现方式

### 方式 1：HDL 数组推断（最常用）

Vivado 综合器自动识别代码模式，推断为 BRAM 或分布式 ROM：

```verilog
module rom_infer #(
    parameter DEPTH = 256,
    parameter WIDTH = 8
) (
    input  wire                       clk,
    input  wire [$clog2(DEPTH)-1:0]   addr,
    output reg  [WIDTH-1:0]           dout
);

reg [WIDTH-1:0] rom [0:DEPTH-1];

initial begin
    $readmemh("sine_data.mem", rom);   // 从外部文件加载
end

always @(posedge clk)
    dout <= rom[addr];                 // 同步读取 → 推断 BRAM

endmodule
```

### 方式 2：Block Memory Generator IP（图形化，大容量）

**Vivado 中没有叫 "ROM" 的 IP 核。** ROM 通过 **Block Memory Generator** 配置为只读模式实现。

#### IP 调用步骤

```
① Flow Navigator → IP Catalog
② 搜索栏输入 "Block Memory Generator" 双击
③ 配置页面分 5 个 Tab 依次设置
④ 点击 OK → Generate 生成 IP
```

#### 各 Tab 配置详解

**Basic 页：**

| 选项 | 说明 | 推荐值 |
|------|------|--------|
| **Component Name** | IP 实例名 | `rom_256x8` |
| **Memory Type** | 存储类型 | **Single Port ROM**（单端口 ROM） |
| | | Simple Dual Port RAM（伪双端口） |
| | | True Dual Port RAM（真双端口） |
| | 选错成 RAM 会导致综合出读写端口 | ✅ 确认选 ROM |

> 其他选项（Single Port RAM / True Dual Port RAM 等）都是 RAM 模式。**只有选 "Single Port ROM" 才是 ROM。**

**Port A Options 页：**

| 选项 | 说明 | 推荐值 |
|------|------|--------|
| **Memory Size → Width** | 数据位宽 | 8 |
| **Memory Size → Depth** | 地址深度 | 256 |
| **Enable Port Type** | 使能方式 | **Always Enabled**（最简单） |
| | | Use EN Pin（需要使能信号） |
| **Output Register** | 输出是否寄存 | ✅ **Primitives Output Register**（推荐，改善时序） |
| **Operating Mode** | 读操作模式 | **READ_FIRST**（先读后写，ROM 固定此值） |

**Port B Options 页：**（单端口 ROM 无此页，双端口才有）

**Other Options 页：**

| 选项 | 说明 | 操作 |
|------|------|------|
| **Load Init File** | 加载初始化文件 | ✅ **勾选**，点击 Browse 选择 .coe 文件 |
| **Fill Remaining Locations** | 未初始化地址填充值 | 默认 0 |

**Summary 页：** 检查最终配置概览，确认无误后点击 **OK**。

#### .coe 文件准备

在工程目录下创建 `.coe` 文件（如 `rom_data.coe`），格式如下：

```coe
; 分号开头是注释（可省略）
memory_initialization_radix=16;      ; 数据进制: 2/10/16
memory_initialization_vector=
80, 83, 86, 89, 8C, 8F, 92, 95,    ; 用逗号或空格分隔
98, 9B, 9E, A1, A4, A7, AA, AD,
00, 00, 00, 00, 00, 00, 00, 00,
FF, FF, FF, FF, FF, FF, FF, FF;
```

> `.coe` 文件存放路径不能有**中文或空格**，否则 IP 生成报错。

#### 例化模板

IP 生成后，在 `Sources → IP Sources` 中找到 `.veo` 文件（Verilog 例化模板），复制到顶层：

```verilog
// 从 .veo 文件中复制 (路径: IP Sources → rom_256x8.veo)
rom_256x8 u_rom (
    .clka   (clk),              // input  clka
    .addra  (addr),             // input  [7:0] addra
    .douta  (data_out)          // output [7:0] douta
);
```

#### IP 输出文件说明

IP 生成后，`IP Sources` 中产生以下文件：

| 文件 | 说明 |
|------|------|
| `rom_256x8.veo` | **Verilog 例化模板**（复制用） |
| `rom_256x8.vho` | VHDL 例化模板 |
| `rom_256x8.xci` | IP 配置文件（核心，同步到工程） |
| `rom_256x8.mif` | 内存初始化文件（仿真用） |
| `rom_256x8_sim_netlist.v` | 仿真模型 |

#### IP 重新配置

如需修改 ROM 参数（深度/位宽/初始化数据）：

```
右键 IP → Customize IP → 修改配置 → OK → Regenerate
```

修改 .coe 文件后，右键 IP → **Refresh IP Catalog** 重新加载。

---

### 方式 3：Distributed Memory Generator IP（图形化，小容量）

小容量 ROM（深度 < 64）可用此 IP，用 LUT 而非 BRAM 实现。

#### IP 调用步骤

```
IP Catalog → Distributed Memory Generator
```

**Basic 页：**

| 选项 | 说明 | 推荐值 |
|------|------|--------|
| **Component Name** | IP 实例名 | `dist_rom_16x8` |
| **Memory Type** | 存储类型 | **ROM**（选 Distributed RAM 就是 RAM 了） |
| **Data Width** | 数据位宽 | 8 |
| **Depth** | 地址深度 | 16 |
| **Output Width** | 输出位宽 | 同 Data Width |

**Memory Init 页：**

| 选项 | 说明 |
|------|------|
| **Load Init File** | 加载 .coe 文件 |
| **Global Init Value** | 全局初始化值（不勾选 Load Init File 时有效） |

#### 性能对比（同容量 16×8）

| 对比项 | Block ROM (BRAM) | Distributed ROM (LUT) |
|--------|-----------------|----------------------|
| 资源消耗 | 0.5 个 BRAM（浪费） | 8 个 LUT |
| 延迟 | 1 clk | 0 clk（组合输出） |
| 适合场景 | ≥64 深度 | ≤64 深度 |

---

## 三、Vivado 综合推断规则

| 代码写法 | 综合结果 |
|---------|---------|
| `depth ≥ 64` + `同步读 (posedge clk)` | **BRAM (BROM)** |
| `depth < 64` + `同步读` | **分布式 ROM** |
| `组合逻辑读 (always @*)` | **分布式 ROM**（费 LUT） |

### 查看综合结果

1. **Schematic** 中搜索 `RAMB18E1` / `RAMB36E1`（BROM 标志）
2. **Report Utilization** → `BRAM` 数量确认
3. **综合日志** → 搜索 `RAMB` / `ROM` / `infer`

```tcl
# Tcl 命令检查
report_utilization -hierarchical
```

---

## 四、实际应用场景

### DDS 正弦波发生器

```verilog
module dds_sine (
    input  wire         clk,
    input  wire [31:0]  freq_word,   // 频率控制字
    output reg  [9:0]   sine_data
);

reg [31:0] phase_acc;
reg [9:0]  sine_rom [0:1023];

initial $readmemh("sine_1024.mem", sine_rom);

always @(posedge clk) begin
    phase_acc <= phase_acc + freq_word;
    sine_data <= sine_rom[phase_acc[31:22]];  // 高 10 位查表
end
endmodule
```

### 数码管段码译码

已在 [[13查找表在FPGA中的应用与设计技巧]] 中详细讲解，本质就是小 ROM：

```verilog
always @(*) begin
    case (hex)
        4'h0: seg = 7'b1000000;
        4'h1: seg = 7'b1111001;
        // ...
    endcase
end
```

### 字符点阵字库

```verilog
module char_rom (
    input  wire         clk,
    input  wire [7:0]   char,      // ASCII
    input  wire [3:0]   row,       // 行号 0-15
    output reg  [7:0]   pixels     // 8 像素列
);

reg [7:0] font [0:4095];          // 256 字符 × 16 行

initial $readmemh("ascii_font.mem", font);

always @(posedge clk)
    pixels <= font[{char, row}];  // 拼接地址
endmodule
```

### 查找表类应用

| 应用 | 表大小 | 说明 |
|------|-------|------|
| 伽马校正 | 256×8 | 图像处理 |
| CRC 校验 | 256×8 | 通信 |
| BCD 转换 | 1024×12 | 二进制→十进制 |
| 温度补偿 | 512×16 | ADC 校准 |

---

## 五、性能对比 (XC7A35T, 256×8 ROM)

| 实现方式 | 延迟 | 资源消耗 | 最大频率 |
|---------|------|---------|---------|
| BROM (BRAM 推断) | 1 clk | 0.5 个 BRAM | 400 MHz+ |
| 分布式 ROM (LUT 同步) | 1 clk | ~64 LUT | 300 MHz |
| 分布式 ROM (LUT 组合) | 0 clk | ~256 LUT | 组合路径限制 |

---

## 六、常见问题

### Q: ROM 在 FPGA 里到底存在吗？
**不存在专用 ROM 硬件。** BRAM 和 LUT 都是 RAM，通过**不写入**（WE 接地）来模拟 ROM 行为。

### Q: BROM 和 BRAM 是什么关系？
**BROM = BRAM + 只读配置。** 同样一块硬件，RAM 模式可读写，ROM 模式只读。

### Q: 为什么 IP Catalog 搜不到 ROM？
因为 Vivado 没有独立的 ROM IP。用 **Block Memory Generator** → Memory Type 选 "ROM"。

### Q: 大 ROM (>36Kb) 怎么实现？
Vivado 自动级联多个 BRAM。例如 64Kb = 2×36Kb BRAM 拼接。

### Q: ROM 数据能在线更新吗？
FPGA 运行时不行。需 `重新综合 + 重新烧录 BIT`。如需在线更新 → 用 RAM 或 Flash。

### Q: 双端口 ROM 怎么做？
Block Memory Generator → `True Dual Port ROM`，或 HDL 声明双端口数组。

---

## 七、与已有笔记的关联

| 笔记 | 关联点 |
|------|--------|
| [[13查找表在FPGA中的应用与设计技巧]] | ROM 是查找表的物理实现，段码表即小 ROM |
| [[12HC595详解_串并转换与Verilog实现]] | 串并转换的控制表可用 ROM 实现 |
| 小梅哥 ch13：`ch13_acx720_rom_ip.rar` | ROM IP 完整工程例程 |
| 小梅哥 ch26/27 | ROM 在 TFT/HDMI 显示中的应用（字库/图像） |

---

## 参考

- UG901 (Vivado Synthesis Guide): RAM/ROM 推断章节
- 小梅哥 ACX720 例程：ch13 ROM IP / ch26 ROM 图像 / ch27 ROM 字符
- [[13查找表在FPGA中的应用与设计技巧]]
- [[15FPGA ROM 详解]]
