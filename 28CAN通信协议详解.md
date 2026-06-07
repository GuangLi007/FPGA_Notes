---
tags:
  - FPGA
  - CAN
  - 通信协议
  - 汽车电子
  - 串行总线
---

## 核心结论

- **CAN** = Controller Area Network，由 Bosch 开发，广泛用于汽车和工业现场
- **差分信号**：CAN_H / CAN_L，抗共模干扰强，支持远距离（~1km）
- **多主总线**：所有节点均可发起通信，通过 ID 仲裁决定优先级
- **帧格式**：标准帧 2.0A（11bit ID），扩展帧 2.0B（29bit ID），CAN FD（最多 64 字节数据）
- **硬件错误处理**：CRC + 错误帧 + 自动重传，可靠性远高于 I²C/SPI/UART
- FPGA 通常**不推荐纯逻辑实现完整 CAN 控制器**，建议用 MCU 或购买 CAN IP 核

---

## 一、CAN 与其他协议对比

详见 [[27I2C通信协议详解#一、四大串行协议对比]]，此处列出 CAN 最突出的特点：

| 特性 | CAN | 与 I²C/SPI 的本质区别 |
|:----:|:---:|:---------------------|
| 差分信号 | CAN_H / CAN_L | 非共地，抗干扰强，距离远 |
| 多主仲裁 | CSMA/CR（载波监听/冲突解决） | 无冲突，优先级低的节点自动退让 |
| 错误处理 | CRC + 错误帧 + 状态转换 | 发生错误时可自动重发，节点可离线 |
| 实时性 | 确定性的消息延迟 | 高优先级消息最坏等待时间可计算 |
| 帧数据量 | 2.0A/B: 8 字节，FD: 64 字节 | 单帧数据少，但可靠性高 |

---

## 二、物理层

### 差分信号

```
CAN 总线状态：
  显性 (Dominant) = 逻辑 0: CAN_H=3.5V, CAN_L=1.5V，差分 ≈ 2V
  隐性 (Recessive) = 逻辑 1: CAN_H=2.5V, CAN_L=2.5V，差分 ≈ 0V
```

```
          显性 (0)        隐性 (1)        显性 (0)
CAN_H ﹉﹉﹍﹍﹍﹍﹍﹍﹉﹉﹉﹉﹍﹍﹍﹍﹍﹍﹉﹉
              ↑ 3.5V         ↑ 2.5V
CAN_L ﹉﹉﹉﹉﹉﹉﹉﹍﹍﹍﹉﹉﹉﹉﹉﹉﹉﹍﹍﹍﹉
              ↑ 1.5V         ↑ 2.5V
```

- **显性覆盖隐性**：只要有一个节点拉显性，总线就是显性（线与逻辑）
- **收发器**：FPGA 的 UART 或 GPIO 不能直接连 CAN 总线，必须用 CAN 收发器芯片

### CAN 收发器

| 芯片 | 速率 | 供电 | 特点 |
|:----:|:----:|:----:|------|
| TJA1050 | 1 Mbps | 5V | 经典，兼容 3.3V MCU |
| SN65HVD230 | 1 Mbps | 3.3V | 3.3V 供电，适合 FPGA 直连 |
| TJA1040 | 1 Mbps | 5V | 低功耗待机 |
| ISO1050 | 1 Mbps | 5V/3.3V | 隔离型 |

```
FPGA                       CAN 收发器                  CAN 总线
┌────────┐                ┌──────────┐                ┌──────┐
│ CAN_TX ├───────────────→│ TXD      │                │      │
│        │                │          ├── CAN_H ───────→│      │
│ CAN_RX │←───────────────│ RXD      │                │ 总线  │
│        │                │          ├── CAN_L ───────→│      │
│  GND   ├───────────────│ GND      │                │      │
└────────┘                └──────────┘                └──────┘
```

### 终端电阻

CAN 总线两端必须各接一个 **120Ω** 电阻，用于匹配阻抗、抑制信号反射。

```
┌──────┐     ┌──────┐     ┌──────┐     ┌──────┐
│节点1 │─────│节点2 │─────│节点3 │─────│节点4 │
└──┬───┘     └──────┘     └──────┘     └──┬───┘
   │                                       │
  120Ω                                   120Ω
   │                                       │
  GND                                     GND
```

---

## 三、CAN 帧格式

### 标准帧（CAN 2.0A）— 11bit ID

```
 ┌─ 1 ─┬─ 11 ─┬─ 1 ─┬─ 1 ─┬─ 1 ─┬─ 4 ─┬─ 0~8 字节 ─┬─ 15 ─┬─ 1 ─┬─ 1 ─┬─ 7 ─┐
 │ SOF │  ID   │ RTR │ IDE │ r0  │ DLC │   Data     │ CRC  │ ACK │ EOF │ IFS  │
 └─────┴───────┴─────┴─────┴─────┴─────┴─────────────┴──────┴─────┴─────┴──────┘
```

| 字段 | 长度 | 说明 |
|:----:|:----:|------|
| SOF | 1 bit | Start of Frame，显性 (0) |
| ID | 11 bit | 标识符，决定优先级（数值越小优先级越高） |
| RTR | 1 bit | Remote Transmission Request（0=数据帧，1=远程帧） |
| IDE | 1 bit | Identifier Extension（0=标准帧） |
| r0 | 1 bit | 保留位 |
| DLC | 4 bit | 数据长度码（0~8） |
| Data | 0~64 bit | 数据（长度由 DLC 指定） |
| CRC | 15 bit | 包含 CRC 分隔符共 16 bit |
| ACK | 2 bit | ACK 槽 + ACK 分隔符 |
| EOF | 7 bit | End of Frame，隐性 (1) |
| IFS | 3 bit | Inter-Frame Space |

### 扩展帧（CAN 2.0B）— 29bit ID

```
 ┌─ 1 ─┬─ 11 ─┬─ 1 ─┬─ 1 ─┬─ 18 ─┬─ 1 ─┬─ 1 ─┬─ 1 ─┬─ 4 ─┬─ 0~8 ─┬─ 15 ─┬─ 1 ─┬─ 1 ─┬─ 7 ─┐
 │ SOF │ Base │ SRR │ IDE │Extend│ r1  │ r0  │ RTR │ DLC │  Data  │ CRC  │ ACK │ EOF │ IFS  │
 └─────┴──────┴─────┴─────┴──────┴─────┴─────┴─────┴─────┴────────┴──────┴─────┴─────┴──────┘
```

| 字段 | 说明 |
|:----:|------|
| SRR | Substitute Remote Request（替代标准帧的 RTR 位） |
| IDE | 1 = 扩展帧 |
| Extend | 18bit 扩展 ID |
| r1, r0 | 保留位 |

### CAN FD 帧

CAN FD (Flexible Data Rate) 在 2.0 基础上增加：
- **最多 64 字节数据**（DLC 编码不同）
- **速率切换**：数据段可用更高比特率（如 5 Mbps），仲裁段仍用原速率

### 远程帧

- RTR = 1：请求对方发送数据
- 无数据段，DLC 为请求的数据长度
- 接收方收到远程帧后，发送对应 ID 的数据帧

---

## 四、仲裁机制（CSMA/CR）

### 原则

- **显性 (0) 覆盖隐性 (1)**：同时发送时，发送 0 的节点获胜
- **ID 即优先级**：ID 数值越小，优先级越高
- **非破坏性仲裁**：失败的节点自动转为接收模式，不破坏总线数据

### 仲裁过程

```
                bit10      bit9      bit8      bit7
节点A ID=0x123:  0         0         0         1     ← 仲裁失败
节点B ID=0x120:  0         0         0         0     ← 仲裁获胜！

物理总线:         0         0         0         0     （显性覆盖隐性）

节点A 发送 1 但总线是 0 → 检测到冲突 → 退出 → 转接收
节点B 继续发送剩余位
```

### 优先级计算

```
ID 范围: 0x000 ~ 0x7FF（标准帧），0x00000000 ~ 0x1FFFFFFF（扩展帧）
最小 ID = 最高优先级

汽车常见优先级示例：
  ID=0x001: 引擎关键数据（最高）
  ID=0x100: 刹车系统
  ID=0x200: 转向角度
  ID=0x500: 车窗控制
  ID=0x7FF: 娱乐信息（最低）
```

---

## 五、错误处理

### 错误检测机制

| 检测方式 | 说明 |
|---------|------|
| Bit Error | 发送时监控总线，读到与自己发送不同的位（仲裁期间除外） |
| Stuff Error | 连续 5 个相同位后必须插入 1 个相反位（位填充） |
| CRC Error | 接收方计算 CRC 与发送方不符 |
| Form Error | 帧格式错误（如 EOF 应为隐性但出现显性） |
| ACK Error | 发送方未收到 ACK（无节点响应） |

### 位填充规则

```
原数据: 11111 00000 1 → 连续 5 个相同位
填充后: 111110 000001 1 → 插入相反位（用下划线标出）
```

接收方收到连续 6 个相同位 → Stuff Error → 发送错误帧。

### 错误状态转换

```
每个节点维护两个计数器：
  TEC (Transmit Error Counter)
  REC (Receive Error Counter)

         TEC>127 或 REC>127
  错误主动 ─────────────────→ 错误被动
      │                          │
      │  TEC≤127 且 REC≤127      │  TEC≤127 且 REC≤127
      └──────────────────────────┘
      
  错误主动: 发送活动错误帧（6 个显性位）
  错误被动: 发送被动错误帧（6 个隐性位）
  总线关闭: TEC>255 → 节点完全断开总线
```

---

## 六、位时序与同步

### CAN 位时间

```
1 bit 时间 = Tq × (Sync_Seg + Prop_Seg + Phase_Seg1 + Phase_Seg2)

                ┌── 1 bit ──┐
          ┌─────┼─────┬─────┼─────┐
          │Sync │ Prop│Ph1  │Ph2  │
          └─────┴─────┴─────┴─────┘
采样点 ────────↑
```

| 段 | 作用 |
|:---:|------|
| Sync_Seg | 同步边沿检测，固定 1 Tq |
| Prop_Seg | 补偿传播延迟 |
| Phase_Seg1 | 采样前相位缓冲 |
| Phase_Seg2 | 采样后相位缓冲 |
| 采样点 | Prop+Ph1 边界，通常在 70%~80% 位时间 |

### 波特率计算

```
例: 50MHz FPGA, 500kbps CAN
  Tq = 50MHz / (8 分频) = 6.25 MHz → 160 ns
  位时间 = 16 Tq = 2.56 μs → 390.625 kbps ← 调整分频得到 500kbps
  
  实际: 50MHz / (10 分频) = 5 MHz → 200 ns
  位时间 = 20 Tq = 4 μs → 250 kbps ← 分频 4: 50/4/20=625k...
  
  50MHz / 分频比 / (1+Prop+Ph1+Ph2) = 目标波特率
```

### 同步

- **硬同步**：SOF 下降沿触发，所有节点对齐 Sync_Seg
- **重同步**：后续边沿若不在 Sync_Seg 内，调整 Ph1/Ph2 宽度（±SJW）

---

## 七、FPGA 实现方案

### 方案 A：外部 CAN 控制器 MCU（推荐）

```
FPGA                  MCU (带 CAN)          CAN 收发器        CAN 总线
┌────────┐   SPI/    ┌───────────┐        ┌──────────┐     ┌──────┐
│ 数据处 ├──UART────→│ STM32    ├───────→│ TJA1050  ├───→│      │
│ 逻辑   │           │ CAN 外设 │        │          │     │ 总线  │
└────────┘           └───────────┘        └──────────┘     └──────┘
```

优势：MCU 的 CAN 外设经过充分验证，FPGA 只需处理应用逻辑。

### 方案 B：FPGA 实现 CAN 控制器 + 外部收发器

```
FPGA                                CAN 收发器        CAN 总线
┌──────────────────────┐           ┌──────────┐     ┌──────┐
│ CAN 控制器 (Verilog)  ├──CAN_TX→│ TJA1050  ├───→│      │
│                      │←─CAN_RX─│          │     │ 总线  │
│ - 位时序 & 同步       │           └──────────┘     └──────┘
│ - 仲裁 & 错误检测     │
│ - CRC 计算            │
│ - FIFO / 过滤         │
└──────────────────────┘
```

### CAN 控制器模块划分

```
CAN_TOP
├── 位时序层 (Bit Timing)
│   ├── Tq 时钟分频
│   ├── 采样点生成
│   └── 同步逻辑 (Sync / Resync)
│
├── 协议控制器 (Protocol Controller)
│   ├── 发送 FSM (SOF → ID → ... → CRC → ACK → EOF)
│   ├── 接收 FSM
│   └── 仲裁逻辑
│
├── 错误管理 (Error Management)
│   ├── TEC / REC 计数器
│   ├── 错误状态转换
│   └── 错误帧生成
│
├── CRC 计算
│   ├── 发送时计算并附加 CRC
│   └── 接收时校验 CRC
│
├── 位填充 (Bit Stuffing)
│   ├── 发送时每 5 个相同位插入 1 个反相位
│   └── 接收时删除填充位
│
├── 验收滤波 (Acceptance Filter)
│   └── 按 ID 过滤，只接收关心的帧
│
└── 接口层
    ├── 发送 FIFO（FPGA 用户 → CAN 总线）
    └── 接收 FIFO（CAN 总线 → FPGA 用户）
```

### Verilog 框架（协议控制器核心 FSM）

```verilog
module can_controller (
    input  wire        clk,          // 系统时钟（如 50MHz）
    input  wire        rst_n,
    // 用户接口
    input  wire        send_req,
    input  wire [10:0] send_id,      // 标准 11bit ID
    input  wire [7:0]  send_data[7:0],
    input  wire [3:0]  send_dlc,
    output reg         send_done,
    // CAN 物理接口
    output reg         can_tx,
    input  wire        can_rx
);
    // 位时序参数（以 500kbps 为例，50MHz/8/12.5=500k）
    localparam TQ_DIV  = 4;          // Tq = 4 clk = 80ns @50MHz
    localparam SYNC    = 1;          // 1 Tq
    localparam PROP    = 3;          // 3 Tq
    localparam PH1     = 4;          // 4 Tq
    localparam PH2     = 4;          // 4 Tq
    localparam BIT_TOTAL = SYNC + PROP + PH1 + PH2;  // = 12 Tq

    reg [3:0] tq_cnt;                // Tq 计数器
    reg [4:0] bit_cnt;               // 当前帧中的 bit 索引
    reg [3:0] state;

    localparam IDLE     = 0;
    localparam SEND_SOF = 1;
    localparam SEND_ID  = 2;
    localparam SEND_RTR = 3;
    localparam SEND_DLC = 4;
    localparam SEND_DAT = 5;
    localparam SEND_CRC = 6;
    localparam SEND_ACK = 7;
    localparam SEND_EOF = 8;
    localparam SEND_IFS = 9;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            state <= IDLE;
            tq_cnt <= 0;
            bit_cnt <= 0;
            can_tx <= 1'b1;  // 隐性（空闲）
        end else case (state)
            IDLE: begin
                if (send_req) begin
                    state <= SEND_SOF;
                end else begin
                    can_tx <= 1'b1;  // 监听状态
                end
            end

            SEND_SOF: begin
                // 发送 SOF = 显性 (0)
                if (tq_cnt == 0) begin
                    can_tx <= 1'b0;
                end
                if (tq_cnt == BIT_TOTAL - 1) begin
                    tq_cnt <= 0;
                    bit_cnt <= 10;  // 11bit ID，从 bit10 开始发
                    state <= SEND_ID;
                end else begin
                    tq_cnt <= tq_cnt + 1;
                end
            end

            SEND_ID: begin
                if (tq_cnt == 0) begin
                    // 采样点：发送当前 bit 并检测仲裁
                    can_tx <= send_id[bit_cnt];
                    if (can_rx != send_id[bit_cnt]) begin
                        // 仲裁失败，转接收模式
                    end
                end
                if (tq_cnt == BIT_TOTAL - 1) begin
                    tq_cnt <= 0;
                    if (bit_cnt == 0) state <= SEND_RTR;
                    else bit_cnt <= bit_cnt - 1;
                end else begin
                    tq_cnt <= tq_cnt + 1;
                end
            end

            // ... 后续状态 SEND_RTR, SEND_DLC, SEND_DAT (含位填充),
            //     SEND_CRC (15bit), SEND_ACK (等 ACK 槽),
            //     SEND_EOF (7 隐性), SEND_IFS
        endcase
    end

    // CRC 计算逻辑（15 位 CRC，生成多项式: x^15 + x^14 + x^10 + x^8 + x^7 + x^4 + x^3 + 1）
    // ...

    // 位填充：连续 5 个相同位时插入反相位
    // ...

    // 错误计数器 TEC / REC
    // ...

endmodule
```

> 完整 CAN 控制器 Verilog 实现约 2000~3000 行，以上仅为框架示意。

---

## 八、开源 CAN 控制器 IP

| IP 名称 | 语言 | 说明 |
|:-------:|:----:|------|
| [SJA1000](https://opencores.org/projects/can) | Verilog | OpenCores 上最流行的 CAN 控制器，兼容 NXP SJA1000 |
| [CAN controller](https://opencores.org/projects/can_controller) | Verilog | 简化版，适合学习 |
| Xilinx CAN | 原语 | Zynq/ZynqMP 自带 CAN 外设，MicroBlaze 可挂 CAN |
| SimpleCAN | Verilog | GitHub 开源，最小实现约 1000 行 |

> FPGA 工程中集成开源 CAN IP + 外部 TJA1050 是最实用的方案。

---

## 九、FPGA 实现 CAN 的注意事项

### 1. 收发器选型

- FPGA 通常 3.3V IO → 选 **SN65HVD230** 或 **TJA1050**（需电平转换）
- 隔离型选 **ISO1050**，适用于工业现场

### 2. 时钟精度

CAN 位时序要求时钟精度 ±1.5% 以内。50MHz 晶振足够满足要求。如果使用内部 OSC，需确认频率误差。

### 3. 仲裁逻辑

仲裁是 CAN 最核心的机制，Verilog 实现时需注意：
- **逐位比较**：发送位与接收位比较，不一致即退出
- **延迟补偿**：发送到接收的环路延迟（包括收发器 + 总线）
- 采样点必须在 Prop_Seg + Phase_Seg1 边界

### 4. 位填充

发送时在连续 5 个相同位后插入反相位，接收时删除。位填充覆盖：SOF ~ CRC 之间所有位（包括 CRC）。

### 5. ACK 等待

发送方在 ACK 槽释放总线（输出隐性），等待任意节点拉显性 ACK。超时未收到 ACK → ACK Error。

### 6. 验收滤波

硬件过滤不需要的帧，减少 FPGA 处理负担。通常实现为：
- 全接受（调试用）
- 单 ID 匹配
- 掩码匹配（ID & Mask == Pattern）

### 7. 与汽车网络的实际差距

FPGA 纯逻辑实现的 CAN 控制器通常只能在**非关键场景**使用（如测试台、数据采集）。车规级 ECU 要求：
- 符合 GMLAN / J1939 等高层协议
- 硬件冗余设计
- 通过 EMC 测试
- 高低温 -40°C ~ 125°C

---

## 十、调试方法

### 工具

| 工具 | 用途 | 价格 |
|:----:|------|:----:|
| CANalyst-II | USB ↔ CAN，配合上位机抓包 | ~200 元 |
| PCAN-USB | 德国 PEAK，稳定可靠 | ~1000 元 |
| Saleae Logic + CAN 解码 | 逻辑分析仪，看位时序 | ~300 元 |
| 示波器（差分探头） | 看 CAN_H/CAN_L 波形 | 看预算 |

### 调试步骤

1. **确认物理层**：示波器量 CAN_H/CAN_L，看差分幅度是否 ~2V
2. **确认终端电阻**：总线两端各 120Ω，断电时量 AH 与 AL 之间 ~60Ω
3. **发单帧测试**：发 ID=0x7FF, DLC=0（无数据），看有无 ACK
4. **逻辑分析仪抓包**：确认 SOF/ID/CRC/EOF 时序正确
5. **环回测试**：FPGA 的 CAN_TX 直接连 CAN_RX（不经收发器），验证协议控制器

### 常见问题

| 现象 | 可能原因 |
|:----|:---------|
| 无 ACK | 总线只有单节点，CAN 协议要求至少 2 节点才能 ACK |
| 总线一直显性 | 收发器损坏 / TXD 一直拉低 / 短路 |
| 间歇性错误帧 | 终端电阻缺失 / 线缆过长 / 波特率不匹配 |
| 节点进入总线关闭 | TEC > 255，检查发送帧格式或总线干扰 |
| 采样点偏移 | 位时序参数配置不当，Prop_Seg 太小 |

---

## 十一、CAN 高层协议

| 协议 | 应用领域 | 说明 |
|:----:|:--------:|------|
| SAE J1939 | 商用车、工程机械 | 基于 CAN 2.0B，29bit ID，250kbps |
| CANopen | 工业自动化 | CiA 标准，对象字典 + PDO/SDO |
| OBD-II | 汽车诊断 | 标准 11bit ID 500kbps，读取故障码 |
| OSEK/VDX | 汽车 OS | 直接运行在 CAN 控制器之上 |
| CANaerospace | 航空电子 | 基于 CAN 的航空总线 |

---

## 十二、与已有笔记的关联

| 笔记 | 关联点 |
|------|--------|
| [[27I2C通信协议详解]] | CAN vs I²C 详细对比，包含四大协议对比表 |
| [[17SPI通信与ADC128S102驱动设计]] | SPI 与 CAN 的同步/异步区别 |
| [[14UART串口发送模块设计与验证]] | UART 与 CAN 的异步通信区别 |
| [[15UART串口接收模块设计与验证]] | 同上 |

---

## 参考

- Bosch CAN 2.0B Specification
- ISO 11898-1: CAN 数据链路层
- ISO 11898-2: CAN 高速物理层
- NXP TJA1050 数据手册
- OpenCores: SJA1000 CAN Controller
-  CAN in Automation (CiA): can-cia.org
