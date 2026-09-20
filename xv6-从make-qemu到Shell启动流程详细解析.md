# xv6：从 `make qemu` 到 Shell 的完整启动流程

本文以以下仓库为准：

```text
/home/wangxin/xv6-labs-2025
```

分析对象是该仓库当前的 2025 版本代码。当前 `conf/lab.mk` 配置为：

```make
LAB=util
```

这个版本与一些经典 xv6 教材代码存在一个重要区别：第一个进程不再先执行内嵌的 `initcode`，而是在 `forkret()` 中直接通过 `kexec("/init", ...)` 从文件系统加载 `/init`。

---

## 1. 总体启动路线

从宿主机命令到 xv6 Shell，整个流程可以概括为：

```text
宿主 Ubuntu
   │
   │ make qemu
   ▼
GNU Make 解析 Makefile
   ├── 检查 QEMU 版本
   ├── 备份旧 fs.img
   ├── 编译并链接 kernel/kernel
   ├── 编译用户程序
   └── 创建新的 fs.img
           │
           ▼
qemu-system-riscv64
   ├── 创建 RISC-V virt 虚拟机
   ├── 创建 3 个 hart
   ├── 提供 128 MiB 内存
   ├── 加载 kernel/kernel
   └── 将 fs.img 接为 VirtIO 磁盘
           │
           ▼
物理地址 0x80000000：_entry
           │
           ▼
entry.S：为每个 hart 建立启动栈
           │
           ▼
start.c：M-mode 初始化并执行 mret
           │
           ▼
main.c：进入 S-mode，初始化内核
           │
           ▼
userinit()：创建第一个进程
           │
           ▼
scheduler() → swtch() → forkret()
           │
           ▼
fsinit() → kexec("/init")
           │
           ▼
trampoline.S：执行 sret 进入 U-mode
           │
           ▼
/init → fork() → exec("sh")
           │
           ▼
出现 xv6 Shell 提示符 `$`
```

---

## 2. `make qemu` 从哪里开始

Makefile 中的目标为：

```make
# makes a new fs.img
qemu: check-qemu-version newfs.img $K/kernel fs.img
	$(QEMU) $(QEMUOPTS)
```

GNU Make 规则的基本形式是：

```make
目标: 依赖项
	构建命令
```

因此，`make qemu` 不会立即启动 QEMU。Make 必须先处理以下依赖：

1. `check-qemu-version`
2. `newfs.img`
3. `$K/kernel`
4. `fs.img`

当这些依赖都准备完成后，才会运行：

```make
$(QEMU) $(QEMUOPTS)
```

变量定义为：

```make
K=kernel
QEMU=qemu-system-riscv64
```

所以 `$K/kernel` 展开为：

```text
kernel/kernel
```

---

## 3. 检查 QEMU 版本

Makefile 要求 QEMU 至少为 7.2：

```make
MIN_QEMU_VERSION = 7.2

QEMU_VERSION := $(shell $(QEMU) --version | head -n 1 | \
  sed -E 's/^QEMU emulator version ([0-9]+\.[0-9]+)\..*/\1/')

check-qemu-version:
	@if [ "$(shell echo "$(QEMU_VERSION) >= $(MIN_QEMU_VERSION)" | bc)" -eq 0 ]; then \
		echo "ERROR: Need qemu version >= $(MIN_QEMU_VERSION)"; \
		exit 1; \
	fi
```

处理过程为：

```text
qemu-system-riscv64 --version
        │
        ▼
提取主版本和次版本，例如 8.2
        │
        ▼
使用 bc 判断 8.2 >= 7.2
```

当前机器上的 QEMU 是 8.2.2，满足要求。

`check-qemu-version` 位于 `.PHONY` 列表中，所以即使目录中偶然存在同名文件，每次执行 `make qemu` 时仍会进行版本检查。

---

## 4. `newfs.img` 为什么会备份旧文件系统

规则如下：

```make
newfs.img:
	-mv -f fs.img fs.img.bk
```

它执行的是：

```bash
mv -f fs.img fs.img.bk
```

效果是：

```text
旧 fs.img ──移动──► fs.img.bk
```

命令前面的 `-` 是 Makefile 的特殊前缀，表示忽略该命令的失败状态。第一次运行时可能还没有 `fs.img`，此时 `mv` 会失败，但 Make 仍继续构建。

这个规则的目标名是 `newfs.img`，但命令没有真正生成这个文件。通常桌面目录中也不存在 `newfs.img`，因此下次执行 `make qemu` 时，该规则还会再次运行。

旧 `fs.img` 被移动后，后面的 `fs.img` 依赖便需要重新构建。因此：

```text
make qemu    → 通常创建全新的文件系统镜像
make qemu-fs → 保留已有的 fs.img（如果它仍是最新的）
```

相关规则是：

```make
qemu-fs: check-qemu-version $K/kernel fs.img
	$(QEMU) $(QEMUOPTS)
```

如果希望保留上一次在 xv6 中创建的文件，应使用 `make qemu-fs`。

### 一个并行构建注意事项

普通的 `make qemu` 按顺序处理这些依赖时，先移动旧镜像，再重新生成镜像。不要轻易对这个目标使用 `make -j qemu`，因为 `newfs.img` 和 `fs.img` 没有用显式依赖边建立严格顺序，并行执行时存在竞争可能。

---

## 5. 内核对象文件如何生成

内核对象文件保存在 `OBJS` 和 `OBJS_KCSAN` 中，例如：

```make
OBJS = \
  $K/entry.o \
  $K/kalloc.o \
  $K/string.o \
  $K/main.o \
  $K/vm.o \
  $K/proc.o \
  $K/swtch.o \
  $K/trampoline.o \
  $K/trap.o \
  $K/syscall.o \
  ...
```

模式规则负责将 C 和汇编源文件编译为对象文件：

```make
$K/%.o: $K/%.c
	$(CC) $(CFLAGS) $(EXTRAFLAG) -c -o $@ $<

$K/%.o: $K/%.S
	$(CC) -g -c -o $@ $<
```

例如：

```text
kernel/main.c       ──编译──► kernel/main.o
kernel/proc.c       ──编译──► kernel/proc.o
kernel/entry.S      ──汇编──► kernel/entry.o
kernel/trampoline.S ──汇编──► kernel/trampoline.o
```

其中自动变量的含义为：

- `$@`：当前目标文件，例如 `kernel/main.o`
- `$<`：第一个依赖文件，例如 `kernel/main.c`

### 5.1 交叉编译工具链

Makefile 会尝试寻找以下工具链前缀：

```text
riscv64-unknown-elf-
riscv64-elf-
riscv64-linux-gnu-
riscv64-unknown-linux-gnu-
```

当前系统可以使用：

```text
riscv64-linux-gnu-gcc
riscv64-linux-gnu-ld
riscv64-linux-gnu-objdump
```

这些程序在 x86-64 Ubuntu 上运行，但生成的是 RISC-V 机器代码，因此称为交叉工具链。

### 5.2 关键编译选项

```make
CFLAGS += -mcmodel=medany
CFLAGS += -ffreestanding
CFLAGS += -fno-common -nostdlib
CFLAGS += -fno-stack-protector
```

主要含义：

- `-mcmodel=medany`：允许代码在较大地址范围中以 PC 相对方式访问符号，适合位于 `0x80000000` 的内核。
- `-ffreestanding`：告诉编译器这是没有普通宿主运行环境的独立程序。
- `-nostdlib`：不自动链接 Ubuntu C 标准库和启动代码。
- `-fno-stack-protector`：不插入依赖宿主运行时的栈保护逻辑。
- `-ggdb -gdwarf-2`：保留调试信息，便于 GDB 和 `objdump -S` 使用。
- `-fno-omit-frame-pointer`：保留帧指针，便于调用栈回溯。

---

## 6. 链接 `kernel/kernel`

链接规则是：

```make
$K/kernel: $(OBJS) $(OBJS_KCSAN) $K/kernel.ld
	$(LD) $(LDFLAGS) -T $K/kernel.ld \
	  -o $K/kernel $(OBJS) $(OBJS_KCSAN)
	$(OBJDUMP) -S $K/kernel > $K/kernel.asm
	$(OBJDUMP) -t $K/kernel | \
	  sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $K/kernel.sym
```

链接器根据 `kernel/kernel.ld` 合并所有对象文件，生成最终 RISC-V ELF 内核：

```text
kernel/*.o + kernel/kernel.ld
              │
              ▼
       RISC-V 链接器
              │
              ▼
       kernel/kernel
          │       │
          │       └──► kernel/kernel.sym
          └──────────► kernel/kernel.asm
```

链接脚本开头是：

```ld
OUTPUT_ARCH("riscv")
ENTRY(_entry)

SECTIONS
{
  . = 0x80000000;

  .text : {
    kernel/entry.o(_entry)
    *(.text .text.*)
    ...
  }
}
```

它规定：

1. 输出文件属于 RISC-V 架构。
2. ELF 入口符号是 `_entry`。
3. 内核从地址 `0x80000000` 开始布局。
4. `entry.o` 中的 `_entry` 放在代码段最前面。
5. 后面依次放置 `.text`、`.rodata`、`.data` 和 `.bss`。

`kernel/kernel` 是 QEMU 实际加载的内核；`.asm` 与 `.sym` 只是分析辅助文件。

---

## 7. 用户程序和 `fs.img` 的构建

文件系统镜像规则为：

```make
fs.img: mkfs/mkfs README $(UEXTRA) $(UPROGS)
	mkfs/mkfs fs.img README $(UEXTRA) $(UPROGS)
```

构建链为：

```text
用户程序源文件
      │
      ▼
user/_init、user/_sh、user/_cat 等 ELF 程序
      │
      ├── README
      ├── UEXTRA
      └── mkfs/mkfs
              │
              ▼
            fs.img
```

当前是 `LAB=util`，所以 Makefile 还加入：

```make
UEXTRA += user/findtest.sh
UEXTRA += user/sixfive.txt
UPROGS += $U/_memdump
```

`mkfs/mkfs` 在宿主 Ubuntu 中运行，创建 xv6 文件系统，然后把用户程序和普通文件写入根目录。宿主文件 `user/_init` 写入镜像时会去掉下划线，最终成为 xv6 中的：

```text
/init
```

同理：

```text
user/_sh  → /sh
user/_cat → /cat
user/_ls  → /ls
```

这一步直接决定了内核启动后能否执行 `kexec("/init")`。

---

## 8. QEMU 命令行展开

Makefile 中定义：

```make
QEMUOPTS = -machine virt -bios none -kernel $K/kernel \
           -m 128M -smp $(CPUS) -nographic
QEMUOPTS += -global virtio-mmio.force-legacy=false
QEMUOPTS += -drive file=fs.img,if=none,format=raw,id=x0
QEMUOPTS += -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0
```

当前 `LAB=util`，没有触发 `LAB=fs` 的单核设置，因此：

```make
CPUS=3
```

最终命令近似为：

```bash
qemu-system-riscv64 \
  -machine virt \
  -bios none \
  -kernel kernel/kernel \
  -m 128M \
  -smp 3 \
  -nographic \
  -global virtio-mmio.force-legacy=false \
  -drive file=fs.img,if=none,format=raw,id=x0 \
  -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0
```

### 8.1 `-machine virt`

创建 QEMU 的通用 RISC-V 虚拟开发板。它提供 UART、PLIC、VirtIO MMIO 和物理内存等设备。

xv6 的 `kernel/memlayout.h` 按照该虚拟机布局定义地址：

```text
0x02000000  CLINT
0x0c000000  PLIC
0x10000000  UART0
0x10001000  VirtIO 磁盘 MMIO
0x80000000  RAM 和内核起始位置
```

### 8.2 `-bios none`

表示不加载 OpenSBI 等外部固件。xv6 从机器态开始运行，并在自己的 `start.c` 中配置机器态寄存器、定时器、PMP 和特权级切换。

### 8.3 `-kernel kernel/kernel`

QEMU 读取内核 ELF，按照其中的段地址将内核装载到物理内存。链接脚本把入口 `_entry` 安排在 `0x80000000`。

### 8.4 `-m 128M`

提供 128 MiB RAM，与 xv6 定义一致：

```c
#define KERNBASE 0x80000000L
#define PHYSTOP (KERNBASE + 128*1024*1024)
```

### 8.5 `-smp 3`

创建三个 RISC-V hart。hart 是 RISC-V 对硬件线程或逻辑 CPU 的称呼，编号通常为 0、1、2。

### 8.6 `-nographic`

不创建图形窗口，把虚拟 UART 串口连接到当前终端。因此 `printf()` 输出和 Shell 输入都出现在执行 `make qemu` 的终端中。

### 8.7 VirtIO 磁盘参数

```text
-drive file=fs.img,if=none,format=raw,id=x0
```

创建一个以 `fs.img` 为内容的磁盘后端，但暂时不自动连接某种控制器。

```text
-device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0
```

创建 VirtIO 块设备前端，将它连接到 RISC-V virt 机器的 MMIO 总线，并使用前面的 `x0` 磁盘后端。

```text
-global virtio-mmio.force-legacy=false
```

强制使用现代 VirtIO MMIO 接口。xv6 驱动会检查设备版本是否为 2。

---

## 9. QEMU 把控制权交给 `_entry`

QEMU 启动后，把内核加载到 `0x80000000`。链接脚本又指定：

```ld
ENTRY(_entry)
. = 0x80000000;
```

因此每个 hart 最终从 `kernel/entry.S` 的 `_entry` 开始执行。

这一刻的关键状态是：

- CPU 处于机器态 M-mode。
- 分页没有启用。
- 还没有 C 语言运行栈。
- 全局 C 初始化函数还没有执行。
- 每个 hart 都运行相同的 `_entry` 代码。

---

## 10. `entry.S`：为每个 hart 建立栈

入口代码是：

```asm
.section .text
.global _entry
_entry:
        la sp, stack0
        li a0, 1024*4
        csrr a1, mhartid
        addi a1, a1, 1
        mul a0, a0, a1
        add sp, sp, a0
        call start
spin:
        j spin
```

`stack0` 定义在 `start.c`：

```c
__attribute__((aligned(16))) char stack0[4096 * NCPU];
```

每个 hart 的初始栈指针计算公式为：

```text
sp = stack0 + (mhartid + 1) × 4096
```

例如：

```text
hart 0：sp = stack0 + 4096
hart 1：sp = stack0 + 8192
hart 2：sp = stack0 + 12288
```

栈向低地址增长，因此这里把 `sp` 设置到每块 4096 字节栈空间的顶部。

使用不同的栈非常重要。如果所有 hart 共用同一个栈，并发执行 C 函数时，返回地址和局部变量会互相覆盖。

栈准备完成后：

```asm
call start
```

跳转到 `kernel/start.c` 中的 `start()`。如果 `start()` 意外返回，代码会在 `spin` 中无限循环，避免继续执行未知内存。

---

## 11. `start()`：机器态初始化

`start()` 仍然运行在 M-mode，但目标是准备好 S-mode 环境并跳入 `main()`。

### 11.1 设置 `mret` 返回到 S-mode

```c
unsigned long x = r_mstatus();
x &= ~MSTATUS_MPP_MASK;
x |= MSTATUS_MPP_S;
w_mstatus(x);
```

`mstatus.MPP` 表示执行 `mret` 后进入的特权级。这里将其设置为 Supervisor。

### 11.2 指定 `mret` 的目标地址

```c
w_mepc((uint64)main);
```

`mepc` 是机器态异常程序计数器。执行 `mret` 时，PC 将从 `mepc` 恢复，所以控制流将进入：

```text
kernel/main.c: main()
```

### 11.3 暂时关闭分页

```c
w_satp(0);
```

此时地址转换关闭，内核暂时使用物理地址。真正的内核页表将在 `main()` 中创建。

### 11.4 将异常和中断委托给 S-mode

```c
w_medeleg(0xffff);
w_mideleg(0xffff);
w_sie(r_sie() | SIE_SEIE | SIE_STIE);
```

作用是尽可能让系统调用、异常和外部/定时器中断直接进入 S-mode，由 xv6 内核处理，而不是始终返回 M-mode。

### 11.5 配置 PMP

```c
w_pmpaddr0(0x3fffffffffffffull);
w_pmpcfg0(0xf);
```

PMP 是 Physical Memory Protection。这里建立一个覆盖极大地址范围的可读、可写、可执行区域，使 S-mode 可以访问 QEMU 提供的全部物理内存和设备地址。

如果不进行该配置，进入 S-mode 后访问内存可能立即产生权限异常。

### 11.6 初始化定时器

```c
timerinit();
```

`timerinit()` 完成：

```c
w_mie(r_mie() | MIE_STIE);
w_menvcfg(r_menvcfg() | (1L << 63));
w_mcounteren(r_mcounteren() | 2);
w_stimecmp(r_time() + 1000000);
```

含义为：

- 允许监督态定时器中断；
- 启用 SSTC 扩展的 `stimecmp`；
- 允许 S-mode 读取时间寄存器；
- 设置第一次定时器中断的时间点。

定时器中断以后会驱动 `ticks` 增长，并触发进程让出 CPU。

### 11.7 把 hart ID 保存到 `tp`

```c
int id = r_mhartid();
w_tp(id);
```

xv6 把 `tp` 寄存器当作每 CPU 标识。后面的 `cpuid()` 和 `mycpu()` 依赖它找到当前 hart 对应的 `struct cpu`。

### 11.8 执行 `mret`

```c
asm volatile("mret");
```

根据前面设置的寄存器：

```text
当前特权级：M-mode
        │
        │ mret
        ▼
目标特权级：S-mode
目标 PC：main
```

---

## 12. `main()`：只让 hart 0 初始化全局资源

三个 hart 都会进入 `main()`：

```c
void
main()
{
  if(cpuid() == 0){
    ...
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    ...
  }

  scheduler();
}
```

hart 0 执行全局初始化；hart 1 和 hart 2 自旋等待 `started` 变成 1。

这样避免多个 hart 同时初始化同一个全局链表、页表或设备。

`__sync_synchronize()` 是完整内存屏障，保证 hart 0 对全局数据结构的写入先于 `started=1` 对其他 hart 可见。

---

## 13. hart 0 的初始化顺序

### 13.1 控制台和输出

```c
consoleinit();
printfinit();
printf("\n");
printf("xv6 kernel is booting\n");
printf("\n");
```

`consoleinit()` 会调用 `uartinit()`，配置位于 `0x10000000` 的 UART，并把控制台读写函数注册到设备表。

初始化完成后，内核才能安全使用 `printf()` 输出启动信息。

### 13.2 物理页分配器

```c
kinit();
```

链接脚本提供符号 `end`，表示内核映像末尾。`kinit()` 把：

```text
[PGROUNDUP(end), PHYSTOP)
```

范围内的物理页加入空闲页链表。

此后 `kalloc()` 可以为页表、进程 trapframe、内核栈和用户内存分配页面。

### 13.3 创建并启用内核页表

```c
kvminit();
kvminithart();
```

`kvminit()` 创建共享的内核页表，映射：

- UART MMIO
- VirtIO MMIO
- PLIC MMIO
- 内核代码段，只读且可执行
- 内核数据和剩余 RAM，可读写
- trampoline
- 每个进程的内核栈

内核主要采用直接映射：

```text
虚拟地址 VA = 物理地址 PA
```

例如内核地址 `0x80001000` 仍映射到物理地址 `0x80001000`。因此打开分页后，正在运行的代码地址仍然有效。

`kvminithart()` 执行：

```c
sfence_vma();
w_satp(MAKE_SATP(kernel_pagetable));
sfence_vma();
```

把页表根地址和 Sv39 模式写入 `satp`，并清空旧 TLB 项。

### 13.4 初始化进程表

```c
procinit();
```

它初始化：

- PID 分配锁；
- `wait_lock`；
- 每个 `struct proc` 的锁；
- 每个进程的初始状态；
- 每个进程的内核栈虚拟地址。

### 13.5 初始化陷阱处理

```c
trapinit();
trapinithart();
```

`trapinit()` 初始化 `tickslock`。

`trapinithart()` 执行：

```c
w_stvec((uint64)kernelvec);
```

从此以后，在内核态发生的中断和异常会先进入 `kernel/kernelvec.S`，保存寄存器后调用 `kerneltrap()`。

### 13.6 初始化 PLIC

```c
plicinit();
plicinithart();
```

`plicinit()` 为 UART IRQ 10 和 VirtIO IRQ 1 设置非零优先级。

`plicinithart()` 为当前 hart 的 S-mode PLIC 上下文启用这两个 IRQ，并把优先级阈值设为 0。

### 13.7 初始化文件系统内存结构

```c
binit();
iinit();
fileinit();
```

分别初始化：

- `binit()`：磁盘块缓存链表和锁；
- `iinit()`：内存 inode 表和每个 inode 的睡眠锁；
- `fileinit()`：全局打开文件表。

这里还没有完整读取磁盘超级块；真正的 `fsinit()` 会稍后在第一个进程上下文中执行。

### 13.8 初始化 VirtIO 磁盘

```c
virtio_disk_init();
```

驱动通过地址 `0x10001000` 访问 QEMU VirtIO MMIO 寄存器，并检查：

- Magic value；
- VirtIO 版本必须为 2；
- 设备类型必须为块设备；
- QEMU vendor ID。

随后驱动协商功能，分配 descriptor、available ring 和 used ring，设置队列物理地址，并把设备状态置为 `DRIVER_OK`。

### 13.9 创建第一个进程

```c
userinit();
```

它创建系统中的第一个 `struct proc`，但此时还没有进入用户态。

---

## 14. 其他 hart 如何启动

hart 1 和 hart 2 执行：

```c
while(started == 0)
  ;

__sync_synchronize();
printf("hart %d starting\n", cpuid());
kvminithart();
trapinithart();
plicinithart();
```

它们共享 hart 0 创建的内核页表和全局数据结构，但仍需要分别完成当前 CPU 的硬件寄存器配置：

- 把共享内核页表写入自己的 `satp`；
- 把自己的 `stvec` 设为 `kernelvec`；
- 配置自己的 PLIC S-mode 上下文。

最后所有 hart 都进入：

```c
scheduler();
```

---

## 15. `userinit()` 创建第一个进程

2025 版代码为：

```c
void
userinit(void)
{
  struct proc *p;

  p = allocproc();
  initproc = p;

  p->cwd = namei("/");
  p->state = RUNNABLE;

  release(&p->lock);
}
```

`allocproc()` 完成以下工作：

1. 找到状态为 `UNUSED` 的进程槽。
2. 分配 PID。
3. 把状态改为 `USED`。
4. 分配一页 trapframe。
5. 创建初始用户页表。
6. 映射 `TRAMPOLINE` 和 `TRAPFRAME`。
7. 设置进程第一次运行时的内核上下文。

其中最重要的两行是：

```c
p->context.ra = (uint64)forkret;
p->context.sp = p->kstack + PGSIZE;
```

这表示调度器第一次切换到该进程时：

- 使用该进程自己的内核栈；
- `swtch()` 最终通过 `ret` 进入 `forkret()`。

初始用户页表暂时没有 `/init` 的代码，只有：

```text
TRAMPOLINE：用户态与内核态切换代码
TRAPFRAME ：保存该进程用户寄存器
```

最后把工作目录设置为根目录，并将状态改为 `RUNNABLE`。

---

## 16. `scheduler()` 如何选中第一个进程

每个 hart 都运行自己的调度器：

```c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();

  c->proc = 0;
  for(;;){
    intr_on();
    intr_off();

    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++){
      acquire(&p->lock);
      if(p->state == RUNNABLE){
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context);
        c->proc = 0;
        found = 1;
      }
      release(&p->lock);
    }

    if(found == 0)
      asm volatile("wfi");
  }
}
```

某个 hart 获取第一个进程的锁并发现它为 `RUNNABLE` 后，将其改为 `RUNNING`，然后执行：

```c
swtch(&c->context, &p->context);
```

`swtch.S` 保存调度器的 `ra`、`sp` 和 `s0`～`s11`，再加载进程上下文中的相同寄存器。

由于该进程的上下文被设置为：

```text
ra = forkret
sp = 进程内核栈顶部
```

所以 `swtch()` 末尾执行 `ret` 时，控制流进入 `forkret()`。

如果没有可运行进程，调度器执行 `wfi`，让当前 hart 等待中断，避免无意义地持续占用宿主 CPU。

---

## 17. `forkret()` 为什么在这里初始化文件系统

第一次进入 `forkret()` 时：

```c
if(first){
  fsinit(ROOTDEV);
  first = 0;
  __sync_synchronize();

  p->trapframe->a0 = kexec("/init", (char *[]){ "/init", 0 });
  if(p->trapframe->a0 == -1)
    panic("exec");
}
```

### 17.1 `fsinit(ROOTDEV)`

文件系统初始化需要从磁盘读取超级块。磁盘 I/O 可能等待 VirtIO 中断并调用 `sleep()`，所以必须存在一个正常的进程上下文和调度器。

这就是为什么 `fsinit()` 没有直接放进 `main()`：`main()` 当时还不是普通进程上下文，若文件系统初始化需要睡眠，将无法按正常进程调度机制工作。

### 17.2 `first` 的作用

所有新进程第一次被调度时都可能从 `forkret()` 开始，但全局文件系统只能初始化一次。

```c
static int first = 1;
```

确保只有第一个进入 `forkret()` 的进程运行 `fsinit()` 和初始 `/init` 加载逻辑。

---

## 18. `kexec("/init")` 如何建立用户程序

`kexec()` 执行以下过程。

### 18.1 查找 `/init`

```c
begin_op();
ip = namei("/init");
ilock(ip);
```

它通过 xv6 文件系统从 `fs.img` 找到 `/init` 对应 inode。

### 18.2 验证 ELF

```c
readi(ip, 0, (uint64)&elf, 0, sizeof(elf));
if(elf.magic != ELF_MAGIC)
  goto bad;
```

确保文件是有效 ELF 可执行文件。

### 18.3 创建新的用户页表

```c
pagetable = proc_pagetable(p);
```

新页表最初包含 trampoline 和 trapframe 映射。

### 18.4 加载 ELF 段

`kexec()` 遍历程序头，对每个 `ELF_PROG_LOAD` 段：

1. 根据虚拟地址和大小分配用户页。
2. 根据 ELF flags 设置读、写、执行权限。
3. 调用 `loadseg()` 从 inode 读取程序内容。

结果大致为：

```text
低地址
┌─────────────────────┐
│ /init 代码段 R-X     │
├─────────────────────┤
│ 只读数据段 R--       │
├─────────────────────┤
│ 数据和 BSS 段 RW-    │
├─────────────────────┤
│ guard page           │
├─────────────────────┤
│ 用户栈               │
├─────────────────────┤
│ 未映射空间           │
├─────────────────────┤
│ TRAPFRAME            │
├─────────────────────┤
│ TRAMPOLINE           │
└─────────────────────┘
高地址
```

### 18.5 创建用户栈

内核为用户栈分配页面，并把第一页面设为不可访问的 guard page。随后把参数字符串和 `argv` 指针数组复制到用户栈。

RISC-V 调用约定要求栈指针按 16 字节对齐，所以代码反复执行：

```c
sp -= sp % 16;
```

### 18.6 设置初始用户寄存器

```c
p->trapframe->a1 = sp;
p->trapframe->epc = elf.entry;
p->trapframe->sp = sp;
```

并将 `kexec()` 返回的 `argc` 写入：

```c
p->trapframe->a0
```

因此 `/init` 开始运行时符合普通 C 程序入口约定：

```text
a0 = argc
a1 = argv
sp = 用户栈顶部
pc = ELF entry
```

最后释放旧用户页表，并提交新地址空间。

---

## 19. 从 S-mode 返回 U-mode

`forkret()` 在加载 `/init` 后调用：

```c
prepare_return();
```

### 19.1 配置用户陷阱入口

```c
uint64 trampoline_uservec = TRAMPOLINE + (uservec - trampoline);
w_stvec(trampoline_uservec);
```

以后用户态发生系统调用、中断或异常时，CPU 会先跳到 trampoline 页中的 `uservec`。

### 19.2 准备返回用户态时所需的信息

内核在 trapframe 中保存：

```c
p->trapframe->kernel_satp = r_satp();
p->trapframe->kernel_sp = p->kstack + PGSIZE;
p->trapframe->kernel_trap = (uint64)usertrap;
p->trapframe->kernel_hartid = r_tp();
```

这些值会在下一次从用户态陷入内核时被 `uservec` 使用。

### 19.3 配置 `sret`

```c
x &= ~SSTATUS_SPP;
x |= SSTATUS_SPIE;
w_sstatus(x);
w_sepc(p->trapframe->epc);
```

- `SPP=0`：`sret` 后进入 U-mode。
- `SPIE=1`：回到用户态后允许中断。
- `sepc=elf.entry`：返回地址是 `/init` 的入口。

### 19.4 进入 `userret`

```c
uint64 satp = MAKE_SATP(p->pagetable);
uint64 trampoline_userret = TRAMPOLINE + (userret - trampoline);
((void (*)(uint64))trampoline_userret)(satp);
```

`userret` 位于 `trampoline.S`，它：

1. 执行 `sfence.vma`。
2. 把用户页表写入 `satp`。
3. 再次刷新 TLB。
4. 从 trapframe 恢复全部用户寄存器。
5. 执行 `sret`。

执行 `sret` 后：

```text
S-mode 内核
    │
    │ sret
    ▼
U-mode /init
```

这是真正第一次开始运行用户代码。

---

## 20. `/init` 如何启动 Shell

`user/init.c` 的第一项任务是准备控制台：

```c
if(open("console", O_RDWR) < 0){
  mknod("console", CONSOLE, 0);
  open("console", O_RDWR);
}
dup(0);  // stdout
dup(0);  // stderr
```

最终得到：

```text
文件描述符 0 → console，标准输入
文件描述符 1 → console，标准输出
文件描述符 2 → console，标准错误
```

随后进入无限循环：

```c
for(;;){
  printf("init: starting sh\n");
  pid = fork();

  if(pid == 0){
    exec("sh", argv);
    ...
  }

  ... wait ...
}
```

父进程 `/init` 保持运行，子进程通过：

```c
exec("sh", argv);
```

把地址空间替换成 `/sh`。

如果 Shell 退出，`init` 的 `wait()` 会返回，外层循环再次创建 Shell。因此 `/init` 同时承担：

- 启动 Shell；
- 回收孤儿进程和退出进程；
- Shell 退出后重新启动 Shell。

---

## 21. 正常启动输出分别来自哪里

通常可以看到：

```text
xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$
```

来源如下：

| 输出 | 产生位置 | 含义 |
|---|---|---|
| `xv6 kernel is booting` | `kernel/main.c` | hart 0 已初始化 UART 和 printf |
| `hart 1 starting` | `kernel/main.c` | hart 1 结束等待并完成本地初始化 |
| `hart 2 starting` | `kernel/main.c` | hart 2 结束等待并完成本地初始化 |
| `init: starting sh` | `user/init.c` | 已进入用户态，`/init` 正在创建 Shell |
| `$` | `user/sh.c` | Shell 已经运行并等待输入 |

如果只看到第一行而看不到 `init: starting sh`，问题通常发生在页表、磁盘、文件系统、调度器或 `/init` 加载阶段。

---

## 22. 定时器中断与调度

`start()` 已经设置第一次定时器中断。内核处理定时器中断后调用：

```c
clockintr();
```

hart 0 更新全局 tick：

```c
ticks++;
wakeup(&ticks);
```

然后设置下一次中断：

```c
w_stimecmp(r_time() + 1000000);
```

如果定时器中断发生时存在当前进程，`kerneltrap()` 或 `usertrap()` 最终会调用：

```c
yield();
```

`yield()` 把进程状态改回 `RUNNABLE`，再通过 `sched()` 和 `swtch()` 返回调度器。这构成 xv6 的抢占式时间片调度基础。

---

## 23. 内核态和用户态陷阱入口为什么不同

当内核正在执行时：

```text
stvec = kernelvec
```

发生中断后，CPU 使用当前内核页表和内核栈进入 `kernelvec`。

当用户程序运行时：

```text
stvec = TRAMPOLINE + uservec 偏移
```

用户态发生系统调用、中断或异常后，CPU 仍暂时使用用户页表，所以陷阱入口必须位于同时映射在用户页表与内核页表中的 trampoline 页。

`uservec` 保存用户寄存器，切换到内核页表和进程内核栈，然后调用 `usertrap()`。

返回用户态时，`userret` 做相反工作：切换到用户页表，恢复用户寄存器并执行 `sret`。

---

## 24. `make qemu-gdb` 与普通启动的区别

调试目标为：

```make
qemu-gdb: $K/kernel .gdbinit fs.img
	@echo "*** Now run 'gdb' in another window."
	$(QEMU) $(QEMUOPTS) -S $(QEMUGDB)
```

其中：

- `-S`：QEMU 创建 CPU 后立即暂停，不执行第一条指令；
- `-gdb tcp::端口`：启动 QEMU GDB server；
- `.gdbinit`：配置 RISC-V 架构、内核符号文件和远程端口。

适合观察启动过程的断点包括：

```gdb
b _entry
b start
b main
b userinit
b scheduler
b forkret
b kexec
b prepare_return
```

推荐按以下顺序单步：

```text
_entry
  → start
  → main
  → userinit
  → scheduler
  → swtch
  → forkret
  → kexec
  → prepare_return
  → userret
```

`userret` 执行 `sret` 后已经进入用户态。如果需要调试用户程序，还需要加载对应用户 ELF 的符号，并处理用户虚拟地址与进程页表。

---

## 25. 常见问题定位

### 25.1 找不到 RISC-V 工具链

典型信息：

```text
Error: Couldn't find a riscv64 version of GCC/binutils.
```

说明 Makefile 没找到交叉编译器。可检查：

```bash
riscv64-linux-gnu-gcc --version
riscv64-linux-gnu-objdump -i
```

### 25.2 QEMU 版本过低

典型信息：

```text
ERROR: Need qemu version >= 7.2
```

检查：

```bash
qemu-system-riscv64 --version
```

### 25.3 没有出现任何 xv6 输出

优先检查：

- QEMU 是否成功加载 `kernel/kernel`；
- `_entry` 是否位于 `0x80000000`；
- `entry.S` 是否建立了正确栈；
- `start()` 是否成功执行 `mret`；
- `consoleinit()` 是否正常访问 UART。

可以使用 `make qemu-gdb` 在 `_entry` 和 `main` 设置断点。

### 25.4 出现启动信息，但找不到 `/init`

如果在 `forkret()` 中 `panic("exec")`，通常检查：

- `fs.img` 是否成功生成；
- `user/_init` 是否属于 `UPROGS`；
- `mkfs/mkfs` 是否把 `_init` 写成 `/init`；
- VirtIO 磁盘驱动是否成功初始化；
- 文件系统超级块是否可读。

### 25.5 想保留 xv6 中创建的文件

使用：

```bash
make qemu-fs
```

而不是：

```bash
make qemu
```

因为 `make qemu` 会先把旧 `fs.img` 移动为 `fs.img.bk` 并创建新镜像。

### 25.6 如何退出 `-nographic` QEMU

通常使用 QEMU monitor 转义序列：

```text
Ctrl-a x
```

先按 `Ctrl-a`，松开后再按 `x`。

---

## 26. 关键文件索引

| 文件 | 启动过程中的职责 |
|---|---|
| `Makefile` | 构建内核、用户程序、文件系统并启动 QEMU |
| `kernel/kernel.ld` | 指定入口 `_entry` 和内核地址布局 |
| `kernel/entry.S` | 建立每 hart 启动栈并调用 `start()` |
| `kernel/start.c` | M-mode 初始化并切换到 S-mode |
| `kernel/main.c` | 初始化全部内核子系统 |
| `kernel/memlayout.h` | 定义 RAM 和设备 MMIO 地址 |
| `kernel/vm.c` | 创建并启用内核页表 |
| `kernel/proc.c` | 创建首进程、调度和 `forkret()` |
| `kernel/swtch.S` | 保存和恢复内核上下文 |
| `kernel/exec.c` | 从文件系统加载 `/init` ELF |
| `kernel/trap.c` | 设置陷阱入口和返回用户态状态 |
| `kernel/trampoline.S` | 在用户页表和内核页表间切换 |
| `kernel/virtio_disk.c` | 驱动 QEMU VirtIO 磁盘 |
| `user/init.c` | 创建控制台、启动并维护 Shell |
| `user/sh.c` | xv6 命令行 Shell |

---

## 27. 最终总结

`make qemu` 实际包含两条相互衔接的主线。

第一条是宿主机上的构建线：

```text
C/汇编源码
   → RISC-V 对象文件
   → kernel/kernel
   → 用户程序
   → fs.img
   → qemu-system-riscv64
```

第二条是虚拟 RISC-V 机器中的执行线：

```text
QEMU 复位
   → _entry（M-mode）
   → start（M-mode）
   → mret
   → main（S-mode）
   → 内存、页表、中断、磁盘和进程初始化
   → scheduler
   → forkret
   → kexec("/init")
   → sret
   → /init（U-mode）
   → exec("sh")
   → Shell 提示符
```

理解这两条线的连接点非常关键：

- `kernel/kernel` 通过 QEMU 的 `-kernel` 参数成为 CPU 执行的内核。
- `fs.img` 通过 VirtIO 设备成为 xv6 看到的根磁盘。
- `kernel.ld` 把 `_entry` 放到 QEMU 预期的 `0x80000000`。
- `start()` 完成 M-mode 到 S-mode 的切换。
- `main()` 建立内核运行环境。
- `scheduler()` 让第一个进程获得 CPU。
- `kexec("/init")` 把磁盘中的用户程序装入地址空间。
- trampoline 和 `sret` 完成 S-mode 到 U-mode 的切换。
- `/init` 最终创建 Shell，完成整个系统启动。
