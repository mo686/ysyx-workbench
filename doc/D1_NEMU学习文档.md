# D1: 支持 RV32IM 的 NEMU — 详细学习文档

## 一、阶段目标

D1 阶段的核心目标是实现一个能够正确执行 RISC-V 32 位整数指令集 (RV32IM) 的全系统模拟器 NEMU。完成后你将：

1. 深入理解"取指→译码→执行→更新PC"的指令执行循环
2. 掌握 RISC-V 指令编码格式和语义
3. 实现一个功能完备的简易调试器 (表达式求值、监视点)
4. 理解程序在裸金属环境下的执行过程

---

## 二、NEMU 整体架构

### 2.1 四大模块

```
┌──────────────────────────────────────────────────────────────┐
│                          NEMU                                 │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │ Monitor  │  │   CPU    │  │  Memory  │  │  Device  │    │
│  │ (调试器) │  │ (执行核心)│  │ (内存模拟)│  │ (外设模拟)│    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
│                                                              │
│  启动流程: main() → init_monitor() → engine_start()          │
│           → sdb_mainloop() (交互式调试)                       │
│           或 cmd_c() (批处理模式直接运行)                      │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 启动流程 (`nemu-main.c`)

```c
int main(int argc, char *argv[]) {
  init_monitor(argc, argv);  // 初始化: 加载镜像、初始化内存、初始化设备
  engine_start();            // 进入主循环: sdb 交互 或 直接运行
  return is_exit_status_bad();
}
```

`init_monitor()` 完成：
- 解析命令行参数 (镜像文件路径等)
- 初始化物理内存 (`init_mem()`)
- 加载用户程序 (.bin 镜像) 到内存起始地址 (0x80000000)
- 初始化 CPU 寄存器 (PC = 0x80000000)
- 初始化调试器 (编译正则表达式、初始化监视点池)

---

## 三、CPU 状态定义

### 3.1 问题：如何用 C 语言表示 CPU 的硬件状态？

**解决方案**：用结构体表示 CPU 的所有可编程寄存器。

```c
// src/isa/riscv32/include/isa-def.h
typedef struct {
  word_t gpr[32];   // 32 个通用寄存器 (x0-x31)
  vaddr_t pc;       // 程序计数器
  word_t csr[4];    // CSR 寄存器: mstatus, mtvec, mepc, mcause
} riscv32_CPU_state;
```

**知识点**：
- `word_t` = `uint32_t` (32-bit 无符号整数)
- `gpr[0]` 即 `x0` 寄存器，硬连线为 0 (每条指令执行后强制清零)
- RV32I 标准定义 32 个寄存器，RV32E 精简为 16 个

### 3.2 寄存器别名

```c
// src/isa/riscv32/reg.c
const char *regs[] = {
  "$0", "ra", "sp", "gp", "tp", "t0", "t1", "t2",
  "s0", "s1", "a0", "a1", "a2", "a3", "a4", "a5",
  "a6", "a7", "s2", "s3", "s4", "s5", "s6", "s7",
  "s8", "s9", "s10", "s11", "t3", "t4", "t5", "t6"
};
```

| 别名 | 编号 | 用途 |
|------|------|------|
| `$0` (zero) | x0 | 常量 0 |
| `ra` | x1 | 返回地址 |
| `sp` | x2 | 栈指针 |
| `a0-a7` | x10-x17 | 函数参数/返回值 |
| `t0-t6` | x5-x7, x28-x31 | 临时寄存器 |
| `s0-s11` | x8-x9, x18-x27 | 被调用者保存 |

---

## 四、指令执行循环 (核心)

### 4.1 问题：CPU 如何一条条执行指令？

**解决方案**：实现 `cpu_exec()` → `execute()` → `exec_once()` 三层调用。

```c
// src/cpu/cpu-exec.c

/* 模拟 CPU 工作 */
void cpu_exec(uint64_t n) {
  // 1. 检查状态：如果已经结束/中止，直接返回
  switch (nemu_state.state) {
    case NEMU_END: case NEMU_ABORT:
      printf("Program execution has ended.\n");
      return;
    default: nemu_state.state = NEMU_RUNNING;
  }

  // 2. 计时开始
  uint64_t timer_start = get_time();

  // 3. 执行 n 条指令
  execute(n);

  // 4. 计时结束，统计信息
  g_timer += get_time() - timer_start;

  // 5. 检查退出状态 (GOOD TRAP / BAD TRAP)
  ...
}
```

### 4.2 execute() — 执行 n 条指令的循环

```c
static void execute(uint64_t n) {
  Decode s;  // 译码结果结构体
  for (; n > 0; n--) {
    exec_once(&s, cpu.pc);          // 取指+译码+执行 一条指令
    g_nr_guest_inst++;              // 指令计数++
    trace_and_difftest(&s, cpu.pc); // 追踪+差分测试+监视点检查
    if (nemu_state.state != NEMU_RUNNING) break;  // 遇到 trap 或错误则停止
    IFDEF(CONFIG_DEVICE, device_update());  // 更新设备状态
  }
}
```

### 4.3 exec_once() — 一条指令的完整生命周期

```c
static void exec_once(Decode *s, vaddr_t pc) {
  s->pc = pc;        // 记录当前 PC
  s->snpc = pc;      // snpc: static next PC (顺序下一条)
  isa_exec_once(s);  // ISA 相关的取指+译码+执行
  cpu.pc = s->dnpc;  // dnpc: dynamic next PC (可能被跳转修改)
  // ... ITRACE 记录、IRINGBUF 更新
}
```

**关键概念**：
- `pc`: 当前指令地址
- `snpc` (Static Next PC): `pc + 4` (RISC-V 固定 4 字节指令)
- `dnpc` (Dynamic Next PC): 实际下一条指令地址 (跳转时 ≠ snpc)

### 4.4 isa_exec_once() — ISA 相关的指令处理

```c
// src/isa/riscv32/inst.c
int isa_exec_once(Decode *s) {
  s->isa.inst.val = inst_fetch(&s->snpc, 4);  // 取指: 从内存读 4 字节
  return decode_exec(s);                        // 译码+执行
}
```

---

## 五、指令译码框架 (INSTPAT 模式匹配)

### 5.1 问题：如何优雅地实现几十条指令的译码？

**解决方案**：使用宏定义实现"指令模式匹配"框架 (`INSTPAT`)。

```c
// 模式: "funct7  rs2   rs1  funct3 rd    opcode"
INSTPAT("0000000 ????? ????? 000 ????? 01100 11", add, R, R(rd) = src1 + src2);
//       ─────── ───── ───── ─── ───── ───── ──
//       7位     5位   5位  3位  5位   7位       = 32 位
```

- `?` 表示"匹配任意位"
- 固定位用于区分不同指令
- 最后的 C 表达式是执行体

### 5.2 六种指令格式的立即数提取

```c
#define immI() do { *imm = SEXT(BITS(i, 31, 20), 12); } while(0)
#define immU() do { *imm = SEXT(BITS(i, 31, 12), 20) << 12; } while(0)
#define immS() do { *imm = (SEXT(BITS(i, 31, 25), 7) << 5) | BITS(i, 11, 7); } while(0)
#define immJ() do { *imm = (SEXT(BITS(i, 31, 31), 1) << 20) | (BITS(i, 19, 12) << 12) 
                          | (BITS(i, 20, 20) << 11) | (BITS(i, 30, 21) << 1); } while(0)
#define immB() do { *imm = (SEXT(BITS(i, 31, 31), 1) << 12) | (BITS(i, 7, 7) << 11) 
                          | (BITS(i, 30, 25) << 5) | (BITS(i, 11, 8) << 1); } while(0)
```

**知识点**：
- `BITS(x, hi, lo)` — 提取 x 的第 hi 到 lo 位
- `SEXT(x, len)` — 符号扩展: 将 len 位的数扩展为 32 位 (保留符号)
- J/B 类型的立即数位域是打散的 (RISC-V 设计的 trade-off: 简化硬件 vs 复杂软件)

### 5.3 decode_operand() — 根据指令类型提取操作数

```c
static void decode_operand(Decode *s, int *rd, word_t *src1, word_t *src2, word_t *imm, int type) {
  uint32_t i = s->isa.inst.val;         // 32 位指令原始值
  int rs1 = BITS(i, 19, 15);           // 源寄存器 1 编号
  int rs2 = BITS(i, 24, 20);           // 源寄存器 2 编号
  *rd     = BITS(i, 11, 7);            // 目的寄存器编号
  switch (type) {
    case TYPE_I: src1R();          immI(); break;  // I 型: rs1 + 立即数
    case TYPE_U:                   immU(); break;  // U 型: 仅立即数
    case TYPE_S: src1R(); src2R(); immS(); break;  // S 型: rs1 + rs2 + 立即数
    case TYPE_J:                   immJ(); break;  // J 型: 仅立即数(跳转偏移)
    case TYPE_R: src1R(); src2R();         break;  // R 型: rs1 + rs2
    case TYPE_B: src1R(); src2R(); immB(); break;  // B 型: rs1 + rs2 + 立即数
  }
}
```

---

## 六、RV32IM 指令实现详解

### 6.1 算术指令 (R 型)

```c
INSTPAT("0000000 ????? ????? 000 ????? 01100 11", add,  R, R(rd) = src1 + src2);
INSTPAT("0100000 ????? ????? 000 ????? 01100 11", sub,  R, R(rd) = src1 - src2);
INSTPAT("0000000 ????? ????? 001 ????? 01100 11", sll,  R, R(rd) = src1 << BITS(src2, 4, 0));
INSTPAT("0000000 ????? ????? 010 ????? 01100 11", slt,  R, R(rd) = ((sword_t)src1 < (sword_t)src2));
INSTPAT("0000000 ????? ????? 011 ????? 01100 11", sltu, R, R(rd) = (src1 < src2));
INSTPAT("0000000 ????? ????? 100 ????? 01100 11", xor,  R, R(rd) = src1 ^ src2);
INSTPAT("0000000 ????? ????? 101 ????? 01100 11", srl,  R, R(rd) = src1 >> BITS(src2, 4, 0));
INSTPAT("0100000 ????? ????? 101 ????? 01100 11", sra,  R, R(rd) = (sword_t)src1 >> BITS(src2, 4, 0));
INSTPAT("0000000 ????? ????? 110 ????? 01100 11", or,   R, R(rd) = src1 | src2);
INSTPAT("0000000 ????? ????? 111 ????? 01100 11", and,  R, R(rd) = src1 & src2);
```

**易错点**：
- `sra` 必须用 `(sword_t)` 强制有符号右移 (算术右移 vs 逻辑右移)
- 移位量只取低 5 位: `BITS(src2, 4, 0)`

### 6.2 M 扩展 (乘除法)

```c
INSTPAT("0000001 ????? ????? 000 ????? 01100 11", mul,   R, R(rd) = src1 * src2);
INSTPAT("0000001 ????? ????? 001 ????? 01100 11", mulh,  R, R(rd) = (SEXT(src1,32) * SEXT(src2,32)) >> 32);
INSTPAT("0000001 ????? ????? 011 ????? 01100 11", mulhu, R, R(rd) = ((uint64_t)src1 * (uint64_t)src2) >> 32);
INSTPAT("0000001 ????? ????? 100 ????? 01100 11", div,   R, R(rd) = (sword_t)src1 / (sword_t)src2);
INSTPAT("0000001 ????? ????? 101 ????? 01100 11", divu,  R, R(rd) = src1 / src2);
INSTPAT("0000001 ????? ????? 110 ????? 01100 11", rem,   R, R(rd) = (sword_t)src1 % (sword_t)src2);
INSTPAT("0000001 ????? ????? 111 ????? 01100 11", remu,  R, R(rd) = src1 % src2);
```

**易错点**：
- `mulh` 需要将操作数符号扩展到 64 位再相乘，取高 32 位
- `div` 和 `rem` 要用有符号类型 `sword_t`
- 除零行为：RISC-V 规范规定除零不触发异常，结果为全 1 (div) 或被除数 (rem)

### 6.3 立即数算术 (I 型)

```c
INSTPAT("??????? ????? ????? 000 ????? 00100 11", addi, I, R(rd) = src1 + imm);
INSTPAT("??????? ????? ????? 010 ????? 00100 11", slti, I, R(rd) = ((sword_t)src1 < (sword_t)imm));
INSTPAT("??????? ????? ????? 011 ????? 00100 11", sltiu,I, R(rd) = (src1 < imm));
INSTPAT("??????? ????? ????? 100 ????? 00100 11", xori, I, R(rd) = src1 ^ imm);
INSTPAT("??????? ????? ????? 110 ????? 00100 11", ori,  I, R(rd) = src1 | imm);
INSTPAT("??????? ????? ????? 111 ????? 00100 11", andi, I, R(rd) = src1 & imm);
INSTPAT("000000? ????? ????? 001 ????? 00100 11", slli, I, R(rd) = src1 << BITS(imm, 5, 0));
INSTPAT("000000? ????? ????? 101 ????? 00100 11", srli, I, R(rd) = src1 >> BITS(imm, 5, 0));
INSTPAT("010000? ????? ????? 101 ????? 00100 11", srai, I, R(rd) = (sword_t)src1 >> BITS(imm, 5, 0));
```

### 6.4 访存指令 (Load/Store)

```c
// Load 指令 (I 型)
INSTPAT("??????? ????? ????? 000 ????? 00000 11", lb,  I, R(rd) = SEXT(Mr(src1+imm, 1), 8));
INSTPAT("??????? ????? ????? 001 ????? 00000 11", lh,  I, R(rd) = SEXT(Mr(src1+imm, 2), 16));
INSTPAT("??????? ????? ????? 010 ????? 00000 11", lw,  I, R(rd) = Mr(src1+imm, 4));
INSTPAT("??????? ????? ????? 100 ????? 00000 11", lbu, I, R(rd) = Mr(src1+imm, 1));
INSTPAT("??????? ????? ????? 101 ????? 00000 11", lhu, I, R(rd) = Mr(src1+imm, 2));

// Store 指令 (S 型)
INSTPAT("??????? ????? ????? 000 ????? 01000 11", sb, S, Mw(src1+imm, 1, BITS(src2, 7, 0)));
INSTPAT("??????? ????? ????? 001 ????? 01000 11", sh, S, Mw(src1+imm, 2, BITS(src2, 15, 0)));
INSTPAT("??????? ????? ????? 010 ????? 01000 11", sw, S, Mw(src1+imm, 4, src2));
```

**知识点**：
- `Mr(addr, len)` = `vaddr_read(addr, len)` 读内存
- `Mw(addr, len, data)` = `vaddr_write(addr, len, data)` 写内存
- `lb` 读 1 字节并**符号扩展** (负数保持), `lbu` 读 1 字节并**零扩展**

### 6.5 跳转和分支

```c
// 无条件跳转
INSTPAT("??????? ????? ????? ??? ????? 11011 11", jal,  J, R(rd) = s->pc+4, s->dnpc = s->pc+imm);
INSTPAT("??????? ????? ????? 000 ????? 11001 11", jalr, I, t=s->pc+4, s->dnpc=(src1+imm)&~1, R(rd)=t);

// 条件分支
INSTPAT("??????? ????? ????? 000 ????? 11000 11", beq,  B, s->dnpc = (src1==src2) ? s->pc+imm : s->snpc);
INSTPAT("??????? ????? ????? 001 ????? 11000 11", bne,  B, s->dnpc = (src1!=src2) ? s->pc+imm : s->snpc);
INSTPAT("??????? ????? ????? 100 ????? 11000 11", blt,  B, s->dnpc = ((sword_t)src1<(sword_t)src2) ? s->pc+imm : s->snpc);
INSTPAT("??????? ????? ????? 101 ????? 11000 11", bge,  B, s->dnpc = ((sword_t)src1>=(sword_t)src2) ? s->pc+imm : s->snpc);
INSTPAT("??????? ????? ????? 110 ????? 11000 11", bltu, B, s->dnpc = (src1<src2) ? s->pc+imm : s->snpc);
INSTPAT("??????? ????? ????? 111 ????? 11000 11", bgeu, B, s->dnpc = (src1>=src2) ? s->pc+imm : s->snpc);
```

**关键理解**：
- `jal`: 保存返回地址 (`pc+4`) 到 `rd`，跳转到 `pc + imm`
- `jalr`: 保存返回地址到 `rd`，跳转到 `(rs1 + imm) & ~1` (清除最低位)
- 分支指令修改 `dnpc`，不满足条件时 `dnpc = snpc` (顺序执行)

### 6.6 特殊指令

```c
// 上地址立即数
INSTPAT("??????? ????? ????? ??? ????? 01101 11", lui,   U, R(rd) = imm);
INSTPAT("??????? ????? ????? ??? ????? 00101 11", auipc, U, R(rd) = s->pc + imm);

// 系统指令
INSTPAT("0000000 00001 00000 000 00000 11100 11", ebreak, N, NEMUTRAP(s->pc, R(10)));
INSTPAT("0000000 00000 00000 000 00000 11100 11", ecall,  I, s->dnpc = isa_raise_intr(0x0b, s->pc));

// 兜底: 匹配所有未实现的指令
INSTPAT("??????? ????? ????? ??? ????? ????? ??", inv, N, INV(s->pc));
```

**`ebreak` 的特殊作用**：在 NEMU 中用作程序结束标志。`R(10)` 即 `a0` 寄存器的值：
- `a0 == 0` → HIT GOOD TRAP (程序正确结束)
- `a0 != 0` → HIT BAD TRAP (程序出错)

---

## 七、内存子系统

### 7.1 问题：如何模拟物理内存？

**解决方案**：用一个大数组模拟物理内存。

```c
// src/memory/paddr.c
static uint8_t pmem[CONFIG_MSIZE] PG_ALIGN = {};  // 例如 128MB

// 客户物理地址 → 主机虚拟地址 的转换
uint8_t* guest_to_host(paddr_t paddr) {
  return pmem + paddr - CONFIG_MBASE;  // CONFIG_MBASE = 0x80000000
}
```

### 7.2 内存读写

```c
word_t paddr_read(paddr_t addr, int len) {
  if (likely(in_pmem(addr))) {        // 在物理内存范围内
    return pmem_read(addr, len);
  }
  IFDEF(CONFIG_DEVICE, return mmio_read(addr, len));  // MMIO 设备
  out_of_bound(addr);  // 越界 → panic
  return 0;
}
```

**知识点**：
- `likely()` 是分支预测提示宏，优化常见路径
- 地址访问分为三种情况：物理内存 / MMIO 设备 / 越界
- `CONFIG_MBASE = 0x80000000` 是 RISC-V 物理内存的约定起始地址

### 7.3 MTRACE (内存访问追踪)

```c
#ifdef CONFIG_MTRACE
  if (addr >= CONFIG_MTRACE_START && addr <= CONFIG_MTRACE_START + CONFIG_MTRACE_SIZE) {
    Log("MTRACE: Read Memory Address: 0x%x, len: %d, data: 0x%x", addr, len, ret);
  }
#endif
```

在 Kconfig 中设置监控的地址范围，即可打印所有落在该范围内的内存访问。

---

## 八、调试器 (SDB) 实现

### 8.1 问题：如何在模拟器中实现类似 GDB 的调试功能？

### 8.2 命令系统 (`sdb.c`)

```c
static struct {
  const char *name;
  const char *description;
  int (*handler)(char *);
} cmd_table[] = {
  {"help", "Display information about all supported commands", cmd_help},
  {"c",    "Continue the execution of the program", cmd_c},
  {"q",    "Exit NEMU", cmd_q},
  {"si",   "Step over, default N=1", cmd_si},
  {"info", "info r: print register; info w: print watchpoint", cmd_info},
  {"x",    "x N EXPR: print N*4 bytes from addr=EXPR", cmd_x},
  {"p",    "p EXPR: evaluate the expression", cmd_p},
  {"w",    "w EXPR: set a watchpoint", cmd_w},
  {"d",    "d N: delete watchpoint N", cmd_d},
};
```

主循环使用 `readline` 库提供命令行补全和历史记录。

### 8.3 表达式求值 (`expr.c`) — 重要难点

**实现步骤**：

**Step 1: 词法分析 (Tokenize)**

使用 POSIX 正则表达式将输入字符串切分为 token：
```c
static struct rule {
  const char *regex;
  int token_type;
} rules[] = {
  {" +", TK_NOTYPE},                    // 空格 (忽略)
  {"0[xX][0-9a-fA-F]+", TK_HEXNUM},   // 十六进制数
  {"[0-9]+", TK_NUM},                   // 十进制数
  {"\\$[$]?[0-9a-z]+", TK_REG},       // 寄存器 ($pc, $a0...)
  {"\\+", '+'},                         // 加号
  {"\\-", '-'},                         // 减号 (或负号)
  {"\\*", '*'},                         // 乘号 (或解引用)
  {"\\/", '/'},                         // 除号
  {"\\(", '('},                         // 左括号
  {"\\)", ')'},                         // 右括号
  {"==", TK_EQ},                        // 相等
  {"!=", TK_NOEQ},                      // 不等
  {"&&", TK_AND},                       // 逻辑与
  {"\\|\\|", TK_OR},                    // 逻辑或
};
```

**Step 2: 一元运算符识别**

负号和解引用需要根据上下文判断：
```c
// 如果 '-' 前面不是数字/寄存器/右括号，则为负号 (而非减号)
if (tokens[i].type == '-' && (i == 0 || prev_is_not_operand)) {
  tokens[i].type = TK_NEG;
}
// 如果 '*' 前面不是数字/寄存器/右括号，则为解引用 (而非乘号)
if (tokens[i].type == '*' && (i == 0 || prev_is_not_operand)) {
  tokens[i].type = TK_DEREF;
}
```

**Step 3: 递归求值 (Recursive Descent)**

```c
static word_t eval(int p, int q, bool *success) {
  if (p > q) { *success = false; return 0; }   // 非法
  
  if (p == q) {  // 单个 token: 数字或寄存器
    switch (tokens[p].type) {
      case TK_NUM:    sscanf(tokens[p].str, "%u", &num); break;
      case TK_HEXNUM: sscanf(tokens[p].str, "%x", &num); break;
      case TK_REG:    num = isa_reg_str2val(tokens[p].str, success); break;
    }
    return num;
  }
  
  if (check_parentheses(p, q)) {  // 被括号包围
    return eval(p + 1, q - 1, success);
  }
  
  // 找主运算符 (优先级最低的)
  int op = dominant_op(p, q);
  word_t val1 = eval(p, op - 1, success);  // 递归求左子表达式
  word_t val2 = eval(op + 1, q, success);  // 递归求右子表达式
  
  switch (tokens[op].type) {
    case '+': return val1 + val2;
    case '-': return val1 - val2;
    case '*': return val1 * val2;
    case '/': return val1 / val2;  // 注意除零检查
    case TK_DEREF: return vaddr_read(val2, 4);  // 读内存
    case TK_NEG:   return -val2;
    ...
  }
}
```

**关键算法 — 找主运算符** (`dominant_op`)：
- 从左到右扫描，跳过括号内的部分
- 选择优先级**最低**的运算符 (它最后执行 = 表达式树的根)
- 同优先级取**最右边**的 (左结合性)

### 8.4 监视点 (`watchpoint.c`)

**数据结构**：使用链表池管理

```c
#define NR_WP 32
static WP wp_pool[NR_WP];      // 预分配的监视点池
static WP *head = NULL;        // 活跃监视点链表
static WP *free_ = NULL;       // 空闲监视点链表
```

**工作原理**：每执行一条指令后调用 `check_wp()`：
```c
bool check_wp() {
  WP *tmp = head;
  while (tmp != NULL) {
    word_t ans = expr(tmp->e, &success);  // 重新求值表达式
    if (ans != tmp->value) {               // 值变化了!
      printf("Hit watchpoint %d: %s\n", tmp->NO, tmp->e);
      printf("Old = %u, New = %u\n", tmp->value, ans);
      tmp->value = ans;
      return true;  // 暂停执行
    }
    tmp = tmp->next;
  }
  return false;
}
```

---

## 九、指令追踪 (ITRACE + IRINGBUF)

### 9.1 ITRACE — 实时打印每条指令

```c
// 在 exec_once() 中格式化指令信息到 logbuf
// 格式: "PC: 机器码字节  反汇编"
// 例如: "0x80000000: 00000297  auipc t0, 0"
```

### 9.2 IRINGBUF — 环形缓冲

**问题**：程序崩溃时如何回溯最近执行的指令？

**解决方案**：维护一个固定大小的环形缓冲区，只保留最近 N 条指令。

```c
#define IRINGBUF_SIZE 32  // 保留最近 8 条指令 (32字节/4字节=8条)
RingBuffer *rb = NULL;

// 每条指令执行后，写入环形缓冲
if (RingBuffer_IsFull(rb)) {
  RingBuffer_Out(rb, &tmp, 4);   // 满了就丢弃最老的
}
RingBuffer_In(rb, &s->isa.inst.val, 4);  // 写入新指令
```

**崩溃时 (`assert_fail_msg()`) 打印**：
```c
printf("[Last several instructions for debug.]\n");
for (int i = 0; i < IRINGBUF_SIZE / 4; i++) {
  RingBuffer_Out(rb, &dest, 4);
  printf("instr[%d]: 0x%08x\n", i, dest);
}
```

---

## 十、常见问题与解决方案

### Q1: `addi` 和 `add` 的区别是什么？
- `add rd, rs1, rs2` — 两个寄存器相加，结果写入 rd
- `addi rd, rs1, imm` — 寄存器加立即数 (12 位符号扩展)
- 在 opcode 中: `add` = `0110011`, `addi` = `0010011`

### Q2: 为什么 `x0` 每条指令后要清零？
```c
R(0) = 0; // reset $zero to 0 (在 decode_exec 最后)
```
因为有些指令可能将 `x0` 作为目的寄存器 (如 `beq x0, x1, offset` 后面不需要写入)，硬件中 x0 硬连线为 0，软件模拟必须显式保持。

### Q3: 有符号 vs 无符号的混淆
- `slt` / `blt` / `bge` — 有符号比较 (cast to `sword_t`)
- `sltu` / `bltu` / `bgeu` — 无符号比较 (直接比较 `word_t`)
- `sra` — 算术右移 (保留符号位)
- `srl` — 逻辑右移 (高位补零)

### Q4: `jalr` 为什么要 `& ~1`？
RISC-V 规范要求: `jalr` 的目标地址低位清零 (对齐到偶数地址)，虽然 RV32I 中指令总是 4 字节对齐，但规范为压缩指令预留了 2 字节对齐的可能。

### Q5: `lui` 和 `auipc` 配合 `addi` 构建 32 位常量
```asm
lui  t0, 0x12345      # t0 = 0x12345000
addi t0, t0, 0x678    # t0 = 0x12345678
```
`auipc` 则是 PC 相对寻址，常用于生成地址。

### Q6: 如何调试指令实现错误？
1. 运行 `am-kernels/tests/cpu-tests` 中的测试
2. 如果某个测试 HIT BAD TRAP:
   - 开启 ITRACE 查看执行了哪些指令
   - 对比反汇编和 RISC-V ISA 手册
   - 用 `info r` 检查寄存器是否符合预期
   - 开启差分测试 (DiffTest) 与 Spike 对比

### Q7: 表达式求值中的坑
- 正则匹配顺序很重要: `0x` 前缀的十六进制必须在纯数字之前匹配
- 负号 `-3` vs 减法 `5-3`: 看前一个 token 是否为操作数
- 指针解引用 `*0x80000000` vs 乘法 `3*4`: 同理看上下文

---

## 十一、测试与验证

### 11.1 cpu-tests 测试流程

```bash
cd am-kernels/tests/cpu-tests
make ARCH=riscv32-nemu ALL=add run    # 运行单个测试 (add)
make ARCH=riscv32-nemu run            # 运行所有测试
```

每个测试程序以 `ebreak` 结束:
- `a0 == 0` → GOOD TRAP (通过)
- `a0 != 0` → BAD TRAP (失败)

### 11.2 表达式测试

项目中提供了表达式随机生成器 (`tools/gen-expr/`)，生成大量测试用例验证表达式求值的正确性：

```c
// nemu-main.c 中的 test_expr()
// 从 /tmp/.input 读取 "expected_value expression" 格式的测试数据
// 逐行验证 expr() 的输出是否等于 expected_value
```

### 11.3 用 gen-expr 生成测试

```bash
cd tools/gen-expr
make
./gen-expr 10000 > /tmp/.input   # 生成 10000 个随机表达式
```

---

## 十二、关键代码路径总结

一条 `addi a0, a1, 5` 指令的完整执行路径：

```
main()
 └─ engine_start()
     └─ sdb_mainloop()
         └─ cmd_si("1")       // 用户输入 si
             └─ cpu_exec(1)
                 └─ execute(1)
                     └─ exec_once(&s, cpu.pc)
                         ├─ isa_exec_once(s)
                         │   ├─ inst_fetch(&s->snpc, 4)  // 从 pmem[pc-0x80000000] 读 4 字节
                         │   └─ decode_exec(s)
                         │       ├─ INSTPAT 匹配到 "addi"
                         │       ├─ decode_operand(s, &rd, &src1, &src2, &imm, TYPE_I)
                         │       │   ├─ rs1 = BITS(inst, 19, 15) → 11 (a1)
                         │       │   ├─ *rd = BITS(inst, 11, 7) → 10 (a0)
                         │       │   ├─ src1R() → *src1 = R(11) = gpr[11] 的值
                         │       │   └─ immI() → *imm = SEXT(BITS(inst, 31, 20), 12) = 5
                         │       ├─ R(rd) = src1 + imm  // gpr[10] = gpr[11] + 5
                         │       └─ R(0) = 0            // 强制 x0 = 0
                         └─ cpu.pc = s->dnpc            // PC += 4
                     └─ trace_and_difftest(&s, cpu.pc)
                         ├─ ITRACE: 打印指令日志
                         ├─ difftest_step(): 与参考模型对比
                         └─ check_wp(): 检查监视点
```

---

## 十三、学习建议

1. **先读代码再写代码**：RTFSC (Read The Fucking Source Code) 是一生一芯的核心方法论
2. **增量开发**：先实现 `addi`、`sw`、`lw`、`jal` 等基本指令，跑通 `dummy` 测试，再逐步添加
3. **善用差分测试**：一旦实现够多指令，开启 DiffTest 可以快速发现语义错误
4. **查阅 ISA 手册**：RISC-V 官方 spec (riscv-spec-20191213.pdf) 是最权威参考
5. **表达式求值**是独立的编程练习，可以先写好单独测试，再集成到 sdb

---

*参考资料：*
- *RISC-V ISA Specification v2.2*
- *南京大学 ICS PA 实验讲义*
- *一生一芯官方文档: https://ysyx.oscc.cc*
- *项目源码: nemu/src/*


---

## 十四、PA 讲义思考题回答 (PA1 RTFSC 部分)

> 以下问题均来自南京大学 ICS PA 讲义，结合本项目 RISC-V 32 实现进行回答。

---

### PA1.3 RTFSC 思考题

#### Q: 需要多费口舌吗？一个程序从哪里开始执行？

**回答**：C 程序从 `main()` 函数开始执行（准确说是从 `_start` 符号开始，`_start` 会调用 C 运行时库的初始化代码，最终调用 `main()`）。NEMU 的入口在 `nemu/src/nemu-main.c` 的 `main()` 函数：

```c
int main(int argc, char *argv[]) {
  init_monitor(argc, argv);
  engine_start();
  return is_exit_status_bad();
}
```

---

#### Q: kconfig 生成的宏 `IFDEF`/`MUXDEF` 是如何工作的？

**回答**：这些宏定义在 `nemu/include/macro.h` 中，利用了 C 预处理器的 token paste 和条件展开技巧：

- `IFDEF(CONFIG_XXX, code)` — 如果 `CONFIG_XXX` 被定义（值为非空），则展开为 `code`；否则展开为空
- `MUXDEF(CONFIG_XXX, a, b)` — 如果 `CONFIG_XXX` 被定义，展开为 `a`；否则展开为 `b`

核心原理：利用宏参数拼接来选择不同的分支。例如 `CONFIG_TRACE` 被定义为某个值时，通过 `##` 拼接产生一个能选择第一个参数的宏；未定义时走另一个分支。这比 `#ifdef` 更灵活，因为可以在表达式中使用。

---

#### Q: 为什么 `init_monitor()` 全部都是函数调用？展开也不影响正确性，使用函数有什么好处？

**回答**：

1. **可读性**：函数名即文档，一眼就能看出每步在做什么
2. **可维护性**：修改某个初始化步骤只需改对应函数，不影响其他代码
3. **调试便利**：可以对单个函数设断点，用 GDB 的 `step`/`next` 精确控制
4. **复用性**：某些初始化函数可能在其他地方被调用
5. **编译优化**：编译器可以独立优化每个函数，也可以自行决定是否 inline

---

#### Q: `parse_args()` 中的参数是从哪里来的？

**回答**：参数来自**运行 NEMU 时的命令行**。当你执行：

```bash
./build/riscv32-nemu-interpreter --batch path/to/image.bin
```

操作系统将命令行拆分为 `argv[]` 数组传给 `main(argc, argv)`，然后 `main()` 传给 `init_monitor(argc, argv)`，最终传给 `parse_args(argc, argv)`。

在 AM 构建系统中，`make run` 命令会自动拼接这些参数（如 `--batch` 和镜像文件路径）。

---

#### Q: 究竟要执行多久？`cmd_c()` 传入 `-1` 是什么意思？

**回答**：

```c
static int cmd_c(char *args) {
  cpu_exec(-1);   // 传入 (uint64_t)-1 = 0xFFFFFFFFFFFFFFFF
  return 0;
}
```

`cpu_exec()` 的参数类型是 `uint64_t`，`-1` 被隐式转换为 `uint64_t` 的最大值 `18446744073709551615`。这意味着"执行极其大量的指令"——实际上等价于"一直执行直到遇到 trap 或错误"。因为在 `execute()` 的循环中，每执行一条 `n--`，在有限的程序执行完毕触发 `nemu_state.state != NEMU_RUNNING` 后才会 break 退出。

---

#### Q: 传入 `-1` 属于未定义行为吗？

**回答**：**不是未定义行为**。C99 标准 §6.3.1.3 规定：将有符号整数转换为无符号整数时，结果是对 `2^N` 取模（N 是目标类型的位宽）。所以 `(uint64_t)(-1)` 的结果是明确定义的，等于 `UINT64_MAX`。

---

#### Q: 谁来指示程序的结束？凭什么 `main()` 返回程序就结束了？

**回答**：

在普通操作系统上：`main()` 返回后，C 运行时库 (crt) 中的 `_start` 代码会调用 `exit()` 系统调用，通知操作系统回收进程资源。所以并非 `main()` 返回程序就自然消失，而是有额外的代码帮你调用了 `exit()`。

在 NEMU 的裸金属环境中：
```c
void _trm_init() {
  int ret = main(mainargs);
  halt(ret);  // ← 这里显式调用 halt 来结束程序
}
```
`halt()` 中执行 `ebreak` 指令，NEMU 拦截后设置 `nemu_state.state = NEMU_END`，cpu_exec 循环因此退出。

**结论**：程序的结束永远需要"某人"来指示——在 OS 上是 crt + exit syscall，在 NEMU 上是 `halt()` + `ebreak`。

---

#### Q: 优美地退出 — 直接键入 `q` 退出 NEMU 时的错误信息是什么原因？如何修复？

**回答**：

**原因分析**：输入 `q` 时，`cmd_q()` 返回 `-1` 使 `sdb_mainloop()` 退出，但此时 `nemu_state.state` 仍然是 `NEMU_STOP`（而非 `NEMU_QUIT`）。在 `main()` 末尾调用的 `is_exit_status_bad()` 函数会判断：如果状态不是 `NEMU_QUIT` 或 `NEMU_END`（且 halt_ret == 0），就认为是异常退出。

**修复方法**：在 `cmd_q()` 中设置状态为 `NEMU_QUIT`：
```c
static int cmd_q(char *args) {
  nemu_state.state = NEMU_QUIT;  // ← 添加这行
  return -1;
}
```

从项目代码中可以看到，这个修复已经实现了。

---

#### Q: 物理内存的起始地址为什么是 `0x80000000`？

**回答**：这是 RISC-V 的约定。在 RISC-V 特权级规范中，`0x80000000` 是 RAM 的典型起始地址（SiFive 等标准平台的地址映射）。`0x00000000-0x7FFFFFFF` 通常留给 ROM、设备映射等。NEMU 通过 `guest_to_host()` 函数将 guest 地址 `0x80000000` 映射到 `pmem[0]`：

```c
uint8_t* guest_to_host(paddr_t paddr) {
  return pmem + paddr - CONFIG_MBASE;  // CONFIG_MBASE = 0x80000000
}
```

---

### PA1.4 基础设施 思考题

#### Q: 单步执行 `si` 命令如何实现？

**回答**：

```c
static int cmd_si(char *args) {
  char *arg = strtok(NULL, " ");
  int steps;
  if (arg == NULL) { cpu_exec(1); return 0; }  // 默认执行 1 步
  sscanf(arg, "%d", &steps);
  cpu_exec(steps);  // 执行 N 步
  return 0;
}
```

本质就是调用 `cpu_exec(N)`，让指令执行循环只跑 N 次就停下来。

---

#### Q: `info r` 如何打印所有寄存器？

**回答**：调用 ISA 相关的 `isa_reg_display()` 函数：

```c
void isa_reg_display() {
  for (int i = 0; i < 32; i++) {
    printf("%-3s: " FMT_WORD " ", regs[i], cpu.gpr[i]);
    if (i % 4 == 3) printf("\n");
  }
  printf("$pc: " FMT_WORD "\n", cpu.pc);
}
```

---

### PA1.5 表达式求值 思考题

#### Q: 如何区分负号和减号？如何区分解引用 `*` 和乘号？

**回答**：通过观察**前一个 token 的类型**来判断。如果 `-` 或 `*` 前面是：
- 操作数（数字 `TK_NUM`、十六进制 `TK_HEXNUM`、寄存器 `TK_REG`、右括号 `)`）→ 是**二元运算符**（减号/乘号）
- 其他情况（运算符、左括号、位于表达式开头）→ 是**一元运算符**（负号/解引用）

```c
if (tokens[i].type == '-' && (i == 0 || prev_not_operand)) {
  tokens[i].type = TK_NEG;   // 标记为负号
}
if (tokens[i].type == '*' && (i == 0 || prev_not_operand)) {
  tokens[i].type = TK_DEREF; // 标记为解引用
}
```

---

#### Q: 如何找到主运算符 (dominant operator)？

**回答**：主运算符 = 表达式树的根节点 = **优先级最低、最后执行**的运算符。

算法：
1. 从左到右扫描 token 列表
2. 跳过括号内的部分（维护括号深度计数器）
3. 对于括号外的运算符，比较优先级：
   - 选择优先级数值**最大**的（项目中优先级数值越大 = 实际优先级越低）
   - 同优先级取**最右边**的（保证左结合性）
4. 一元运算符特殊处理：取最左边的

---

### PA1.6 监视点 思考题

#### Q: 监视点如何检测值的变化？

**回答**：在**每条指令执行后**调用 `check_wp()`，对所有活跃的监视点表达式重新求值，与上一次保存的值比较：

```c
bool check_wp() {
  WP *tmp = head;
  while (tmp != NULL) {
    word_t ans = expr(tmp->e, &success);  // 重新求值
    if (ans != tmp->value) {               // 值变了！
      // 打印信息，暂停执行
      tmp->value = ans;                    // 更新保存的值
      return true;
    }
    tmp = tmp->next;
  }
  return false;
}
```

**性能影响**：每条指令后都要对所有监视点求值，如果表达式复杂或监视点多，会显著降低模拟速度。这和硬件调试器的监视点不同——硬件监视点通过比较电路实现，不影响执行速度。

---

### PA2 相关思考题 (对应 D1 核心内容)

#### Q: RISC-V 指令的 opcode 字段在哪里？如何从指令中提取？

**回答**：RISC-V 指令的低 7 位 (bits[6:0]) 是 opcode，用于初步区分指令类型：

| opcode[6:0] | 类型 | 指令举例 |
|-------------|------|----------|
| `0110011` | R 型 (寄存器-寄存器) | add, sub, mul |
| `0010011` | I 型 (立即数算术) | addi, slli |
| `0000011` | I 型 (Load) | lw, lb |
| `0100011` | S 型 (Store) | sw, sb |
| `1100011` | B 型 (Branch) | beq, bne |
| `0110111` | U 型 | lui |
| `0010111` | U 型 | auipc |
| `1101111` | J 型 | jal |
| `1100111` | I 型 (Jump) | jalr |
| `1110011` | System | ecall, ebreak, csr |

进一步由 `funct3` (bits[14:12]) 和 `funct7` (bits[31:25]) 精确区分具体指令。

---

#### Q: INSTPAT 宏展开后实际做了什么？

**回答**：`INSTPAT` 宏的核心行为是**位模式匹配**。展开后大致等价于：

```c
if (instruction_matches_pattern(inst, "0000000 ????? ????? 000 ????? 01100 11")) {
  decode_operand(s, &rd, &src1, &src2, &imm, TYPE_R);
  R(rd) = src1 + src2;  // 执行体
}
```

具体实现中，框架代码会将 `?` 编码为 mask 的 0 位（不关心），固定位编码为 mask 的 1 位，然后通过 `(inst & mask) == pattern` 进行匹配。

---

#### Q: 为什么最后有一条 `R(0) = 0`？

**回答**：RISC-V 规范要求 `x0` 寄存器**永远为 0**。如果某条指令将 `x0` 作为目的寄存器（如 `add x0, x1, x2`——虽然没实际意义但语法合法），执行后 `gpr[0]` 可能被写入非零值。所以在每条指令执行完毕后，必须将其强制恢复为 0：

```c
R(0) = 0; // reset $zero to 0
```

在真实硬件中，x0 是通过硬连线实现的（写入被忽略），但软件模拟必须显式处理。

---

*注：以上回答基于 NJU ICS PA 2024 讲义中的思考题，结合本项目 riscv32 实现进行分析。*
*PA 讲义来源：https://nju-projectn.github.io/ics-pa-gitbook/ics2024/*


---

## 十五、本阶段 Git 版本记录

| Commit | 说明 | 完成内容 |
|--------|------|----------|
| `0d70386` | init repo | 初始化项目仓库 |
| `2af74ac` | PA1 | sdb 调试器、表达式求值、监视点实现 |
| `1c8f0fb` | PA2.1 | RV32IM 指令集实现，cpu-exec 主循环 |
| `33c5971` | PA2.2 w/o trace | 完善所有指令，通过 cpu-tests (不含 trace) |

**查看对应版本代码**：
```bash
git show 2af74ac:nemu/src/monitor/sdb/sdb.c       # PA1: sdb 调试器
git show 1c8f0fb:nemu/src/isa/riscv32/inst.c       # PA2.1: 指令实现
git show 33c5971:nemu/src/isa/riscv32/inst.c       # PA2.2: 完善指令
```


---

## 十六、Git 版本代码演进详解

### 16.1 PA1 (`2af74ac`) — 框架初始版本

此时 `inst.c` **只有 3 条指令**：

```c
// 只支持: lui, lw, sw, ebreak
INSTPAT("??????? ????? ????? ??? ????? 01101 11", lui   , U, R(rd) = imm);
INSTPAT("??????? ????? ????? 010 ????? 00000 11", lw    , I, R(rd) = Mr(src1+imm, 4));
INSTPAT("??????? ????? ????? 010 ????? 01000 11", sw    , S, Mw(src1+imm, 4, src2));
INSTPAT("0000000 00001 00000 000 00000 11100 11", ebreak, N, NEMUTRAP(s->pc, R(10)));
```

- 只定义了 3 种指令格式: `TYPE_I, TYPE_U, TYPE_S`
- 只有 `immI`, `immU`, `immS` 三个立即数提取宏
- 能运行内置客户程序 (仅做 lui+sw 然后 ebreak)
- **这就是 PA1 的完成标志**——能让 NEMU 启动并跑完内置程序

**sdb 调试器变化** (本次新增 +439 行):
- `expr.c`: 实现了完整的表达式求值 (词法分析 + 递归下降)
- `sdb.c`: 实现 si/info/x/p/w/d 命令
- `watchpoint.c`: 实现监视点池 (链表管理 + check_wp)

---

### 16.2 PA2.1 (`1c8f0fb`) — 完整 RV32IM 指令集

**新增指令类型**: `TYPE_J, TYPE_R, TYPE_B` + 对应立即数宏 `immJ, immB`

**新增指令 (从 3 条 → 37 条)**:

| 类别 | 新增指令 |
|------|----------|
| U 型 | `auipc` |
| I 型算术 | `addi`, `sltiu`, `andi`, `srai`, `slli`, `srli`, `xori` |
| I 型 Load | `lbu`, `lh`, `lhu` |
| I 型跳转 | `jalr` |
| S 型 | `sb`, `sh` |
| J 型 | `jal` |
| B 型 | `beq`, `bne`, `bge`, `bgeu`, `bltu`, `blt` |
| R 型 | `add`, `sub`, `or`, `sltu`, `xor`, `sll`, `and`, `slt`, `sra`, `srl` |
| R 型 M | `mul`, `mulh`, `rem`, `div`, `remu`, `divu` |

**关键代码变化**:
- `static uint64_t t;` — 新增临时变量用于 jalr 的"先保存再跳转"
- J/B 型立即数的位拼接很容易出错 — 这是 PA2.1 的主要难点

---

### 16.3 PA2.2 (`33c5971`) — 完善与修复

在 PA2.1 基础上增量改动很小 (+55 行):
- `inst.c`: 增加了 `slti`, `ori`, `mulhu`, `lb`, `csrrw`, `csrrs`, `ecall`, `mret` 等指令
- `difftest/dut.c`: 修复 DiffTest 寄存器同步

这说明 PA2.1 已经完成了大部分工作，PA2.2 是查漏补缺。
