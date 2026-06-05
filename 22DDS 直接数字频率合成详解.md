---
tags:
  - FPGA
  - DDS
  - 信号生成
  - 数字电路
---

## 核心结论

- **DDS** = Direct Digital Synthesis，直接数字频率合成，从数字域生成可控频率/相位/幅度的模拟信号
- **三要素**：相位累加器 → ROM 查表 → DAC/量化输出
- **频率控制字 (FCW/Freq Word)** 决定输出频率，`f_out = FCW × f_clk / 2^N`
- **频率分辨率** = `f_clk / 2^N`，仅取决于累加器位宽 N，N 越大精度越高

---

## 一、DDS 基本原理

### 系统框图

```
         FCW ──→┌──────────┐    addr    ┌──────────┐ 幅度  ┌──────────┐
    clk ──→    │  相位累加器  │───→──────→│  波形 ROM  │───→──→│  DAC /   │──→ f_out
               │   N bit    │           │ M × W    │      │  滤波器   │
               └──────────┘            └──────────┘      └──────────┘
```

### 工作流程

| 步骤     | 说明                              |
| ------ | ------------------------------- |
| ① 相位累加 | 每个时钟周期，N 位累加器加上频率控制字 FCW        |
| ② 相位截断 | 取累加器高 M 位作为 ROM 地址（可选，可节省资源）    |
| ③ 波形查表 | 用地址查 ROM，得到当前相位对应的幅度值           |
| ④ 数模转换 | 数字幅度 → DAC → 模拟信号（或直接 PWM/量化输出） |
| ⑤ 低通滤波 | 滤除 DAC 引入的阶梯波高频分量，恢复平滑波形        |

### 关键公式

| 公式                          | 含义          |
| --------------------------- | ----------- |
| `f_out = FCW × f_clk / 2^N` | 输出频率        |
| `f_res = f_clk / 2^N`       | 频率分辨率（最小步进） |
| `FCW = f_out × 2^N / f_clk` | 已知目标频率反推控制字 |
| `Δφ = FCW × 2π / 2^N`       | 每时钟相位增量（弧度） |

---

## 二、DDS 核心参数

| 参数        | 说明             | 典型值                   |
| --------- | -------------- | --------------------- |
| **N**     | 相位累加器位宽        | 24~48 bit             |
| **M**     | ROM 地址位宽（查表精度） | 8~14 bit              |
| **W**     | ROM 数据位宽（幅度精度） | 8~16 bit              |
| **f_clk** | 系统时钟频率         | 50~200 MHz            |
| **f_out** | 输出信号频率         | DC ~ f_clk/2          |
| **SFDR**  | 无杂散动态范围        | 随 M 增大而改善，约 6.02×M dB |

### 相位截断

```
N bit 累加器:  [N-1 .. M .. 1 0]
               │  高 M 位 → ROM 地址  │  低 N-M 位丢弃 │
```

截断会引入相位截断噪声，但将 M 取到 10~14 bit 可忽略不计。

---

## 三、DDS 的 Verilog 实现

### 基础 DDS 模块

```verilog
module dds #(
    parameter N = 32,          // 相位累加器位宽
    parameter M = 10,          // ROM 地址位宽
    parameter W = 10           // 幅度数据位宽
) (
    input  wire               clk,
    input  wire               rst_n,
    input  wire [N-1:0]       freq_word,   // 频率控制字
    input  wire [N-1:0]       phase_word,  // 相位偏置（可选）
    output reg  [W-1:0]       dout         // 幅度输出
);

reg [N-1:0] phase_acc;

always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        phase_acc <= 0;
    else
        phase_acc <= phase_acc + freq_word;
end

wire [M-1:0] rom_addr = phase_acc[N-1:N-M] + phase_word[N-1:N-M];

// 调用波形 ROM（见下方）
wave_rom #(
    .M (M),
    .W (W)
) u_rom (
    .clk  (clk),
    .addr (rom_addr),
    .dout (dout)
);

endmodule
```

### 正弦波 ROM（Matlab/Coe 生成）

```verilog
module wave_rom #(
    parameter M = 10,
    parameter W = 10
) (
    input  wire              clk,
    input  wire [M-1:0]      addr,
    output reg  [W-1:0]      dout
);

reg [W-1:0] rom [0:2**M-1];

initial $readmemh("sine_1024.mem", rom);

always @(posedge clk)
    dout <= rom[addr];

endmodule
```

#### 生成 .mem 文件（Python）

```python
import math

M, W = 10, 10  # 地址位宽 10, 数据位宽 10
depth = 1 << M

with open("sine_1024.mem", "w") as f:
    for i in range(depth):
        phase = 2 * math.pi * i / depth
        val = int((math.sin(phase) + 1) * (2**W - 1) / 2)
        f.write(f"{val:03x}\n")
```

#### 生成 .coe 文件（Vivado IP 用）

```python
with open("sine_1024.coe", "w") as f:
    f.write("memory_initialization_radix=16;\n")
    f.write("memory_initialization_vector=\n")
    for i in range(depth):
        phase = 2 * math.pi * i / depth
        val = int((math.sin(phase) + 1) * (2**W - 1) / 2)
        f.write(f"{val:02x}" + ("," if i < depth - 1 else ";"))
```

---

## 四、Vivado 中的 DDS IP 核

Vivado 提供 **DDS Compiler** IP 核，无需手写累加器 + ROM：

```
IP Catalog → DDS Compiler (6.0)
```

### 配置页详解

**Configuration 页：**

| 选项 | 说明 | 推荐值 |
|------|------|--------|
| **System Clock** | 系统时钟频率 | 实际值 (如 50 MHz) |
| **Number of Channels** | 通道数 | 1（多通道测相位差等场景用） |
| **Mode of Operation** | 工作模式 | **Standard**（标准 DDS）或 **Rasterized**（栅格化） |
| **Parameter Selection** | 参数选择方式 | **Hardware Parameters**（手动设位宽）或 **System Parameters**（设频率分辨率） |
| **Phase Width** | 相位位宽 N | 16~32 |
| **Output Width** | 幅度位宽 W | 8~16 |

**Implementation 页：**

| 选项 | 说明 | 推荐值 |
|------|------|--------|
| **Noise Shaping** | 噪声整形 | None（普通）或 Taylor Series Corrected（改善 SFDR） |
| **Memory Type** | ROM 实现方式 | **Auto**（自动选 BRAM/Distributed） |
| **Optimization Goal** | 优化目标 | **Area**（省资源）或 **Speed**（高性能） |

**Output Frequencies 页：**

可直接输入目标频率，IP 自动计算 FCW。Standard 模式下需手动配 FCW。

### 例化模板

```verilog
dds_compiler_0 u_dds (
    .aclk                 (clk),
    .s_axis_phase_tvalid  (1'b1),
    .s_axis_phase_tdata   (freq_word),   // 频率控制字
    .m_axis_data_tvalid   (dds_valid),
    .m_axis_data_tdata    (sine_data)    // 正弦波幅度
);
```

---

## 五、DDS 性能优化

### 相位截断 vs SFDR

| M (地址位宽) | SFDR 理论值 | 说明 |
|:-----------:|:-----------:|------|
| 8 | ~48 dB | 可用，谐波较明显 |
| 10 | ~60 dB | 一般应用 |
| 12 | ~72 dB | 通信系统 |
| 14 | ~84 dB | 高精度仪表 |

SFDR ≈ 6.02 × M (dB)（相位截断主导时）

### 幅度量化噪声

幅度位宽 W 决定信噪比：`SNR ≈ 6.02 × W + 1.76 (dB)`

| W (bit) | SNR 理论值 |
|:-------:|:----------:|
| 8 | ~50 dB |
| 10 | ~62 dB |
| 12 | ~74 dB |
| 14 | ~86 dB |
| 16 | ~98 dB |

### 输出频率限制

- **最大输出频率** ≈ `f_clk / 2`（奈奎斯特限制）
- **推荐** `f_out ≤ f_clk / 4`，便于重建滤波
- 实际 DDS 输出阶梯波，含 `f_clk ± f_out` 镜像频率，需 LPF 滤除

---

## 六、多波形 DDS

修改波形 ROM 内容即可输出不同波形：

```verilog
module multi_wave_dds #(
    parameter N = 32,
    parameter M = 10,
    parameter W = 10
) (
    input  wire               clk,
    input  wire               rst_n,
    input  wire [N-1:0]       freq_word,
    input  wire [1:0]         wave_sel,    // 00: sine, 01: saw, 10: square, 11: triangle
    output reg  [W-1:0]       dout
);

reg [N-1:0] phase_acc;

always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        phase_acc <= 0;
    else
        phase_acc <= phase_acc + freq_word;
end

wire [M-1:0] addr = phase_acc[N-1:N-M];

// 用 4 个 ROM 或 1 个大 ROM（地址高位选波形）
reg [W-1:0] rom [0:4*2**M-1];

initial $readmemh("multi_wave.mem", rom);

always @(posedge clk)
    dout <= rom[{wave_sel, addr}];

endmodule
```

| 波形 | ROM 生成公式 | 特点 |
|------|-------------|------|
| 正弦波 | `sin(θ)` | 基频纯净，需 DAC |
| 方波 | `sin(θ) > 0 ? max : 0` | 数字输出即可，谐波丰富 |
| 三角波 | `|θ/π| * max` | 线性变化 |
| 锯齿波 | `θ/(2π) * max` | 扫频用 |
| 任意波 | 采样自定义波形 | 任意波形发生器 |

---

## 七、实际工程要点

### DDS 时钟频率选择

| 应用 | 推荐 f_clk | 说明 |
|------|:----------:|------|
| 音频信号源 | 1~50 MHz | 20 kHz 输出足够 |
| 低频测试信号 | 10~50 MHz | 100 kHz 级输出 |
| 中频通信信号 | 50~200 MHz | 10~50 MHz 输出 |
| 高频 RF 信号 | 200 MHz+ | 需高速 DAC + 专用 DDS 芯片 |

### 与 DAC 的接口

```verilog
// DDS → DAC 输出（以 8 位并行 DAC 为例）
wire [W-1:0] dds_data;
wire         dds_valid;

dds #(32, 10, 8) u_dds (
    .clk       (clk),
    .rst_n     (rst_n),
    .freq_word (32'd429496730),  // f_out = 1 MHz @ 50 MHz clk
    .dout      (dds_data)
);

assign dac_data  = dds_data;
assign dac_clk   = clk;         // DAC 时钟与 DDS 同源
```

### 频率切换响应

- FCW 随时可改，下一时钟生效
- 频率切换**相位连续**（DDS 最大优势之一）
- 跳频速度 = 时钟周期级别（ns 级）

---

## 八、DDS vs PLL 对比

| 对比项 | DDS | PLL |
|--------|-----|-----|
| 频率分辨率 | 极高（`f_clk/2^N`） | 受限于鉴相器频率 |
| 跳频速度 | 时钟周期（ns） | 环路锁定时间（μs~ms） |
| 相位连续 | ✅ 连续 | ❌ 可能失锁 |
| 高频输出 | 受限（`f_clk/2`） | 可倍频至 GHz |
| 杂散/噪声 | 量化噪声为主 | 相位噪声为主 |
| 面积 | 逻辑 + BRAM | 模拟电路 + 电容 |
| FPGA 实现 | 纯数字，易集成 | 需专用硬核 (MMCM/PLL) |

**典型场景：**
- DDS：信号源、跳频通信、调制器、扫频仪
- PLL：时钟生成、高频本振、频率综合

---

## 九、常见问题

### Q: DDS 输出频率能到多少？
理论上限 `f_clk/2`（奈奎斯特），建议 `f_clk/4` 以内。50 MHz 时钟 → 25 MHz 理论上限，实测推荐 ≤ 12.5 MHz。

### Q: 相位累加器位宽 N 取多大？
**N ≥ 24** 为好。50 MHz 时钟下：
- N=24：分辨率 ≈ 2.98 Hz
- N=32：分辨率 ≈ 0.0116 Hz
- N=48：分辨率 ≈ 0.18 μHz

### Q: 相位截断会怎样？
会引入相位截断噪声，表现为杂散谱线。一般 M ≥ 10 即可满足大部分应用。Vivado DDS Compiler 可加 Taylor Series Correction 改善。

### Q: DDS ROM 用 BRAM 还是分布式？
深度 ≥ 64 时 BRAM 更省资源。<64 时分布式 ROM（LUT）更优。多波形 ROM 建议用 BRAM（容量大）。

### Q: 怎么调幅度？
乘法器乘幅度系数，或 DAC 的参考电压控制。Vivado DDS IP 提供 `m_axis_phase_tdata` 和可选的幅度控制通道。

### Q: 扫频怎么实现？
```verilog
// 线性扫频：每步增加 FCW
always @(posedge clk) begin
    if (sweep_en)
        freq_word <= freq_word + sweep_step;
end
```

### Q: 输出正弦波有台阶怎么办？
提高采样率、加外部 LPF。FPGA 内也可做插值滤波（CIC/FIR）提高等效采样率。

---

## 十、与已有笔记的关联

| 笔记 | 关联点 |
|------|--------|
| [[20FPGA ROM 详解]] | DDS 的核心是波形 ROM，ROM 是实现基础 |
| [[21FPGA FIFO 详解]] | 连续输出数据流可用 FIFO 缓存 |
| [[19查找表在FPGA中的应用与设计技巧]] | DDS 查表即查找表的经典应用 |
| [[10呼吸灯PWM设计]] | PWM + 低通滤波可替代 DAC 实现简易 DDS |
| [[5线性序列机原理与应用]] | 相位累加器本质是线性序列机 |

---

## 参考

- Xilinx PG141: DDS Compiler IP 手册
- UG901: RAM/ROM 推断
- 小梅哥 ch17: DDS 信号发生器例程
- 小梅哥 ch26/27: DDS 在 TFT 显示中的应用
- [[20FPGA ROM 详解]]
- [Analog Devices: DDS 教程 MT-085](https://www.analog.com/en/education/education-library/tutorials/mt085.html)
