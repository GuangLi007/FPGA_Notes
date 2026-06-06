---
tags:
  - FPGA
  - HDMI
  - 显示
  - TMDS
  - DVI
  - 时序
---

## 核心结论

- **HDMI** = High-Definition Multimedia Interface，数字视频/音频接口，基于 **DVI 1.0** 规范
- **TMDS** = Transition Minimized Differential Signaling，HDMI 的物理层编码方式
- **差分信号**：每个通道一对 P/N 差分线，抗干扰强，支持更高频率
- **三数据通道 + 一时钟通道**：R/G/B 各占一个 TMDS 通道
- **ACX720 板载双 HDMI**（HDMI1 + HDMI2），无需外接模块

---

## 一、HDMI vs VGA 对比

| 特性 | VGA | HDMI |
|------|:---:|:----:|
| 信号类型 | 模拟（RGB + HSync/VSync） | 数字（TMDS 差分） |
| 颜色深度 | 通常 4~8bit/通道 | 8~16bit/通道 |
| 抗干扰 | 差（模拟信号易衰减） | 好（差分信号） |
| 最高分辨率 | 受限于线缆和 DAC | 最高 4K@60Hz+ |
| 音频传输 | 不支持 | 支持 |
| 即插即用 | DDC（I2C）读取 EDID | 同 VGA（DDC + EDID） |
| FPGA 实现复杂度 | 低（直接输出 RGB + 同步信号） | 高（TMDS 8b/10b 编码 + 串行化） |
| ACX720 板载 | ❌ 需 ACM7123 模块 (GPIO2) | ✅ 双 HDMI 接口（FGG484 直连） |

### 核心区别

VGA 直接将像素的 RGB 值经电阻分压输出模拟电压，时序仅需 HSync/VSync。
HDMI 则需经过 **3 层处理**：

```
RGB 8bit → TMDS 8b/10b 编码 → 并串转换 (10:1) → OBUFDS 差分输出
```

---

## 二、ACX720 HDMI 硬件

### 引脚定义（HDMI1）

| 信号 | FPGA 引脚 | IOSTANDARD | 说明 |
|------|:---------:|:----------:|------|
| hdmi1_clk_p | K4 | TMDS_33 | 时钟差分 P |
| hdmi1_clk_n | J4 | TMDS_33 | 时钟差分 N |
| hdmi1_data_p[0] | L3 | TMDS_33 | 蓝色通道 P |
| hdmi1_data_n[0] | K3 | TMDS_33 | 蓝色通道 N |
| hdmi1_data_p[1] | L5 | TMDS_33 | 绿色通道 P |
| hdmi1_data_n[1] | L4 | TMDS_33 | 绿色通道 N |
| hdmi1_data_p[2] | M3 | TMDS_33 | 红色通道 P |
| hdmi1_data_n[2] | M2 | TMDS_33 | 红色通道 N |
| hdmi1_oe | H5 | LVCMOS33 | 输出使能（高有效） |

### 引脚定义（HDMI2）

| 信号 | FPGA 引脚 | IOSTANDARD |
|------|:---------:|:----------:|
| hdmi2_clk_p | H4 | TMDS_33 |
| hdmi2_clk_n | G4 | TMDS_33 |
| hdmi2_data_p[0] | E2 | TMDS_33 |
| hdmi2_data_n[0] | D2 | TMDS_33 |
| hdmi2_data_p[1] | F3 | TMDS_33 |
| hdmi2_data_n[1] | E3 | TMDS_33 |
| hdmi2_data_p[2] | H3 | TMDS_33 |
| hdmi2_data_n[2] | G3 | TMDS_33 |
| hdmi2_oe | K2 | LVCMOS33 |

> **OE 引脚必须拉高**（`assign hdmi_oe = 1'b1`），否则无输出。
> 引脚号以 `ACX720_FPGA管脚分配表20201221.xlsx` 为准，不同板版本可能不同。

### 板级通路

```
FPGA (XC7A35T FGG484)
  ┌─ HDMI1: TMDS_33 差分直连 HDMI 连接器
  │    OE = H5 → 控制 HDMI 输出缓冲
  │
  └─ HDMI2: TMDS_33 差分直连 HDMI 连接器
       OE = K2 → 控制 HDMI 输出缓冲
```

---

## 三、TMDS 编码原理

### 为什么需要 TMDS？

HDMI 传输的是**数字信号**，裸传 8bit RGB + HSync/VSync 会产生以下问题：
- 频谱集中在低频，EMI 大
- 没有 DC 平衡，接收端难以恢复时钟
- 无控制字符，无法区分"有效像素"和"控制信号"

TMDS 通过 **8b/10b 编码**解决上述问题。

### TMDS 编码流程（两阶段）

```
Stage 1: 8 bit → 9 bit（最小化跳变）
─────────────────────────────────
  din[7:0] → 统计 1 的个数 N1
  if (N1 > 4 或 (N1 == 4 && din[0] == 0)):
    q_m[0] = din[0]
    q_m[i] = q_m[i-1] XNOR din[i]  // 减少跳变
    q_m[8] = 0
  else:
    q_m[0] = din[0]
    q_m[i] = q_m[i-1] XOR din[i]   // 原始编码
    q_m[8] = 1

Stage 2: 9 bit → 10 bit（DC 平衡）
─────────────────────────────────
  统计 q_m[7:0] 的 1 和 0 个数
  根据累积差异 cnt 决定是否取反：
    if (cnt == 0 || N1q_m == N0q_m):
      dout[9] = ~q_m[8], dout[8] = q_m[8]
      dout[7:0] = q_m[8] ? q_m[7:0] : ~q_m[7:0]
    else if ((cnt > 0 && N1 > N0) || cnt < 0 && N0 > N1):
      dout[9] = 1, dout[8] = q_m[8], dout[7:0] = ~q_m[7:0]
    else:
      dout[9] = 0, dout[8] = q_m[8], dout[7:0] = q_m[7:0]
```

### 控制字符（非显示区）

当 de = 0 时（消隐区），TMDS 发送控制字符，HSync/VSync 嵌入在蓝色通道：

| {c1, c0} | 10bit 控制码 |
|:--------:|:------------:|
| 00 | 1101010100 |
| 01 | 0010101011 |
| 10 | 0101010100 |
| 11 | 1010101011 |

---

## 四、工程架构（Acx720_Hdmi_Test）

### 模块层级

```
hdmi_top (顶层)
├── clk_gen              — MMCME2_BASE: 50MHz → 74.219MHz (像素) + 371.094MHz (5x串行)
│
├── colour_bar_dat_gen   — 8 色彩条发生器（4 行 × 2 列）
│
├── disp_driver           — 显示时序驱动（1280×720@60Hz 标准 VESA 时序）
│   └── disp_parameter_cfg  — 时序参数定义（使用 `include）
│
└── dvi_encoder × 2      — HDMI1 和 HDMI2 各一个
    ├── tmds_encoder × 3   — R/G/B 各一个 TMDS 8b/10b 编码器
    └── serdes_4b_10to1    — 4 路 10:1 串行化（ODDR + OBUFDS）
```

### 各模块功能

| 模块                 | 文件                   | 功能                               |
| ------------------ | -------------------- | -------------------------------- |
| clk_gen            | clk_gen.v            | MMCME2_BASE 原语，生成像素时钟和 5x 串行时钟   |
| colour_bar_dat_gen | colour_bar_dat_gen.v | 在 1280×720 有效区生成 4 行×2 列 = 8 色彩条 |
| disp_driver        | disp_driver.v        | 行/场计数器，生成 DE、HSync、VSync，提供显示地址  |
| disp_parameter_cfg | disp_parameter_cfg.v | `include 宏定义文件，包含 VESA 时序参数      |
| dvi_encoder        | dvi_encoder.v        | 集成 3 个 TMDS 编码器 + 1 个串行化器        |
| tmds_encoder       | tmds_encoder.v       | 标准的 DVI 1.0 TMDS 8b/10b 编码器      |
| serdes_4b_10to1    | serdes_4b_10to1.v    | 4 路 10:1 串行化，使用 ODDR + OBUFDS 原语 |

### 信号路径

```
                         clk_gen
                     ┌──────────────┐
50MHz ──────────────→│  MMCME2_BASE  │──→ pixelclk (74.219 MHz)
                     │              │──→ pixelclk5x (371.094 MHz)
                     └──────────────┘
                           │
                           ▼
 colour_bar_dat_gen   disp_driver
 ┌──────────────┐    ┌────────────────┐
 │ 彩条图案数据  │───→│ 行/场时序驱动   │──→ disp_hs, disp_vs, disp_de
 │ 24bit RGB    │    │ 地址 → 图案     │──→ disp_red[7:0], grn[7:0], blu[7:0]
 └──────────────┘    └────────────────┘
                           │
                           ▼
                   dvi_encoder (×2: HDMI1 + HDMI2)
              ┌──────────────────────────┐
              │  tmds_encoder (蓝 + 绿 + 红) │──→ 10bit × 3
              │  serdes_4b_10to1           │──→ ODDR 串行化 → OBUFDS 差分
              └──────────────────────────┘
                           │
                           ▼
                    HDMI 连接器 → 显示器
```

### 时钟方案

由于 MMCME2_BASE 参数限制（CLKFBOUT_MULT_F 范围 2~64，步长 0.125）和 VCO 范围（600~1440 MHz），从 **50 MHz** 无法精确生成 **74.25 MHz**。本工程采用最近似配置：

```
DIVCLK_DIVIDE = 4
CLKFBOUT_MULT_F = 59.375 (= 475/8)
VCO = 50 × 59.375 / 4 = 742.1875 MHz
CLKOUT0_DIVIDE_F = 10 → pixelclk = 74.21875 MHz（误差 -0.042%）
CLKOUT1_DIVIDE = 2 → pixelclk5x = 371.09375 MHz
```

误差远小于显示器可接受范围（±0.5%），实测正常工作。

---

## 五、HDMI 与 VGA 实现的 3 个关键区别

### 1. 信号类型

| | VGA | HDMI |
|---|---|---|
| 电信号 | 单端 LVCMOS33 | 差分 TMDS_33 |
| FPGA 原语 | 普通 IO + 电阻网络 | OBUFDS（差分缓冲） |
| 引脚分配 | GPIO 扩展口 | 专用 HDMI 引脚 |

### 2. 需要串行化

VGA 在像素时钟每个周期直接输出多位 RGB 并行数据。HDMI 需要将 10bit 编码数据在 **5 倍像素时钟**的 DDR 模式下逐 bit 串行输出：

```
VGA:  像素时钟沿 → 直接输出 RGB[3:0] × 3
HDMI: 像素时钟 × 5 → ODDR 逐 bit 输出 (10bit → 1bit × 10 个半周期)
```

### 3. HSync/VSync 嵌入方式

VGA 的 HSync/VSync 是独立的信号线。
HDMI 将 HSync/VSync 嵌入到蓝色通道的 TMDS **控制字符**中：

```
VGA:  独立的 HSync, VSync 引脚
HDMI: HSync → tmds_encoder 的 c0 输入（蓝色通道）
      VSync → tmds_encoder 的 c1 输入（蓝色通道）
      消隐区自动发送控制码，有效区发送像素数据
```

---

## 六、ifdef / elsif / endif 在 FPGA 中的应用

### 作用

Verilog 的 ``ifdef``/`elsif`/``endif` 是**预编译指令**（类似于 C 语言的 `#ifdef`），在综合前决定哪些代码被编译。常用于：
- 同一文件适配多种分辨率/颜色深度
- 调试/发布版本切换
- 多平台兼容

### 语法

```verilog
`ifdef MACRO_NAME
  // 当 MACRO_NAME 被定义时，编译这段代码
`elsif ANOTHER_MACRO
  // 当 ANOTHER_MACRO 被定义时，编译这段代码
`else
  // 以上都不满足时，编译这段代码
`endif
```

### 定义宏的方法

**方法 1：在代码中用 ``define**（推荐）

```verilog
`define Resolution_1280x720
```

**方法 2：在 XDC 中定义**

```tcl
set_property VerilogMacro "Resolution_1280x720=1" [get_filesets sources_1]
```

**方法 3：在 Vivado 综合设置中设置**

```
Settings → Synthesis → verilog_macros → 添加 Resolution_1280x720
```

**方法 4：TCL 脚本**

```tcl
set_property GENERATE {Resolution_1280x720=1} [current_project]
```

### 典型应用：多分辨率适配

以下是 `disp_parameter_cfg.v` 中最典型的应用：

```verilog
// =====================================
// 选择分辨率（只需 define 一个，其他自动匹配）
// =====================================
`ifdef Resolution_640x480
  `define H_Total_Time    12'd800
  `define H_Front_Porch   12'd8
  `define H_Sync_Time     12'd96
  `define H_Back_Porch    12'd40
  // ... 其他 640x480 参数

`elsif Resolution_800x600
  `define H_Total_Time    12'd1056
  `define H_Front_Porch   12'd40
  `define H_Sync_Time     12'd128
  `define H_Back_Porch    12'd88
  // ... 其他 800x600 参数

`elsif Resolution_1280x720
  `define H_Total_Time    12'd1650
  `define H_Front_Porch   12'd110
  `define H_Sync_Time     12'd40
  `define H_Back_Porch    12'd220
  // ... 其他 1280x720 参数

`elsif Resolution_1920x1080
  `define H_Total_Time    12'd2200
  `define H_Front_Porch   12'd88
  `define H_Sync_Time     12'd44
  `define H_Back_Porch    12'd148
  // ... 其他 1920x1080 参数

`endif
```

使用时只需定义一个宏：

```verilog
`define Resolution_1280x720 1   // ← 切换到这里
// `define Resolution_640x480 1 // ← 注释掉其他
```

### 典型应用：RGB565 vs RGB888

```verilog
`ifdef MODE_RGB888
  `define Red_Bits   8
  `define Green_Bits 8
  `define Blue_Bits  8
  // disp_driver 中 Reg 宽度自动变为 24bit

`elsif MODE_RGB565
  `define Red_Bits   5
  `define Green_Bits 6
  `define Blue_Bits  5
  // disp_driver 中 Reg 宽度自动变为 16bit
`endif
```

### 注意点

1. **``define 顺序敏感**：按文件编译顺序生效，A 在 B 之前编译则 A 定义的宏 B 能看见，反之则不行

   ```verilog
   // ❌ 错误：先编译 display.v，后编译 config.v
   // display.v
   reg [`H_TOTAL-1:0] h_cnt;  // H_TOTAL 未定义！

   // config.v
   `define H_TOTAL 1650

   // ✅ 正确：先编译 config.v，再编译 display.v
   // config.v  — 先编译
   `define H_TOTAL 1650

   // display.v  — 后编译
   reg [`H_TOTAL-1:0] h_cnt;  // OK
   ```

   **三种方案的可靠性对比**：

   | 方案 | 是否可靠 | 说明 |
   |-----|:--------:|------|
   | 靠编译顺序 | ❌ | Vivado 文件顺序可拖动，容易搞乱 |
   | 全放 TOP | ❌ | TOP 本身也是一个文件，同样有顺序问题 |
   | include 头文件 | ✅ | 不受编译顺序影响，每个文件显式包含 |

   ```verilog
   // ✅ 最安全：被依赖的文件用 `include 显式包含
   // params.v
   `define H_TOTAL 1650

   // disp_driver.v
   `include "params.v"       // 展开后 H_TOTAL 就有定义了
   reg [`H_TOTAL-1:0] h_cnt; // OK，无论文件编译顺序如何

   // hdmi_top.v
   `include "params.v"       // 需要的地方都 include 一次
   ```

   `include 本质是**文本替换**——在预处理阶段直接把被包含文件内容插入到当前位置，所以不依赖 Vivado 的文件编译顺序。

2. 宏名不要和变量名冲突，习惯加 `_` 或 `RES_` 前缀
3. `endif 必须成对出现，缺失会导致综合报错
4. 宏条件中不要用 `==` 比较值，如 ``ifdef MACRO` 只检查**是否定义**，不检查值。如果需要值比较，用： 
   ```verilog
   `ifdef MY_MACRO
     if (`MY_MACRO == 1) ... // 在运行时判断，不是编译期判断
   `endif
   ```
5. 参数传递更推荐使用 **Verilog parameter** 重写，宏只是编译期选择的手段

### 与 parameter + generate 的对比

| | `ifdef | generate + parameter |
|---|---|---|
| 作用阶段 | 编译前预处理 | 综合时 |
| 适用范围 | 整个代码块任意内容 | 只能对模块实例化/assign/always 等结构化元素 |
| 灵活性 | 高（可控制任意代码是否编译） | 较低 |
| 可读性 | 较差（多个分支难追踪） | 较好 |
| 典型场景 | 多平台/多分辨率/调试开关 | 参数化模块宽度、条件实例化 |

---

## 七、1280×720@60Hz 时序参数（VESA 标准）

### 行时序

| 段 | 时钟数 | 说明 |
|:---:|:-----:|------|
| H_Active | 1280 | 有效像素 |
| H_Front_Porch | 110 | 前廊 |
| H_Sync | 40 | 行同步 |
| H_Back_Porch | 220 | 后廊 |
| **H_Total** | **1650** | **行总周期** |

### 场时序

| 段 | 行数 | 说明 |
|:---:|:----:|------|
| V_Active | 720 | 有效行 |
| V_Front_Porch | 5 | 前廊 |
| V_Sync | 5 | 场同步 |
| V_Back_Porch | 20 | 后廊 |
| **V_Total** | **750** | **帧总行数** |

### 计算

```
像素时钟  = 74.25 MHz
行频     = 74.25 MHz / 1650 = 45 kHz
帧频     = 45 kHz / 750 ≈ 60 Hz
实际帧频  = 74.21875 MHz / 1650 / 750 ≈ 59.97 Hz
```

---

## 八、常见问题

### Q: 屏幕显示"无信号" / 黑屏
- 检查 HDMI 线是否插紧
- 确保 `hdmi_oe` 已拉高（`1'b1`）
- 检查 PLL 是否锁定（`locked` 信号应拉高）
- 用示波器查看 HDMI CLK P 引脚有无 74 MHz 方波
- 检查 XDC 引脚号与板版本匹配

### Q: 画面闪烁 / 雪花
- TMDS 差分对 P/N 极性接反？检查 XDC 约束
- 像素时钟和 5x 时钟相位未对齐？确保 MMCM 同一 VCO 分频
- OBUFDS 约束未加 SLEW？添加 `set_property SLEW SLOW`

### Q: 颜色不对（偏色 / 通道错位）
- `dvi_encoder` 中 R/G/B 通道顺序：datain_0 = B, datain_1 = G, datain_2 = R
- 与显示器 EDID 协商的颜色空间不匹配（本工程固定 RGB）

### Q: 分辨率/刷新率不对
- 检查 `disp_parameter_cfg.v` 中的时序参数
- 确认 MMCM 实际输出频率（Vivado 中运行 report_clocks）
- 部分显示器需要 EDID 模拟，本工程未实现，显示器使用默认检测

### Q: TMDS 编码报时序违规
- 确保 `serdes_4b_10to1` 中 ODDR 的时钟域正确（pixelclk5x 必须 = 5 × pixelclk）
- 减少组合逻辑路径：在 tmds_encoder 输出加寄存器打拍

---

## 九、与已有笔记的关联

| 笔记 | 关联点 |
|------|--------|
| [[23VGA 显示驱动详解]] | 显示时序原理相同，HDMI 增加 TMDS 编码 + 串行化 |
| [[20FPGA ROM 详解]] | 图片显示需 ROM 存储像素，2560×720 需大容量 BRAM |
| [[21FPGA FIFO 详解]] | 高速 HDMI 可用 FIFO 做跨时钟域处理 |
| [[22DDS 直接数字频率合成详解]] | MMCM 时钟生成与 DDS 同源（PLL/MMCM 原语） |
| [[19查找表在FPGA中的应用与设计技巧]] | Gamma 校正可以用 LUT 实现 |

---

## 参考

- DVI 1.0 Specification (Digital Display Working Group)
- Xilinx UG471: 7 Series SelectIO Resources（OSERDESE2 / ODDR / OBUFDS）
- Xilinx UG472: 7 Series Clocking Resources（MMCME2_BASE）
- ACX720_FPGA管脚分配表20201221.xlsx
- 小梅哥 ch25: acx720_hdmi_colour_bar
- [[23VGA 显示驱动详解]]
