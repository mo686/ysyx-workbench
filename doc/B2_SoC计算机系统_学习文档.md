# B2: SoC 计算机系统 — 详细学习文档

## 一、阶段目标

B2 阶段的核心目标是将 NPC 处理器核集成到完整的 SoC (System-on-Chip) 中，对接 ysyxSoC 框架和真实外设。完成后你将：

1. 理解 SoC 的组成要素（CPU + Bus + Memory + Peripherals）
2. 完成从 AXI-Lite 到 AXI4 Full 总线的升级
3. 实现 CPU Wrapper（标准化处理器核对外接口）
4. 对接 ysyxSoC（Flash, PSRAM, SDRAM, UART16550, SPI）
5. 实现两级 Bootloader（Flash → SRAM → SDRAM）
6. 理解多种存储器的地址映射和启动流程
7. 掌握条件编译管理多模式 SoC 设计

---

## 二、B2 与 B1 的承接关系

### 2.1 B1 完成了什么？

B1 阶段 (commits `32fd6d1` → `8dc82bc`) 完成了：
- IFU/LSU 从 DPI-C 瞬时访存 → AXI-Lite 多周期协议
- AXI-Lite Arbiter (IFU 和 LSU 的请求仲裁)
- AXI-Lite Xbar (地址解码，路由到 SRAM/UART/CLINT)
- AXI-Lite UART 和 CLINT 外设实现
- LFSR 随机延迟验证

### 2.2 B2 要解决什么新问题？

| B1 的局限 | B2 的解决 |
|-----------|-----------|
| AXI-Lite 不支持 burst，ICache 填充效率低 | 升级到 AXI4 Full，支持 burst 传输 |
| 内存仍为 DPI-C 模型，不真实 | 对接真实存储器模型 (Flash/PSRAM/SDRAM) |
| 从 RAM 直接启动，不符合硬件现实 | 实现 Bootloader，从 Flash 启动 |
| 顶层是自建的简单互联 | 对接 ysyxSoC 标准框架 |
| 外设只有简化的 UART/CLINT | 对接 UART16550/SPI 等工业标准外设 |
| CPU 接口不标准 | 实现 CPU Wrapper 标准化接口 |


### 2.3 技术演进路线图

```
B1 完成态:                              B2 完成态:
┌──────────────────┐                   ┌──────────────────────────────────────┐
│ IFU ──┐          │                   │ IFU ──┐                              │
│ LSU ──┤          │                   │ LSU ──┤                              │
│       ▼          │                   │       ▼                              │
│  AXI-Lite Arbiter│                   │  AXI4 Interconnect (PULP Demux+Arb) │
│       │          │                   │       │                              │
│  AXI-Lite Xbar   │                   │  ┌────┼────┬─────┬─────┬────┐       │
│   ┌───┼───┐      │                   │  ▼    ▼    ▼     ▼     ▼    ▼       │
│   ▼   ▼   ▼      │       ──→         │ SRAM Flash PSRAM SDRAM UART CLINT   │
│ SRAM UART CLINT  │                   │  (真实RTL存储器模型)                  │
│ (DPI-C内存)      │                   │                                      │
└──────────────────┘                   │  + CPU Wrapper 标准接口              │
                                       │  + 两级 Bootloader                   │
                                       │  + ysyxSoC 框架集成                  │
                                       └──────────────────────────────────────┘
```

---

## 三、SoC 整体架构

### 3.1 最终系统结构

```
┌──────────────────────────────────────────────────────────────────────┐
│                           ysyxSoC                                     │
│                                                                      │
│  ┌────────────────────────────────────────────────────────┐          │
│  │                    NPC CPU Core                         │          │
│  │  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐        │          │
│  │  │ IFU │→│ IDU │→│ EXU │→│ LSU │→│ WBU │        │          │
│  │  └──┬──┘  └─────┘  └─────┘  └──┬──┘  └─────┘        │          │
│  │     │ AXI4 Master               │ AXI4 Master          │          │
│  └─────┼────────────────────────────┼────────────────────┘          │
│        │                            │                                │
│        ▼                            ▼                                │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    AXI4 Interconnect                           │   │
│  │  (PULP Demux + Arbiter + AXI-Lite Bridge)                     │   │
│  └──┬──────────┬──────────┬──────────┬──────────┬───────────────┘   │
│     │          │          │          │          │                    │
│     ▼          ▼          ▼          ▼          ▼                    │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌───────┐  ┌──────┐                │
│  │ SRAM │  │Flash │  │PSRAM │  │UART   │  │CLINT │                │
│  │(8KB) │  │(16MB)│  │(4MB) │  │16550  │  │(Timer)│               │
│  └──────┘  └──────┘  └──────┘  └───────┘  └──────┘                │
│                                                                      │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌───────┐                           │
│  │SDRAM │  │ SPI  │  │ MROM │  │PS2/VGA│                           │
│  │(64MB)│  │      │  │(4KB) │  │       │                           │
│  └──────┘  └──────┘  └──────┘  └───────┘                           │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.2 地址映射

| 地址范围 | 设备 | 大小 | 读写属性 | 用途 |
|----------|------|------|----------|------|
| `0x02000000–0x0200FFFF` | CLINT | 64 KB | R/W | 定时器/中断控制 |
| `0x0F000000–0x0F001FFF` | SRAM | 8 KB | R/W | 高速暂存 + 栈空间 |
| `0x10000000–0x10000FFF` | UART16550 | 4 KB | R/W | 串口通信 |
| `0x10001000–0x10001FFF` | SPI | 4 KB | R/W | Flash 控制器 |
| `0x10011000–0x10011FFF` | PS2 键盘 | 4 KB | R | 键盘输入 |
| `0x20000000–0x20000FFF` | MROM | 4 KB | R | 第一级 Bootloader |
| `0x21000000–0x211FFFFF` | VGA 帧缓冲 | 2 MB | W | 显示输出 |
| `0x30000000–0x3FFFFFFF` | Flash | 16 MB | R | 程序存储 (XIP) |
| `0x80000000–0x803FFFFF` | PSRAM | 4 MB | R/W | 堆空间 |
| `0xA0000000–0xA3FFFFFF` | SDRAM | 64 MB | R/W | 主内存 (程序运行) |

> **注意**：地址映射来自项目实际链接脚本 `linker_ysyxsoc.ld` 和设备头文件 `ysyxsoc.h`。

### 3.3 设备地址宏定义 (来自 `ysyxsoc.h`)

```c
// 串口 UART16550
#define UART16550            0x10000000
#define UART_REG_TX          UART16550 + 0x0   // 发送数据寄存器
#define UART_REG_RX          UART16550 + 0x0   // 接收数据寄存器
#define UART_REG_LC          UART16550 + 0x3   // 行控制寄存器
#define UART_REG_LS          UART16550 + 0x5   // 行状态寄存器
#define UART_REG_DL1         UART16550 + 0x0   // 分频器 LSB (DLAB=1时)
#define UART_REG_DL2         UART16550 + 0x1   // 分频器 MSB (DLAB=1时)

// 定时器 CLINT
#define CLINT                0x02000000         // mtime 低32位

// SPI 控制器
#define SPI                  0x10001000
#define SPI_TX0              SPI + 0x0          // 发送缓冲 [31:0]
#define SPI_TX1              SPI + 0x4          // 发送缓冲 [63:32]
#define SPI_RX0              SPI + 0x0          // 接收缓冲 [31:0]
#define SPI_RX1              SPI + 0x4          // 接收缓冲 [63:32]
#define SPI_CTRL             SPI + 0x10         // 控制寄存器
#define SPI_DIV              SPI + 0x14         // 分频器
#define SPI_SS               SPI + 0x18         // 片选

// PS2 键盘
#define PS2_KBD_ADDR         0x10011000

// VGA
#define VGA_FB_ADDR          0x21000000         // 帧缓冲起始
#define VGA_CTL_ADDR         0x211FFFF0         // 控制寄存器
```


---

## 四、从 AXI-Lite 升级到 AXI4 Full

### 4.1 为什么需要升级？

| 需求 | AXI-Lite 能力 | AXI4 Full 能力 |
|------|--------------|----------------|
| 单次读写 | ✅ | ✅ |
| 突发传输 | ❌ | ✅ (一次传多拍数据) |
| 乱序完成 | ❌ | ✅ (通过 ID 追踪) |
| ICache 填充 | 逐字读取（慢） | 4/8 字 burst（快） |
| 与 SoC 互联 | 不兼容 | ysyxSoC 标准接口 |

**核心动机**: 
1. **ICache 效率**: ICache miss 时需要从内存读取一整个 cache line (如 16 字节/4 字)。AXI-Lite 每次只读 4 字节需要 4 次独立事务；AXI4 用 burst 一次完成。
2. **SoC 兼容**: ysyxSoC 框架使用 AXI4 互联，CPU 必须提供 AXI4 Master 接口。

### 4.2 AXI4 相比 AXI-Lite 新增的信号

| 信号 | 说明 | AXI-Lite 等效值 |
|------|------|-----------------|
| `awid/arid` | 事务 ID (4-bit) | 固定为 0 |
| `awlen/arlen` | 突发长度-1 (8-bit) | 0 (单拍) |
| `awsize/arsize` | 每拍字节数 log2 (3-bit) | 2 (4字节) |
| `awburst/arburst` | 突发类型 (2-bit) | 01 (INCR) |
| `wlast` | 写数据最后一拍 | 始终为 1 |
| `rlast` | 读数据最后一拍 | 始终为 1 |
| `bid/rid` | 响应 ID | 固定为 0 |

### 4.3 升级的本质：接口扩展 + 信号保持

对于不使用 burst 的简单操作（如 LSU 的单字读写），AXI4 的行为与 AXI-Lite 完全相同——只需将新增信号设为默认值：

```verilog
// LSU AXI4 Master — 不使用 burst 时的默认值
assign lsu_axi_awid    = 4'b0001;    // 固定 ID (区分 IFU)
assign lsu_axi_awlen   = 8'd0;       // 单拍传输
assign lsu_axi_awsize  = 3'b010;     // 4 字节
assign lsu_axi_awburst = 2'b01;      // INCR 类型
assign lsu_axi_wlast   = 1'b1;       // 唯一的数据拍即为最后一拍
```

只有 ICache 填充时才真正使用 burst：

```verilog
// ICache miss → 发起 4 字 burst 读取
assign icache_arlen   = 8'd3;        // 4 拍 burst (len = 4-1 = 3)
assign icache_arsize  = 3'b010;      // 每拍 4 字节
assign icache_arburst = 2'b01;       // INCR (地址递增)
// 效果: 一次读取 4×4 = 16 字节 = 1 个 cache line
```

### 4.4 项目中的 AXI4 接口 (来自 `top.v`)

```verilog
// top.v — ysyxSoC 模式下的 AXI4 Master 接口
`ifdef YSYXSOC
  output                  io_master_awvalid,
  output [`CPU_WIDTH-1:0] io_master_awaddr,
  output [           3:0] io_master_awid,
  output [           7:0] io_master_awlen,
  output [           2:0] io_master_awsize,
  output [           1:0] io_master_awburst,
  // ... 完整的 AXI4 五通道信号 ...
`endif
```


---

## 五、CPU Wrapper — 处理器核标准化封装

### 5.1 为什么需要 Wrapper？

在 ysyxSoC 框架中，CPU 核是一个可替换的组件。为了让不同学员设计的 CPU 都能接入同一个 SoC 框架，需要统一的对外接口标准：

```
ysyxSoC 框架要求的接口：
  ┌──────────────────────────────────────┐
  │           cpu_wrapper                 │
  │                                      │
  │  输入: clock, reset, io_interrupt    │
  │                                      │
  │  io_master_* (AXI4 Master 接口)      │  → 连接到 SoC 总线
  │  io_slave_*  (AXI4 Slave 接口)       │  → 连接 DMA (本项目未使用)
  │                                      │
  └──────────────────────────────────────┘
```

### 5.2 项目中的 cpu_wrapper.v

```verilog
module cpu_wrapper (
  input  clock, reset, io_interrupt,
  
  // AXI4 Master 接口 — CPU 发出的读写请求
  input  io_master_awready,  output io_master_awvalid,
  output [`CPU_WIDTH-1:0] io_master_awaddr,
  output [3:0] io_master_awid,  output [7:0] io_master_awlen,
  output [2:0] io_master_awsize, output [1:0] io_master_awburst,
  // ... W, B, AR, R 通道 ...
  
  // AXI4 Slave 接口 — 外部设备对 CPU 的访问 (DMA/调试)
  output io_slave_awready,  input io_slave_awvalid,
  // ... (本项目未实际使用) ...
);
  // 内部例化完整的 CPU 顶层模块
  top u_top (
    .clock(clock), .reset(reset),
    .io_master_awready(io_master_awready),
    .io_master_awvalid(io_master_awvalid),
    // ... 所有 AXI4 信号透传 ...
  );
endmodule
```

### 5.3 封装层次关系

```
ysyxSoCFull (ysyxSoC 顶层，Chisel 生成)
  └── cpu_wrapper (Verilog 封装层)
        └── top (CPU 实际顶层)
              ├── ifu (取指单元 + AXI4 Master)
              ├── idu (译码)
              ├── exu (执行)
              ├── lsu (访存单元 + AXI4 Master)
              ├── wbu (写回)
              ├── bru (分支)
              ├── iru (中断/异常)
              ├── regfile (寄存器堆)
              ├── csrfile (CSR 寄存器)
              ├── axi_demux (IFU 地址分路)
              ├── axi_icache (指令缓存)
              ├── axi_interconnect (AXI4 互联)
              ├── axi_axil_adapter (协议桥)
              └── axi_lite_clint (定时器)
```

### 5.4 独立仿真 vs SoC 模式的接口差异

| 方面 | 独立仿真模式 | SoC 模式 |
|------|-------------|-----------|
| 顶层模块 | `top` | `cpu_wrapper` → `top` |
| 对外接口 | 仅 `clock` + `reset` | 完整 AXI4 Master/Slave |
| 内存/外设 | 内置在 `top` 中 (DPI-C) | 由 ysyxSoC 框架提供 |
| AXI 信号 | 内部互联完成 | 通过端口暴露给 SoC |


---

## 六、两级 Bootloader — 从 Flash 启动

### 6.1 为什么需要 Bootloader？

真实嵌入式系统中，程序存储在 Flash（断电不丢失），但：
- Flash 是只读的（运行时不能写 .data/.bss 段）
- Flash 访问速度比 SRAM/SDRAM 慢得多
- 部分代码需要在 RAM 中执行（性能要求）

因此需要 Bootloader 完成**从 Flash 到 RAM 的代码搬运**。

### 6.2 本项目的启动流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                        启动流程 (时间轴)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Stage 0: CPU 复位                                                  │
│  ┌──────────────────────┐                                          │
│  │ PC = 0x30000000       │  CPU 从 Flash 起始地址取第一条指令        │
│  └──────────┬───────────┘                                          │
│             │                                                       │
│  Stage 1: 第一级 Bootloader (.bl.f 段, 在 Flash 中执行)             │
│  ┌──────────▼───────────┐                                          │
│  │ 将 .bl.s 段从 Flash  │  把第二级代码从 Flash 复制到 SRAM         │
│  │ 复制到 SRAM           │  (因为第二级需要写 SDRAM, 不能在 Flash)   │
│  │ 跳转到 SRAM 继续      │                                          │
│  └──────────┬───────────┘                                          │
│             │                                                       │
│  Stage 2: 第二级 Bootloader (.bl.s 段, 在 SRAM 中执行)             │
│  ┌──────────▼───────────┐                                          │
│  │ 将 .text 从 Flash    │                                          │
│  │ 复制到 SDRAM          │  主程序代码搬运到快速 RAM                 │
│  │ 将 .data 从 Flash    │  已初始化全局变量搬运                     │
│  │ 复制到 SDRAM          │                                          │
│  │ 清零 .bss            │  未初始化全局变量清零                     │
│  │ 设置栈指针 sp         │  栈在 SRAM 中 (最快)                     │
│  │ 跳转 _trm_init       │  进入 C 运行时初始化                     │
│  └──────────┬───────────┘                                          │
│             │                                                       │
│  Stage 3: 主程序在 SDRAM 中运行                                     │
│  ┌──────────▼───────────┐                                          │
│  │ _trm_init()          │                                          │
│  │  → init_uart(115200) │  初始化串口                               │
│  │  → brandShow()       │  显示 CPU 品牌信息                        │
│  │  → main(mainargs)    │  执行用户主程序                           │
│  │  → halt(ret)         │  程序结束                                 │
│  └──────────────────────┘                                          │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.3 为什么是"两级"？

**问题**: 搬运 .text 到 SDRAM 的代码本身不能在 Flash 中执行。因为搬运操作需要**写内存**，而如果搬运代码本身也在被搬运的范围内，就会产生自我覆盖问题。

**解决**: 
1. **第一级** (.bl.f): 极小的搬运代码，将第二级从 Flash 搬到 SRAM。因为第一级不搬运自己，所以安全。
2. **第二级** (.bl.s): 在 SRAM 中运行，负责搬运主程序到 SDRAM。因为它在 SRAM 中，不会被 SDRAM 写操作影响。

### 6.4 启动代码详解 (`start.S`)

```asm
.section .bl.f, "ax"         # 第一级: 放在 Flash 中
_start:
  mv s0, zero                # 初始化

  # === 将 .bl.s 段从 Flash 复制到 SRAM ===
  la a0, _bl_s               # 目标地址 (SRAM 中)
  la a1, _bl_s_load          # 源地址 (Flash 中)
  la a2, _ebl_s
  sub a2, a2, a0             # 大小 = 结束 - 开始
  mv t0, zero
0:
  bgeu t0, a2, 1f            # 复制完毕?
  add t1, a1, t0
  lw t2, 0(t1)               # 从 Flash 读一个字
  add t1, a0, t0
  sw t2, 0(t1)               # 写到 SRAM
  addi t0, t0, 4
  j 0b
1:
  la ra, _do_bl_ss           # 跳转到 SRAM 中的第二级
  jalr ra

.section .bl.s, "ax"         # 第二级: 运行时在 SRAM 中
_do_bl_ss:
  # === 将 .text 从 Flash 复制到 SDRAM ===
  la a0, _text               # 目标: SDRAM (0xa0000000)
  la a1, _text_load          # 源: Flash
  la a2, _etext
  jal _bl_ss_load_align4     # 通用的 4 字节对齐复制函数

  # === 将 .data 从 Flash 复制到 SDRAM ===
  la a0, _data
  la a1, _data_load
  la a2, _edata
  jal _bl_ss_load_align4

  # === 清零 .bss 段 ===
  la a0, _bss_start
  la a2, _ebss
  sub a2, a2, a0
  beqz a2, 1f
  mv t0, zero
0:
  bgeu t0, a2, 1f
  add t1, a0, t0
  sw zero, 0(t1)             # 逐字清零
  addi t0, t0, 4
  j 0b
1:
  # === 设置栈并跳转到 C 代码 ===
  la sp, _stack_pointer      # 栈在 SRAM 中 (0x0F000000 + 8KB)
  la ra, _trm_init           # C 入口
  jalr ra
```


### 6.5 链接脚本详解 (`linker_ysyxsoc.ld`)

链接脚本定义了程序各段在存储器中的**运行地址 (VMA)** 和**加载地址 (LMA)**：

```ld
ENTRY(_start)

MEMORY {
  sram  (rwx) : ORIGIN = 0x0f000000, LENGTH = 8K     # 高速暂存
  mrom  (r x) : ORIGIN = 0x20000000, LENGTH = 4K     # 只读 Boot ROM
  flash (rwx) : ORIGIN = 0x30000000, LENGTH = 16M    # Flash 存储
  psram (rwx) : ORIGIN = 0x80000000, LENGTH = 4M     # PSRAM (堆)
  sdram (rwx) : ORIGIN = 0xa0000000, LENGTH = 64M    # SDRAM (主内存)
}

SECTIONS {
  . = _flash_start;                    # 从 Flash 起始

  .bl.f : { *(.bl.f*) } > flash       # 第一级 BL: 存在 Flash, 也在 Flash 中执行
  
  .bl.s : { *(.bl.s*) }               # 第二级 BL: 运行地址=SRAM, 加载地址=Flash
         > sram AT> flash              # (VMA=SRAM, LMA=Flash)

  .text : { *(.text*) }               # 代码段: 运行地址=SDRAM, 加载地址=Flash
         > sdram AT> flash

  .rodata : { *(.rodata*) } > flash   # 只读数据: 直接放 Flash (不需要搬运)

  .data : { *(.data*) }               # 数据段: 运行地址=SDRAM, 加载地址=Flash
         > sdram AT> flash

  .bss (NOLOAD) : { *(.bss*) }        # BSS段: 只在 SDRAM 中, 不占 Flash 空间
         > sdram

  _heap_start = _psram_start;          # 堆从 PSRAM 开始 (0x80000000)
  
  .stack (NOLOAD) : {                  # 栈空间: 在 SRAM 中 (最快)
    _stack_top = .;
    . += _stack_size;                  # 1KB 栈 (由 Makefile 传入)
    _stack_pointer = .;
  } > sram
}
```

**关键概念 — VMA vs LMA**:
- **VMA (Virtual Memory Address)**: 程序运行时代码/数据的地址（CPU 看到的地址）
- **LMA (Load Memory Address)**: 程序在存储介质中的物理位置
- `> sdram AT> flash` 表示: 代码运行在 SDRAM 中，但加载（存储）在 Flash 中
- Bootloader 的工作就是把数据从 LMA (Flash) 搬到 VMA (SDRAM)

### 6.6 存储器使用分工

```
┌──────────────────────────────────────────────────────────────────┐
│  Flash (0x30000000, 16MB, 只读)                                   │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ .bl.f (第一级 BL)                                             ││
│  │ .bl.s (第二级 BL 的 Flash 副本)                               ││
│  │ .text (主程序代码的 Flash 副本)                                ││
│  │ .rodata (只读数据, 直接从 Flash 读取)                          ││
│  │ .data (初始化数据的 Flash 副本)                                ││
│  └──────────────────────────────────────────────────────────────┘│
├──────────────────────────────────────────────────────────────────┤
│  SRAM (0x0F000000, 8KB, 读写, 最快)                              │
│  ┌─────────────────────────────────┐                             │
│  │ .bl.s 运行地址 (第二级 BL)       │                             │
│  │ 栈 (_stack_pointer, 1KB)        │                             │
│  └─────────────────────────────────┘                             │
├──────────────────────────────────────────────────────────────────┤
│  SDRAM (0xA0000000, 64MB, 读写, 主内存)                          │
│  ┌─────────────────────────────────┐                             │
│  │ .text 运行地址 (主程序代码)       │                             │
│  │ .data 运行地址 (全局变量)         │                             │
│  │ .bss (清零的全局变量)             │                             │
│  └─────────────────────────────────┘                             │
├──────────────────────────────────────────────────────────────────┤
│  PSRAM (0x80000000, 4MB, 读写)                                   │
│  ┌─────────────────────────────────┐                             │
│  │ 堆 (_heap_start → _psram_end)   │   malloc/动态内存           │
│  └─────────────────────────────────┘                             │
└──────────────────────────────────────────────────────────────────┘
```


---

## 七、独立仿真模式 vs SoC 模式

### 7.1 编译时切换

```makefile
# 独立仿真模式 (开发调试用)
make ARCH=riscv32e-npc run

# SoC 集成模式 (对接 ysyxSoC)
make ARCH=riscv32e-ysyxsoc run
```

NPC 的 Makefile 根据 ARCH 选择不同的顶层模块：
```makefile
# npc/Makefile
ifeq ($(ARCH), riscv32e-npc)
  TOPNAME = top              # 独立仿真: CPU + 内置互联 + DPI-C 内存
else ifeq ($(ARCH), riscv32e-ysyxsoc)
  TOPNAME = ysyxSoCFull      # SoC 集成: CPU Wrapper 接入 ysyxSoC
  CXXFLAGS += -DYSYXSOC
  VERILATOR_FLAGS += +define+YSYXSOC
endif
```

### 7.2 条件编译

RTL 中使用 `ifdef YSYXSOC` 区分两种模式的行为差异：

```verilog
// npc/vsrc/top.v
`ifdef YSYXSOC
  // SoC 模式: 暴露 AXI4 Master 接口给外部 SoC
  // IFU 和 LSU 的 AXI 信号连接到顶层端口
  // CLINT 由 SoC 提供 (地址 0x02000000)
  // UART 由 SoC 提供 (UART16550, 地址 0x10000000)
`else
  // 独立仿真模式: 内置完整的 AXI4 互联 + 存储/外设
  // 内置 AXI SRAM (DPI-C 后端)
  // 内置 AXI-Lite UART ($write 输出)
  // 内置 AXI-Lite CLINT (mtime 计数器)
`endif
```

### 7.3 两种模式全面对比

| 方面 | 独立仿真 (`riscv32e-npc`) | SoC 集成 (`riscv32e-ysyxsoc`) |
|------|--------------------------|-------------------------------|
| 顶层模块 | `top` | `ysyxSoCFull` (含 `cpu_wrapper`) |
| 内存后端 | DPI-C 调用主机内存 | Flash/PSRAM/SDRAM RTL 模型 |
| UART | 自建 `axi_lite_uart.v` ($write) | UART16550 (寄存器级精确模型) |
| 定时器 | 自建 `axi_lite_clint.v` | SoC 的 CLINT (地址不同!) |
| UART 地址 | `0xa0000000` (B1遗留) | `0x10000000` (UART16550) |
| Timer 地址 | `0xa0002000` (B1遗留) | `0x02000000` (标准 CLINT) |
| 启动地址 | `0x80000000` (直接从 RAM) | `0x30000000` (从 Flash) |
| Bootloader | 不需要 | 必须 (Flash→SRAM→SDRAM) |
| ICache | 内置在 `top.v` 中 | 内置 (随 CPU 一起进入 SoC) |
| 地址解码 | 自建 `axi_interconnect.v` | SoC 框架 + 内部 Demux |
| DiffTest | 直接支持 | 支持 (需对齐初始状态 + skip 设备访问) |
| 调试便利性 | ⭐⭐⭐ 高 (直接看波形) | ⭐⭐ 中 (SoC 模块多，波形复杂) |
| 硬件真实性 | ⭐⭐ 中 | ⭐⭐⭐ 高 (接近流片状态) |

### 7.4 开发策略

**推荐流程**: 
1. 在独立仿真模式下完成功能开发和调试 (DiffTest + 波形)
2. 功能正确后切换 SoC 模式验证集成
3. SoC 模式下只调试接口适配问题

```bash
# 开发调试阶段: 快速迭代
make ARCH=riscv32e-npc run DIFFTEST=1 WAVE=1

# 集成验证阶段: 确认 SoC 对接
make ARCH=riscv32e-ysyxsoc run
```


---

## 八、AXI4 Interconnect — 完整互联子系统

### 8.1 互联架构 (独立仿真模式)

在独立仿真模式下，`top.v` 内置了完整的 AXI4 互联系统：

```
IFU (AXI4 Master)
  │
  ▼
AXI Demux (PULP axi_demux_intf)
  │             │
  │ port[0]     │ port[1]
  ▼             ▼
ICache      AXI Interconnect (3×2)
  │              ↑ (IFU bypass)  ↑ (LSU)
  │              │               │
  ▼              │               │
ICache AXI ─────┘               │
                                 │
LSU (AXI4 Master) ──────────────┘
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
    AXI SRAM    AXI→AXI-Lite    AXI→AXI-Lite
    (主存)      Bridge           Bridge
                   │                │
                   ▼                ▼
              AXI-Lite          AXI-Lite
               UART              CLINT
```

### 8.2 PULP AXI Demux — IFU 的请求分路

项目使用 PULP Platform 的开源 `axi_demux` IP (SystemVerilog):

```systemverilog
// 地址规则: 根据 PC 地址决定走 ICache 还是直接访问内存
localparam rule_t [NoRules-1:0] addr_map = '{
  '{idx: 32'd1, start_addr: 32'h30000000, end_addr: 32'h40000000},  // Flash → port[1] 直接内存
  '{idx: 32'd0, start_addr: 32'ha0000000, end_addr: 32'hc0000000},  // SDRAM → port[0] 走 ICache
  '{idx: 32'd1, start_addr: 32'h0f000000, end_addr: 32'h10000000}   // SRAM  → port[1] 直接内存
};
```

**设计考量**:
- SDRAM 地址范围的取指经过 ICache (频繁访问，值得缓存)
- Flash/SRAM 地址范围的取指绕过 ICache (启动阶段，只执行一次)

### 8.3 AXI4 → AXI-Lite 桥

CLINT 和 UART 使用 AXI-Lite 接口（简单外设不需要 burst），但 CPU 发出 AXI4 请求。桥接器处理协议降级：

```
CPU (AXI4) → AXI Interconnect → AXI4→AXI-Lite Bridge → CLINT (AXI-Lite)
                                                       → UART  (AXI-Lite)
```

桥接器的核心工作：
- **Burst 展开**: AXI4 的 burst (如 arlen=3) 转换为 4 次独立的 AXI-Lite 事务
- **ID 管理**: AXI-Lite 无 ID，桥接器记住请求 ID 并在响应时回填
- **Last 信号生成**: 将多次 AXI-Lite 响应合并为带 rlast 的 AXI4 响应

项目实现：
```
npc/vsrc/amba/
├── axi_axil_adapter.v        # 顶层桥接器 (读写通道合并)
├── axi_axil_adapter_rd.v     # 读通道适配器 (494 行)
└── axi_axil_adapter_wr.v     # 写通道适配器 (557 行)
```

### 8.4 AXI Access Fault 检测

```verilog
// npc/vsrc/amba/axi_access_fault.v
// 监控 AXI 响应通道的 bresp/rresp:
//   2'b00 (OKAY)   → 正常
//   2'b10 (SLVERR) → Slave 错误
//   2'b11 (DECERR) → 地址解码错误 (访问了不存在的地址)

// 当检测到非 OKAY 响应时:
assign access_fault = (rvalid && rready && rresp != 2'b00) ||
                      (bvalid && bready && bresp != 2'b00);
```

**用途**: `access_fault` 连接到 BRU，可触发处理器异常处理流程。当 CPU 访问不存在的地址时，不会静默挂死，而是产生异常。


---

## 九、UART16550 — 工业标准串口

### 9.1 从简化 UART 到 UART16550

| 方面 | B1 自建 `axi_lite_uart.v` | B2 ysyxSoC 的 UART16550 |
|------|--------------------------|--------------------------|
| 发送 | 直接 $write 输出 | 需要查询发送缓冲状态 |
| 接收 | 不支持 | 支持 (查询接收状态) |
| 波特率 | 无概念 | 需要配置分频器 |
| 中断 | 不支持 | 支持 (本项目未用) |
| 地址 | 0xa0000000 | 0x10000000 |

### 9.2 UART16550 寄存器模型

| 偏移 | 名称 | DLAB=0 | DLAB=1 |
|------|------|--------|--------|
| 0x0 | THR/RBR/DLL | 发送/接收数据 | 分频器低字节 |
| 0x1 | IER/DLM | 中断使能 | 分频器高字节 |
| 0x3 | LCR | 行控制 (含 DLAB 位) | — |
| 0x5 | LSR | 行状态 | — |

> **DLAB (Divisor Latch Access Bit)**: LCR 的 bit[7]。当 DLAB=1 时，偏移 0x0 和 0x1 变为分频器寄存器，用于设置波特率。

### 9.3 初始化代码 (来自 `trm.c`)

```c
void init_uart(uint32_t baud_rate) {
  // 1. 设置 DLAB=1，打开分频器寄存器的访问
  outb(UART_REG_LC, inb(UART_REG_LC) | 0x80);
  
  // 2. 计算分频值: 系统时钟 / (16 × 波特率)
  uint16_t divisor = 50000000 / (16 * baud_rate);  // 50MHz 系统时钟
  outb(UART_REG_DL2, divisor >> 8);    // 高字节
  outb(UART_REG_DL1, divisor);         // 低字节
  
  // 3. 清除 DLAB，恢复正常寄存器访问
  outb(UART_REG_LC, inb(UART_REG_LC) & (~0x80));
}
```

### 9.4 发送字符

```c
void putch(char ch) {
  // 1. 等待发送缓冲区空闲 (LSR bit[5] = THRE)
  while ((inb(UART_REG_LS) & 0x20) == 0);
  
  // 2. 将字符写入发送寄存器
  outb(UART_REG_TX, ch);
}
```

**与 B1 的关键区别**: B1 的 `putch` 直接写一个地址就完成了（$write 即时输出），B2 需要**先查询状态再写入**。这是真实硬件行为——如果 UART 正在发送上一个字符，新字符必须等待。

### 9.5 接收字符

```c
void __am_uart_rx(AM_UART_RX_T *cfg) {
  // 检查 LSR bit[0] = DR (Data Ready)
  if ((inb(UART_REG_LS) & 0x1) == 0x1) {
    cfg->data = inb(UART_REG_RX);  // 有数据，读取
  } else {
    cfg->data = 0xff;               // 无数据
  }
}
```

---

## 十、SPI 控制器 — Flash 访问接口

### 10.1 SPI 的作用

在 ysyxSoC 中，Flash 存储器通过 SPI 接口连接。虽然 Flash 的地址空间映射到 `0x30000000`（可以直接通过 AXI 总线读取，即 XIP 模式），但 SPI 控制器提供了更底层的访问方式。

### 10.2 SPI 控制器初始化

```c
void init_spi(uint32_t spi_clock, uint8_t spi_ss, 
              uint8_t char_len, uint8_t tx_neg, 
              uint8_t rx_neg, uint8_t lsb) {
  // 1. 设置 SPI 时钟分频
  uint16_t divider = 50000000 / (2 * spi_clock) - 1;
  outw(SPI_DIV, divider);
  
  // 2. 设置片选
  outb(SPI_SS, spi_ss);
  
  // 3. 配置控制寄存器
  //    ASS=1: 自动片选
  //    TX_NEG: 发送数据在 SCK 下降沿
  //    RX_NEG: 接收数据在 SCK 上升沿
  //    CHAR_LEN: 传输位数
  uint32_t ctrl = 1<<13 | 0<<12 | lsb<<11 | tx_neg<<10 | rx_neg<<9 | 0<<8 | char_len;
  outl(SPI_CTRL, ctrl);
}
```

### 10.3 SPI 传输操作

```c
// 发送数据
void spi_tx(uint8_t *tx_data, uint8_t len) {
  if (len == 8)       outb(SPI_TX0, *tx_data);
  else if (len == 16) outw(SPI_TX0, *(uint16_t *)tx_data);
  else if (len == 32) outl(SPI_TX0, *(uint32_t *)tx_data);
  else if (len == 64) {
    outl(SPI_TX0, *(uint64_t *)tx_data);
    outl(SPI_TX1, *(uint64_t *)tx_data >> 32);
  }
}

// 启动传输
void spi_tx_start() {
  outl(SPI_CTRL, inl(SPI_CTRL) | 0x100);  // 设置 GO_BSY 位
}

// 接收数据 (等待传输完成)
void spi_rx(uint8_t *rx_data, uint8_t len) {
  while ((inl(SPI_CTRL) & 0x100) == 0x100);  // 等待 GO_BSY 清除
  if (len == 8)       *rx_data = inb(SPI_RX0);
  else if (len == 16) *(uint16_t *)rx_data = inw(SPI_RX0);
  else if (len == 32) *(uint32_t *)rx_data = inl(SPI_RX0);
  else if (len == 64) *(uint64_t *)rx_data = (uint64_t)inl(SPI_RX1)<<32 | inl(SPI_RX0);
}
```


---

## 十一、AM ysyxSoC 平台适配

### 11.1 平台源文件结构

```
abstract-machine/am/src/riscv/ysyxsoc/
├── start.S              # 两级 Bootloader 启动代码
├── trm.c               # TRM 层 (putch/halt/init_uart/init_spi)
├── serial.c            # 串口接收 (UART16550)
├── timer.c             # 定时器 (CLINT mtime)
├── ioe.c              # IOE 框架注册
├── input.c            # PS2 键盘输入
├── gpu.c              # VGA 显示
├── cte.c              # CTE 异常处理
├── trap.S             # 异常入口汇编
├── include/ysyxsoc.h  # 设备地址宏定义
└── libgcc/            # 软件乘除法 (RV32E 无硬件乘除)
    ├── div.S           # 整数除法
    ├── muldi3.S        # 64位乘法
    ├── multi3.c        # 128位乘法
    ├── ashldi3.c       # 64位左移
    └── unused.c        # 占位
```

### 11.2 TRM 初始化流程 (`trm.c`)

```c
void _trm_init() {
#ifndef DIFFTEST_ON
  init_uart(115200);     // 初始化 UART (DiffTest 模式下跳过)
  brandShow();           // 读取 mvendorid/marchid 并输出 CPU 品牌
#endif
  int ret = main(mainargs);  // 执行用户主程序
  halt(ret);                  // ebreak 终止
}
```

**brandShow()** — 读取 CSR 显示 CPU 品牌信息:
```c
void brandShow() {
  uint32_t mvendorid, marchid;
  asm volatile("csrr %0, mvendorid" : "=r"(mvendorid));
  asm volatile("csrr %0, marchid" : "=r"(marchid));
  // 输出 4 字节 vendor ID 作为字符 (如 "YSYX")
  for (int i = 3; i >= 0; i--)
    putch((char)((mvendorid >> i*8) & 0xFF));
  // 输出 marchid 的十进制表示
  // ...
  putch('\n');
}
```

### 11.3 定时器实现 (`timer.c`)

```c
#define CLINT 0x02000000    // 注意: 不同于独立仿真的 0xa0002000!

static uint64_t boot_time = 0;

static uint64_t get_time() {
  uint32_t lo = inl(CLINT);         // 读低32位
  uint32_t hi = inl(CLINT + 4);     // 读高32位
  return ((uint64_t)hi << 32) + lo;
}

void __am_timer_init() {
  boot_time = get_time();            // 记录启动时间
}

void __am_timer_uptime(AM_TIMER_UPTIME_T *uptime) {
  uptime->us = get_time() - boot_time;  // 返回自启动以来的时间
}
```

### 11.4 IOE 设备注册 (`ioe.c`)

```c
// 设备查找表 — 通过 AM_xxx 常量索引
static void *lut[128] = {
  [AM_TIMER_CONFIG] = __am_timer_config,
  [AM_TIMER_RTC]    = __am_timer_rtc,
  [AM_TIMER_UPTIME] = __am_timer_uptime,
  [AM_INPUT_CONFIG] = __am_input_config,
  [AM_INPUT_KEYBRD] = __am_input_keybrd,
  [AM_UART_CONFIG]  = __am_uart_config,
  [AM_UART_RX]      = __am_uart_rx,
  [AM_GPU_CONFIG]   = __am_gpu_config,
  [AM_GPU_STATUS]   = __am_gpu_status,
  [AM_GPU_FBDRAW]   = __am_gpu_fbdraw,
};

bool ioe_init() {
  for (int i = 0; i < LENGTH(lut); i++)
    if (!lut[i]) lut[i] = fail;    // 未注册的设备访问触发 panic
  __am_timer_init();
  __am_gpu_init();
  return true;
}

void ioe_read(int reg, void *buf)  { ((handler_t)lut[reg])(buf); }
void ioe_write(int reg, void *buf) { ((handler_t)lut[reg])(buf); }
```

### 11.5 构建系统 (`platform/ysyxsoc.mk`)

```makefile
# 源文件列表
AM_SRCS := riscv/ysyxsoc/start.S \
           riscv/ysyxsoc/trm.c \
           riscv/ysyxsoc/ioe.c \
           riscv/ysyxsoc/timer.c \
           riscv/ysyxsoc/input.c \
           riscv/ysyxsoc/cte.c \
           riscv/ysyxsoc/trap.S \
           riscv/ysyxsoc/serial.c \
           riscv/ysyxsoc/gpu.c \
           platform/dummy/vme.c \      # VME 不实现
           platform/dummy/mpe.c        # MPE 不实现

# 链接配置
LDFLAGS += -T $(AM_HOME)/scripts/linker_ysyxsoc.ld
LDFLAGS += --defsym=_stack_size=1K     # 栈大小 = 1KB
LDFLAGS += --gc-sections -e _start     # 入口符号

# DiffTest 支持
ifneq ($(DIFFTEST),)
  CFLAGS += -DDIFFTEST_ON              # DiffTest 模式标志
endif

# 构建流程: ELF → 反汇编 → bin
image: $(IMAGE).elf
	$(OBJDUMP) -d $(IMAGE).elf > $(IMAGE).txt
	$(OBJCOPY) -S --set-section-flags .bss=alloc,contents -O binary $(IMAGE).elf $(IMAGE).bin
```


---

## 十二、DiffTest 在 SoC 模式下的适配

### 12.1 挑战

SoC 模式引入了新的 DiffTest 难题：

| 问题 | 原因 | 解决 |
|------|------|------|
| UART 初始化指令不一致 | NPC 的 UART16550 需要初始化，NEMU 不需要 | `#ifndef DIFFTEST_ON` 跳过初始化 |
| 设备地址不同 | NPC 用 0x10000000, NEMU 用不同地址 | DiffTest skip 设备访问 |
| 启动 PC 不同 | NPC 从 0x30000000, NEMU 从 0x80000000 | 对齐 REF 起始 PC |
| Bootloader 期间 | BL 代码在 NEMU 中不存在 | BL 结束后再开始对比 |

### 12.2 解决策略

```c
// trm.c — DiffTest 模式下跳过设备初始化
void _trm_init() {
#ifndef DIFFTEST_ON
  init_uart(115200);   // 真实 UART 初始化 (只在非 DiffTest 时)
  brandShow();         // 读 CSR 显示品牌 (只在非 DiffTest 时)
#endif
  int ret = main(mainargs);
  halt(ret);
}
```

在 DiffTest 模式下：
- 跳过 UART 初始化（避免 NEMU 没有对应行为）
- 跳过 brandShow（CSR 读取在 DiffTest 下有特殊处理）
- Bootloader 完成后再开始逐指令对比

### 12.3 设备访问的 DiffTest Skip

当 NPC 执行对设备地址的访存时，需要调用 `difftest_skip()`：

```verilog
// 在 AXI-Lite CLINT/UART 中:
if (arvalid && !arready && ...) begin
  difftest_skip();       // 通知 DiffTest 框架跳过本次对比
  // ... 正常的设备读操作 ...
end
```

**原因**: NPC 读设备时的行为（AXI 协议多周期）与 NEMU 读设备时的行为（C 函数瞬时返回）天然不一致。跳过这些指令的对比是必要的。

---

## 十三、存储器类型详解

### 13.1 SoC 中的存储层次

```
速度快 ←─────────────────────────────────────────────────→ 速度慢
容量小 ←─────────────────────────────────────────────────→ 容量大

  ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
  │ RegFile│   │  SRAM  │   │ PSRAM  │   │ SDRAM  │   │ Flash  │
  │ (寄存器)│   │ (8KB)  │   │ (4MB)  │   │ (64MB) │   │ (16MB) │
  └────────┘   └────────┘   └────────┘   └────────┘   └────────┘
   1 cycle      1~2 cycles   多 cycles    多 cycles    很慢(XIP)
   16×32-bit    R/W           R/W          R/W         只读
```

### 13.2 各存储器特性

| 存储器 | 类型 | 特性 | 在 SoC 中的用途 |
|--------|------|------|-----------------|
| **MROM** | ROM | 只读, 上电内容固定 | 第一级 Bootloader (可选) |
| **SRAM** | 静态RAM | 读写快, 断电丢失, 小 | 栈 + Bootloader 暂存区 |
| **Flash** | NOR Flash | 只读(XIP), 断电不丢 | 程序存储 + 只读数据 |
| **PSRAM** | 伪静态RAM | 读写, 速度中等 | 堆空间 (动态分配) |
| **SDRAM** | 动态RAM | 读写, 容量大, 需刷新 | 主内存 (代码+数据运行) |

### 13.3 XIP (Execute-In-Place)

Flash 支持 XIP 模式——CPU 可以直接从 Flash 地址取指执行，不需要先复制到 RAM。本项目中 Bootloader 的第一级 (.bl.f) 就是 XIP 执行的：

```
CPU PC = 0x30000000 → AXI 总线 → Flash 控制器 → 返回指令
```

**缺点**: Flash 读取延迟高（可能需要 10+ 个时钟周期），不适合频繁执行的代码。所以主程序需要搬运到 SDRAM。

### 13.4 SDRAM 的特殊性

SDRAM (Synchronous Dynamic RAM) 与 SRAM 的核心区别：
- **需要刷新**: 电容存储电荷会泄露，必须定期读出再写回
- **需要初始化**: 上电后需要配置模式寄存器
- **Bank/Row/Column**: 按 Bank→Row→Column 层次组织
- **行缓冲**: 打开一行后，同行内连续访问很快；跨行需要额外开销

ysyxSoC 框架提供了 SDRAM 控制器 (Chisel 实现)，CPU 只需通过 AXI 接口访问 SDRAM 的地址范围即可，控制器自动处理刷新、初始化等底层操作。


---

## 十四、Git 版本代码演进详解

### 14.1 版本总览

| Commit | 阶段 | 核心变化 | 新增关键内容 |
|--------|------|----------|-------------|
| `5608222` | 初始化 | 引入 ysyxSoC 框架 | Chisel 构建系统 + SoC Verilog |
| `94a3e40` | 初始化 | SoC 环境配置 | 仿真环境适配 |
| `9eeef29` | **v1.0** | MROM + SRAM + Bootloader | +3696/-608 行, AXI4 升级, cpu_wrapper |
| `d51d2d4` | **v2.0** | PSRAM + 两级 Bootloader | 完整启动链路 |
| `960c134` | **v3.0** | SDRAM | 大容量主内存 |
| `0a781ec` | **完成** | SoC 系统联调 | AM ysyxSoC 平台全部就位 |

### 14.2 v1.0 (`9eeef29`) — 核心架构升级

**这是 B2 最重要的 commit** (+3696/-608 行)，完成了多项关键改造：

**1. IFU/LSU 从 AXI-Lite 升级到 AXI4 Full**:
- 新增 `awid/arid/awlen/arlen/awsize/arsize/awburst/arburst/wlast/rlast/bid/rid` 信号
- 对于不用 burst 的场景（LSU 单字读写），新增信号设为默认值
- ICache 可以利用 burst 一次读取整个 cache line

**2. 新增 `cpu_wrapper.v`**:
- 封装 `top.v` 为标准 SoC 接口
- 增加 `io_slave_*` 端口（DMA 预留，未实际连接）
- 增加 `io_interrupt` 输入（中断预留）

**3. 新增/升级 AXI 互联模块**:
```
新增:
  npc/vsrc/amba/axi_interconnect.v          (972 行, AXI4 完整互联)
  npc/vsrc/amba/axi_interconnect_wrap_3x2.v (3 Master × 2 Slave)
  npc/vsrc/amba/axi_interconnect_wrap_3x3.v (3 Master × 3 Slave)
  npc/vsrc/amba/axi_axil_adapter.v          (AXI4→AXI-Lite 桥)
  npc/vsrc/amba/axi_axil_adapter_rd.v       (读通道适配, 494 行)
  npc/vsrc/amba/axi_axil_adapter_wr.v       (写通道适配, 557 行)
  npc/vsrc/peripheral/axi_sram.v            (AXI4 接口 SRAM)

升级:
  npc/vsrc/top.v                            (+ifdef YSYXSOC 条件编译)
  npc/vsrc/ifu.v                            (AXI-Lite → AXI4 信号)
  npc/vsrc/lsu.v                            (AXI-Lite → AXI4 信号)
```

**4. 目录重组**:
```
之前:                          之后:
  npc/vsrc/mem/                  npc/vsrc/amba/        (总线相关)
    ├── axi_lite_arbiter.v       npc/vsrc/peripheral/  (外设+存储)
    ├── axi_lite_sram.v          npc/vsrc/deprecated/  (废弃代码)
    └── ...
```

### 14.3 v2.0 (`d51d2d4`) — PSRAM + 两级 Bootloader

- 添加 PSRAM DPI-C 内存模型（仿真用）
- 实现完整的两级 Bootloader (`start.S`)
- 修改 AM 链接脚本 (`linker_ysyxsoc.ld`)
- 验证: Bootloader 成功将程序从 Flash 搬到 PSRAM

### 14.4 v3.0 (`960c134`) — SDRAM

- 添加 SDRAM 控制器（ysyxSoC 框架提供）
- 链接脚本中 .text/.data 目标地址改为 SDRAM (0xa0000000)
- 验证: 程序在 SDRAM 中正确运行

### 14.5 完成版 (`0a781ec`) — 全系统联调

- AM ysyxSoC 平台所有源文件就位
- UART16550 初始化和收发正常
- CLINT 定时器工作正常
- cpu-tests 全部通过
- hello 程序正确输出

### 14.6 查看各版本代码

```bash
# v1.0: AXI4 升级 + CPU Wrapper
git show 9eeef29:npc/vsrc/cpu_wrapper.v
git show 9eeef29:npc/vsrc/top.v
git show 9eeef29:npc/vsrc/amba/axi_interconnect.v

# v2.0: 两级 Bootloader
git show d51d2d4:abstract-machine/am/src/riscv/ysyxsoc/start.S
git show d51d2d4:abstract-machine/scripts/linker_ysyxsoc.ld

# 完成版: 全部平台代码
git show 0a781ec:abstract-machine/am/src/riscv/ysyxsoc/trm.c
git show 0a781ec:abstract-machine/am/src/riscv/ysyxsoc/include/ysyxsoc.h
```


---

## 十五、详解：AXI4 Full 互联实现

### 15.1 `axi_interconnect.v` (972 行) 架构

项目采用 Alex Forencich 的开源 Verilog-AXI 库实现互联：

```
输入 (Master 端口):
  S00: IFU (通过 ICache 或 Demux 之后)
  S01: LSU
  S02: IFU (ICache miss 的直接内存路径)

输出 (Slave 端口):
  M00: 主存储器 (SRAM / ysyxSoC 外部存储)
  M01: AXI-Lite 桥 → CLINT + UART

内部:
  - 每个 Master 到每个 Slave 有独立的仲裁通道
  - 地址解码决定请求路由
  - 写通道和读通道独立仲裁
```

### 15.2 互联参数配置

```verilog
// axi_interconnect_wrap_3x2.v (项目实际使用的配置)
parameter S_COUNT = 3,          // 3 个 Master 输入
parameter M_COUNT = 2,          // 2 个 Slave 输出
parameter DATA_WIDTH = 32,      // 数据宽度 32-bit
parameter ADDR_WIDTH = 32,      // 地址宽度 32-bit
parameter ID_WIDTH = 4,         // ID 宽度 4-bit
parameter M_BASE_ADDR = {32'ha0000000, 32'h80000000},  // Slave 基地址
parameter M_ADDR_WIDTH = {32'd29, 32'd29}              // 地址空间大小
```

### 15.3 与 PULP AXI Demux 的配合

项目同时使用了两种不同来源的开源 IP:
- **PULP Platform 的 `axi_demux`** (SystemVerilog): 用于 IFU 的地址分路（ICache vs 直接内存）
- **Alex Forencich 的 `axi_interconnect`** (Verilog): 用于多 Master 到多 Slave 的交叉连接

这种组合提供了最大灵活性：Demux 在 IFU 出口处做第一层路由，Interconnect 做全局的交叉连接。

---

## 十六、ysyxSoC 框架介绍

### 16.1 框架来源

ysyxSoC 是"一生一芯"项目提供的标准 SoC 框架，使用 Chisel (Scala) 编写，通过 `mill` 构建工具生成 Verilog。

### 16.2 框架提供的组件

| 组件 | 实现方式 | 说明 |
|------|----------|------|
| AXI Crossbar | Chisel | 总线互联 |
| Flash 控制器 | Chisel + DPI-C | 模拟 SPI NOR Flash |
| PSRAM 控制器 | Chisel + DPI-C | 模拟伪静态 RAM |
| SDRAM 控制器 | Chisel + DPI-C | 模拟同步动态 RAM |
| UART16550 | Chisel | 寄存器级精确模型 |
| SPI Master | Chisel | 通用 SPI 控制器 |
| GPIO | Chisel | 通用输入输出 |
| CLINT | Chisel | 定时器/中断 |
| MROM | Chisel | 只读启动 ROM |

### 16.3 CPU 接入要求

ysyxSoC 对 CPU 核的要求非常明确：

1. **模块名**: `cpu_wrapper`
2. **接口**: 标准 AXI4 Master (`io_master_*`) + AXI4 Slave (`io_slave_*`)
3. **时钟/复位**: `clock` (上升沿) + `reset` (高有效)
4. **中断**: `io_interrupt` (暂未使用)
5. **复位后 PC**: 由 SoC 配置（通常为 Flash 起始 0x30000000）

只要满足这些要求，任何设计的 CPU 都可以接入 ysyxSoC。

### 16.4 仿真流程

```bash
# 1. 交叉编译用户程序 (生成 .bin)
cd am-kernels/tests/cpu-tests
make ARCH=riscv32e-ysyxsoc image

# 2. Verilator 编译 SoC (包含 cpu_wrapper + ysyxSoC)
cd npc
make ARCH=riscv32e-ysyxsoc

# 3. 运行仿真 (将 .bin 加载到 Flash 模型)
make ARCH=riscv32e-ysyxsoc run IMAGE=path/to/test.bin
```


---

## 十七、RV32E 的软件乘除法

### 17.1 为什么需要 libgcc？

本项目 CPU 采用 RV32E 基本指令集（**不含 M 扩展**），意味着硬件中**没有乘除法器**。编译器生成的乘除法操作需要由软件库实现：

```makefile
# riscv32e-ysyxsoc.mk
AM_SRCS += riscv/ysyxsoc/libgcc/div.S \       # 整数除法 (汇编实现)
           riscv/ysyxsoc/libgcc/muldi3.S \     # 64位乘法 (汇编实现)
           riscv/ysyxsoc/libgcc/multi3.c \     # 128位乘法
           riscv/ysyxsoc/libgcc/ashldi3.c \    # 64位左移
           riscv/ysyxsoc/libgcc/unused.c       # 占位
```

### 17.2 编译选项

```makefile
COMMON_CFLAGS += -march=rv32e_zicsr -mabi=ilp32e
```

- `rv32e`: 16 个通用寄存器 (x0~x15), 无 M 扩展
- `_zicsr`: 支持 CSR 指令 (csrrw/csrrs/csrrc)
- `ilp32e`: 对应的 ABI (int=32bit, long=32bit, pointer=32bit, 16 regs)

### 17.3 对性能的影响

软件乘除法比硬件慢 10-30 倍：
- 硬件乘法: 1-3 个时钟周期
- 软件乘法: 约 30-50 个时钟周期 (循环累加)
- 硬件除法: 1-32 个时钟周期
- 软件除法: 约 100+ 个时钟周期 (试商法)

这是 RV32E（面积优化）与 RV32IM（性能优化）的 trade-off。

---

## 十八、常见问题与解决

### Q1: SoC 模式下程序不启动 / 卡在 Bootloader

**检查清单**:
1. ✅ Flash 中是否正确加载了 .bin 镜像？(检查仿真环境的 Flash 模型)
2. ✅ CPU 复位 PC 是否为 `0x30000000`？(检查 `defines.vh` 中的 `RESET_PC`)
3. ✅ 链接脚本中 `.bl.f` 是否在 Flash 起始？(检查 `linker_ysyxsoc.ld`)
4. ✅ SRAM 地址 (0x0F000000) 是否在 SoC 地址映射中？
5. ✅ 第一级 BL 的 `sw` 指令能否写入 SRAM？(确认 SRAM 是 R/W)

**调试方法**:
```bash
# 查看程序 ELF 段分布
riscv64-linux-gnu-objdump -h xxx.elf

# 查看反汇编
riscv64-linux-gnu-objdump -d xxx.elf | head -100

# 波形观察 PC 是否在预期地址范围
make run WAVE=1 MAXCYCLE=1000
```

### Q2: 从 AXI-Lite 升级到 AXI4 后 DiffTest 失败

**原因**: AXI4 的 burst 改变了访存时序，但功能行为应该不变。
**检查**:
1. `wlast` 是否在正确的位置拉高？(单拍传输必须为 1)
2. `arlen/awlen` 对于非 burst 操作是否为 0？
3. ID 信号是否正确传递？(Interconnect 可能修改 ID)
4. DiffTest 是否在指令 commit 后才对比？

### Q3: UART 无输出 / 输出乱码

**检查**:
1. ✅ UART 初始化波特率配置是否正确？
2. ✅ DLAB 位设置/清除顺序是否正确？
3. ✅ 发送前是否查询了 LSR 状态？
4. ✅ 地址是否使用了 `0x10000000` (不是 B1 的 `0xa0000000`)？

### Q4: 定时器读取值为 0 或不增长

**检查**:
1. ✅ CLINT 地址是否为 `0x02000000` (不是 B1 的 `0xa0002000`)？
2. ✅ SoC 的 CLINT 模块是否正常工作？
3. ✅ 读低32位和高32位的偏移是否正确？(低=+0, 高=+4)

### Q5: Bootloader 复制到 SDRAM 后程序 crash

**可能原因**:
- SDRAM 控制器未正确初始化
- 复制长度计算错误 (_etext - _text ≠ 实际代码大小)
- .bss 清零范围不对
- 栈指针设置不在有效 SRAM 范围内

### Q6: ICache + SoC 模式下取指异常

**原因**: ICache 缓存了 Flash 地址的指令，但程序已经搬到 SDRAM。
**解决**: Bootloader 跳转到 SDRAM 前执行 `fence.i` 清空 ICache。

---

## 十九、与后续阶段的关系

### 19.1 B2 为 B3 (时序分析) 提供的基础

- AXI4 互联的关键路径需要时序优化
- Spill Register 插入点在 AXI Demux 中已预留 (`SPILL_AW`, `SPILL_AR` 参数)
- CPU Wrapper 的边界清晰，便于 STA 分析

### 19.2 B2 为 B4 (ICache) 提供的基础

- AXI4 burst 支持是 ICache 高效填充的前提
- AXI Demux 在 IFU 出口做 ICache/直接内存的分路
- `fence.i` 后 ICache 需要无效化已有内容

### 19.3 B2 为 B5 (流水线) 提供的基础

- Valid/Ready 弹性握手已贯穿全设计
- 多周期 AXI 访存自然产生流水线 stall
- IFU 和 LSU 的独立 AXI Master 支持并行访问

---

## 二十、PA 讲义相关思考题

### Q: 为什么真实的计算机系统需要多种不同类型的存储器？

**回答**:
这是**速度-容量-成本三角**的体现：
- **SRAM**: 最快 (1-2周期), 但每bit成本高、面积大 → 只能做小容量 (几KB)
- **SDRAM**: 速度中等, 容量/成本比高 → 做主内存 (MB~GB级)
- **Flash**: 断电不丢失, 但只能读 → 做程序存储
- **寄存器堆**: 最快, 但数量极少 (16~32个)

SoC 设计就是在这些存储器之间建立合适的层次结构，用总线将它们连接起来。

---

### Q: Bootloader 和操作系统 Loader 有什么区别？

**回答**:
| 方面 | Bootloader (BL) | OS Loader |
|------|-----------------|-----------|
| 运行环境 | 裸金属, 无 OS | OS 内部 |
| 功能 | 代码搬运, 硬件初始化 | 加载 ELF, 设置虚拟内存 |
| 程序格式 | 纯二进制 (.bin) | ELF (含符号/段信息) |
| 地址计算 | 链接时固定 | 运行时重定位 |
| 本项目 | `start.S` | 不涉及 (裸金属系统) |

---

### Q: 为什么 .rodata 可以直接放在 Flash 中不搬运？

**回答**:
- `.rodata` 是只读数据 (如字符串常量 "Hello World")
- 只读意味着运行时不需要修改，所以不需要搬到 RAM
- CPU 读取时直接通过 AXI 总线从 Flash 地址读取即可 (XIP)
- 这节省了 RAM 空间和 Bootloader 搬运时间
- 但代价是: 每次读取 .rodata 都是 Flash 访问 (慢于 RAM)

---

### Q: 条件编译 (`ifdef YSYXSOC`) 的设计哲学是什么？

**回答**:
1. **一份代码维护两种模式** — 避免代码分叉
2. **开发期用独立模式** — DiffTest 方便, 波形简洁
3. **验证期用 SoC 模式** — 贴近真实硬件
4. **差异最小化** — 大部分 CPU 逻辑在两种模式下相同
5. **差异局部化** — 只有 `top.v` 的最外层互联有条件编译

---

## 二十一、学习建议

1. **先理解 B1 的 AXI-Lite 再学 B2** — AXI4 只是 AXI-Lite 的超集
2. **理解 VMA vs LMA** — 这是 Bootloader 设计的核心概念
3. **画完整的地址空间图** — SoC 设计中地址映射是最容易出错的地方
4. **从简单到复杂** — 先让 Bootloader + SRAM 工作，再加入 Flash/PSRAM/SDRAM
5. **善用条件编译** — 在独立模式下调通功能，再切 SoC 模式
6. **DiffTest 是你的安全网** — SoC 模式下也要保持 DiffTest 可用
7. **阅读开源 IP 文档** — PULP AXI 和 Verilog-AXI 库都有详细文档

---

## 二十二、RTL 源文件清单

### 22.1 B2 阶段新增/修改的 RTL 文件

```
npc/vsrc/
├── cpu_wrapper.v              ← 新增: SoC 标准封装
├── top.v                      ← 修改: 添加 ifdef YSYXSOC + AXI4 端口
├── ifu.v                      ← 修改: AXI-Lite → AXI4 信号
├── lsu.v                      ← 修改: AXI-Lite → AXI4 信号
├── amba/
│   ├── axi_interconnect.v     ← 新增: AXI4 完整互联 (972行)
│   ├── axi_interconnect_wrap_3x2.v  ← 新增
│   ├── axi_interconnect_wrap_3x3.v  ← 新增
│   ├── axi_axil_adapter.v    ← 新增: AXI4→AXI-Lite 桥
│   ├── axi_axil_adapter_rd.v ← 新增: 读通道 (494行)
│   ├── axi_axil_adapter_wr.v ← 新增: 写通道 (557行)
│   ├── axi_demux.sv          ← 新增: PULP AXI Demux
│   ├── axi_demux_simple.sv   ← 新增
│   ├── axi_intf.sv           ← 新增: SV interface 定义
│   └── axi_delayer.sv        ← 新增: AXI 延迟器 (调试用)
├── peripheral/
│   └── axi_sram.v            ← 新增: AXI4 接口 SRAM
└── libs/
    ├── addr_decode.sv         ← 新增: 地址解码器
    ├── addr_decode_dync.sv    ← 新增: 动态地址解码
    ├── rr_arb_tree.sv         ← 新增: Round-Robin 仲裁树
    ├── spill_register.sv      ← 新增: 流水线寄存器 (时序优化)
    └── spill_register_flushable.sv  ← 新增
```

### 22.2 B2 阶段新增的 AM 平台文件

```
abstract-machine/
├── scripts/
│   ├── linker_ysyxsoc.ld              ← 新增: ysyxSoC 链接脚本
│   ├── riscv32e-ysyxsoc.mk           ← 新增: 架构配置
│   └── platform/ysyxsoc.mk           ← 新增: 平台构建规则
└── am/src/riscv/ysyxsoc/
    ├── start.S                         ← 新增: 两级 Bootloader
    ├── trm.c                           ← 新增: TRM (含 UART/SPI 初始化)
    ├── serial.c                        ← 新增: UART16550 收发
    ├── timer.c                         ← 新增: CLINT 定时器
    ├── ioe.c                           ← 新增: IOE 框架
    ├── input.c                         ← 新增: PS2 键盘
    ├── gpu.c                           ← 新增: VGA 显示
    ├── cte.c                           ← 新增: CTE 异常处理
    ├── trap.S                          ← 新增: 异常入口
    ├── include/ysyxsoc.h              ← 新增: 设备地址定义
    └── libgcc/                         ← 新增: 软件乘除法
        ├── div.S
        ├── muldi3.S
        ├── multi3.c
        ├── ashldi3.c
        └── unused.c
```

---

*参考资料：*
- *AMBA AXI Protocol Specification (ARM IHI 0022)*
- *ysyxSoC 官方文档: https://ysyx.oscc.cc*
- *PULP Platform AXI IP: https://github.com/pulp-platform/axi*
- *Verilog-AXI 开源库: https://github.com/alexforencich/verilog-axi*
- *项目源码: npc/vsrc/amba/, npc/vsrc/peripheral/, npc/vsrc/cpu_wrapper.v*
- *AM 平台代码: abstract-machine/am/src/riscv/ysyxsoc/*
- *链接脚本: abstract-machine/scripts/linker_ysyxsoc.ld*
