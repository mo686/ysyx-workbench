# C4: ELF 文件和链接 — 详细学习文档

## 一、阶段目标

C4 阶段的核心目标是深入理解可执行文件的组织结构、链接过程，以及如何利用 ELF 信息实现 FTRACE（函数调用追踪）。完成后你将：

1. 理解 ELF 文件的结构（文件头、段表、符号表、字符串表）
2. 理解链接器如何将 .o 文件组合成可执行文件
3. 理解重定位 (Relocation) 的概念
4. 实现 FTRACE — 利用 ELF 符号表追踪函数调用/返回
5. 理解 AM 构建系统中从 .c → .o → .elf → .bin 的完整链路

---

## 二、ELF 文件格式

### 2.1 什么是 ELF？

ELF (Executable and Linkable Format) 是 Unix/Linux 系统通用的可执行文件格式。一个 ELF 文件同时服务于两个视图：

```
         链接视图 (Linking View)          执行视图 (Execution View)
         ┌──────────────────┐            ┌──────────────────┐
         │    ELF Header    │            │    ELF Header    │
         ├──────────────────┤            ├──────────────────┤
         │ Program Header   │            │ Program Header   │
         │ Table (optional) │            │ Table            │
         ├──────────────────┤            ├──────────────────┤
         │    .text         │            │                  │
         ├──────────────────┤            │   Segment 1     │
         │    .rodata       │            │   (LOAD, R+X)   │
         ├──────────────────┤            │                  │
         │    .data         │            ├──────────────────┤
         ├──────────────────┤            │                  │
         │    .bss          │            │   Segment 2     │
         ├──────────────────┤            │   (LOAD, R+W)   │
         │    .symtab       │            │                  │
         ├──────────────────┤            ├──────────────────┤
         │    .strtab       │            │ Section Header   │
         ├──────────────────┤            │ Table (optional) │
         │ Section Header   │            └──────────────────┘
         │ Table            │
         └──────────────────┘
```

- **链接视图**：以 Section (段) 为单位，链接器使用
- **执行视图**：以 Segment 为单位，加载器使用

### 2.2 ELF 文件头

```bash
$ riscv64-linux-gnu-readelf -h build/add-riscv32e-npc.elf

ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 ...    # 魔数 "\x7fELF"
  Class:                             ELF32  # 32位
  Data:                              2's complement, little endian
  Type:                              EXEC   # 可执行文件
  Machine:                           RISC-V
  Entry point address:               0x80000000  # 入口地址 (_start)
  Start of program headers:          52
  Start of section headers:          ...
  Number of program headers:         2       # 2个加载段
  Number of section headers:         ...
```

### 2.3 Section (节/段)

```bash
$ riscv64-linux-gnu-readelf -S build/add-riscv32e-npc.elf

Section Headers:
  [Nr] Name              Type     Addr       Off    Size   ES Flg
  [ 0]                   NULL     00000000   000000 000000 00
  [ 1] .text             PROGBITS 80000000   001000 000xxx 00 AX   # 代码
  [ 2] .rodata           PROGBITS 800xxxxx   00xxxx 000xxx 00  A   # 只读数据
  [ 3] .data             PROGBITS 800xxxxx   00xxxx 000xxx 00 WA   # 可写数据
  [ 4] .bss              NOBITS   800xxxxx   00xxxx 000xxx 00 WA   # 未初始化数据
  [ 5] .symtab           SYMTAB   ...                              # 符号表
  [ 6] .strtab           STRTAB   ...                              # 字符串表
```

**Flags 含义**：A=Alloc (需加载), W=Write, X=Execute

### 2.4 Program Header (程序头)

```bash
$ riscv64-linux-gnu-readelf -l build/add-riscv32e-npc.elf

Program Headers:
  Type    Offset   VirtAddr   PhysAddr   FileSiz  MemSiz   Flg  Align
  LOAD    0x001000 0x80000000 0x80000000 0x00xxxx 0x00xxxx R E  0x1000
  LOAD    0x00xxxx 0x800xxxxx 0x800xxxxx 0x000xxx 0x000xxx RW   0x1000
```

- 第1个 LOAD 段：代码 + 只读数据 (R+E)
- 第2个 LOAD 段：可写数据 (R+W)
- `objcopy -O binary` 就是把这些段按地址顺序拼成纯二进制

---

## 三、符号表 (Symbol Table)

### 3.1 查看符号表

```bash
$ riscv64-linux-gnu-readelf -s build/add-riscv32e-npc.elf

Symbol table '.symtab' contains N entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     1: 80000000     0 NOTYPE  LOCAL  DEFAULT    1 _start
     5: 80000014    28 FUNC    GLOBAL DEFAULT    1 _trm_init
     8: 80000030    16 FUNC    GLOBAL DEFAULT    1 add
     9: 80000040    96 FUNC    GLOBAL DEFAULT    1 main
    12: 80000xxx     0 NOTYPE  GLOBAL DEFAULT    4 _heap_start
    15: 80000xxx     0 NOTYPE  GLOBAL DEFAULT  ABS _stack_pointer
```

### 3.2 符号表字段含义

| 字段 | 含义 | 示例 |
|------|------|------|
| Value | 符号地址 | `0x80000030` (函数 add 的入口) |
| Size | 符号大小（字节） | `16` (add 函数占 16 字节) |
| Type | 类型 | FUNC=函数, OBJECT=全局变量, NOTYPE=标号 |
| Bind | 绑定 | GLOBAL=全局可见, LOCAL=文件内可见 |
| Ndx | 所在节的索引 | 1=.text, 4=.bss, ABS=绝对地址 |
| Name | 符号名 | "add", "main", "_heap_start" |

### 3.3 FTRACE 如何利用符号表

FTRACE 的核心思想：
- `jal` 指令跳转到某个地址 → 查符号表找到该地址对应的函数名 → 打印 "call funcname"
- `jalr ra` (ret 的等价形式) → 打印 "ret"

```c
// FTRACE 伪代码
void ftrace(uint32_t pc, uint32_t target, bool is_call) {
  if (is_call) {
    const char *name = find_func_by_addr(target);  // 查符号表
    printf("0x%x: call [%s@0x%x]\n", pc, name, target);
    call_depth++;
  } else {
    call_depth--;
    printf("0x%x: ret\n", pc);
  }
}
```

---

## 四、链接过程

### 4.1 从 .c 到 .elf 的完整流程

```
  add.c          start.S        string.c (klib)
    │              │               │
    ▼ gcc -c      ▼ gcc -c        ▼ gcc -c
  add.o          start.o         string.o
    │              │               │
    │              ▼               ▼
    │         am-riscv32e-npc.a   klib-riscv32e-npc.a
    │              │               │
    └──────────────┼───────────────┘
                   │
                   ▼  ld (链接器)
            add-riscv32e-npc.elf
                   │
                   ▼  objcopy -O binary
            add-riscv32e-npc.bin
```

### 4.2 链接器做了什么？

1. **合并同名节**：所有 .o 文件的 `.text` 段合并为一个大的 `.text` 段
2. **符号解析**：将 `jal add` 中的 `add` 查找到具体地址
3. **重定位**：填充指令中的地址字段（编译时不知道最终地址，链接时确定）
4. **布局安排**：按照链接脚本 (`linker.ld`) 的规则排列各段

### 4.3 重定位 (Relocation)

**问题**：编译 `add.c` 时，编译器不知道 `main` 函数最终在内存哪个位置。

**解决**：
1. 编译时在指令中**留空**（填 0 或临时值）
2. 生成**重定位条目**：记录"这里需要填入某个符号的地址"
3. 链接时，链接器**确定所有符号的最终地址**，回填到指令中

```bash
# 查看 .o 文件中的重定位信息
$ riscv64-linux-gnu-readelf -r add.o

Relocation section '.rela.text':
  Offset     Info    Type                Sym. Value  Symbol's Name + Addend
  00000010   ...     R_RISCV_JAL         00000000    main + 0
  00000014   ...     R_RISCV_CALL        00000000    halt + 0
```

**RISC-V 重定位类型**：
| 类型 | 含义 |
|------|------|
| R_RISCV_JAL | JAL 指令的 20-bit 偏移 |
| R_RISCV_BRANCH | 分支指令的 12-bit 偏移 |
| R_RISCV_HI20 | LUI/AUIPC 的高 20 位 |
| R_RISCV_LO12_I | ADDI/LW 的低 12 位 |
| R_RISCV_CALL | 函数调用 (AUIPC+JALR 组合) |

### 4.4 链接脚本的作用

```ld
SECTIONS {
  . = 0x80000000;       // 起始地址
  .text : { *(entry) *(.text*) }   // 代码段: entry 最先
  .rodata : { *(.rodata*) }
  .data : { *(.data) }
  .bss : { *(.bss*) }
  _stack_pointer = . + 0x8000;     // 栈指针
  _heap_start = ALIGN(0x1000);     // 堆起始
}
```

链接器根据脚本确定：
- `.text` 段从 `0x80000000` 开始
- `_start` 的地址 = `0x80000000`（因为 `entry` 段排第一）
- `main` 的地址 = `.text` 段中的某个偏移
- `_stack_pointer` = `.bss` 结束后再加 32KB

---

## 五、objcopy: ELF → 纯二进制

### 5.1 为什么需要 objcopy？

ELF 文件包含大量元数据（段表、符号表、调试信息等），这些对裸金属执行没有意义。`objcopy -O binary` 只保留需要加载到内存的**实际内容**。

### 5.2 转换逻辑

```
ELF 文件中的 LOAD 段:
  Segment 1: VirtAddr=0x80000000, FileSiz=0x1234 (代码+只读数据)
  Segment 2: VirtAddr=0x80001240, FileSiz=0x0056 (可写数据)

objcopy -O binary 的输出:
  偏移 0x0000: Segment 1 的内容 (0x1234 字节)
  偏移 0x1240: Segment 2 的内容 (0x0056 字节)
  中间用 0 填充

最终 .bin 文件大小 = 0x1240 + 0x0056 = 0x1296 字节
```

### 5.3 NEMU 加载 .bin 文件

```c
// nemu/src/monitor/monitor.c
fread(guest_to_host(RESET_VECTOR), size, 1, fp);
// .bin 文件的第一个字节 → 内存地址 0x80000000
// .bin 文件的第 N 个字节 → 内存地址 0x80000000 + N - 1
```

因此 `.bin` 文件的偏移 0 正好对应内存地址 `0x80000000`，即 `_start` 的位置。

---

## 六、FTRACE 实现

### 6.1 设计思路

FTRACE 通过监控 `jal`/`jalr` 指令来跟踪函数调用关系：

| 指令 | 条件 | 解释 |
|------|------|------|
| `jal rd, offset` | rd != x0 | 函数调用 (保存返回地址) |
| `jalr rd, rs1, 0` | rs1 == ra, rd == x0 | 函数返回 (ret = jalr x0, ra, 0) |
| `jalr rd, rs1, imm` | rd != x0 | 间接调用 (函数指针) |

### 6.2 地址到函数名的映射

在 NEMU 启动时解析 ELF 符号表，构建地址→函数名的映射：

```c
// 伪代码
typedef struct {
  uint32_t addr;
  uint32_t size;
  char name[64];
} FuncEntry;

FuncEntry func_table[MAX_FUNCS];
int nr_func = 0;

void load_elf_symbols(const char *elf_file) {
  // 1. 读取 ELF 文件头
  // 2. 找到 .symtab 节和 .strtab 节
  // 3. 遍历符号表，筛选 Type==FUNC 的条目
  // 4. 保存 {addr, size, name} 到 func_table
}

const char* find_func(uint32_t addr) {
  for (int i = 0; i < nr_func; i++) {
    if (addr >= func_table[i].addr &&
        addr < func_table[i].addr + func_table[i].size) {
      return func_table[i].name;
    }
  }
  return "???";
}
```

### 6.3 FTRACE 输出格式

```
0x80000030: call [add@0x80000030]
0x80000040:   call [check@0x80000060]
0x80000070:   ret  [check]
0x80000044:   call [check@0x80000060]
0x80000070:   ret  [check]
0x8000004c: ret  [add]
```

缩进表示调用深度，方便直观看出调用层次。

---

## 七、`--gc-sections` 与链接优化

### 7.1 什么是 gc-sections？

```makefile
CFLAGS  += -fdata-sections -ffunction-sections
LDFLAGS += --gc-sections
```

- `-ffunction-sections`：每个函数放入独立的 `.text.funcname` 节
- `-fdata-sections`：每个全局变量放入独立的 `.data.varname` 节
- `--gc-sections`：链接时移除**未被引用**的节

**效果**：如果 klib 中实现了 20 个函数但你只用了 3 个，其余 17 个不会出现在最终二进制中。这对裸金属程序的大小优化很重要。

---

## 八、常见问题与解决

### Q1: readelf 报错 "Not an ELF file"

**原因**：你对 .bin 文件执行了 readelf。readelf 只能处理 .elf 文件。

### Q2: 符号表中找不到某个函数

**可能原因**：
1. 被 `--gc-sections` 移除了（没有被调用）
2. `static` 函数可能被内联优化掉了
3. `-O2` 优化导致函数被合并

**解决**：用 `-O0` 编译，或给函数加 `__attribute__((noinline))`

### Q3: FTRACE 输出的函数名是 "???"

**原因**：跳转目标地址在符号表中找不到对应的 FUNC 类型符号。可能是：
- 跳转到了库函数内部（符号在 .a 文件中）
- 间接调用（函数指针），目标地址在运行时才确定

### Q4: objcopy 后的 .bin 文件太大

**原因**：.data 段和 .text 段之间有大量填充（对齐）
**解决**：检查链接脚本中的对齐要求是否合理

---

## 九、PA 讲义相关思考题

#### Q: 为什么编译时不能确定函数的最终地址？

**回答**：因为编译是**分别**对每个 .c 文件进行的。编译 `add.c` 时，编译器不知道 `main()` 在 `main.c` 编译后会排在 .text 段的什么位置。只有链接器将所有 .o 文件的 .text 段合并后，才能确定每个函数的最终地址。

---

#### Q: 为什么 RISC-V 的 JAL 指令偏移只有 20 位？如果函数太远怎么办？

**回答**：20 位有符号偏移 = ±1MB 的跳转范围。如果目标超过这个范围：
- 编译器生成 `auipc + jalr` 的组合（可跳转 ±2GB）
- 链接器在重定位时选择合适的重定位类型 (`R_RISCV_CALL` = auipc + jalr)

这也是为什么 RISC-V 有 `R_RISCV_CALL` 这种"组合重定位"类型。

---

#### Q: `_start` 为什么一定要排在最前面？

**回答**：因为 NEMU/NPC 的 PC 复位值是 `0x80000000`，即内存的起始地址。CPU 上电后从 `0x80000000` 开始取指，所以 `_start` 必须在这个地址。

链接脚本通过 `*(entry)` 确保 `_start` 所在的 `entry` 段排在 `.text` 最前面：
```ld
.text : {
  *(entry)      ← _start 在这里
  *(.text*)     ← 其他函数在后面
}
```

---

#### Q: 为什么 .bss 段在 ELF 中不占空间但在 .bin 中需要？

**回答**：
- **ELF 中**：.bss 段的 `Type = NOBITS`，不占文件空间（只记录大小），因为它全是 0
- **.bin 中**：`objcopy --set-section-flags .bss=alloc,contents` 强制将 .bss 实际输出为 0 字节

为什么？因为裸金属环境没有加载器来帮你清零 .bss。如果 .bin 中不包含这些零字节，加载到内存后 .bss 区域可能是随机值。

---

## 十、学习建议

1. **动手用 readelf 分析你的 ELF 文件** — 理解各节的地址和大小
2. **对比 .elf 和 .bin** — 理解 objcopy 做了什么
3. **实现 FTRACE** — 它是理解符号表最好的练习
4. **手动做一次重定位** — 看 .o 文件中的重定位条目，确认链接后被正确填充
5. **阅读链接脚本** — 修改入口地址或栈大小，观察对程序的影响

---

*参考资料：*
- *System V ABI (ELF 格式规范)*
- *GNU ld Linker Scripts 手册*
- *RISC-V ELF psABI Document*
- *项目源码: abstract-machine/scripts/linker.ld, abstract-machine/Makefile*


---

## 十一、Git 版本代码演进详解

### 11.1 AM 框架中的链接基础设施

C4 (ELF 文件和链接) 的核心代码在 AM 框架引入时 (`fa79548`) 就已经存在:

| 文件 | 功能 | 行数 |
|------|------|------|
| `scripts/linker.ld` | 链接脚本 (段布局) | ~30 行 |
| `scripts/platform/npc.mk` | 平台构建规则 (objcopy) | ~30 行 |
| `scripts/isa/riscv.mk` | 交叉编译器设置 | ~8 行 |
| `Makefile` | AM 主构建系统 | ~140 行 |

### 11.2 链接脚本在不同阶段的变化

链接脚本 (`linker.ld`) 在整个项目中**几乎没有变化**，说明它的设计一次到位:

```ld
ENTRY(_start)
SECTIONS {
  . = _pmem_start + _entry_offset;  // 由 LDFLAGS --defsym 传入
  .text : { *(entry) *(.text*) }    // 代码段
  .rodata : { *(.rodata*) }
  .data : { *(.data) }
  .bss : { *(.bss*) *(.sbss*) }
  _stack_top = ALIGN(0x1000);
  . = _stack_top + 0x8000;          // 32KB 栈
  _stack_pointer = .;
  _heap_start = ALIGN(0x1000);
}
```

唯一变化在 `--defsym` 参数中：
- D 阶段: `_pmem_start=0x80000000` (RAM 起始)
- B2 SoC: `_pmem_start=0x30000000` (Flash 起始，ysyxSoC 模式)

### 11.3 FTRACE 相关

本项目中 FTRACE 在 NEMU 的 Kconfig 中定义 (`CONFIG_FTRACE`)，但从 git 历史看未被完整实现 (PA2.2 标注为 "TODO")。

FTRACE 的实现需要:
1. 启动时解析 ELF 符号表 (在 `init_monitor` 中)
2. 在 `exec_once` 中检测 `jal`/`jalr` 指令
3. 查符号表输出函数名

如果你需要实现 FTRACE，参考 C4 文档中的设计思路即可。
