# C5: 异常处理和 RT-Thread — 详细学习文档

## 一、阶段目标

C5 阶段的核心目标是实现 RISC-V Machine-mode 特权级机制，支持异常/中断处理和上下文切换，最终能运行 RT-Thread 等实时操作系统。完成后你将：

1. 理解 RISC-V 特权级架构 (Machine Mode)
2. 实现 CSR 寄存器文件 (mtvec, mepc, mcause, mstatus)
3. 实现 `ecall` / `mret` 指令及其硬件逻辑
4. 实现 `csrrw` / `csrrs` / `csrrc` CSR 读写指令
5. 理解 AM 的 CTE (Context Extension) 如何建立异常处理框架
6. 理解上下文切换机制，支持运行简单的多任务内核

---

## 二、RISC-V 特权级概述

### 2.1 特权级层次

```
┌────────────────────────────────────────────┐
│        Application (用户程序)                │  U-mode (User)
├────────────────────────────────────────────┤
│        Operating System (操作系统)           │  S-mode (Supervisor)
├────────────────────────────────────────────┤
│        Firmware / SEE (执行环境)             │  M-mode (Machine)
└────────────────────────────────────────────┘
```

**本项目只实现 M-mode** — 所有代码运行在最高特权级，无保护。这是嵌入式系统和教学系统的典型配置。

### 2.2 为什么需要异常处理？

| 事件 | 来源 | 处理方式 |
|------|------|----------|
| `ecall` | 软件主动触发 | 陷入异常处理函数 |
| 非法指令 | 指令无法识别 | 报错 / 终止 |
| 地址不对齐 | 访存地址错误 | 报错 |
| 外部中断 | 定时器/设备 | 转入中断处理 |

在本阶段，主要关注 **ecall (环境调用)** — 它是用户程序请求系统服务的标准方式。

---

## 三、CSR 寄存器

### 3.1 本项目实现的 CSR

| CSR | 地址 | 读写 | 功能 |
|-----|------|------|------|
| `mstatus` | 0x300 | RW | 机器状态 (MIE, MPIE 等) |
| `mtvec` | 0x305 | RW | 异常入口地址 (trap vector) |
| `mepc` | 0x341 | RW | 异常返回地址 |
| `mcause` | 0x342 | RW | 异常原因码 |
| `mvendorid` | 0xF11 | RO | 厂商 ID (只读) |
| `marchid` | 0xF12 | RO | 架构 ID (只读) |

### 3.2 mstatus 寄存器位域

```
31    22  21  20  19  18  17  16:15 14:13 12:11  10:9  8   7   6   5   4   3   2   1   0
┌──────┬───┬───┬───┬───┬───┬─────┬─────┬─────┬────┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│  0   │TSR│TW │TVM│MXR│SUM│MPRV │ XS  │ FS  │ MPP│ 0 │SPP│MPIE│ 0 │SPIE│ 0 │MIE│ 0 │SIE│ 0 │
└──────┴───┴───┴───┴───┴───┴─────┴─────┴─────┴────┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
                                                 ↑              ↑             ↑
                                            bits[12:11]      bit[7]        bit[3]
                                            MPP=11(M-mode)  MPIE          MIE
```

本项目主要关注：
- `MIE` (bit 3): Machine Interrupt Enable — 全局中断使能
- `MPIE` (bit 7): Previous MIE — 进入异常前的 MIE 值
- `MPP` (bits 12:11): Previous Privilege — 进入异常前的特权级 (M-mode=11)

### 3.3 CSR 硬件实现 (`csrfile.v`)

```verilog
module csrfile (
  // CSR 读写接口 (来自 csrrw/csrrs/csrrc 指令)
  input  i_ren, input [`CSR_ADDRW-1:0] i_raddr, output [`CPU_WIDTH-1:0] o_rdata,
  input  i_wen, input [`CSR_ADDRW-1:0] i_waddr, input  [`CPU_WIDTH-1:0] i_wdata,

  // 异常专用写接口 (来自 IRU)
  input  i_mepc_wen,    input [`CPU_WIDTH-1:0] i_mepc_wdata,
  input  i_mcause_wen,  input [`CPU_WIDTH-1:0] i_mcause_wdata,
  input  i_mstatus_wen, input [`CPU_WIDTH-1:0] i_mstatus_wdata,

  // 异常专用读接口 (给 BRU/IRU)
  output [`CPU_WIDTH-1:0] o_mtvec,    // ecall 跳转目标
  output [`CPU_WIDTH-1:0] o_mstatus,  // 当前状态
  output [`CPU_WIDTH-1:0] o_mepc      // mret 返回地址
);
```

**双写入源设计**：每个 CSR 都有两个写入来源：
1. `csrrw`/`csrrs`/`csrrc` 指令的正常写入 (`i_wen + i_wdata`)
2. 异常发生时 IRU 的强制写入 (`i_mepc_wen + i_mepc_wdata`)

```verilog
// 异常写入优先级高于指令写入
wire wen_mepc = (i_wen & (i_waddr == `ADDR_MEPC)) | i_mepc_wen;
wire [`CPU_WIDTH-1:0] wdata_mepc = i_mepc_wen ? i_mepc_wdata : i_wdata;
```

---

## 四、ecall 指令 — 触发异常

### 4.1 ecall 的语义

```
ecall 执行时:
  1. mepc  ← 当前 PC            (保存异常返回地址)
  2. mcause ← 11                (异常原因: Environment call from M-mode)
  3. mstatus.MPIE ← mstatus.MIE (保存当前中断使能状态)
  4. mstatus.MIE ← 0            (关闭中断)
  5. PC ← mtvec                  (跳转到异常处理入口)
```

### 4.2 NEMU 中的 ecall 实现

```c
// nemu/src/isa/riscv32/inst.c
INSTPAT("0000000 00000 00000 000 00000 11100 11", ecall, I,
  s->dnpc = isa_raise_intr(0x0b, s->pc));

// nemu/src/isa/riscv32/system/intr.c
word_t isa_raise_intr(word_t NO, vaddr_t epc) {
  csr(MCAUSE) = NO;      // mcause = 11 (M-mode ecall)
  csr(MEPC) = epc;       // mepc = 当前PC
  return csr(MTVEC);     // 返回异常入口地址作为下一条PC
}
```

### 4.3 NPC 中的 ecall 硬件实现

IRU 模块负责 ecall 的 CSR 写入：

```verilog
// npc/vsrc/iru.v
// 当 ecall 信号有效时，写入 mepc 和 mcause
input i_ecall,
input [`CPU_WIDTH-1:0] i_pc,

// mepc = 当前 PC
assign o_mepc_wen = ecall_delayed && i_lsu_valid;
assign o_mepc_wdata = pc_delayed;

// mcause = 11 (IRQ_ECALL)
assign o_mcause_wen = ecall_delayed && i_lsu_valid;
assign o_mcause_wdata = `IRQ_ECALL;  // 32'd11
```

BRU 模块负责 PC 跳转：
```verilog
// npc/vsrc/bru.v
if (i_ecall) next_pc = i_mtvec;  // PC ← mtvec
```

---

## 五、mret 指令 — 从异常返回

### 5.1 mret 的语义

```
mret 执行时:
  1. PC ← mepc                   (返回到异常发生的位置)
  2. mstatus.MIE ← mstatus.MPIE (恢复中断使能状态)
  3. mstatus.MPIE ← 1           (MPIE 设为 1)
```

### 5.2 实现

```c
// NEMU: inst.c
INSTPAT("0011000 00010 00000 000 00000 11100 11", mret, R, s->dnpc = csr(MEPC));
```

```verilog
// NPC: bru.v
if (i_mret) next_pc = i_mepc;  // PC ← mepc
```

---

## 六、CSR 读写指令

### 6.1 三条 CSR 指令

| 指令 | 行为 | 用途 |
|------|------|------|
| `csrrw rd, csr, rs1` | t=CSR; CSR=rs1; rd=t | 读后写 (原子交换) |
| `csrrs rd, csr, rs1` | t=CSR; CSR=t\|rs1; rd=t | 读后置位 (Set bits) |
| `csrrc rd, csr, rs1` | t=CSR; CSR=t&~rs1; rd=t | 读后清位 (Clear bits) |

### 6.2 NEMU 实现

```c
INSTPAT("??????? ????? ????? 001 ????? 11100 11", csrrw, I,
  t = csr(imm), csr(imm) = src1, R(rd) = t);
INSTPAT("??????? ????? ????? 010 ????? 11100 11", csrrs, I,
  t = csr(imm), csr(imm) = t | src1, R(rd) = t);
```

注意：`imm` 在这里就是 12-bit CSR 地址（inst[31:20]），不需要符号扩展。

### 6.3 NPC 中 CSR 操作的数据通路

```
IDU ──csrsid──▶ CSRFile (读) ──csrs──▶ EXU (CSR运算)
IDU ──csrdid──▶ WBU ──▶ CSRFile (写回) ──wbu_csrd──▶
                                                  
EXU 中的 CSR 运算:
  csrrw: csrd = rs1          (新值 = rs1)
  csrrs: csrd = csrs | rs1   (新值 = 旧值 OR rs1)
  csrrc: csrd = csrs & ~rs1  (新值 = 旧值 AND NOT rs1)
  
rd 写回: rd = csrs (CSR 的旧值)
```

---

## 七、AM 的 CTE (Context Extension)

### 7.1 初始化异常处理 (`cte_init`)

```c
// abstract-machine/am/src/riscv/npc/cte.c
extern void __am_asm_trap(void);  // 汇编异常入口

bool cte_init(Context*(*handler)(Event, Context*)) {
  // 将 mtvec 设为汇编异常入口地址
  asm volatile("csrw mtvec, %0" : : "r"(__am_asm_trap));
  // 注册用户的事件处理函数
  user_handler = handler;
  return true;
}
```

### 7.2 汇编异常入口 (`trap.S`)

```asm
.globl __am_asm_trap
__am_asm_trap:
  # 1. 在栈上分配 Context 结构体空间
  addi sp, sp, -CONTEXT_SIZE

  # 2. 保存所有通用寄存器
  sw x1,  1*4(sp)    # ra
  sw x3,  3*4(sp)    # gp
  sw x4,  4*4(sp)    # tp
  ...
  sw x15, 15*4(sp)   # a5 (RV32E最后一个)

  # 3. 保存 CSR
  csrr t0, mcause
  csrr t1, mstatus
  csrr t2, mepc
  sw t0, OFFSET_CAUSE(sp)
  sw t1, OFFSET_STATUS(sp)
  sw t2, OFFSET_EPC(sp)

  # 4. 调用 C 语言异常处理函数
  mv a0, sp              # 参数 = Context 指针
  jal __am_irq_handle    # 调用 C 处理函数
  mv sp, a0              # 返回值 = (可能不同的) Context 指针

  # 5. 恢复 CSR
  lw t1, OFFSET_STATUS(sp)
  lw t2, OFFSET_EPC(sp)
  csrw mstatus, t1
  csrw mepc, t2

  # 6. 恢复所有通用寄存器
  lw x1,  1*4(sp)
  lw x3,  3*4(sp)
  ...
  lw x15, 15*4(sp)

  # 7. 释放栈空间并返回
  addi sp, sp, CONTEXT_SIZE
  mret                   # PC ← mepc, 恢复特权级
```

### 7.3 C 语言异常处理 (`__am_irq_handle`)

```c
Context* __am_irq_handle(Context *c) {
  if (user_handler) {
    Event ev = {0};
    switch (c->mcause) {
      case 0x0b:                        // Environment call from M-mode
        ev.event = EVENT_YIELD;
        c->mepc += 4;                   // ecall 返回到下一条指令
        break;
      default:
        ev.event = EVENT_ERROR;
        break;
    }
    c = user_handler(ev, c);            // 调用用户注册的处理函数
    assert(c != NULL);
  }
  return c;                             // 返回要恢复的上下文
}
```

**关键**: `c->mepc += 4` — ecall 返回时应该跳过 ecall 指令本身，执行其下一条。

### 7.4 Context 结构体

```c
// abstract-machine/am/include/arch/riscv.h
struct Context {
  uintptr_t gpr[GPR_NUM];     // 通用寄存器 (x0-x15 for RV32E)
  uintptr_t mcause;           // 异常原因
  uintptr_t mstatus;          // 机器状态
  uintptr_t mepc;             // 异常返回地址
  void *pdir;                 // 页表指针 (VME用，目前未用)
};
```

---

## 八、yield — 主动让出 CPU

### 8.1 实现

```c
void yield() {
  asm volatile("li a5, -1; ecall");  // RV32E: 用 a5 传参数
}
```

**完整流程**:
```
yield()
  → ecall 指令
    → 硬件: mepc=PC, mcause=11, PC=mtvec
      → __am_asm_trap (保存上下文)
        → __am_irq_handle (识别为 EVENT_YIELD)
          → user_handler (操作系统调度逻辑)
            → 可能切换到另一个 Context
          → 返回新的 Context 指针
        → 恢复新 Context 的寄存器
      → mret (PC = 新 Context 的 mepc)
```

### 8.2 上下文切换原理

如果 `user_handler` 返回一个**不同的 Context 指针**，`mret` 后就会恢复另一个任务的寄存器状态——这就是上下文切换。

```c
// 简化的调度器
Context* schedule(Event ev, Context *prev) {
  // 保存当前任务的 Context
  tasks[current].ctx = prev;
  // 切换到下一个任务
  current = (current + 1) % NUM_TASKS;
  // 返回下一个任务的 Context
  return tasks[current].ctx;
}
```

---

## 九、kcontext — 创建内核线程上下文

```c
Context *kcontext(Area kstack, void (*entry)(void *), void *arg) {
  // 在内核栈顶部构造一个 "假" Context
  Context *c = (Context *)(kstack.end - sizeof(Context));
  c->mepc = (uintptr_t)entry;     // "返回地址" 设为线程入口
  c->mstatus = 0x1800;            // MPP=M-mode
  c->gpr[10] = (uintptr_t)arg;   // a0 = 参数
  return c;
}
```

当调度器第一次切换到这个 Context 时，`mret` 会将 PC 设为 `entry`，开始执行新线程。

---

## 十、完整异常处理时序

```
时间轴 →

     ┌─ ecall ─┐
     │         │
  ···指令N··· ecall ··· __am_asm_trap(保存) ··· handler() ··· __am_asm_trap(恢复) ··· mret ··· 指令N+1···
     │         │           │                      │                │                    │
     PC=A    mepc=A      PC=mtvec            C 代码处理        mepc可能被修改        PC=mepc
             mcause=11                                         (+4 或切换任务)
             PC→mtvec
```

---

## 十一、常见问题与解决

### Q1: ecall 后 PC 跳到 0x00000000

**原因**：`mtvec` 未被正确初始化。确保 `cte_init()` 在 ecall 之前被调用。

### Q2: mret 后程序执行同一条 ecall 导致无限循环

**原因**：`mepc` 没有 +4。`__am_irq_handle` 中需要：
```c
c->mepc += 4;  // 跳过 ecall，返回到下一条
```

### Q3: DiffTest 在 ecall/mret 时报 CSR 不一致

**检查**：
1. NEMU 的 `isa_raise_intr` 是否正确设置了 mcause 和 mepc
2. NPC 的 IRU 是否在正确的时钟周期写入 CSR
3. mstatus 的 MPIE/MIE 位操作是否与 NEMU 一致

### Q4: 上下文切换后寄存器值被破坏

**检查**：
1. `trap.S` 是否保存了所有寄存器（包括 x2/sp）
2. Context 结构体中寄存器顺序是否与 trap.S 中的 PUSH/POP 顺序一致
3. `CONTEXT_SIZE` 是否正确计算

### Q5: csrrw 指令无法正确读写 CSR

**检查**：CSR 的"先读后写"必须是原子的——先把旧值存到临时变量/rd，再写入新值。在 RTL 中，读是组合逻辑（当周期可得），写是时序逻辑（下周期生效），天然满足这个要求。

---

## 十二、PA 讲义相关思考题

#### Q: 为什么 ecall 保存的是当前 PC 而不是 PC+4？

**回答**：这是 RISC-V 的设计选择。对于异常（exception），`mepc` 保存触发异常的指令地址，方便：
1. 处理完异常后可以**重新执行**该指令（如缺页异常：加载页面后重试访存）
2. 如果需要跳过，软件可以手动 `mepc += 4`

对于中断（interrupt），`mepc` 保存被中断的下一条指令地址（因为当前指令已经执行完了）。

---

#### Q: 为什么 trap.S 不保存 x0 和 x2(sp)？

**回答**：
- `x0` = 0，永远不变，不需要保存
- `x2 (sp)` 特殊处理：进入 trap 时先用 `addi sp, sp, -SIZE` 分配空间，sp 本身的旧值可以通过 `sp + SIZE` 恢复。或者在 Context 中额外记录

---

#### Q: CTE 的抽象意义是什么？

**回答**：CTE 将异常处理抽象为"事件驱动"模型：

```c
// 应用层只关心事件类型，不关心硬件细节
Context* handler(Event ev, Context *ctx) {
  switch (ev.event) {
    case EVENT_YIELD:   return schedule(ctx);  // 调度
    case EVENT_SYSCALL: return do_syscall(ctx); // 系统调用
  }
}
```

底层的 CSR 操作、寄存器保存/恢复全部被 AM 封装。操作系统开发者无需了解 `mcause` 的编码细节。

---

## 十三、学习建议

1. **先实现 CSR 读写指令** (csrrw/csrrs/csrrc) — 不涉及异常，纯数据操作
2. **再实现 ecall + mret** — 最基本的异常进入和返回
3. **用 `am-tests` 中的 yield-test 验证** — 它只做 yield() 然后返回
4. **最后实现 mstatus 的 MPIE/MIE** — 这是 DiffTest 最容易报不一致的地方
5. **阅读 trap.S** — 理解每一行汇编的作用

---

*参考资料：*
- *RISC-V Privileged ISA Specification (Volume II)*
- *PA3 "穿越时空的旅程" 讲义*
- *项目源码: npc/vsrc/csrfile.v, npc/vsrc/iru.v, abstract-machine/am/src/riscv/npc/cte.c, trap.S*


---

## 十四、本阶段 Git 版本记录

| Commit | 说明 | 完成内容 |
|--------|------|----------|
| `1c64d37` | feat: nemu rt-thread | NEMU 实现 ecall/mret/CSR，成功运行 RT-Thread |
| `51b3f3b` | feat: NPC rt-thread | NPC 硬件实现 CSRFile + IRU，RT-Thread 启动 |

**查看对应版本代码**：
```bash
git show 1c64d37:nemu/src/isa/riscv32/inst.c         # NEMU ecall/mret/csrrw
git show 1c64d37:nemu/src/isa/riscv32/system/intr.c  # NEMU isa_raise_intr
git show 51b3f3b:npc/vsrc/csrfile.v                  # NPC CSR 寄存器文件
git show 51b3f3b:npc/vsrc/iru.v                      # NPC 中断/异常处理单元
git show 51b3f3b:abstract-machine/am/src/riscv/npc/cte.c  # AM CTE 实现
```


---

## 十五、Git 版本代码演进详解

### 15.1 NEMU RT-Thread (`1c64d37`)

此 commit 在 NEMU 中实现了异常处理所需的全部指令:

**inst.c 新增的指令**:
```c
// CSR 读写指令
INSTPAT("??????? ????? ????? 001 ????? 11100 11", csrrw, I, t=csr(imm), csr(imm)=src1, R(rd)=t);
INSTPAT("??????? ????? ????? 010 ????? 11100 11", csrrs, I, t=csr(imm), csr(imm)=t|src1, R(rd)=t);

// 异常/返回指令
INSTPAT("0000000 00000 00000 000 00000 11100 11", ecall, I, s->dnpc=isa_raise_intr(0x0b, s->pc));
INSTPAT("0011000 00010 00000 000 00000 11100 11", mret,  R, s->dnpc=csr(MEPC));
```

**intr.c 新增**:
```c
word_t isa_raise_intr(word_t NO, vaddr_t epc) {
  csr(MCAUSE) = NO;    // 异常原因 = 11 (ecall from M-mode)
  csr(MEPC) = epc;     // 保存触发异常的 PC
  return csr(MTVEC);   // 返回异常入口地址
}
```

**验证**: NEMU 成功运行 RT-Thread，输出 `msh />` 提示符。

---

### 15.2 NPC RT-Thread (`51b3f3b`) — 完整硬件重构

这是项目中**最大的单次重构** (+1107/-803 行)。

**新增核心模块**:

| 新文件 | 行数 | 功能 |
|--------|------|------|
| `csrfile.v` | 121 行 | 4 个 CSR 寄存器 + 双写入源 |
| `idu/idu_system.v` | 47 行 | ecall/mret/csrrw/csrrs/csrrc 译码 |
| `exu.v` | 136 行 | 含 CSR 运算 (RW/RS/RC) |
| `top.v` | 191 行 | 全新顶层 (含 CSR 信号连接) |

**CSR 文件初始版本** (commit `51b3f3b`) 要点:
- 4 个 CSR: mstatus (复位值=0x1800), mtvec, mepc, mcause
- 双写入源: `csrrw 指令写入` OR `ecall 异常写入` (异常优先)
- `sim_csr[4]` 数组通过 DPI-C 暴露给 C++ (用于 DiffTest)
- mstatus 复位值 `0x1800` = MPP 字段为 M-mode

**PCU (后改名 BRU) 中的跳转逻辑**:
```verilog
if (i_ecall)      next_pc = i_mtvec;   // ecall → 跳转到异常入口
else if (i_mret)  next_pc = i_mepc;    // mret → 返回异常前位置
else if (brch && zero) next_pc = pc + imm;  // 分支
else              next_pc = pc + 4;     // 顺序
```

**验证结果**: NPC 成功启动 RT-Thread，但存在末尾 `msh />` 重复输出的问题（commit message 中提到）。

---

### 15.3 第一版 CSR vs 最终版 CSR 对比

| 方面 | 初版 (`51b3f3b`) | 最终版 (当前 HEAD) |
|------|-----------------|-------------------|
| CSR 数量 | 4 个 (mstatus/mtvec/mepc/mcause) | 6 个 (+mvendorid/marchid) |
| mstatus 复位值 | `0x1800` | `0x0` (由软件初始化) |
| 寄存器实现 | 直接 `reg` 声明 | 统一用 `stdreg` 模块 |
| IRU | 无独立模块 | 独立 `iru.v` (含流水线寄存器) |
| DPI-C 接口 | `set_csr_ptr(sim_csr)` | 同 (保持兼容) |
