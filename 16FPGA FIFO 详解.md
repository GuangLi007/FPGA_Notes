---
tags:
  - FPGA
  - FIFO
  - 存储
  - 跨时钟域
---

## 核心结论

- **FIFO** = First In First Out，先进先出缓存，无地址线
- **同步 FIFO** = 单时钟域，同进同出
- **异步 FIFO** = 双时钟域，解决跨时钟域数据传递
- **空/满标志**是 FIFO 的核心控制信号，设计关键

---

## 一、什么是 FIFO

FIFO 是一种**无地址线**的存储器件，数据按写入顺序读出：

```
写入侧:            读出侧:
  wdata ─→┌──────┐→ rdata
  wclk   │ FIFO │  rclk
  wr_en  │      │  rd_en
         |满/空  │
         │ 标志  │
         └──────┘
           full empty
```

与 RAM/ROM 的区别：

| 对比 | RAM | ROM | FIFO |
|------|-----|-----|------|
| 地址线 | ✅ 有 | ✅ 有 | ❌ 无 |
| 读写方向 | 可读可写 | 只读 | 可读可写 |
| 数据顺序 | 任意地址 | 任意地址 | **先进先出** |
| 溢出保护 | 无 | — | 空/满标志 |

---

## 二、同步 FIFO vs 异步 FIFO

### 同步 FIFO

| 特性   | 说明                     |
| ---- | ---------------------- |
| 时钟   | 读写共用同一时钟               |
| 实现难度 | 简单（计数器判空满）             |
| 适用场景 | 数据缓冲（如 UART 接收缓存）      |
| 资源   | BRAM 或 Distributed RAM |

### 异步 FIFO

| 特性   | 说明                   |
| ---- | -------------------- |
| 时钟   | 读写不同时钟域              |
| 实现难度 | 较难（需格雷码 + 同步器）       |
| 适用场景 | 跨时钟域数据传递（如 ADC→DDR）  |
| 核心技巧 | 读写指针用**格雷码**编码后跨时钟同步 |

### 对比

| 对比项   | 同步 FIFO | 异步 FIFO    |
| ----- | ------- | ---------- |
| 时钟数   | 1       | 2          |
| 判空满   | 计数器直接减  | 格雷码比较      |
| 亚稳态风险 | 低       | 高（需 2 级同步） |
| 资源消耗  | 少       | 略多（同步器）    |
| 应用    | 数据缓存    | 跨时钟域桥接     |

---

## 三、FIFO 的核心参数

| 参数 | 说明 | 示例 |
|------|------|------|
| **深度 (Depth)** | 最多存多少笔数据 | 1024 |
| **位宽 (Width)** | 每笔数据多少 bit | 8 |
| **满标志 (full)** | FIFO 已满，不能再写 | 高有效 |
| **空标志 (empty)** | FIFO 已空，不能读 | 高有效 |
| **几乎满 (almost_full)** | 即将满，预警 | 可配置阈值 |
| **几乎空 (almost_empty)** | 即将空，预警 | 可配置阈值 |
| **写计数 (wr_count)** | 当前已写入数量 | 调试用 |
| **读计数 (rd_count)** | 当前可读数量 | 调试用 |

---

## 四、同步 FIFO 的 Verilog 实现

```verilog
module sync_fifo #(
    parameter DEPTH = 16,
    parameter WIDTH = 8
) (
    input  wire                clk,
    input  wire                rst_n,
    input  wire                wr_en,
    input  wire [WIDTH-1:0]    wdata,
    input  wire                rd_en,
    output reg  [WIDTH-1:0]    rdata,
    output wire                full,
    output wire                empty
);

localparam AW = $clog2(DEPTH);

reg [WIDTH-1:0] mem [0:DEPTH-1];
reg [AW:0]      wr_ptr;       // 多 1 bit 判满
reg [AW:0]      rd_ptr;

wire [AW-1:0]   waddr = wr_ptr[AW-1:0];
wire [AW-1:0]   raddr = rd_ptr[AW-1:0];

// 写操作
always @(posedge clk) begin
    if (wr_en && !full)
        mem[waddr] <= wdata;
end

// 读操作
always @(posedge clk) begin
    if (rd_en && !empty)
        rdata <= mem[raddr];
end

// 指针递增
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        wr_ptr <= 0;
        rd_ptr <= 0;
    end else begin
        if (wr_en && !full) wr_ptr <= wr_ptr + 1;
        if (rd_en && !empty) rd_ptr <= rd_ptr + 1;
    end
end

// 空满判断
assign full  = (wr_ptr[AW] != rd_ptr[AW]) &&
               (wr_ptr[AW-1:0] == rd_ptr[AW-1:0]);
assign empty = (wr_ptr == rd_ptr);

endmodule
```

### 空满判断原理

```
wr_ptr 比 rd_ptr 多 1 bit（最高位做标志位）：

空: wr_ptr == rd_ptr
满: wr_ptr[AW] != rd_ptr[AW] 且 低位相等
    → 表示 wr_ptr 比 rd_ptr 多跑了一圈
```

---

## 五、Vivado FIFO IP 核使用

**FIFO 有独立 IP 核**（不像 ROM 需要拐弯），直接搜索即可：

```
IP Catalog → FIFO Generator
```

### 配置页详解

**Basic 页：**

| 选项 | 说明 | 推荐值 |
|------|------|--------|
| **Interface Type** | 接口类型 | **Native**（标准 FIFO 接口） |
| | | AXI-Stream（数据流） |
| **FIFO Type** | 类型 | **Common Clock**（同步） |
| | | **Independent Clocks**（异步） |

**Native Port Options 页：**

| 选项 | 说明 | 推荐值 |
|------|------|--------|
| **Read Mode** | 读模式 | **First Word Fall Through**（FWFT，数据提前就绪） |
| | | **Standard**（需 rd_en 才出数据） |
| **Data Port Size → Width** | 位宽 | 8 |
| **Data Port Size → Depth** | 深度 | 1024 |
| **Enable** | 使能引脚 | 勾选则添加 wr_en/rd_en |

**Status Flags 页：**

| 选项 | 说明 |
|------|------|
| **Full Flag** | 满标志 |
| **Empty Flag** | 空标志 |
| **Almost Full / Almost Empty** | 可设置阈值（如 almost_full ≥ 1008） |
| **Data Count** | 当前数据量计数 |

### 例化模板

IP 生成后从 `.veo` 复制：

```verilog
fifo_generator_0 u_fifo (
    .clk        (clk),
    .rst        (~rst_n),
    .din        (wdata),
    .wr_en      (wr_en),
    .rd_en      (rd_en),
    .dout       (rdata),
    .full       (full),
    .empty      (empty),
    .data_count (data_cnt)
);
```

---

## 六、异步 FIFO 核心原理

### 跨时钟域问题

读写时钟不同，直接比较指针会有**亚稳态**风险：

```
写时钟域 → 写指针 → [同步到读时钟域] → 与读指针比较 → empty
读时钟域 → 读指针 → [同步到写时钟域] → 与写指针比较 → full
```

### 格雷码编码

二进制指针跨时钟传输时可能多位同时变化，格雷码**每次只变 1 bit**：

```verilog
// 二进制 → 格雷码
wire [AW:0] wr_ptr_gray;
assign wr_ptr_gray = wr_ptr ^ (wr_ptr >> 1);

// 格雷码同步到读时钟域（2 级触发器）
reg [AW:0] wr_ptr_gray_sync1, wr_ptr_gray_sync2;
always @(posedge rd_clk or negedge rd_rst_n) begin
    if (!rd_rst_n) begin
        wr_ptr_gray_sync1 <= 0;
        wr_ptr_gray_sync2 <= 0;
    end else begin
        wr_ptr_gray_sync1 <= wr_ptr_gray;
        wr_ptr_gray_sync2 <= wr_ptr_gray_sync1;
    end
end
```

Vivado 的 FIFO Generator 选 `Independent Clocks` 就自动处理了这些，一般不需要手写异步 FIFO。

---

## 七、实际应用场景

### UART 接收缓存

```verilog
// UART RX → FIFO → 主控读取
uart_rx u_rx (
    .rx     (uart_rx_line),
    .clk    (clk),
    .data   (rx_data),
    .valid  (rx_valid)
);

fifo_generator_0 u_fifo (
    .clk    (clk),
    .din    (rx_data),
    .wr_en  (rx_valid),
    .rd_en  (read_en),
    .dout   (read_data),
    .empty  (fifo_empty),
    .full   (fifo_full)
);
```

### SPI / ADC 数据采集

```verilog
// ADC 转换结果 (慢时钟域) → FIFO → 处理模块 (快时钟域)
// 异步 FIFO 解决跨时钟域问题
```

### 跨时钟域数据流

| 场景 | FIFO 类型 | 说明 |
|------|----------|------|
| UART 16x 过采样 → 系统时钟 | 同步 | 同频缓存 |
| ADC 10MHz → DDR 200MHz | 异步 | 跨时钟域 |
| 视频像素 → 显示控制器 | 异步 | 不同时钟域 |
| PCIe → 用户逻辑 | 异步 | 大位宽转换 |

### 位宽转换

FIFO Generator 支持**读写位宽不同**（Native 模式下），如：
- 写 8bit × 1024 → 读 16bit × 512
- 常用于串并/并串转换

---

## 八、常见问题

### Q: FIFO 深度怎么选？
**至少大于最坏情况下的突发数据量 × 2。** 如一次突发写 100 笔，深度建议 ≥ 256。

### Q: FIFO 与 RAM 比有什么好处？
**无地址管理，硬件自动判空满。** 用 RAM 实现 FIFO 行为需要额外写地址计数器、读地址计数器、空满判断逻辑，而 FIFO IP 全内置。

### Q: FIFO 与环形缓冲区 (Ring Buffer) 什么关系？
**FIFO = 环形缓冲区的硬件实现。** 本质是同一个东西——都靠读写指针在环状存储空间中移动。区别只是抽象层次：

| 对比 | FIFO (硬件) | 环形缓冲区 (软件) |
|------|-----------|-----------------|
| 载体 | BRAM / LUT 寄存器 | 内存数组 |
| 空满判断 | 硬件自动产生 full/empty | 软件算 `(wptr - rptr) & MASK` |
| 指针溢出 | 硬件自动处理 | 手动 `& (SIZE-1)` 掩码 |
| 使用方式 | 拉信号 wr_en/rd_en | 函数调用 push/pop |

软件实现参考：
```c
// C 环形缓冲区, 与 FIFO 原理完全相同
#define SIZE 256
#define MASK (SIZE-1)
uint8_t buf[SIZE];
uint16_t wptr = 0, rptr = 0;

void push(uint8_t d) {
    if ((wptr - rptr) < SIZE)    // 判满
        buf[wptr++ & MASK] = d;  // 写指针绕回
}

uint8_t pop(void) {
    if (wptr != rptr)            // 判空
        return buf[rptr++ & MASK];
    return 0;
}
```
FPGA 的 FIFO 就是把这段 C 的指针逻辑用硬件电路实现，再多了 full/empty 标志位。

### Q: almost_full / almost_empty 有什么用？
**流水线预警。** 如 almost_full 设 1008（深度 1024），还剩 16 个空位时通知上游暂停，避免溢出。

### Q: FWFT 模式是什么？
**First Word Fall Through** — 数据在 rd_en 有效前就出现在 dout 上，减少读延迟。适合高性能流水线。

### Q: 复位需要几个时钟？
FIFO IP 复位后需至少 **3~5 个时钟** 才能稳定操作，复位后先检查 empty 是否为 1。

### Q: 异步 FIFO 深度最小是多少？
格雷码判空满要求深度**至少为 2**（即 2 个地址位），实际最小一般取 4 或 8。

### Q: 满标志为 1 时还能写吗？
**不能！** 满时写入会**覆盖**已有数据，这是严重的逻辑错误。必须等 full 变低再写。

---

## 九、与已有笔记的关联

| 笔记 | 关联点 |
|------|--------|
| [[14SPI通信与ADC128S102驱动设计]] | SPI 采集数据→FIFO→处理 |
| [[15FPGA ROM 详解]] | ROM+FIFO 构成数据通路 |
| [[13查找表在FPGA中的应用与设计技巧]] | LUT 可做小 FIFO 控制逻辑 |
| 小梅哥 ch16：`ch16_acx720_fifo_ip.rar` | FIFO IP 完整工程例程 |

---

## 参考

- UG901: RAM/ROM/FIFO 推断
- PG057: FIFO Generator (Vivado IP 手册)
- 小梅哥 ch16 FIFO IP 例程
- [[FPGA 跨时钟域设计]]
