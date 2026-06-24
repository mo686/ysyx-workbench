# D4: 用 RTL 实现迷你 RISC-V 处理器 — 详细学习文档

## 一、阶段目标

D4 阶段的核心目标是用 Verilog 硬件描述语言实现一个能运行 RISC-V 程序的处理器核（NPC），并通过 Verilator 仿真验证其正确性。完成后你将：

1. 掌握 Verilog 数字电路设计基础 (组合逻辑 + 时序逻辑)
2. 理解处理器数据通路的设计 (取指→译码→执行→访存→写回)
3. 实现一个可运行 `dummy` 测试的最小处理器
4. 掌握 Verilator 仿真环境搭建 (Verilog → C++ 仿真)
5. 理解 DPI-C 接口如何打通硬件与软件世界

---

## 二、设计概述

### 2.1 从 NEMU 到 NPC 的转变

| 方面 | NEMU (软件模拟) | NPC (硬件 RTL) |
|------|----------------|----------------|
| 实现语言 | C | Verilog/SystemVerilog |
| 执行方式 | 解释执行 (逐条翻译) | 并行电路 (组合逻辑计算) |
| 时间概念 | 无时钟 | 有时钟周期 (posedge clock) |
| 调试方式 | printf / GDB | 波形 (VCD / GTKWave) |
| 验证方式 | DiffTest vs Spike | DiffTest vs NEMU (.so) |
| 内存模型 | C 数组 (pmem[]) | DPI-C 回调 / AXI 总线 |

### 2.2 NPC 处理器参数

```verilog
// npc/vsrc/include/defines.vh
`define CPU_WIDTH 32        // 数据宽度 32 位
`define INS_WIDTH 32        // 指令宽度 32 位
`define REG_ADDRW 5         // 寄存器地址位宽 (32 个寄存器)
`define RESET_PC  32'h80000000  // PC 复位值
```

---

## 三、数据通路设计

### 3.1 单周期处理器结构

```
                        ┌─────────────────────────────────────────────────────┐
                        │                     top.v                            │
    clock ──────────────┤                                                     │
    reset ──────────────┤                                                     │
                        │                                                     │
                        │   ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐         │
                PC ───▶│   │ IFU │──▶│ IDU │──▶│ EXU │──▶│ LSU │──▶ WBU    │
                ◀───────│   │取指  │    │译码  │    │执行  │    │访存  │   写回   │
                  BRU   │   └─────┘    └──┬──┘    └─────┘    └─────┘         │
                        │                  │                                   │
                        │            ┌─────▼─────┐                            │
                        │            │ RegFile   │                            │
                        │            │ CSRFile   │                            │
                        │            └───────────┘                            │
                        └─────────────────────────────────────────────────────┘
```

### 3.2 各模块信号连接 (来自 `top.v`)

```verilog
// 核心数据通路信号
wire [`INS_WIDTH-1:0] instr;          // IFU → IDU: 取出的指令
wire [`CPU_WIDTH-1:0] pc;             // BRU → IFU: 当前 PC
wire [`REG_ADDRW-1:0] rs1id, rs2id;   // IDU → RegFile: 源寄存器编号
wire [`CPU_WIDTH-1:0] rs1, rs2, imm;  // RegFile/IDU → EXU: 操作数
wire [`EXU_OPT_WIDTH-1:0] exu_opt;    // IDU → EXU: 运算类型
wire [`EXU_SEL_WIDTH-1:0] exu_src_sel;// IDU → EXU: 操作数来源选择
wire [`CPU_WIDTH-1:0] exu_res;        // EXU → LSU/WBU: 运算结果
wire [`CPU_WIDTH-1:0] lsu_res;        // LSU → WBU: 内存读取结果
wire [`CPU_WIDTH-1:0] rd;             // WBU → RegFile: 写回数据
wire brch, jal, jalr;                 // IDU → BRU: 分支/跳转控制
wire zero;                            // EXU → BRU: 比较结果
```

---

## 四、寄存器堆 (RegFile)

### 4.1 设计要点

```verilog
// npc/vsrc/regfile.v
module regfile (
  input                   i_clk,
  input                   i_wen,       // 写使能
  input  [`REG_ADDRW-1:0] i_waddr,    // 写地址 (rd)
  input  [`CPU_WIDTH-1:0] i_wdata,    // 写数据
  input  [`REG_ADDRW-1:0] i_raddr1,   // 读地址 1 (rs1)
  input  [`REG_ADDRW-1:0] i_raddr2,   // 读地址 2 (rs2)
  output [`CPU_WIDTH-1:0] o_rdata1,   // 读数据 1
  output [`CPU_WIDTH-1:0] o_rdata2,   // 读数据 2
  output                  s_a0zero    // a0==0 标志 (仿真用)
);
  reg [`CPU_WIDTH-1:0] rf[`REG_COUNT-1:0];  // 32 个 32-bit 寄存器

  // 写操作 (时钟上升沿)
  always @(posedge i_clk) begin
    if (i_wen) begin
      if (i_waddr == 0) rf[i_waddr] <= 0;   // x0 始终为 0
      else rf[i_waddr] <= i_wdata;
    end
  end

  // 读操作 (组合逻辑，无延迟)
  assign o_rdata1 = rf[i_raddr1];
  assign o_rdata2 = rf[i_raddr2];

  // 仿真辅助: 判断 a0 是否为 0 (用于 GOOD/BAD TRAP)
  assign s_a0zero = ~|rf[10];
endmodule
```

### 4.2 关键设计决策

| 问题 | 决策 | 原因 |
|------|------|------|
| 读端口数量 | 2 个 | 同时读 rs1 和 rs2 |
| 写端口数量 | 1 个 | 单发射每周期最多写一个 rd |
| x0 写入处理 | 写 0 而非忽略 | 硬件简单，等效于不可写 |
| 读操作类型 | 组合逻辑 | 同周期读取，无延迟 |
| 写操作类型 | 时序逻辑 | 上升沿写入，避免竞争 |

### 4.3 DPI-C 接口 (仿真用)

```verilog
// 将寄存器数组暴露给 C++ testbench
import "DPI-C" function void set_reg_ptr(input bit [31:0] a[]);
initial set_reg_ptr(rf);
```

C++ 端通过此指针可以直接读取 RTL 中的寄存器值，用于 DiffTest 对比。

---

## 五、指令译码单元 (IDU)

### 5.1 设计结构

IDU 是纯**组合逻辑** — 输入指令，输出控制信号，无时钟依赖。

```verilog
// npc/vsrc/idu/idu_normal.v
wire [6:0] opcode = i_instr[6:0];    // 操作码
wire [2:0] func3  = i_instr[14:12];  // funct3
wire [6:0] func7  = i_instr[31:25];  // funct7
wire [4:0] rs1id  = i_instr[19:15];  // 源寄存器 1
wire [4:0] rs2id  = i_instr[24:20];  // 源寄存器 2
wire [4:0] rdid   = i_instr[11:7];   // 目的寄存器
```

### 5.2 立即数提取 (硬件实现)

```verilog
case (opcode)
  `TYPE_I: o_imm = {{20{i_instr[31]}}, i_instr[31:20]};           // 符号扩展 12 位
  `TYPE_S: o_imm = {{20{i_instr[31]}}, i_instr[31:25], i_instr[11:7]};
  `TYPE_B: o_imm = {{20{i_instr[31]}}, i_instr[7], i_instr[30:25], i_instr[11:8], 1'b0};
  `TYPE_U_LUI: o_imm = {i_instr[31:12], 12'b0};                   // 高 20 位
  `TYPE_J: o_imm = {{12{i_instr[31]}}, i_instr[19:12], i_instr[20], i_instr[30:21], 1'b0};
endcase
```

**对比 NEMU**：
- NEMU 中用 C 宏 `SEXT(BITS(i, 31, 20), 12)` 提取立即数
- RTL 中用位拼接 `{{20{i_instr[31]}}, i_instr[31:20]}` 实现相同功能
- `{20{i_instr[31]}}` = 将最高位复制 20 次 = **硬件符号扩展**

### 5.3 控制信号生成

IDU 根据 opcode/funct3/funct7 输出：

| 输出信号 | 含义 | 驱动模块 |
|----------|------|----------|
| `o_exu_opt` | ALU 运算类型 (ADD/SUB/AND/...) | EXU |
| `o_exu_src_sel` | ALU 输入来源 (REG/IMM/PC+4/PC+IMM) | EXU |
| `o_lsu_opt` | 访存类型 (LB/LH/LW/SB/SH/SW/NOP) | LSU |
| `o_rdwen` | 是否写回寄存器 | WBU |
| `o_brch/o_jal/o_jalr` | 分支跳转控制 | BRU |

### 5.4 EXU 源操作数选择

```verilog
`define EXU_SEL_REG  2'b00   // src2 = rs2 (R 型指令)
`define EXU_SEL_IMM  2'b01   // src2 = imm (I/S/U 型)
`define EXU_SEL_PC4  2'b10   // result = PC + 4 (JAL/JALR 保存返回地址)
`define EXU_SEL_PCI  2'b11   // result = PC + imm (AUIPC)
```

---

## 六、算术逻辑单元 (ALU)

### 6.1 实现

```verilog
// npc/vsrc/alu.v
module alu (
  input  [`CPU_WIDTH-1:0]     i_src1,
  input  [`CPU_WIDTH-1:0]     i_src2,
  input  [`EXU_OPT_WIDTH-1:0] i_opt,
  output reg [`CPU_WIDTH-1:0] o_alu_res,
  output reg                  o_sububit   // 无符号减法借位
);
  wire [63:0] tmp_res;  // 算术右移辅助

  always @(*) begin
    o_alu_res = 0;
    o_sububit = 0;
    case (i_opt)
      `ALU_ADD:  o_alu_res = i_src1 + i_src2;
      `ALU_SUB:  o_alu_res = i_src1 - i_src2;
      `ALU_AND:  o_alu_res = i_src1 & i_src2;
      `ALU_OR:   o_alu_res = i_src1 | i_src2;
      `ALU_XOR:  o_alu_res = i_src1 ^ i_src2;
      `ALU_SLL:  o_alu_res = i_src1 << i_src2[4:0];
      `ALU_SRL:  o_alu_res = i_src1 >> i_src2[4:0];
      `ALU_SRA:  o_alu_res = tmp_res[31:0];  // 算术右移
      `ALU_SUBU: {o_sububit, o_alu_res} = {1'b0,i_src1} - {1'b0,i_src2};
    endcase
  end

  // 算术右移: 符号扩展到 64 位后逻辑右移
  assign tmp_res = {{{32{i_src1[31]}}, i_src1} >> i_src2[5:0]};
endmodule
```

### 6.2 算术右移的硬件技巧

Verilog 中 `>>` 是逻辑右移（高位补 0），要实现算术右移（高位补符号位）：

```verilog
// 方法: 将 32 位数符号扩展到 64 位，逻辑右移后取低 32 位
wire [63:0] tmp = {{32{src[31]}}, src} >> shamt;
// 结果 = tmp[31:0]
```

这等效于 C 语言中 `(int32_t)src >> shamt`。

---

## 七、Verilator 仿真环境

### 7.1 Verilator 的工作原理

```
Verilog RTL (.v/.sv)
       │
       │ verilator --cc --exe
       ▼
C++ 模型 (Vtop.h, Vtop.cpp)
       │
       │ g++ 编译链接
       ▼
可执行仿真程序 (./build/top)
       │
       │ 运行
       ▼
波形文件 (build/sim.vcd) + 终端输出
```

### 7.2 Testbench 主循环 (`csrc/main.cpp`)

```cpp
int main(int argc, char **argv) {
  // 1. 初始化 Verilator
  VerilatedContext* contextp = new VerilatedContext;
  TOP* top = new TOP;
  VerilatedVcdC* tfp = new VerilatedVcdC;

  // 2. 启用波形记录
  contextp->traceEverOn(true);
  top->trace(tfp, 0);
  tfp->open("build/sim.vcd");

  // 3. 加载程序镜像、初始化 NPC
  npc_init(argc, argv);

  // 4. 复位
  top->reset = 1;
  for (int i = 0; i < 100; i++) {
    top->clock = !top->clock;
    top->eval();
    contextp->timeInc(1);
    tfp->dump(contextp->time());
  }
  top->reset = 0;

  // 5. 主仿真循环
  while (!contextp->gotFinish()) {
    top->clock = !top->clock;         // 翻转时钟
    top->eval();                      // 计算组合逻辑
    contextp->timeInc(1);             // 推进仿真时间
    tfp->dump(contextp->time());      // 记录波形

    // DiffTest: 在时钟上升沿检查状态
    if (top->clock && dut_status) {
      if (!difftest_check()) break;   // 状态不一致 → 停止
      difftest_step();                // REF 执行一步
    }
  }

  // 6. 清理
  tfp->close();
  statistics();  // 打印 IPC 等性能数据
  return 0;
}
```

### 7.3 仿真时钟的理解

```
时间:  0  1  2  3  4  5  6  7  8  9 ...
clock: 0  1  0  1  0  1  0  1  0  1 ...
       ↑     ↑     ↑     ↑     ↑
      上升沿  上升沿  上升沿  上升沿  上升沿

每两个时间步 (time step) = 一个完整时钟周期
contextp->time() / 2 = 已经过的时钟周期数
```

---

## 八、DPI-C 接口

### 8.1 问题：Verilog 仿真中如何访问主机内存？如何读取 RTL 内部信号？

**解决方案**：DPI-C (Direct Programming Interface for C) 允许 Verilog 调用 C 函数，或 C 调用 Verilog 导出的任务。

### 8.2 从 Verilog 调用 C 函数

```verilog
// Verilog 端声明
import "DPI-C" function void set_reg_ptr(input bit [31:0] a[]);
initial set_reg_ptr(rf);   // 将寄存器数组指针传递给 C++
```

```cpp
// C++ 端实现
static uint32_t *cpu_reg = NULL;
extern "C" void set_reg_ptr(uint32_t *a) {
  cpu_reg = a;   // 保存指针，后续可以直接读取寄存器
}
```

### 8.3 内存访问 DPI-C

当 NPC 的 IFU/LSU 需要读写内存时，通过 DPI-C 回调到 C++ 的内存模型：

```cpp
// C++ 内存模型
static uint8_t pmem[PMEM_MSIZE];

uint32_t _pmem_read(uint32_t addr, int len) {
  uint8_t *p = guest_to_host(addr);
  switch (len) {
    case 1: return *p;
    case 2: return *(uint16_t *)p;
    case 4: return *(uint32_t *)p;
  }
}
```

---

## 九、从 NEMU 到 NPC 的对应关系

| NEMU (C 语言) | NPC (Verilog) | 说明 |
|---------------|---------------|------|
| `isa_exec_once()` | 一个时钟周期 | 都是执行一条指令 |
| `inst_fetch(&snpc, 4)` | IFU 模块 | 从内存读指令 |
| `decode_operand()` | IDU 模块 | 立即数提取 + 寄存器读取 |
| `INSTPAT(...)` → 执行体 | EXU + ALU | 运算执行 |
| `Mr()/Mw()` | LSU 模块 | 内存读写 |
| `R(rd) = result` | WBU → RegFile | 写回结果 |
| `cpu.pc = dnpc` | BRU | 更新 PC |
| `R(0) = 0` | RegFile 写入逻辑 | x0 始终为 0 |

---

## 十、构建与运行

### 10.1 Makefile 核心

```makefile
# npc/Makefile
VERILATOR = verilator
VERILATOR_FLAGS += -cc --exe      # 生成 C++ 可执行文件
VERILATOR_FLAGS += --trace        # 启用波形记录
VERILATOR_FLAGS += --assert       # 启用断言检查
VERILATOR_FLAGS += -Wno-fatal     # 警告不致命

$(BIN): $(VSRCS) $(CSRCS)
    $(VERILATOR) $(VERILATOR_FLAGS) --top-module $(TOPNAME) $^ \
        $(addprefix -CFLAGS , $(CXXFLAGS)) --exe -o $(abspath $(BIN))
```

### 10.2 运行命令

```bash
make                              # 编译 NPC
make run                          # 运行默认测试 (dummy)
make run DIFFTEST=1               # 启用差分测试
make run WAVE=1                   # 记录波形
make run DIFFTEST=1 WAVE=1        # 两者同时启用
make wave                         # 用 GTKWave 打开波形
```

---

## 十一、常见问题与解决方案

### Q1: 仿真一开始就报 DiffTest 不一致

**原因**：复位不充分。RTL 中的寄存器需要足够的复位周期才能稳定。
**解决**：确保 `reset(100)` 执行足够多的时钟周期。

### Q2: 指令执行结果与 NEMU 不一致 — 如何定位？

**流程**：
1. DiffTest 报错时会打印：哪个寄存器、DUT 值是多少、REF 值是多少
2. 从 DiffTest 报错的 PC 值找到出错的指令
3. 用 `objdump -d` 确认该指令的预期行为
4. 打开波形 (WAVE=1)，在 GTKWave 中定位到该时刻
5. 检查 IDU 输出的控制信号是否正确
6. 检查 EXU/ALU 的计算结果是否正确

### Q3: Verilog 中 `>>>` vs `>>` 的区别

- `>>` — 逻辑右移（高位补 0），用于无符号数
- `>>>` — 算术右移（高位补符号位），但**仅当操作数声明为 `signed` 时有效**

本项目未使用 `>>>`,而是通过符号扩展 + 逻辑右移实现算术右移。

### Q4: 为什么 RegFile 写入在上升沿，读取是组合逻辑？

- **写入用时序逻辑**：保证同一周期内 WBU 写回的值不会影响当前指令的读取（避免读写竞争）
- **读取用组合逻辑**：允许 IDU 在当前周期内立即获得寄存器值，无需等待下一周期

### Q5: `s_a0zero` 信号是干什么的？

```verilog
assign s_a0zero = ~|rf[10];  // rf[10] 的所有位取 OR，再取反
```

当 `rf[10]`（即 a0 寄存器）= 0 时，`s_a0zero = 1`。仿真 testbench 通过此信号判断程序是否正确结束：
- a0 == 0 → HIT GOOD TRAP
- a0 != 0 → HIT BAD TRAP

---

## 十二、PA 讲义相关思考题

#### Q: RTL 实现和 NEMU 的 C 实现有什么本质区别？

**回答**：

| 维度 | C (NEMU) | Verilog (NPC) |
|------|----------|---------------|
| 执行模型 | 顺序执行 (一句一句) | 并行计算 (所有 assign 同时生效) |
| 时间 | 无物理时间概念 | 有时钟周期 |
| 数据存储 | C 变量 (无限生命周期) | 寄存器 (边沿触发) / 线网 (组合逻辑) |
| 调试 | 断点、printf | 波形、断言 |

核心区别：**C 代码是串行描述"做什么"，Verilog 是并行描述"电路长什么样"**。

---

#### Q: 为什么处理器需要时钟？没有时钟可以吗？

**回答**：

时钟的作用是**同步**——确保所有组合逻辑在产生稳定结果后，才在时钟边沿将结果写入寄存器。

没有时钟（异步电路）理论上可以，但：
1. 需要复杂的握手协议确保数据稳定
2. 难以设计、难以验证、难以综合
3. 工业界几乎全部采用同步设计

时钟周期的长度由**最长组合逻辑路径**（关键路径）决定。这就是为什么后续 B 阶段要做时序分析。

---

#### Q: 为什么 D4 阶段不实现乘除法 (M 扩展)?

**回答**：

1. **M 扩展的硬件开销大**：乘法器需要大量加法器级联或使用 Booth 编码，除法器更复杂
2. **RV32E 本身是嵌入式子集**：面积敏感，通常不包含硬件乘除
3. **可以用软件替代**：AM 提供了 `libgcc/div.S` 和 `muldi3.S`，用软件模拟乘除法
4. **渐进式设计**：先跑通基本指令，后续阶段再添加 M 扩展

---

#### Q: Verilator 和真正的逻辑仿真器 (ModelSim/VCS) 有什么区别？

**回答**：

| 维度 | Verilator | ModelSim/VCS |
|------|-----------|--------------|
| 仿真模型 | 双状态 (0/1) | 四状态 (0/1/x/z) |
| 速度 | 极快 (编译为 C++) | 较慢 (事件驱动) |
| 精度 | 周期精确 | 支持延迟精确 |
| 开源 | 是 | 否 (商业工具) |
| 适合 | 功能验证、DiffTest | 完整验证、门级仿真 |

Verilator 不支持 `x`/`z` 值，所以无法检测未初始化的信号。但对于功能验证来说，它的速度优势远超精度损失。

---

## 十三、学习建议

1. **先实现最简单的数据通路**：只支持 `addi` + `ebreak`，跑通 `dummy` 测试
2. **逐步增加指令**：add → lw/sw → beq/jal → 完整 RV32I
3. **波形是你最好的朋友**：不确定时打开 GTKWave 观察信号时序
4. **时刻开启 DiffTest**：每增加一条指令就运行回归测试
5. **理解组合逻辑 vs 时序逻辑**：`always @(*)` 是组合，`always @(posedge clk)` 是时序
6. **不要在 D4 阶段追求完美**：单周期+DPI-C 内存就够了，AXI 总线是 B 阶段的事

---

*参考资料：*
- *RISC-V ISA Specification (Volume I)*
- *Verilator 官方文档: https://verilator.org/guide/latest/*
- *一生一芯讲义数字电路部分*
- *项目源码: npc/vsrc/, npc/csrc/*


---

## 十四、本阶段 Git 版本记录

| Commit | 说明 | 完成内容 |
|--------|------|----------|
| `323892b` | PA2.1 RTL | 第一版 NPC (Verilator 仿真，基本指令) |
| `68bd7d0` | PA2.2 RTL w/o trace | NPC 完善，通过 cpu-tests (不含 trace) |
| `8b59731` | replace Verilator with Questasim | 尝试 Questasim (后续回归 Verilator) |
| `d56f8cf` | update RTL | 参考实现优化 NPC 数据通路 |
| `c4ddbe0` | feat: code simplified | RTL 代码简化重构 |

**查看对应版本代码**：
```bash
git show 323892b:npc/vsrc/top.v            # 第一版 NPC 顶层
git show 68bd7d0:npc/vsrc/idu/idu_normal.v # PA2.2 译码器
git show c4ddbe0:npc/vsrc/alu.v            # 简化后的 ALU
```


---

## 十五、Git 版本代码演进详解

### 15.1 第一版 NPC (`323892b`) — PA2.1 RTL

**文件结构** (早期命名风格，与最终版完全不同):
```
npc/vsrc/
├── ALU.v           # 算术逻辑单元 (大写命名)
├── Branch.v        # 分支判断
├── CPU_top.v       # 顶层模块
├── ControlUnit.v   # 控制单元 (相当于后来的 IDU)
├── EXU.v           # 执行单元
├── IDU.v           # 指令译码
├── IFU.v           # 取指单元
├── ImmGen.v        # 立即数生成器
├── include/        # 头文件
├── libs/           # 基础库 (Mux, RegisterFile)
└── sim/            # 仿真相关 (testbench)
```

**特点**:
- 模块命名为大写驼峰 (ALU.v, IFU.v) — 后续重构为小写 (alu.v, ifu.v)
- 有独立的 `ImmGen.v` 立即数生成模块 — 后续合并到 IDU 中
- 有独立的 `ControlUnit.v` 控制单元 — 后续合并到 IDU
- 使用 `sim/testbench_cpu.v` — 后续改为 Verilator C++ testbench

---

### 15.2 重大重构 (`3980d5e` → `51b3f3b`) — 从旧架构到新架构

**删除的文件** (旧架构, -803 行):
```
ALU.v, Branch.v, CPU_top.v, ControlUnit.v, EXU.v,
IDU.v, IFU.v, ImmGen.v, LSU.v, sim/testbench_cpu.v
```

**新增的文件** (新架构, +1107 行):
```
alu.v          # ALU (简洁版)
csrfile.v      # CSR 寄存器文件 (新增!)
exu.v          # 执行单元 (含 CSR 操作)
idu/idu.v      # 顶层译码器
idu/idu_normal.v  # 普通指令译码
idu/idu_system.v  # 系统指令译码 (CSR/ecall/mret)
ifu.v          # 取指 (DPI-C 版)
lsu.v          # 访存单元
pcu.v          # PC 更新单元 (后改名为 bru.v)
regfile.v      # 寄存器堆
top.v          # 新顶层 (所有模块连接)
wbu.v          # 写回单元
libs/stdreg.v  # 标准寄存器模块
libs/stdrst.v  # 同步复位模块
include/defines.v  # 大量宏定义 (从 137 行扩展)
```

**这次重构的意义**:
1. 从"教科书风格"(每个功能一个模块) 转向"流水线风格"(按级划分)
2. 引入了 CSR 支持 (为 RT-Thread 做准备)
3. IDU 分为 normal + system 两部分 (解耦普通指令和特权指令)
4. 引入 `stdreg` 和 `stdrst` 标准化库 (后续所有寄存器都用它)
5. 控制信号用 `defines.v` 中的宏统一管理

---

### 15.3 代码简化 (`c4ddbe0`)

在重构后做清理：
- 统一代码风格
- 去除冗余逻辑
- 优化信号命名

---

### 15.4 关键对比: 第一版 vs 最终版

| 方面 | 第一版 (`323892b`) | 最终版 (`51b3f3b`) |
|------|-------|-------|
| 模块数 | 8 个 | 14 个 |
| 命名风格 | 大写驼峰 | 小写下划线 |
| IDU 结构 | 单一 ControlUnit | idu + idu_normal + idu_system |
| CSR 支持 | 无 | csrfile.v + idu_system.v |
| 立即数生成 | 独立 ImmGen.v | 合并到 idu_normal.v |
| 仿真方式 | Verilog testbench | Verilator C++ |
| 标准化库 | 无 | stdreg, stdrst, MuxKey |
| 复位方式 | 无规范 | stdrst 同步复位 |
