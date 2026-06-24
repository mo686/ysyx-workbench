# 项目开发历程 — 按 Git 提交顺序梳理

本文档基于项目的 git commit history，将每个提交对应到一生一芯的学习阶段，记录从零到完整 SoC 的渐进式开发过程。

---

## D 阶段 — 基础阶段

### D1: 支持 RV32IM 的 NEMU

| Commit | 说明 | 对应内容 |
|--------|------|----------|
| `0d70386` | init repo | 初始化项目仓库 |
| `2af74ac` | PA1 | 实现 NEMU 基础设施：sdb 调试器、表达式求值、监视点 |
| `1c8f0fb` | PA2.1 | 实现 RV32IM 指令集 (取指→译码→执行循环) |
| `33c5971` | PA2.2 w/o trace | 完善指令实现，通过 cpu-tests (不含 trace 功能) |

**里程碑**：NEMU 能够正确执行 RV32IM 指令，通过 am-kernels/tests/cpu-tests

---

### D2: 程序的机器级表示

包含在 PA2.1/PA2.2 中：
- 理解交叉编译 (gcc → riscv)
- 理解链接脚本 (linker.ld)
- 理解 ELF → bin 转换
- 理解 start.S → _trm_init → main → halt 启动链

---

### D3: AM 运行时环境

| Commit | 说明 | 对应内容 |
|--------|------|----------|
| `fa79548` | NJU-ProjectN/abstract-machine ics2023 initialized | 引入 AM 框架 |
| `e5276c7` | fix bug and do real-time clock test | 实现 IOE 定时器设备 |
| `702fcd3` | test benchmarks | 运行 CoreMark/Dhrystone 基准测试 |

**里程碑**：AM IOE 框架工作正常，能运行基准测试

---

### D4: 用 RTL 实现迷你 RISC-V 处理器

| Commit | 说明 | 对应内容 |
|--------|------|----------|
| `323892b` | PA2.1 RTL | 第一版 NPC RTL 设计 (基于 Verilator 仿真) |
| `68bd7d0` | PA2.2 RTL w/o trace | 完善 NPC 指令实现，通过基本测试 |
| `8b59731` | replace Verilator with Questasim | 尝试 Questasim (后续又回到 Verilator) |

**里程碑**：NPC 能运行 dummy 和基本 cpu-tests

---

### D5: 设备和输入输出

| Commit | 说明 | 对应内容 |
|--------|------|----------|
| `807cfb6` | PA2.3 w/o dtrace | NEMU 实现设备 (串口/定时器/键盘/VGA) |
| `3980d5e` | PA2.3 RTL | NPC 支持设备 I/O (DPI-C 方式) |

**里程碑**：NEMU 和 NPC 都能运行带 I/O 的程序 (hello, typing-game)

---

## C 阶段 — 进阶阶段

### C1: 工具和基础设施

贯穿整个开发过程的基础设施：
- DiffTest 在 PA2.1 RTL 时就已引入
- ITRACE/MTRACE 在 PA2.2 实现

### C2: 支持 RV32E 的单周期 NPC

| Commit | 说明 | 对应内容 |
|--------|------|----------|
| `d56f8cf` | update RTL, reference:... | 参考实现优化 NPC |
| `c4ddbe0` | feat: code simplified | 代码简化重构 |

**里程碑**：单周期 NPC 通过所有 cpu-tests + DiffTest

### C3: 调试技巧

贯穿整个开发过程，通过实践逐步积累：
- 波形调试 (WAVE=1)
- DiffTest 报错分析
- objdump 反汇编定位

### C4: ELF 文件和链接

在 AM 框架使用过程中自然掌握：
- linker.ld 理解
- objcopy 转换
- FTRACE (如已实现)

### C5: 异常处理和 RT-Thread

| Commit | 说明 | 对应内容 |
|--------|------|----------|
| `1c64d37` | feat: nemu rt-thread | NEMU 支持 RT-Thread (实现 ecall/mret/CSR) |
| `51b3f3b` | feat: NPC rt-thread | NPC 支持 RT-Thread (硬件实现 CSR + IRU) |

**里程碑**：ecall/mret/csrrw 在 NEMU 和 NPC 中都正确实现，RT-Thread 启动成功

---

## B 阶段 — 高级阶段

### B1: 总线

| Commit | 说明 | 对应内容 |
|--------|------|----------|
| `076d790` | feat: Add handshake signals | 添加 valid/ready 握手信号 (为总线做准备) |
| `75481bf` | feat: multi-cycle cpu (3 cycles) | **三周期 CPU**: IFU→IDU/EXU→LSU/WBU/PCU |
| `32fd6d1` | feat: multi-cycle cpu with axi-lite mem intf **v1.0** | AXI-Lite 内存接口 (IFU/LSU 通过总线访问内存) |
| `5e45834` | feat: random delay in SRAM access **v1.1** | SRAM 加入随机延迟 (模拟真实内存延迟) |
| `456fe5a` | feat: add random delay from LFSR to axi bus **v1.2** | 用 LFSR 产生随机延迟 |
| `cf5bea5` | feat: axi bus **v1.3** add axi lite arbiter | 添加 AXI-Lite 仲裁器 (多 Master 支持) |
| `8dc82bc` | feat: add axi lite xbar, axi lite uart, axi lite clint | AXI-Lite 交叉开关 + UART + CLINT 外设 |

**里程碑**：完整 AXI-Lite 总线系统，多 Master 仲裁，多 Slave 互联

**演进路线详解**：
```
v1.0 (32fd6d1): 单周期 → 多周期 + AXI-Lite 内存接口
                IFU 和 LSU 通过 AXI-Lite 协议读写内存

v1.1 (5e45834): 给 SRAM Slave 加入随机延迟
                验证 CPU 在内存延迟不确定时能正确等待

v1.2 (456fe5a): LFSR 随机延迟 (更真实的随机性)

v1.3 (cf5bea5): 引入 AXI-Lite Arbiter
                IFU 和 LSU 可能同时请求同一个 Slave，需要仲裁
```

---

### B2: SoC 计算机系统

| Commit | 说明 | 对应内容 |
|--------|------|----------|
| `5608222` | feat: ysyxSoC init | 引入 ysyxSoC 框架 |
| `94a3e40` | feat: ysyxSoC init | SoC 初始化 |
| `9eeef29` | feat: mrom and sram added with bootloader v1.0 | 添加 MROM + SRAM + bootloader |
| `d51d2d4` | feat: psram added with 2-stage bootloader | 添加 PSRAM + 两级 bootloader |
| `960c134` | feat: sdram added | 添加 SDRAM |
| `0a781ec` | feat: SoC system finished | **SoC 系统完成** |

**里程碑**：完整 SoC 集成 (MROM→SRAM→PSRAM→SDRAM + UART + SPI)

**启动流程**：
```
CPU 复位 → 从 MROM(0x20000000) 取 bootloader
         → bootloader 将程序从 Flash(0x30000000) 复制到 PSRAM/SDRAM(0x80000000)
         → 跳转到 PSRAM 执行主程序
```

---

### B3: 时序分析和优化

| Commit | 说明 | 对应内容 |
|--------|------|----------|
| `4637c62` | feat: yosys synthesis Fmax=350MHz | Yosys 综合，最高频率 350MHz |
| `fab3c15` | yosys-sta | 静态时序分析 (STA) |

**里程碑**：完成逻辑综合 + STA，评估最高频率 (理想情况 350MHz)

---

### B4: 性能优化和简易缓存

| Commit | 说明 | 对应内容 |
|--------|------|----------|
| `db1eb6e` | feat: add pfc | 添加性能计数器 (Performance Counter) |
| `e84d37f` | feat: add simple icache | 添加简单指令缓存 |
| `8796832` | feat: simple fence.i, invalid all icache | 实现 fence.i (缓存无效化) |
| `c038289` | feat: w-way set-associative icache | **W-way 组相联 ICache** (升级) |

**里程碑**：ICache 实现 + fence.i 一致性 + 性能计数器统计 IPC/CPI/Hit Rate

**演进路线**：
```
直接映射 ICache (e84d37f)  →  组相联 ICache (c038289)
                                ↑
                        性能计数器指导优化 (db1eb6e)
```

---

### B5: 流水线处理器

流水线在 `75481bf` (多周期 CPU) 时就初步引入了三级流水：

```
75481bf: 3 级: IFU → IDU/EXU → LSU/WBU/PCU
```

后续的 AXI 总线和 Cache 都是在这个多级架构基础上持续完善。当前最终版本是带 valid/ready 握手的弹性流水线。

---

## 总结：版本演进一览

```
时间轴 (从下到上 = 从旧到新):

16eb8ab ── 组相联 ICache                          ← B4 完成
c038289 ──┘

8796832 ── fence.i 缓存一致性
e84d37f ── 简单 ICache
db1eb6e ── 性能计数器                              ← B4 开始

0a781ec ── SoC 系统完成                           ← B2 完成
960c134 ── SDRAM
d51d2d4 ── PSRAM + 两级 bootloader
9eeef29 ── MROM + SRAM + bootloader v1.0          ← B2 开始

4637c62 ── Yosys 综合 350MHz                      ← B3

8dc82bc ── AXI-Lite xbar + UART + CLINT           ← B1 完成
cf5bea5 ── AXI-Lite Arbiter v1.3
456fe5a ── LFSR 随机延迟 v1.2
5e45834 ── SRAM 随机延迟 v1.1
32fd6d1 ── AXI-Lite 内存接口 v1.0                 ← B1 开始
75481bf ── 多周期 CPU (3级)                       ← B5 (流水线雏形)
076d790 ── 添加握手信号

51b3f3b ── NPC RT-Thread                          ← C5
1c64d37 ── NEMU RT-Thread                         ← C5

3980d5e ── PA2.3 RTL (设备)                       ← D5
807cfb6 ── PA2.3 NEMU (设备)                      ← D5
68bd7d0 ── PA2.2 RTL                              ← D4 (NPC 基本指令)
323892b ── PA2.1 RTL                              ← D4 (NPC 初版)
33c5971 ── PA2.2 NEMU                             ← D1 (NEMU 完善)
1c8f0fb ── PA2.1 NEMU                             ← D1 (NEMU 指令实现)
2af74ac ── PA1                                    ← D1 (调试器基础设施)
0d70386 ── init repo                              ← 起点
```

---

## 各阶段的关键技术跃迁

| 跃迁 | 从 | 到 | 对应 commit |
|------|----|----|-------------|
| 内存访问方式 | DPI-C (瞬时) | AXI-Lite (多周期) | `32fd6d1` |
| CPU 架构 | 单周期 | 多周期 (3级) | `75481bf` |
| 延迟模型 | 固定延迟 | 随机延迟 (LFSR) | `456fe5a` |
| 总线仲裁 | 无 (单 Master) | Round-Robin Arbiter | `cf5bea5` |
| 外设接口 | DPI-C 直接回调 | AXI-Lite 外设 (UART/CLINT) | `8dc82bc` |
| 启动方式 | 直接加载到 RAM | 两级 bootloader (MROM→Flash→PSRAM) | `d51d2d4` |
| 指令读取 | 每次都从内存 | ICache (组相联) | `c038289` |
| 缓存一致性 | 无 | fence.i 无效化全部 | `8796832` |
| 运行能力 | 裸金属测试 | RT-Thread RTOS | `51b3f3b` |

---

*注：本文档基于 `git log --oneline --all --reverse` 的实际输出整理。*
