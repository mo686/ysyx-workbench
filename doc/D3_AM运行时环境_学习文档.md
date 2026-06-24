# D3: AM 运行时环境 — 详细学习文档

## 一、阶段目标

D3 阶段的核心目标是理解 Abstract-Machine (AM) 如何将硬件差异隐藏在统一的 API 之后，使得同一份应用代码可以运行于不同的硬件平台。完成后你将：

1. 理解 AM 的分层抽象设计 (TRM → IOE → CTE → VME → MPE)
2. 掌握 IOE (I/O Extension) 的设备访问模型
3. 理解设备寄存器抽象 (`amdev.h`) 和查找表分发机制
4. 能够编写使用定时器、键盘、显示设备的裸金属应用
5. 理解 AM 构建系统如何做到"一份代码，多平台运行"

---

## 二、AM 设计哲学

### 2.1 问题：如何让一份贪吃蛇游戏既能跑在 NEMU 上，又能跑在 NPC FPGA 上？

**解决方案**：AM 将计算机硬件抽象为 5 个递进的层次，应用只需调用统一的 API，底层由各平台分别实现。

```
                    ┌────────────────────────────────────────┐
                    │        应用程序 (Application)           │
                    │   typing-game / snake / CoreMark ...    │
                    └──────────────────┬─────────────────────┘
                                       │ 调用统一 API
                    ┌──────────────────▼─────────────────────┐
                    │          Abstract-Machine (am.h)        │
                    │                                        │
                    │  TRM    IOE    CTE    VME    MPE       │
                    └──────────────────┬─────────────────────┘
                                       │ 分平台实现
              ┌────────────┬───────────┼───────────┬────────────┐
              ▼            ▼           ▼           ▼            ▼
         NEMU 平台    NPC 平台    ysyxSoC     QEMU 平台    Native
        (ioe.c...)   (ioe.c...)   (FPGA)     (ioe.c...)   (Linux)
```

### 2.2 五层抽象

| 层级 | 名称 | 功能 | D3 涉及 |
|------|------|------|---------|
| **TRM** | 图灵机 | 最基本的计算 (内存 + 执行 + 停机) | ✅ (D2 已完成) |
| **IOE** | I/O 扩展 | 设备抽象 (定时器/键盘/显示/音频) | ✅ (D3 重点) |
| **CTE** | 上下文扩展 | 中断/异常处理, 上下文切换 | (D5 阶段) |
| **VME** | 虚拟内存 | 页表管理, 地址空间隔离 | (后续阶段) |
| **MPE** | 多处理 | 多核支持, 原子操作 | (后续阶段) |

---

## 三、TRM — 图灵机 (回顾)

TRM 在 D2 阶段已经实现，提供最基本的计算能力：

```c
// am.h 中的 TRM API
extern Area heap;                          // 可用堆内存区域
void putch(char ch);                       // 输出一个字符
void halt(int code) __attribute__((noreturn));  // 停止执行
```

**各平台实现**：

| 平台 | putch 实现 | halt 实现 |
|------|-----------|-----------|
| NEMU | `outb(SERIAL_PORT, ch)` | `mv a0, code; ebreak` |
| NPC | `outb(SERIAL_PORT, ch)` | `mv a0, code; ebreak` |
| Native | `putchar(ch)` (标准库) | `exit(code)` |

---

## 四、IOE — I/O 扩展 (D3 核心)

### 4.1 IOE API

```c
// am.h 中的 IOE API
bool ioe_init(void);                    // 初始化 I/O 子系统
void ioe_read(int reg, void *buf);      // 从设备寄存器读取
void ioe_write(int reg, void *buf);     // 向设备寄存器写入
```

### 4.2 设备寄存器抽象 (`amdev.h`)

AM 将所有设备抽象为"设备寄存器"，每个寄存器有一个 ID 和对应的数据结构：

```c
// abstract-machine/am/include/amdev.h

// 宏定义格式: AM_DEVREG(ID, 名称, 权限, 结构体成员)
AM_DEVREG( 4, TIMER_CONFIG, RD, bool present, has_rtc);
AM_DEVREG( 5, TIMER_RTC,    RD, int year, month, day, hour, minute, second);
AM_DEVREG( 6, TIMER_UPTIME, RD, uint64_t us);
AM_DEVREG( 7, INPUT_CONFIG, RD, bool present);
AM_DEVREG( 8, INPUT_KEYBRD, RD, bool keydown; int keycode);
AM_DEVREG( 9, GPU_CONFIG,   RD, bool present, has_accel; int width, height, vmemsz);
AM_DEVREG(10, GPU_STATUS,   RD, bool ready);
AM_DEVREG(11, GPU_FBDRAW,   WR, int x, y; void *pixels; int w, h; bool sync);
AM_DEVREG(14, AUDIO_CONFIG, RD, bool present; int bufsize);
AM_DEVREG(15, AUDIO_CTRL,   WR, int freq, channels, samples);
AM_DEVREG(16, AUDIO_STATUS, RD, int count);
AM_DEVREG(17, AUDIO_PLAY,   WR, Area buf);
```

**宏展开示例**：
```c
AM_DEVREG(6, TIMER_UPTIME, RD, uint64_t us);
// 展开为:
enum { AM_TIMER_UPTIME = 6 };
typedef struct { uint64_t us; } AM_TIMER_UPTIME_T;
```

### 4.3 便捷读写宏 (`klib-macros.h`)

```c
// 读设备寄存器 (返回结构体值)
#define io_read(reg) \
  ({ reg##_T __io_param; \
    ioe_read(reg, &__io_param); \
    __io_param; })

// 写设备寄存器 (用初始化列表构造结构体)
#define io_write(reg, ...) \
  ({ reg##_T __io_param = (reg##_T) { __VA_ARGS__ }; \
    ioe_write(reg, &__io_param); })
```

**使用示例**：
```c
// 读取系统运行时间 (微秒)
uint64_t uptime = io_read(AM_TIMER_UPTIME).us;

// 在屏幕(100,50)位置绘制像素
io_write(AM_GPU_FBDRAW, 100, 50, pixels, 32, 32, true);

// 读取键盘事件
AM_INPUT_KEYBRD_T ev = io_read(AM_INPUT_KEYBRD);
if (ev.keydown && ev.keycode == AM_KEY_ESCAPE) halt(0);
```

---

## 五、IOE 实现机制 — 查找表分发

### 5.1 问题：`ioe_read(AM_TIMER_UPTIME, &buf)` 是如何找到定时器实现函数的？

**解决方案**：用**函数指针查找表** (Look-Up Table, LUT) 实现分发。

```c
// abstract-machine/am/src/platform/nemu/ioe/ioe.c

typedef void (*handler_t)(void *buf);

// 查找表: 设备寄存器 ID → 处理函数
static void *lut[128] = {
  [AM_TIMER_CONFIG] = __am_timer_config,
  [AM_TIMER_RTC   ] = __am_timer_rtc,
  [AM_TIMER_UPTIME] = __am_timer_uptime,
  [AM_INPUT_CONFIG] = __am_input_config,
  [AM_INPUT_KEYBRD] = __am_input_keybrd,
  [AM_GPU_CONFIG  ] = __am_gpu_config,
  [AM_GPU_FBDRAW  ] = __am_gpu_fbdraw,
  [AM_GPU_STATUS  ] = __am_gpu_status,
  [AM_AUDIO_CONFIG] = __am_audio_config,
  [AM_AUDIO_CTRL  ] = __am_audio_ctrl,
  [AM_AUDIO_STATUS] = __am_audio_status,
  [AM_AUDIO_PLAY  ] = __am_audio_play,
  // ...
};

// 未注册的寄存器访问 → panic
static void fail(void *buf) { panic("access nonexist register"); }

bool ioe_init() {
  // 填充未注册的槽位为 fail 函数
  for (int i = 0; i < LENGTH(lut); i++)
    if (!lut[i]) lut[i] = fail;
  // 初始化各设备
  __am_gpu_init();
  __am_timer_init();
  __am_audio_init();
  return true;
}

// 核心分发: 通过 ID 索引查找表，调用对应处理函数
void ioe_read (int reg, void *buf) { ((handler_t)lut[reg])(buf); }
void ioe_write(int reg, void *buf) { ((handler_t)lut[reg])(buf); }
```

**调用链示例**：
```
io_read(AM_TIMER_UPTIME)
  → ioe_read(6, &buf)
    → lut[6] = __am_timer_uptime
      → __am_timer_uptime(&buf)
        → buf.us = get_time() - boot_time
```

---

## 六、各设备实现详解

### 6.1 定时器 (Timer)

**MMIO 地址**：`RTC_ADDR` (NPC: `0xa0002000`, NEMU: `0xa0000048`)

```c
// abstract-machine/am/src/riscv/npc/timer.c (NPC 平台)
static uint64_t boot_time = 0;

static uint64_t get_time() {
    uint32_t lo = inl(RTC_ADDR);       // 读低 32 位
    uint32_t hi = inl(RTC_ADDR + 4);   // 读高 32 位
    return ((uint64_t)hi << 32) + lo;   // 拼接为 64 位
}

void __am_timer_init() {
    boot_time = get_time();            // 记录启动时间
}

void __am_timer_uptime(AM_TIMER_UPTIME_T *uptime) {
    uptime->us = get_time() - boot_time;  // 返回启动后经过的微秒数
}
```

**应用使用**：
```c
uint64_t t0 = io_read(AM_TIMER_UPTIME).us;
// ... 做一些事情 ...
uint64_t elapsed = io_read(AM_TIMER_UPTIME).us - t0;
printf("耗时 %lld 微秒\n", elapsed);
```

### 6.2 键盘输入 (Input)

**MMIO 地址**：`KBD_ADDR` (NEMU: `0xa0000060`)

```c
// abstract-machine/am/src/platform/nemu/ioe/input.c
#define KEYDOWN_MASK 0x8000

void __am_input_keybrd(AM_INPUT_KEYBRD_T *kbd) {
  uint32_t temp_data = inl(KBD_ADDR);       // 读键盘状态寄存器
  kbd->keycode = temp_data & ~KEYDOWN_MASK;  // 低 15 位 = 键码
  kbd->keydown = temp_data >= KEYDOWN_MASK;  // 最高位 = 按下/释放
}
```

**键码定义** (`amdev.h`)：
```c
#define AM_KEYS(_) \
  _(ESCAPE) _(F1) _(F2) ... _(Q) _(W) _(E) _(R) ... _(SPACE) ...

#define AM_KEY_NAMES(key) AM_KEY_##key,
enum {
  AM_KEY_NONE = 0,
  AM_KEYS(AM_KEY_NAMES)  // AM_KEY_ESCAPE=1, AM_KEY_F1=2, ...
};
```

**应用使用**：
```c
while (1) {
  AM_INPUT_KEYBRD_T ev = io_read(AM_INPUT_KEYBRD);
  if (ev.keycode == AM_KEY_NONE) break;       // 无按键事件
  if (ev.keydown && ev.keycode == AM_KEY_ESCAPE) halt(0);  // ESC 退出
  if (ev.keydown) printf("按下了键: %d\n", ev.keycode);
}
```

### 6.3 GPU / 帧缓冲 (Framebuffer)

**MMIO 地址**：
- `VGACTL_ADDR` — 显示控制 (屏幕宽高)
- `FB_ADDR` — 帧缓冲区起始地址

```c
// abstract-machine/am/src/platform/nemu/ioe/gpu.c

void __am_gpu_config(AM_GPU_CONFIG_T *cfg) {
  uint32_t screen_wh = inl(VGACTL_ADDR);    // 宽高打包在一个 32-bit 寄存器中
  uint32_t w = screen_wh >> 16;             // 高 16 位 = 宽度
  uint32_t h = screen_wh & 0xffff;          // 低 16 位 = 高度
  *cfg = (AM_GPU_CONFIG_T){
    .present = true, .has_accel = false,
    .width = w, .height = h,
    .vmemsz = w * h * sizeof(uint32_t)
  };
}

void __am_gpu_fbdraw(AM_GPU_FBDRAW_T *ctl) {
  int x = ctl->x, y = ctl->y;
  int w = ctl->w, h = ctl->h;
  uint32_t *pixels = ctl->pixels;
  uint32_t screen_w = inl(VGACTL_ADDR) >> 16;

  // 逐像素写入帧缓冲
  for (int i = y; i < y + h; i++) {
    for (int j = x; j < x + w; j++) {
      outl(FB_ADDR + (i * screen_w + j) * 4,
           pixels[w * (i - y) + (j - x)]);
    }
  }

  // 写同步寄存器触发屏幕刷新
  if (ctl->sync) {
    outl(VGACTL_ADDR + 4, 1);
  }
}
```

**应用使用**：
```c
// 获取屏幕尺寸
int w = io_read(AM_GPU_CONFIG).width;
int h = io_read(AM_GPU_CONFIG).height;

// 在 (100, 50) 处绘制一个 32x32 的图块
uint32_t pixels[32 * 32];
// ... 填充像素数据 (0xRRGGBB 格式) ...
io_write(AM_GPU_FBDRAW, 100, 50, pixels, 32, 32, true);
//                       x    y   data   w   h   sync
```

### 6.4 音频 (Audio)

```c
// abstract-machine/am/src/platform/nemu/ioe/audio.c
void __am_audio_ctrl(AM_AUDIO_CTRL_T *ctrl) {
  outl(AUDIO_FREQ_ADDR, ctrl->freq);        // 设置采样率
  outl(AUDIO_CHANNELS_ADDR, ctrl->channels);// 设置通道数
  outl(AUDIO_SAMPLES_ADDR, ctrl->samples);  // 设置样本数
  outl(AUDIO_INIT_ADDR, 1);                 // 启动音频
}

void __am_audio_play(AM_AUDIO_PLAY_T *ctl) {
  // 将音频数据逐字节写入环形缓冲区
  uintptr_t ptr = (uintptr_t)ctl->buf.start;
  uintptr_t end = (uintptr_t)ctl->buf.end;
  int block_size = inl(AUDIO_SBUF_SIZE_ADDR);
  for (; ptr < end;) {
    int count = inl(AUDIO_COUNT_ADDR);
    if (count == block_size) continue;  // 缓冲区满，等待
    for (; ptr < end && count < block_size; ++ptr, ++count) {
      outb(AUDIO_SBUF_ADDR + count, *((unsigned char*)ptr));
    }
    outl(AUDIO_COUNT_ADDR, count);
  }
}
```

---

## 七、NPC 平台 vs NEMU 平台的差异

### 7.1 设备地址不同

| 设备 | NPC 平台 | NEMU 平台 |
|------|----------|-----------|
| 串口 | `0xa0000000` | `0xa00003f8` |
| 键盘 | `0xa0001000` | `0xa0000060` |
| 定时器 | `0xa0002000` | `0xa0000048` |
| 显示控制 | `0xa0003000` | `0xa0000100` |
| 帧缓冲 | `0xa1000000` | `0xa1000000` |

### 7.2 NPC 平台的简化

NPC 平台的某些设备实现是 stub (占位)：

```c
// abstract-machine/am/src/riscv/npc/input.c
void __am_input_keybrd(AM_INPUT_KEYBRD_T *kbd) {
  kbd->keydown = 0;
  kbd->keycode = AM_KEY_NONE;  // 永远返回"无按键"
}
```

这是因为 NPC 的 RTL 仿真中可能还未实现键盘设备。

---

## 八、完整应用示例分析

### 8.1 Hello World (`am-kernels/kernels/hello/`)

```c
// hello.c
#include <am.h>
#include <klib-macros.h>

int main(const char *args) {
  const char *fmt = "Hello, AbstractMachine!\nmainargs = '%'.\n";
  for (const char *p = fmt; *p; p++) {
    (*p == '%') ? putstr(args) : putch(*p);
  }
  return 0;
}
```

Makefile 只需三行：
```makefile
NAME = hello
SRCS = hello.c
include $(AM_HOME)/Makefile
```

运行：
```bash
cd am-kernels/kernels/hello
make ARCH=riscv32e-npc run       # 在 NPC 上运行
make ARCH=riscv32-nemu run       # 在 NEMU 上运行
make ARCH=native run             # 在 Linux 上原生运行
```

### 8.2 打字游戏 (`am-kernels/kernels/typing-game/`)

这是一个完整使用 IOE 各子系统的应用：

```c
// game.c (核心逻辑)
int main() {
  ioe_init();       // 初始化 I/O 子系统
  video_init();     // 初始化显示 (读取屏幕尺寸，清屏)

  // 检查设备是否存在
  panic_on(!io_read(AM_TIMER_CONFIG).present, "requires timer");
  panic_on(!io_read(AM_INPUT_CONFIG).present, "requires keyboard");

  uint64_t t0 = io_read(AM_TIMER_UPTIME).us;  // 记录开始时间

  while (1) {
    // 1. 计算当前帧数 (基于定时器)
    int frames = (io_read(AM_TIMER_UPTIME).us - t0) / (1000000 / FPS);

    // 2. 更新游戏逻辑
    for (; current < frames; current++) {
      game_logic_update(current);
    }

    // 3. 处理键盘输入
    while (1) {
      AM_INPUT_KEYBRD_T ev = io_read(AM_INPUT_KEYBRD);
      if (ev.keycode == AM_KEY_NONE) break;
      if (ev.keydown && ev.keycode == AM_KEY_ESCAPE) halt(0);
      if (ev.keydown && lut[ev.keycode]) check_hit(lut[ev.keycode]);
    }

    // 4. 渲染画面 (写帧缓冲)
    if (current > rendered) {
      render();  // 内部调用 io_write(AM_GPU_FBDRAW, ...)
      rendered = current;
    }
  }
}
```

**这份代码可以不做任何修改，跑在 NEMU、NPC、QEMU、甚至 Linux 上！**

---

## 九、AM 构建系统详解

### 9.1 ARCH 变量决定一切

```bash
make ARCH=riscv32e-npc run    # 编译为 RV32E, 运行在 NPC
make ARCH=riscv32-nemu run    # 编译为 RV32IM, 运行在 NEMU
make ARCH=native run          # 编译为 x86_64, 运行在 Linux
```

`ARCH` 由 `-` 分割为 `ISA` 和 `PLATFORM`：
```makefile
ARCH_SPLIT = $(subst -, ,$(ARCH))
ISA        = $(word 1,$(ARCH_SPLIT))    # riscv32e
PLATFORM   = $(word 2,$(ARCH_SPLIT))    # npc
```

### 9.2 构建层级

```
$AM_HOME/scripts/riscv32e-npc.mk           # 顶层: 组合 ISA + 平台
  ├── include scripts/isa/riscv.mk         # ISA: 交叉编译器、march
  └── include scripts/platform/npc.mk      # 平台: 链接脚本、run 规则、AM_SRCS
```

### 9.3 库链接顺序

```makefile
# 应用会自动链接这些库:
LIBS := $(sort $(LIBS) am klib)
# 最终链接命令:
$(LD) ... --start-group user.o am-riscv32e-npc.a klib-riscv32e-npc.a --end-group
```

`--start-group ... --end-group` 解决循环依赖问题 (klib 依赖 am 的 putch，am 依赖 klib 的 memcpy)。

---

## 十、底层 MMIO 访问机制

### 10.1 `inl` / `outl` 的本质

```c
// abstract-machine/am/src/riscv/riscv.h
static inline uint32_t inl(uintptr_t addr) {
  return *(volatile uint32_t *)addr;   // 对特定地址做内存读
}

static inline void outl(uintptr_t addr, uint32_t data) {
  *(volatile uint32_t *)addr = data;   // 对特定地址做内存写
}
```

在硬件层面，这些地址不对应 RAM，而是对应设备控制器的寄存器。读写这些地址会触发设备的行为。

### 10.2 NEMU 中的设备分发

```c
// nemu/src/memory/paddr.c
word_t paddr_read(paddr_t addr, int len) {
  if (likely(in_pmem(addr))) return pmem_read(addr, len);  // 普通内存
  IFDEF(CONFIG_DEVICE, return mmio_read(addr, len));       // MMIO 设备
  out_of_bound(addr);
}
```

当地址落在 `0xa0000000` 范围时，`mmio_read` 会根据地址查找对应设备并调用其读回调。

---

## 十一、常见问题与解决方案

### Q1: `ioe_init()` 返回 false 或 panic
**原因**：某个设备的初始化函数中访问了未实现的 MMIO 地址
**解决**：确保 NEMU/NPC 已正确处理对应 MMIO 地址的读写

### Q2: 定时器返回值不变或一直为 0
**原因**：NEMU/NPC 的定时器设备未实现，读 `RTC_ADDR` 总是返回 0
**解决**：在模拟器中实现定时器设备 (通常使用 `gettimeofday` 获取真实时间)

### Q3: 帧缓冲写入后屏幕不刷新
**原因**：忘记写同步寄存器 (`sync = true`)
**解决**：
```c
io_write(AM_GPU_FBDRAW, x, y, pixels, w, h, true);  // 最后参数 sync=true
// 或者批量绘制后手动同步:
io_write(AM_GPU_FBDRAW, 0, 0, NULL, 0, 0, true);    // 空绘制 + sync
```

### Q4: 键盘事件丢失
**原因**：键盘事件队列有限，需要在每帧中循环读取直到 `AM_KEY_NONE`
**正确做法**：
```c
while (1) {
  AM_INPUT_KEYBRD_T ev = io_read(AM_INPUT_KEYBRD);
  if (ev.keycode == AM_KEY_NONE) break;  // 队列空了
  // 处理事件...
}
```

### Q5: 为什么 NPC 平台的 `input.c` 是空的？
NPC RTL 设计阶段可能还未实现键盘控制器。AM 的设计允许设备实现为 stub，应用通过 `io_read(AM_INPUT_CONFIG).present` 检测设备是否存在。

### Q6: `io_read` 和 `io_write` 宏为什么用 `({...})` 语法？
这是 GCC 扩展"语句表达式" (Statement Expression)：
- 允许在表达式中嵌入多条语句
- 最后一条表达式的值作为整个 `({...})` 的值
- 变量 `__io_param` 的作用域限制在内部，不会污染外层

---

## 十二、实践：编写一个 AM 应用

### 12.1 最小 IOE 应用 — 打印系统运行时间

```c
// my_timer.c
#include <am.h>
#include <klib.h>
#include <klib-macros.h>

int main() {
  ioe_init();

  if (!io_read(AM_TIMER_CONFIG).present) {
    printf("Timer not present!\n");
    return 1;
  }

  for (int i = 0; i < 10; i++) {
    uint64_t us = io_read(AM_TIMER_UPTIME).us;
    printf("Uptime: %lld us\n", us);
    // 简易忙等 (约 100ms)
    uint64_t target = us + 100000;
    while (io_read(AM_TIMER_UPTIME).us < target);
  }

  return 0;
}
```

Makefile:
```makefile
NAME = my_timer
SRCS = my_timer.c
include $(AM_HOME)/Makefile
```

运行：
```bash
make ARCH=riscv32e-npc run
```

---

## 十三、学习建议

1. **从 hello 开始**：确保最基本的 `putch` 输出正确
2. **然后加入定时器**：实现 `__am_timer_uptime`，验证时间在增长
3. **再加入键盘**：实现设备寄存器读取，用简单程序测试
4. **最后实现 GPU**：帧缓冲写入是最复杂的部分
5. **用 typing-game 作为终极测试**：它综合使用了定时器 + 键盘 + GPU
6. **对比 NEMU 和 NPC 的 IOE 实现**：理解同一 API 如何适配不同硬件

---

*参考资料：*
- *Abstract-Machine 设计文档*
- *RISC-V MMIO 规范*
- *项目源码: abstract-machine/am/src/, am-kernels/kernels/*
- *一生一芯官方文档: https://ysyx.oscc.cc*


---

## 十四、PA 讲义思考题回答 (PA2.3 AM / PA2.5 IOE 部分)

> 以下问题来自 PA 讲义中与 AM 运行时环境和 I/O 相关的核心思考题。

---

### PA2.3 程序, 运行时环境与 AM

#### Q: 程序在裸金属上运行 vs 在操作系统上运行有什么区别？

**回答**：

| 方面 | 裸金属 (AM) | 操作系统 (Linux) |
|------|-------------|-----------------|
| 启动 | start.S → _trm_init → main | _start → __libc_start_main → main |
| 退出 | halt() → ebreak | exit() → sys_exit 系统调用 |
| 内存 | 直接访问物理内存 | 虚拟地址空间, mmap/brk |
| I/O | MMIO (volatile 指针) | 系统调用 (read/write/ioctl) |
| 并发 | 无（单线程,无中断） | 多进程/线程, 抢占调度 |
| 库函数 | klib (自己实现) | glibc (完整实现) |
| 保护 | 无（全权限 M-mode） | 有（用户态不能直接访问硬件） |

---

#### Q: `_trm_init()` 为什么不是 `main()`？为什么不能直接从 main 开始？

**回答**：

因为 `main()` 是**用户程序的入口**，它假设：
1. 栈已经设置好了（需要 start.S 中的 `la sp, _stack_pointer`）
2. 全局变量已初始化（bss 段清零 + data 段就绪）
3. main 返回后有人调用 halt

`_trm_init()` 就是负责这些"main 之前和之后"事情的函数：

```c
void _trm_init() {
  int ret = main(mainargs);   // main 之前的准备由 start.S 完成
  halt(ret);                  // main 返回后调用 halt
}
```

如果直接跳转到 main，main 返回后 PC 会跑飞（因为 `jal main` 的返回地址 ra 指向哪里取决于调用者）。

---

#### Q: heap（堆）是怎么来的？`malloc` 怎么知道从哪里开始分配？

**回答**：

```c
// trm.c
extern char _heap_start;   // 链接脚本中定义的符号
Area heap = RANGE(&_heap_start, PMEM_END);
```

- `_heap_start` 是链接脚本在所有段（.text/.data/.bss/栈）之后定义的符号
- `PMEM_END` = `0x80000000 + 128MB`
- 所以 heap 就是"栈顶到物理内存末尾"之间的全部空间

`malloc` 使用 bump allocator：
```c
static void* addr = NULL;
void* malloc(size_t size) {
  if (addr == NULL) addr = heap.start;
  void* ret = addr;
  addr += size;  // 只增不减
  return ret;
}
```

---

#### Q: `io_read(AM_TIMER_UPTIME)` 这个宏是如何工作的？

**回答**：展开过程：

```c
// 原始调用
uint64_t us = io_read(AM_TIMER_UPTIME).us;

// 宏展开 (klib-macros.h)
uint64_t us = ({
  AM_TIMER_UPTIME_T __io_param;           // 创建临时结构体
  ioe_read(AM_TIMER_UPTIME, &__io_param); // 调用 IOE 读取
  __io_param;                              // 返回结构体
}).us;                                     // 访问 us 成员

// ioe_read 内部
void ioe_read(int reg, void *buf) {
  ((handler_t)lut[reg])(buf);  // lut[6] = __am_timer_uptime
}

// 最终调用
void __am_timer_uptime(AM_TIMER_UPTIME_T *uptime) {
  uptime->us = get_time() - boot_time;  // 读 MMIO 获取时间
}
```

这个设计的精妙之处：**应用程序不需要知道底层是 MMIO、是系统调用、还是 DPI-C，只需要用统一的 `io_read` 接口**。

---

#### Q: 为什么 IOE 使用查找表 (LUT) 而不是 switch-case？

**回答**：

1. **O(1) 分发**：LUT 通过数组索引直接获得函数指针，比 switch-case 的分支跳转更高效
2. **可扩展**：添加新设备只需在 LUT 中添加一行，不用修改 ioe_read/ioe_write 函数
3. **平台解耦**：不同平台只需提供不同的 LUT 内容，ioe_read/ioe_write 代码完全相同
4. **运行时灵活**：可以动态替换处理函数（虽然本项目中没用到这一点）

---

#### Q: 帧缓冲 (framebuffer) 是如何工作的？

**回答**：

帧缓冲 = 一块和屏幕像素一一对应的内存区域。每个 4 字节对应一个像素 (0x00RRGGBB 格式)。

```
FB_ADDR = 0xa1000000

显示器 (400x300):
  像素(0,0)  → FB_ADDR + 0
  像素(1,0)  → FB_ADDR + 4
  像素(2,0)  → FB_ADDR + 8
  ...
  像素(0,1)  → FB_ADDR + 400*4
  像素(x,y)  → FB_ADDR + (y * width + x) * 4
```

**写入过程**：
```c
void __am_gpu_fbdraw(AM_GPU_FBDRAW_T *ctl) {
  for (int i = y; i < y + h; i++)
    for (int j = x; j < x + w; j++)
      outl(FB_ADDR + (i * screen_w + j) * 4, pixels[...]);
  if (ctl->sync) outl(SYNC_ADDR, 1);  // 写同步寄存器触发刷新
}
```

**为什么需要 sync？** 如果每写一个像素就刷新屏幕，会非常卡。所以先批量写入帧缓冲，最后写同步寄存器一次性刷新，避免画面撕裂。

---

#### Q: 键盘输入事件是"推"模型还是"拉"模型？

**回答**：**拉模型 (polling)**。

应用程序主动读取键盘状态寄存器：
```c
AM_INPUT_KEYBRD_T ev = io_read(AM_INPUT_KEYBRD);
```

如果有按键事件，返回键码和按下/释放状态；如果没有，返回 `AM_KEY_NONE`。

这与中断驱动的"推模型"不同——在 D3 阶段还没有中断机制，所以只能用 polling。应用程序必须**主动且频繁地**查询键盘，否则会丢失按键事件。

典型的游戏循环模式：
```c
while (1) {
  // 排空键盘事件队列
  while (1) {
    AM_INPUT_KEYBRD_T ev = io_read(AM_INPUT_KEYBRD);
    if (ev.keycode == AM_KEY_NONE) break;
    handle_key(ev);
  }
  // 更新逻辑 + 渲染
  update();
  render();
}
```

---

#### Q: 如何确保程序在不同平台上有一致的行为？以 typing-game 为例

**回答**：

typing-game 使用**帧率锁定**的方式保证一致性：

```c
uint64_t t0 = io_read(AM_TIMER_UPTIME).us;
while (1) {
  int frames = (io_read(AM_TIMER_UPTIME).us - t0) / (1000000 / FPS);
  for (; current < frames; current++) {
    game_logic_update(current);  // 每 1/30 秒更新一次逻辑
  }
}
```

不管 NEMU 执行指令快还是慢，游戏逻辑的更新频率始终由**真实时间**（定时器）控制。这样：
- 在快速的 native 平台上：渲染流畅，但逻辑更新频率不变
- 在慢速的 RTL 仿真上：渲染卡顿，但逻辑更新频率依然正确

这就是为什么定时器设备返回的是**主机的真实时间**而非模拟时间。

---

#### Q: 裸金属程序没有 OS，`printf` 怎么实现的？

**回答**：

klib 中自己实现了完整的 printf/sprintf（参考了开源的 mini-printf 实现）：

```
printf("hello %d", 42)
  → vsprintf(buf, "hello %d", args)   // 格式化到缓冲区
    → 逐字符解析格式串
    → 遇到 %d → 将 42 转为字符串 "42"
  → putstr(buf)                         // 逐字符输出
    → for each char: putch(ch)          // 调用 TRM 的 putch
      → outb(SERIAL_PORT, ch)           // 写串口 MMIO
```

整条链路**完全不依赖操作系统**——从格式化到物理输出，全部由裸金属代码完成。

---

#### Q: AM 程序能否动态链接？能否使用 glibc？

**回答**：**不能**。

- **动态链接**需要操作系统提供动态链接器 (ld-linux.so)，裸金属没有
- **glibc** 内部大量依赖系统调用 (mmap, write, brk, futex 等)，裸金属没有 OS 来处理

所以 AM 程序必须：
1. 静态链接 (`-static`)
2. 使用自己的 klib 而非 glibc (`-fno-builtin`)
3. 不能用任何需要 OS 支持的功能 (文件系统、网络、线程...)

这也解释了为什么编译选项中有 `-fno-builtin` —— 防止 GCC 将 `memcpy` 等替换为内置版本（内置版本可能依赖 glibc）。

---

*注：以上回答基于 NJU ICS PA 2024 讲义核心思考题，结合本项目源码分析。*
*PA 讲义来源：https://nju-projectn.github.io/ics-pa-gitbook/ics2024/*


---

## 十五、本阶段 Git 版本记录

| Commit | 说明 | 完成内容 |
|--------|------|----------|
| `fa79548` | NJU-ProjectN/abstract-machine ics2023 initialized | AM 框架引入，TRM/IOE API 就绪 |
| `e5276c7` | fix bug and do real-time clock test | IOE 定时器设备实现与验证 |
| `702fcd3` | test benchmarks | 在 AM 上运行 CoreMark/Dhrystone |

**查看对应版本代码**：
```bash
git show e5276c7:abstract-machine/am/src/platform/nemu/ioe/timer.c   # 定时器
git show e5276c7:abstract-machine/am/src/platform/nemu/ioe/ioe.c     # IOE 框架
```


---

## 十六、Git 版本代码演进详解

### 16.1 定时器修复 (`e5276c7`)

这个 commit 修复了 IOE 定时器的实现问题并做了真实时钟测试:

**核心变化**: AM 端的 `timer.c` 正确实现了通过 MMIO 读取 64-bit 时间戳:
```c
static uint64_t get_time() {
    uint32_t lo = inl(RTC_ADDR);      // 先读低32位 (触发时间更新)
    uint32_t hi = inl(RTC_ADDR + 4);  // 再读高32位
    return ((uint64_t)hi << 32) + lo;
}
```

**验证方法**: 运行一个打印系统运行时间的测试程序，确认时间在持续增长。

---

### 16.2 基准测试 (`702fcd3`)

此 commit 成功在 NEMU/NPC 上运行了 CoreMark 和 Dhrystone:

**意义**:
- CoreMark 是嵌入式处理器标准性能基准
- 如果能跑通 CoreMark，说明 RV32IM 指令集 + AM IOE 框架 + klib 全部正确
- 这是 D3 阶段的最终验收——"复杂真实程序"在你的系统上正确运行

**CoreMark 涉及的 AM 功能**:
- `putch()` — 输出结果 (TRM)
- `io_read(AM_TIMER_UPTIME)` — 计时 (IOE)
- `malloc()` — 动态内存分配 (klib)
- `printf()` — 格式化输出 (klib)
