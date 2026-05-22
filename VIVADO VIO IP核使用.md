# VIVADO VIO IP核使用

## 概述

VIO（Virtual Input/Output）是 Vivado 提供的**虚拟输入输出 IP 核**，通过 JTAG 接口实时**观察 FPGA 内部信号**并**驱动控制信号**，无需额外物理引脚。

### VIO vs ILA

|      | VIO          | ILA         |
| ---- | ------------ | ----------- |
| 功能   | 实时观察 + 手动驱动  | 采集波形分析      |
| 交互   | 双向（读+写）      | 单向（只读）      |
| 典型场景 | 模拟按键、调参、观察状态 | 抓取时序波形、协议分析 |
| 触发   | 无触发条件        | 可设置复杂触发条件   |

## 工程结构（本笔记配套工程）

```
VIO_Test/
├── build.tcl          ← 创建项目 + VIO IP + 综合实现
├── program.tcl         ← 烧录
├── src/
│   ├── hdl/
│   │   └── Vio_Top.v   ← 顶层（计数器 + VIO）
│   └── constrs/
│       └── cons.xdc    ← 管脚约束
```

设计内容：1Hz 递增 8 位计数器，LED 显示，VIO 观察和操控。

## 添加 VIO IP 核的两种方式

### 方式一：IP Catalog 图形化添加

1. 打开 IP Catalog，搜索 `vio`
2. 双击 `VIO (Virtual Input/Output)`
3. 配置参数：
   - **Input Probe Count**：输入 probe 数量（观察的信号数）
   - **Output Probe Count**：输出 probe 数量（控制的信号数）
   - **Probe Width**：每个 probe 的位宽
   - **Probe Out Init Value**：输出 probe 的初始值
4. 点击 OK 生成 IP
5. 在 `IP Sources` 中找到 `.veo` 例化模板，复制到代码中

### 方式二：Tcl 脚本添加（推荐，便于复现）

```tcl
create_ip -name vio -vendor xilinx.com -library ip -version 3.0 -module_name vio_0
set_property -dict [list \
    CONFIG.C_NUM_PROBE_IN  {1}     \   # 1 个输入 probe
    CONFIG.C_NUM_PROBE_OUT {2}     \   # 2 个输出 probe
    CONFIG.C_PROBE_IN0_WIDTH  {8}  \   # probe_in0 位宽 = 8
    CONFIG.C_PROBE_OUT0_WIDTH {1}  \   # probe_out0 位宽 = 1
    CONFIG.C_PROBE_OUT0_INIT_VAL {0} \ # probe_out0 初值 = 0
    CONFIG.C_PROBE_OUT1_WIDTH {8}  \   # probe_out1 位宽 = 8
    CONFIG.C_PROBE_OUT1_INIT_VAL {0} \ # probe_out1 初值 = 0
] [get_ips vio_0]

generate_target all [get_ips vio_0]
```

关键参数说明：

| 参数 | 含义 |
|------|------|
| `C_NUM_PROBE_IN` | 输入 probe 数量（最多 256） |
| `C_NUM_PROBE_OUT` | 输出 probe 数量（最多 256） |
| `C_PROBE_IN0_WIDTH` | 输入 probe0 的位宽 |
| `C_PROBE_OUT0_WIDTH` | 输出 probe0 的位宽 |
| `C_PROBE_OUT0_INIT_VAL` | 输出 probe0 的初始值 |

## 顶层模块例化 VIO

```verilog
module Vio_Top(
    input       clk,
    input       rst_n,
    output [7:0] led
);

    reg [25:0] div_cnt;
    reg [7:0] counter;

    wire vio_rst;          // VIO 输出：复位计数器
    wire [7:0] led_set;    // VIO 输出：设置计数器值

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            div_cnt <= 0;
            counter <= 0;
        end else if (vio_rst) begin
            div_cnt <= 0;
            counter <= led_set;    // VIO 控制复位后的初值
        end else if (div_cnt == 50000000 - 1) begin
            div_cnt <= 0;
            counter <= counter + 1;
        end else begin
            div_cnt <= div_cnt + 1;
        end
    end

    assign led = counter;

    // VIO IP 例化
    vio_0 u_vio (
        .clk        (clk),
        .probe_in0  (counter),       // 观察：计数器当前值
        .probe_out0 (vio_rst),       // 控制：复位计数器
        .probe_out1 (led_set)        // 控制：设置计数器初值
    );

endmodule
```

### VIO 端口说明

| 端口           | 方向  | 说明                     |
| ------------ | --- | ---------------------- |
| `clk`        | in  | 必须与监测信号同时钟域            |
| `probe_in0`  | in  | 观察信号（输入到 VIO，显示在硬件管理器） |
| `probe_out0` | out | 控制信号（从 VIO 输出，驱动设计）    |
| `probe_out1` | out | 第二个控制信号                |

probe_in 和 probe_out 可以有多个（probe_in0~15, probe_out0~15），受 `C_NUM_PROBE_IN/OUT` 控制。

## 使用流程

### Step 1：创建项目 + 添加 VIO IP

```bash
vivado -mode batch -source build.tcl
```

`build.tcl` 会完成：创建项目 → 创建并配置 VIO IP → 添加源文件 → 综合 → 实现 → 生成 bitstream。

### Step 2：烧录

```bash
vivado -mode batch -source program.tcl
```

或通过 Vivado GUI：Open Hardware Manager → Auto Connect → Program Device。

### Step 3：打开 VIO 调试窗口

硬件管理器连接后：

1. 在 Hardware 窗口选中设备
2. 右键 → **Debug → Add Probes...** 或直接双击 VIO 模块
3. 或者在 Hardware Manager 底部窗口点击 **VIO 标签**（自动出现）

### Step 4：观察和操控信号

VIO 调试窗口中：

- **probe_in**（输入）：实时显示信号值，有 Activity 指示变化
- **probe_out**（输出）：可手动修改值，回车或点击确认即写入 FPGA

常用操作：

| 操作 | 方法 |
|------|------|
| 修改输出值 | 双击 Value 列，输入新值，回车 |
| 切换 bit 值 | 切换 Radio button 或输入 0/1 |
| 观察变化 | Activity 列显示箭头（↑上升 ↓下降 ⇅变化） |
| 修改显示格式 | 右键 probe → Radix → Hex/Dec/Bin |

## 本实验操作练习

### 现象

- LED 按 1Hz 速度递增计数（二进制显示）
- 8 个 LED 从 `0000_0000` 到 `1111_1111` 循环

### VIO 练习

在 VIO 调试窗口中：

1. **观察** `probe_in0` 的值每秒递增
2. **修改** `probe_out0`（vio_rst）为 `1` → 计数器复位到 `led_set` 的值
3. **修改** `probe_out1`（led_set）为 `0xAA`，再 toggle `probe_out0` → 计数器跳转到 `0xAA`
4. **修改**显示格式为 Binary，观察 LED 和 VIO 显示一致

## 注意事项

1. **时钟域**：VIO 的 clk 必须与被监测信号同时钟域
2. **初始化**：probe_out 的初始值由 `C_PROBE_OUTx_INIT_VAL` 决定，可以在 IP 配置时设置
3. **probe_out 是异步的**：从 VIO 输出的控制信号相对设计时钟是异步的，如果需要同步，可以在设计中加两级同步器
4. **综合优化**：被观察的信号不会被综合优化掉（VIO 会自动保持），但建议在代码中使用 `(* mark_debug = "true" *)` 属性确保
5. **重新编译**：修改 probe 配置（增删、改位宽）需要重新综合实现
6. **不影响设计**：去掉 VIO 只影响观察能力，不影响设计逻辑
