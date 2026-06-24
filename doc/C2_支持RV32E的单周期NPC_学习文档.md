# C2: 支持 RV32E 的单周期 NPC — 详细学习文档

## 一、阶段目标

C2 阶段的核心目标是实现一个完整的单周期 RV32E 处理器，通过所有 cpu-tests 测试用例。完成后你将：

1. 完整实现 RV32I 基础整数指令集（在 RV32E 16 寄存器约束下）
2. 理解单周期处理器中所有数据通路信号的协同工作
3. 掌握 IDU 控制信号生成的完整逻辑
4. 通过 DiffTest 对比验证每条指令的正确性
5. 能够运行 am-kernels 中的所有 CPU 测试

---

## 二、单周期处理器概念

### 2.1 "单周期"的含义

在一个时钟周期内完成：**取指 → 译码 → 执行 → 访存 → 写回**

```
                         一个时钟周期
    ┌─────────────────────────────────────────────────────────┐
    │                                                         │
    │  ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐    │
    │  │ IF  │──▶│ ID  │──▶│ EX  │──▶│ MEM │──▶│ WB  │    │
    │  │取指  │   │译码  │   │执行  │   │访存  │   │写回  │    │
    │  └─────┘   └─────┘   └─────┘   └─────┘   └─────┘    │
    │                                                         │
    └─────────────────────────────────────────────────────────┘
                               ↑
                          posedge clock
                       (所有结果在此刻锁存)
```

**优点**：设计简单，每条指令行为容易理解
**缺点**：时钟周期由最慢的指令决定（通常是 Load 指令），主频低

### 2.2 RV32E vs RV32I

| 特性 | RV32I | RV32E |
|------|-------|-------|
| 通用寄存器 | 32 个 (x0-x31) | **16 个** (x0-x15) |
| 寄存器地址位宽 | 5 位 | 4 位 (但编码仍用 5 位) |
| 指令集 | 完整 RV32I | 与 RV32I 相同 |
| 适用场景 | 通用处理器 | 嵌入式微控制器 |
| ABI | ilp32 | ilp32e |

在硬件实现中，RV32E 仍使用 5 位寄存器地址字段，但只访问 x0-x15。高 16 个寄存器在 RV32E 模式下未定义。

---

## 三、完整数据通路

### 3.1 信号流图

```
                                         ┌─────────────┐
                                    ┌───▶│  RegFile    │◀──── WBU 写回
                                    │    │ (读 rs1/rs2)│
                                    │    └──┬──────┬──┘
                                    │       │rs1   │rs2
         ┌──────┐    ┌──────┐      │       ▼      ▼
  PC ───▶│ IFU  │───▶│ IDU  │──────┤    ┌──────────────┐
         │      │    │      │──imm─┼───▶│    EXU       │
         │ 取指 │    │ 译码 │      │    │ (ALU + MUX)  │
         └──────┘    └──┬───┘      │    └──────┬───────┘
                        │          │           │exu_res
                        │ lsu_opt  │           ▼
                        │          │    ┌──────────────┐
                        └──────────┼───▶│    LSU       │
                                   │    │ (Load/Store) │
                                   │    └──────┬───────┘
                                   │           │lsu_res
                                   │           ▼
                                   │    ┌──────────────┐
                                   └───▶│    WBU       │──── rd → RegFile
                                        │ (选择写回源) │
                                        └──────────────┘

         ┌──────┐
         │ BRU  │◀── brch/jal/jalr/zero
         │ 分支 │───▶ next PC
         └──────┘
```

### 3.2 各模块输入输出总结

| 模块 | 输入 | 输出 |
|------|------|------|
| **IFU** | PC | instr (32-bit 指令) |
| **IDU** | instr | rs1id, rs2id, rdid, imm, exu_opt, exu_src_sel, lsu_opt, brch/jal/jalr, rdwen |
| **EXU** | rs1, rs2/imm, PC, exu_opt, exu_src_sel | exu_res, zero |
| **LSU** | exu_res (地址), rs2 (存储数据), lsu_opt | lsu_res (加载数据) |
| **WBU** | exu_res, lsu_res, lsu_opt | rd (写回数据), rdwen, rdid |
| **BRU** | brch, jal, jalr, zero, rs1, imm, PC | next_PC |

---

## 四、指令实现完整清单

### 4.1 R 型指令 (寄存器-寄存器运算)

| 指令 | opcode | funct3 | funct7 | IDU 输出 | EXU 行为 |
|------|--------|--------|--------|----------|----------|
| `add` | 0110011 | 000 | 0000000 | exu_opt=ADD, sel=REG | rd = rs1 + rs2 |
| `sub` | 0110011 | 000 | 0100000 | exu_opt=SUB, sel=REG | rd = rs1 - rs2 |
| `sll` | 0110011 | 001 | 0000000 | exu_opt=SLL, sel=REG | rd = rs1 << rs2[4:0] |
| `slt` | 0110011 | 010 | 0000000 | exu_opt=SLT, sel=REG | rd = (signed)rs1 < (signed)rs2 |
| `sltu` | 0110011 | 011 | 0000000 | exu_opt=SLTU, sel=REG | rd = rs1 < rs2 |
| `xor` | 0110011 | 100 | 0000000 | exu_opt=XOR, sel=REG | rd = rs1 ^ rs2 |
| `srl` | 0110011 | 101 | 0000000 | exu_opt=SRL, sel=REG | rd = rs1 >> rs2[4:0] |
| `sra` | 0110011 | 101 | 0100000 | exu_opt=SRA, sel=REG | rd = (signed)rs1 >>> rs2[4:0] |
| `or` | 0110011 | 110 | 0000000 | exu_opt=OR, sel=REG | rd = rs1 \| rs2 |
| `and` | 0110011 | 111 | 0000000 | exu_opt=AND, sel=REG | rd = rs1 & rs2 |

### 4.2 I 型指令 (立即数运算)

| 指令 | opcode | funct3 | IDU 输出 | EXU 行为 |
|------|--------|--------|----------|----------|
| `addi` | 0010011 | 000 | exu_opt=ADD, sel=IMM | rd = rs1 + sext(imm) |
| `slti` | 0010011 | 010 | exu_opt=SLT, sel=IMM | rd = (signed)rs1 < (signed)imm |
| `sltiu` | 0010011 | 011 | exu_opt=SLTU, sel=IMM | rd = rs1 < imm |
| `xori` | 0010011 | 100 | exu_opt=XOR, sel=IMM | rd = rs1 ^ imm |
| `ori` | 0010011 | 110 | exu_opt=OR, sel=IMM | rd = rs1 \| imm |
| `andi` | 0010011 | 111 | exu_opt=AND, sel=IMM | rd = rs1 & imm |
| `slli` | 0010011 | 001 | exu_opt=SLL, sel=IMM | rd = rs1 << imm[4:0] |
| `srli` | 0010011 | 101 (f7=000000x) | exu_opt=SRL, sel=IMM | rd = rs1 >> imm[4:0] |
| `srai` | 0010011 | 101 (f7=010000x) | exu_opt=SRA, sel=IMM | rd = (signed)rs1 >>> imm[4:0] |

### 4.3 Load 指令

| 指令 | funct3 | LSU 行为 |
|------|--------|----------|
| `lb` | 000 | rd = sext(mem[rs1+imm][7:0]) |
| `lh` | 001 | rd = sext(mem[rs1+imm][15:0]) |
| `lw` | 010 | rd = mem[rs1+imm][31:0] |
| `lbu` | 100 | rd = zext(mem[rs1+imm][7:0]) |
| `lhu` | 101 | rd = zext(mem[rs1+imm][15:0]) |

### 4.4 Store 指令

| 指令 | funct3 | LSU 行为 |
|------|--------|----------|
| `sb` | 000 | mem[rs1+imm] = rs2[7:0] |
| `sh` | 001 | mem[rs1+imm] = rs2[15:0] |
| `sw` | 010 | mem[rs1+imm] = rs2[31:0] |

### 4.5 分支指令

| 指令 | funct3 | 条件 |
|------|--------|------|
| `beq` | 000 | rs1 == rs2 |
| `bne` | 001 | rs1 != rs2 |
| `blt` | 100 | (signed)rs1 < (signed)rs2 |
| `bge` | 101 | (signed)rs1 >= (signed)rs2 |
| `bltu` | 110 | rs1 < rs2 |
| `bgeu` | 111 | rs1 >= rs2 |

**BRU 逻辑**：若条件成立 → PC = PC + imm；否则 → PC = PC + 4

### 4.6 跳转指令

| 指令 | 行为 |
|------|------|
| `jal` | rd = PC+4; PC = PC + imm |
| `jalr` | rd = PC+4; PC = (rs1 + imm) & ~1 |

### 4.7 上址指令

| 指令 | 行为 |
|------|------|
| `lui` | rd = imm (高 20 位左移 12) |
| `auipc` | rd = PC + imm |

---

## 五、IDU 控制信号生成详解

### 5.1 EXU 操作数来源 (`exu_src_sel`)

```verilog
`define EXU_SEL_REG  2'b00   // src1=rs1, src2=rs2      (R型/B型)
`define EXU_SEL_IMM  2'b01   // src1=rs1, src2=imm      (I型/S型/Load/LUI)
`define EXU_SEL_PC4  2'b10   // result=PC+4             (JAL/JALR 保存返回地址)
`define EXU_SEL_PCI  2'b11   // result=PC+imm           (AUIPC)
```

### 5.2 LSU 操作类型编码

```verilog
// LSU_OPT = {funct3[2:0], is_store}
`define LSU_LB   4'b0000   // Load Byte (符号扩展)
`define LSU_LH   4'b0010   // Load Half (符号扩展)
`define LSU_LW   4'b0100   // Load Word
`define LSU_LBU  4'b1000   // Load Byte Unsigned (零扩展)
`define LSU_LHU  4'b1010   // Load Half Unsigned (零扩展)
`define LSU_SB   4'b0001   // Store Byte
`define LSU_SH   4'b0011   // Store Half
`define LSU_SW   4'b0101   // Store Word
`define LSU_NOP  4'b1111   // 非访存指令
```

**设计技巧**：`lsu_opt[0]` = 1 表示 Store，= 0 表示 Load。WBU 据此选择写回来源：
```verilog
// WBU 中的写回选择
assign rd = (~lsu_opt[0]) ? lsu_res : exu_res;
// Load 指令: 写回 LSU 读取的数据
// 非 Load: 写回 EXU 运算结果
```

### 5.3 分支控制信号

```verilog
// IDU 输出
assign o_brch = (opcode == `TYPE_B);    // 是条件分支?
assign o_jal  = (opcode == `TYPE_J);    // 是 JAL?
assign o_jalr = (opcode == `TYPE_I_JALR); // 是 JALR?
```

BRU 综合这些信号决定下一条 PC：
```verilog
// BRU 伪代码
if (jal)       next_pc = pc + imm;
else if (jalr) next_pc = (rs1 + imm) & ~1;
else if (brch && zero_flag_match) next_pc = pc + imm;
else           next_pc = pc + 4;
```

---

## 六、EXU 中的比较操作

### 6.1 问题：分支指令需要比较 rs1 和 rs2，如何复用 ALU？

**解决方案**：分支指令让 ALU 做减法，然后根据结果的符号位和借位判断大小关系。

```verilog
// EXU 中分支条件判断
always @(*) begin
  case (i_opt)
    `EXU_BEQ:  zero = (alu_res == 0);                    // rs1-rs2==0 → 相等
    `EXU_BNE:  zero = (alu_res != 0);                    // rs1-rs2!=0 → 不等
    `EXU_BLT:  zero = alu_res[31];                       // 结果为负 → rs1 < rs2 (有符号)
    `EXU_BGE:  zero = ~alu_res[31];                      // 结果非负 → rs1 >= rs2 (有符号)
    `EXU_BLTU: zero = sububit;                           // 借位 → rs1 < rs2 (无符号)
    `EXU_BGEU: zero = ~sububit;                          // 无借位 → rs1 >= rs2 (无符号)
  endcase
end
```

ALU 的 `o_sububit` 就是为无符号比较设计的：
```verilog
`ALU_SUBU: {o_sububit, o_alu_res} = {1'b0, i_src1} - {1'b0, i_src2};
// 扩展到 33 位做减法，最高位就是借位
```

---

## 七、SLT/SLTU 的实现

### 7.1 有符号比较 (`slt`)

```verilog
// ALU 做有符号减法
`ALU_SUB: o_alu_res = i_src1 - i_src2;
// EXU 判断结果符号位
// 但要注意溢出! 例如 0x7FFFFFFF - 0x80000000 = 0xFFFFFFFF (结果为负但实际 src1 > src2)
```

正确做法需要考虑溢出，本项目的 EXU 对 SLT 有专门处理。

### 7.2 无符号比较 (`sltu`)

```verilog
`ALU_SUBU: {o_sububit, o_alu_res} = {1'b0, i_src1} - {1'b0, i_src2};
// sububit=1 表示 src1 < src2 (无符号)
// rd = sububit
```

---

## 八、测试与验证

### 8.1 cpu-tests 测试列表

```bash
cd am-kernels/tests/cpu-tests
make ARCH=riscv32e-npc ALL=dummy run   # 最简测试
make ARCH=riscv32e-npc ALL=add run     # 加法测试
make ARCH=riscv32e-npc ALL=sub run     # 减法
make ARCH=riscv32e-npc ALL=shift run   # 移位
make ARCH=riscv32e-npc ALL=load-store run  # 访存
make ARCH=riscv32e-npc ALL=branch run  # 分支
make ARCH=riscv32e-npc run             # 全部测试
```

### 8.2 推荐的指令实现顺序

```
第1步: addi + ebreak                    → 跑通 dummy
第2步: add, sub                          → 跑通 add/sub
第3步: lui, auipc                        → 地址计算正确
第4步: lw, sw                            → 跑通 load-store
第5步: jal, jalr                         → 跑通函数调用
第6步: beq, bne                          → 跑通简单分支
第7步: slt, sltu, blt, bge, bltu, bgeu  → 全部比较/分支
第8步: and, or, xor, sll, srl, sra      → 全部逻辑/移位
第9步: lb, lh, lbu, lhu, sb, sh         → 不同宽度访存
第10步: slti, sltiu, andi, ori, xori    → I型剩余指令
```

### 8.3 DiffTest 驱动开发

```
1. 添加新指令到 IDU
2. 运行 DiffTest (make run DIFFTEST=1)
3. 如果 PASS → 继续下一条
4. 如果 FAIL:
   a. 看报错的 PC 和寄存器
   b. objdump 确认出错的指令
   c. 检查 IDU 控制信号
   d. 必要时看波形
5. 重复直到所有 cpu-tests PASS
```

---

## 九、常见 Bug 与解决

### Q1: `addi` 可以工作但 `add` 不行

**检查**：IDU 中 R 型和 I 型的 `exu_src_sel` 是否正确区分：
- I 型: `EXU_SEL_IMM` (src2 = 立即数)
- R 型: `EXU_SEL_REG` (src2 = rs2 寄存器值)

### Q2: Load 指令数据错误

**检查**：
1. 地址计算：`addr = rs1 + imm`，imm 是否正确符号扩展？
2. 字节/半字 Load：符号扩展 vs 零扩展 (`lb` vs `lbu`)
3. 地址对齐：`sw` 的地址低 2 位应为 0

### Q3: 分支指令跳转到错误地址

**检查**：
1. B 型立即数的拼接顺序：`{sign, [7], [30:25], [11:8], 0}`
2. 分支偏移是**相对 PC** 的，不是绝对地址
3. 有符号比较 vs 无符号比较是否混淆

### Q4: `jal` 的返回地址错误

**检查**：`rd = PC + 4`，注意 EXU 的 `exu_src_sel` 应为 `EXU_SEL_PC4`

### Q5: `lui` 的立即数值不对

**检查**：U 型立即数 = `{instr[31:12], 12'b0}`，**不需要符号扩展**（它本身就是 32 位）

---

## 十、PA 讲义相关思考题

#### Q: 为什么单周期处理器的主频受限于最慢的指令？

**回答**：时钟周期必须足够长，让**任何一条指令**都能在一个周期内完成。最慢的指令（通常是 `lw`：取指+译码+地址计算+内存读取+写回）决定了时钟周期的最小长度。即使大多数指令只需要更短的时间，时钟也不能更快。

---

#### Q: 单周期 NPC 的 CPI 是多少？

**回答**：**CPI = 1.0**（严格的单周期），每条指令恰好用一个时钟周期。但如果使用 DPI-C 内存访问（主机函数调用是"瞬时"的），仿真中确实是每周期执行一条。在真实硬件中如果内存延迟多周期，就不再是单周期了（需要引入 stall 或改为多周期/流水线）。

---

#### Q: x0 寄存器在硬件中如何保证始终为 0？

**回答**：本项目的实现：
```verilog
always @(posedge i_clk) begin
  if (i_wen) begin
    if (i_waddr == 0) rf[i_waddr] <= 0;  // 写入 x0 时强制写 0
    else rf[i_waddr] <= i_wdata;
  end
end
```

另一种实现方式：读出时做判断：
```verilog
assign o_rdata1 = (i_raddr1 == 0) ? 0 : rf[i_raddr1];
```

两种方式等效，第一种更接近真实硬件（x0 的物理寄存器就是硬连线接地）。

---

#### Q: 为什么 EXU 的比较用减法实现而不是直接比较？

**回答**：
1. **复用硬件**：加法器已经存在（用于 ADD/SUB），减法就是 `A + (~B) + 1`
2. **面积节约**：不需要额外的比较器电路
3. **多种比较统一**：BEQ (==0), BNE (!=0), BLT (符号位), BLTU (借位) 都可以从减法结果推导

---

## 十一、学习建议

1. **按推荐顺序实现指令** — 每一步跑通一个测试用例
2. **一次只加一条指令** — 出错时 DiffTest 能精确告诉你哪条新指令有问题
3. **仔细对照 RISC-V ISA 手册** — 特别是符号扩展和移位的细节
4. **记住 R 型用 funct7 区分 add/sub/srl/sra** — 这是最容易遗漏的
5. **Store 指令不写回寄存器** — 确保 `rdwen = 0`

---

*参考资料：*
- *RISC-V ISA Specification Volume I (Chapter 2: RV32I Base Instruction Set)*
- *项目源码: npc/vsrc/idu/idu_normal.v, npc/vsrc/alu.v, npc/vsrc/exu.v*
- *am-kernels/tests/cpu-tests/*


---

## 十二、本阶段 Git 版本记录

| Commit | 说明 | 完成内容 |
|--------|------|----------|
| `68bd7d0` | PA2.2 RTL w/o trace | 单周期 NPC 通过 cpu-tests |
| `d56f8cf` | update RTL | 参考实现优化 |
| `c4ddbe0` | feat: code simplified | 代码简化重构 |
| `076d790` | feat: Add handshake signals | 添加 valid/ready 握手信号 (为多周期做准备) |

**查看对应版本代码**：
```bash
git show 68bd7d0:npc/vsrc/top.v              # 单周期 NPC 完整顶层
git show 076d790:npc/vsrc/top.v              # 添加握手信号后的版本
```


---

## 十三、Git 版本代码演进详解

### 13.1 从 RT-Thread 版到握手版 (`51b3f3b` → `076d790`)

**变化统计**: 13 文件, +168/-211 行 (净减少 43 行 = 重构简化)

**主要变化**:

| 文件 | 变化 | 说明 |
|------|------|------|
| `include/defines.v` | -127 行 (删除) | 改为 `defines.vh` (用 `include 引入) |
| `top.v` | +82/-34 行 | 添加 valid/ready 接口信号 |
| `lsu.v` | +70/-38 行 | LSU 添加握手控制 |
| `wbu.v` | +27/-8 行 | WBU 添加握手逻辑 |
| `ifu.v` | +26/-12 行 | IFU 添加 o_post_valid |
| `exu.v` | +13/-5 行 | EXU 添加握手透传 |

**核心改动——添加 valid/ready 握手**:

在单周期 CPU 中 valid/ready 实际上总为 1 (因为每条指令一周期内完成)，但这一步是为 B1 (多周期/总线) 做**接口预备**:

```verilog
// 076d790 版本的 top.v 中新增的信号
wire ifu_valid;    // IFU 取指完成
wire idu_ready;    // IDU 准备好接收
wire idu_valid;    // IDU 译码完成
wire exu_ready;    // EXU 准备好接收
wire exu_valid;    // EXU 执行完成
wire lsu_ready;    // LSU 准备好接收
wire lsu_valid;    // LSU 访存完成
wire wbu_ready;    // WBU 准备好接收
```

**commit message 原话**: "But valid-ready in single-cycle cpu is meaningless" — 正确! 但接口先建立好，后续只需修改内部实现。

---

### 13.2 单周期到多周期的关键转变 (`076d790` → `75481bf`)

**变化统计**: 12 文件, +324/-133 行

**架构变化**:
```
单周期 (076d790):
  所有操作在一个周期完成
  valid/ready 全为 1 (形式上存在但无实际约束)

多周期 (75481bf):
  周期1: IFU (取指) — 从 SRAM 读指令
  周期2: IDU + EXU (译码+执行)
  周期3: LSU + WBU + PCU (访存+写回+更新PC)
```

**关键新增文件**:
- `npc/vsrc/ifu/ifu_sram.v` — IFU 专用 SRAM (44 行)
- `npc/vsrc/lsu/lsu_sram.v` — LSU 专用 SRAM (60 行)

**RegFile 变化**: 从组合逻辑读改为考虑写后读 (write-first) 行为:
```verilog
// 写操作加入了写地址为0的保护
if (i_waddr == 0) rf[i_waddr] <= 0;
else rf[i_waddr] <= i_wdata;
```

**WBU 大幅扩展** (+49 行): 因为多周期需要在 WBU 中锁存 IDU 传来的控制信号，等 LSU 完成后才能写回:
```verilog
// 锁存 rdid、rdwen、csrdid、csrdwen 等信号
// 等到 lsu_valid 时才真正执行写回
```
