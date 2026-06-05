---
tags:
  - FPGA
  - VGA
  - 显示
  - 时序
---

## 核心结论

- **VGA** = Video Graphics Array，标准模拟显示接口，15 针 D-SUB（DE-15）
- **五线信号**：R/G/B 模拟（各 0~0.714V）+ HSync + VSync
- **核心是时序**：行同步 + 行消隐 + 行有效，帧同步同理
- **分辨率由像素时钟决定**：640×480@60Hz 需 25.175 MHz ≈ 25 MHz

---

## 一、VGA 接口与信号

### 引脚定义 (DE-15)

```
┌─────────────────┐
│  1(R)  2(G)  3(B) │  ─→ 模拟色差 (0~0.714V)
│  4(NC) 5(NC) 6(RGND)│
│  7(GGND) 8(BGND) 9(NC)│
│  10(SGND) 11(NC) 12(DDC_SDA)│
│  13(HSync) 14(VSync) 15(DDC_SCL)│
└─────────────────┘
```

### FPGA 实现中的信号

```verilog
module vga (
    output wire [3:0] vga_r,      // 红色 4bit (或 8bit)
    output wire [3:0] vga_g,      // 绿色 4bit
    output wire [3:0] vga_b,      // 蓝色 4bit
    output wire       vga_hsync,  // 行同步
    output wire       vga_vsync   // 场同步
);
```

### FPGA 驱动 VGA 的关键

| 要素         | 说明                                      |
| ---------- | --------------------------------------- |
| **像素时钟**   | 每个像素一个时钟，由 PLL/MMCM 生成                  |
| **行计数器**   | 从 0 计到「行总像素-1」，控制 hsync 和 hblank        |
| **场计数器**   | 每行结束 +1，从 0 到「场总行数-1」，控制 vsync 和 vblank |
| **有效显示区**  | 行/场计数器在「有效区间」时输出 RGB，否则拉 0              |
| **RGB 电压** | 3.3V 通过 3 个 330Ω 电阻分压 → 约 0.7V（VGA 标准）  |

### 电阻分压网络（4bit 颜色）

```
FPGA 3.3V ──→ 330Ω ──→ VGA_R
              470Ω ──→ VGA_R  (bit3)
              680Ω ──→ VGA_R  (bit2)
              1kΩ  ──→ VGA_R  (bit1)
              1.5kΩ ──→ VGA_R (bit0)
```

或用 R-2R 梯形网络。ACX720 开发板通常直接接 330Ω 串联电阻 + 二极管钳位。

---

## 二、VGA 时序标准 (640×480@60Hz)

### 时序参数

| 参数 | 像素时钟周期 | 时间 |
|------|:-----------:|:----:|
| 像素时钟 | — | 25.175 MHz (~39.7 ns) |
| 行频 | 800 clk | 31.78 μs |
| 帧频 | 525 行 | 16.68 ms (~60 Hz) |

### 行时序 (Horizontal)

```
        ┌──────────────────────┐
HSync ──┘                      └────────
        │← a →│← b →│← c →│← d →│
        │      │      │      │      │
        │ 同步  │ 后廊  │ 有效  │ 前廊  │
```

| 段 | 时钟数 | 说明 |
|:---:|:-----:|------|
| a: HSync 脉冲 | 96 | 同步脉冲低电平 |
| b: H Back Porch | 48 | 消隐后廊 |
| **c: H Active** | **640** | **有效像素** |
| d: H Front Porch | 16 | 消隐前廊 |
| **总和** | **800** | **行总周期** |

### 场时序 (Vertical)

```
        ┌──────────────────────┐
VSync ──┘                      └────────
        │← a →│← b →│← c →│← d →│
```

| 段 | 行数 | 说明 |
|:---:|:----:|------|
| a: VSync 脉冲 | 2 | 同步脉冲低电平 |
| b: V Back Porch | 33 | 消隐后廊 |
| **c: V Active** | **480** | **有效行** |
| d: V Front Porch | 10 | 消隐前廊 |
| **总和** | **525** | **帧总行数** |

---

## 三、Verilog 实现

### VGA 时序核心

```verilog
module vga_timing (
    input  wire         clk_25m,
    input  wire         rst_n,
    output reg          hsync,
    output reg          vsync,
    output reg          active,       // 有效显示区标志
    output reg  [9:0]   h_cnt,        // 行像素计数
    output reg  [9:0]   v_cnt         // 场行计数
);

// 行参数
localparam H_TOTAL  = 800;
localparam H_SYNC   = 96;
localparam H_BP     = 48;
localparam H_ACTIVE = 640;
localparam H_FP     = 16;

// 场参数
localparam V_TOTAL  = 525;
localparam V_SYNC   = 2;
localparam V_BP     = 33;
localparam V_ACTIVE = 480;
localparam V_FP     = 10;

// 行计数器
always @(posedge clk_25m or negedge rst_n) begin
    if (!rst_n)
        h_cnt <= 0;
    else if (h_cnt == H_TOTAL - 1)
        h_cnt <= 0;
    else
        h_cnt <= h_cnt + 1;
end

// 场计数器
always @(posedge clk_25m or negedge rst_n) begin
    if (!rst_n)
        v_cnt <= 0;
    else if (h_cnt == H_TOTAL - 1) begin
        if (v_cnt == V_TOTAL - 1)
            v_cnt <= 0;
        else
            v_cnt <= v_cnt + 1;
    end
end

// HSync
always @(posedge clk_25m or negedge rst_n) begin
    if (!rst_n)
        hsync <= 1;
    else if (h_cnt < H_SYNC)
        hsync <= 0;     // 同步脉冲低电平
    else
        hsync <= 1;
end

// VSync
always @(posedge clk_25m or negedge rst_n) begin
    if (!rst_n)
        vsync <= 1;
    else if (v_cnt < V_SYNC)
        vsync <= 0;
    else
        vsync <= 1;
end

// 有效显示区
always @(posedge clk_25m or negedge rst_n) begin
    if (!rst_n)
        active <= 0;
    else if (h_cnt >= H_SYNC + H_BP &&
             h_cnt <  H_SYNC + H_BP + H_ACTIVE &&
             v_cnt >= V_SYNC + V_BP &&
             v_cnt <  V_SYNC + V_BP + V_ACTIVE)
        active <= 1;
    else
        active <= 0;
end

endmodule
```

### 顶层模块（彩条测试）

```verilog
module vga_top (
    input  wire       clk_50m,     // 板载 50 MHz
    input  wire       rst_n,
    output wire       vga_hsync,
    output wire       vga_vsync,
    output wire [3:0] vga_r,
    output wire [3:0] vga_g,
    output wire [3:0] vga_b
);

reg [3:0] r, g, b;
reg [7:0] div_cnt;

// 25 MHz 分频
wire clk_25m;
assign clk_25m = div_cnt[0];

always @(posedge clk_50m or negedge rst_n) begin
    if (!rst_n)
        div_cnt <= 0;
    else
        div_cnt <= div_cnt + 1;
end

wire        active;
wire [9:0]  h_cnt, v_cnt;

vga_timing u_timing (
    .clk_25m (clk_25m),
    .rst_n   (rst_n),
    .hsync   (vga_hsync),
    .vsync   (vga_vsync),
    .active  (active),
    .h_cnt   (h_cnt),
    .v_cnt   (v_cnt)
);

// 彩条: 用 h_cnt[9:7] 选择 8 色
always @(posedge clk_25m or negedge rst_n) begin
    if (!rst_n) begin
        r <= 0; g <= 0; b <= 0;
    end else if (active) begin
        case (h_cnt[9:7])
            3'b000: {r,g,b} = {4'hF, 4'h0, 4'h0};  // 红
            3'b001: {r,g,b} = {4'h0, 4'hF, 4'h0};  // 绿
            3'b010: {r,g,b} = {4'h0, 4'h0, 4'hF};  // 蓝
            3'b011: {r,g,b} = {4'hF, 4'hF, 4'h0};  // 黄
            3'b100: {r,g,b} = {4'hF, 4'h0, 4'hF};  // 紫
            3'b101: {r,g,b} = {4'h0, 4'hF, 4'hF};  // 青
            3'b110: {r,g,b} = {4'hF, 4'hF, 4'hF};  // 白
            3'b111: {r,g,b} = {4'h0, 4'h0, 4'h0};  // 黑
        endcase
    end else begin
        {r,g,b} <= 0;
    end
end

assign vga_r = r;
assign vga_g = g;
assign vga_b = b;

endmodule
```

---

## 四、常用分辨率时序参数

| 分辨率 | 刷新率 | 像素时钟 | 行总 | 行有效 | HSync | HBP | HFP | 场总 | 场有效 | VSync | VBP | VFP |
|:------:|:-----:|:--------:|:----:|:-----:|:----:|:---:|:---:|:---:|:-----:|:----:|:---:|:---:|
| 640×480 | 60 Hz | 25.175 | 800 | 640 | 96 | 48 | 16 | 525 | 480 | 2 | 33 | 10 |
| 800×600 | 60 Hz | 40.0 | 1056 | 800 | 128 | 88 | 40 | 628 | 600 | 4 | 23 | 1 |
| 1024×768 | 60 Hz | 65.0 | 1344 | 1024 | 136 | 160 | 24 | 806 | 768 | 6 | 29 | 3 |
| 1280×720 | 60 Hz | 74.25 | 1650 | 1280 | 40 | 220 | 110 | 750 | 720 | 5 | 20 | 5 |
| 1920×1080 | 60 Hz | 148.5 | 2200 | 1920 | 44 | 148 | 88 | 1125 | 1080 | 5 | 36 | 4 |

### 参数含义速查

```
行总 = 行有效 + HFP + HSync + HBP
场总 = 场有效 + VFP + VSync + VBP
像素时钟 = 行总 × 场总 × 刷新率
```

---

## 五、VGA 显示图案示例

### 棋盘格

```verilog
// 棋盘格: 32×32 像素格子
wire [4:0] grid_x = h_cnt[9:5];   // 640/32 = 20 格
wire [4:0] grid_y = v_cnt[8:5];   // 480/32 = 15 格

assign color_white = grid_x[0] ^ grid_y[0];

always @(posedge clk_25m) begin
    if (active) begin
        if (color_white)
            {r,g,b} <= {4'hF, 4'hF, 4'hF};  // 白
        else
            {r,g,b} <= {4'h0, 4'h0, 4'h0};  // 黑
    end else begin
        {r,g,b} <= 0;
    end
end
```

### 十字准星

```verilog
// 屏幕中心十字
localparam CX = 320;
localparam CY = 240;

always @(posedge clk_25m) begin
    if (active) begin
        if (h_cnt == CX || v_cnt == CY)
            {r,g,b} <= {4'hF, 4'hF, 4'hF};  // 白线
        else
            {r,g,b} <= {4'h0, 4'h0, 4'h0};  // 黑底
    end else begin
        {r,g,b} <= 0;
    end
end
```

---

## 六、从 ROM 显示图片

```verilog
module vga_image (
    input  wire       clk_25m,
    input  wire       rst_n,
    output wire       vga_hsync,
    output wire       vga_vsync,
    output wire [3:0] vga_r,
    output wire [3:0] vga_g,
    output wire [3:0] vga_b
);

wire        active;
wire [9:0]  h_cnt, v_cnt;
wire [9:0]  rom_addr;
wire [11:0] pixel_data;  // 4+4+4 = 12bit RGB

vga_timing u_timing (.*);

// 将图片数据存入 BRAM ROM
// 地址: v_cnt × 640 + h_cnt（简化版：截取高地址位）
assign rom_addr = (v_cnt[8:4] * 40) + h_cnt[9:5];  // 压缩到 40×30

vga_img_rom u_rom (
    .clk  (clk_25m),
    .addr (rom_addr),
    .dout (pixel_data)
);

always @(posedge clk_25m) begin
    if (active) begin
        {r, g, b} <= pixel_data;
    end else begin
        {r, g, b} <= 0;
    end
end

assign vga_r = r;
assign vga_g = g;
assign vga_b = b;

endmodule
```

### 图片转 COE 工具（Python）

```python
from PIL import Image

img = Image.open("image.png").resize((640, 480)).convert("RGB")
with open("image.coe", "w") as f:
    f.write("memory_initialization_radix=16;\n")
    f.write("memory_initialization_vector=\n")
    for y in range(480):
        for x in range(640):
            r, g, b = img.getpixel((x, y))
            # 4bit 量化
            r4, g4, b4 = r >> 4, g >> 4, b >> 4
            pixel = (r4 << 8) | (g4 << 4) | b4
            f.write(f"{pixel:03x},")
    f.write(";")
```

---

## 七、ACX720 开发板 VGA 电路

ACX720 的 VGA 接口电路特点：

| 项目 | 说明 |
|------|------|
| FPGA 引脚 | RGB 各 4bit（共 12 个 GPIO） |
| 电阻网络 | 4bit 加权电阻分压（如 330Ω/470Ω/680Ω/1kΩ） |
| HSync/VSync | 直接 GPIO 输出（3.3V → 经 330Ω 电阻） |
| 连接器 | 标准 DE-15 母座 |

### XDC 约束示例

```tcl
set_property PACKAGE_PIN R4   [get_ports {vga_r[3]}]
set_property PACKAGE_PIN T4   [get_ports {vga_r[2]}]
set_property PACKAGE_PIN R5   [get_ports {vga_r[1]}]
set_property PACKAGE_PIN T5   [get_ports {vga_r[0]}]
set_property PACKAGE_PIN R6   [get_ports {vga_g[3]}]
set_property PACKAGE_PIN T6   [get_ports {vga_g[2]}]
set_property PACKAGE_PIN R7   [get_ports {vga_g[1]}]
set_property PACKAGE_PIN T7   [get_ports {vga_g[0]}]
set_property PACKAGE_PIN R8   [get_ports {vga_b[3]}]
set_property PACKAGE_PIN T8   [get_ports {vga_b[2]}]
set_property PACKAGE_PIN R9   [get_ports {vga_b[1]}]
set_property PACKAGE_PIN T9   [get_ports {vga_b[0]}]

set_property PACKAGE_PIN R10  [get_ports vga_hsync]
set_property PACKAGE_PIN T10  [get_ports vga_vsync]

set_property IOSTANDARD LVCMOS33 [get_ports {vga_*}]
```

---

## 八、常见问题

### Q: 屏幕无显示 / 黑屏
- 检查像素时钟频率是否正确（25 MHz @ 640×480）
- 检查 HSync/VSync 极性（标准 VGA 是负极性）
- 确认 RGB 电阻网络连接正确
- 用示波器看 HSync 频率是否 ≈ 31.5 kHz

### Q: 画面偏移 / 滚动
行/场消隐时序参数不对。对照 VGA 标准严格匹配 H_SYNC/H_BP/H_FP/V_SYNC/V_BP/V_FP。

### Q: 颜色不对
- 检查 RGB 电阻分压值
- 确认 RGB pin 顺序（R/G/B 不能混）
- 检查 IOSTANDARD 约束

### Q: 有残影 / 拖尾
- VGA 线缆过长（建议 < 3m）
- RGB 输出未在消隐区拉 0
- 像素时钟边沿有抖动

### Q: 分辨率怎么切换？
更换像素时钟 PLL 配置，并更新对应的时序参数。PLL 输出频率 = 行总 × 场总 × 刷新率。

### Q: 12bit 颜色不够用怎么办？
- 增加 RGB 位宽到 8bit（ACX720 需要改电阻网络）
- 用 PWM 做颜色深度扩展（需要足够高频）
- 换 HDMI 接口（数字信号，颜色深度高）

---

## 九、扩展：VGA 字符显示

```verilog
module vga_text (
    input  wire         clk_25m,
    input  wire         rst_n,
    output wire         vga_hsync,
    output wire         vga_vsync,
    output wire [3:0]   vga_r,
    output wire [3:0]   vga_g,
    output wire [3:0]   vga_b
);

// 字符参数: 80×30 字符，每个 8×16 像素
// 字符 ROM: 8'h00 ~ 8'h7F (ASCII), 16 行/字符
// 显存: 80 × 30 = 2400 字节

wire [9:0]  h_cnt, v_cnt;
wire        active;

vga_timing u_timing (.*);

// 计算字符行列
wire [6:0]  char_col = h_cnt[9:3];   // 640/8 = 80
wire [4:0]  char_row = v_cnt[8:4];   // 480/16 = 30
wire [3:0]  char_line = v_cnt[3:0];  // 字符内行号
wire [2:0]  char_pix = h_cnt[2:0];   // 字符内列号

// 显存地址 = char_row × 80 + char_col
wire [12:0] mem_addr = char_row * 80 + char_col;
wire [7:0]  ascii;

// 显存 BRAM（在线可写，实现屏幕输出）
// vga_fb: Frame Buffer

// 像素着色
wire [7:0] font_data;  // 8bit 字模
wire pixel_on = font_data[7 - char_pix];

assign vga_r = active ? (pixel_on ? 4'hF : 4'h0) : 0;
assign vga_g = active ? (pixel_on ? 4'hF : 4'h0) : 0;
assign vga_b = active ? (pixel_on ? 4'hF : 4'h0) : 0;

endmodule
```

---

## 十、与已有笔记的关联

| 笔记 | 关联点 |
|------|--------|
| [[15FPGA ROM 详解]] | 图片/字符需要 ROM 存储像素数据 |
| [[16FPGA FIFO 详解]] | 高速显示可用 FIFO 做行缓冲 |
| [[13查找表在FPGA中的应用与设计技巧]] | 颜色映射、Gamma 校正用查找表 |
| [[17DDS 直接数字频率合成详解]] | VGA 像素时钟由 PLL 生成，与 DDS 同源 |
| PLL/MMCM | 生成不同分辨率所需像素时钟 |

---

## 参考

- [VGA Signal Timing (tinyvga.com)](http://tinyvga.com/vga-timing)
- Xilinx UG901: 时钟资源与 PLL 配置
- ACX720 原理图: VGA 接口电路
- 小梅哥 ch25: VGA 显示驱动例程
- [[17DDS 直接数字频率合成详解]]
