# xv6-labs-2025 Makefile 中文解析

> 分析对象：`/home/wangxin/xv6-labs-2025/Makefile`  
> 当前实验配置：`conf/lab.mk` 中为 `LAB=util`  
> 当前 Git 分支：`util`  
> Makefile 共 398 行

## 1. 这个 Makefile 的核心作用

这个 Makefile 负责把 xv6 的三个主要部分组织成一个可运行的 RISC-V 操作系统：

1. 使用 RISC-V 交叉编译器编译并链接 xv6 内核，生成 `kernel/kernel`。
2. 编译、链接用户程序，生成 `user/_cat`、`user/_sh` 等可执行文件。
3. 使用宿主机程序 `mkfs/mkfs`，把 README、测试文件和用户程序装进 `fs.img`。
4. 使用 `qemu-system-riscv64` 启动内核并挂载 `fs.img`。
5. 为实验提供条件编译、自动评分、GDB 调试和提交打包功能。

整体构建关系可以概括为：

```text
kernel/*.c、kernel/*.S
          │
          ▼
   kernel/*.o ──────────────┐
                            ▼
kernel/kernel.ld ───► kernel/kernel
                            │
                            ├──────────────────────┐
user/*.c                    │                      │
   │                        │                      │
   ▼                        │                      ▼
user/*.o + ULIB             │               QEMU 启动 xv6
   │                        │                      ▲
   ▼                        │                      │
user/_程序 ──► mkfs ──► fs.img ──────────────────┘
```

## 2. 实验选择机制（第 2～6、93～96 行）

```make
-include conf/lab.mk
```

前面的 `-` 表示：即使 `conf/lab.mk` 不存在，make 也不会报错退出。当前该文件内容为：

```make
LAB=util
```

因此本次构建处于 `util` 实验模式。只要定义了 `LAB`，Makefile 就会把实验名转换成大写，并加入两个 C 宏：

```text
-DSOL_UTIL -DLAB_UTIL
```

源代码可以通过 `#ifdef LAB_UTIL` 或 `#ifdef SOL_UTIL` 启用实验专用逻辑。Makefile 中的 `ifeq ($(LAB),...)` 还会按实验增减内核模块、用户程序、CPU 数量及 QEMU 网络参数。

## 3. 基础目录和内核对象（第 8～59 行）

```make
K=kernel
U=user
```

后面用 `$K`、`$U` 代表 `kernel`、`user`。在 GNU make 中，单字符变量可以直接写成 `$K`；更常见也更清晰的写法是 `$(K)`。

`OBJS` 是内核主体对象文件列表，大致分为：

- 启动和底层汇编：`entry.o`、`swtch.o`、`trampoline.o`、`kernelvec.o`
- 内存与进程：`kalloc.o`、`vm.o`、`proc.o`
- 中断和系统调用：`trap.o`、`syscall.o`、`sysproc.o`、`sysfile.o`
- 文件系统：`bio.o`、`fs.o`、`log.o`、`file.o`、`pipe.o`、`exec.o`
- 硬件驱动：`plic.o`、`virtio_disk.o`

`OBJS_KCSAN` 虽然名字中有 KCSAN，但其中的启动、控制台、输出、串口和自旋锁模块在普通构建中也始终参与链接。定义 `KCSAN` 后才额外加入 `kernel/kcsan.o`，并对 `OBJS` 中的对象启用线程消毒器相关参数。

实验专用内核对象：

| `LAB` 值 | 额外内核对象 |
|---|---|
| `lock` | `stats.o`、`sprintf.o` |
| `net` | `e1000.o`、`net.o`、`pci.o` |

## 4. RISC-V 工具链自动探测（第 62～89 行）

Makefile 依次尝试四种常见工具链前缀：

```text
riscv64-unknown-elf-
riscv64-elf-
riscv64-linux-gnu-
riscv64-unknown-linux-gnu-
```

它通过运行相应的 `objdump -i` 并查找 RISC-V ELF 支持来选择 `TOOLPREFIX`。随后得到：

```make
CC      = $(TOOLPREFIX)gcc
AS      = $(TOOLPREFIX)gas
LD      = $(TOOLPREFIX)ld
OBJCOPY = $(TOOLPREFIX)objcopy
OBJDUMP = $(TOOLPREFIX)objdump
```

如果自动探测失败，可以在命令行手工指定，例如：

```bash
make TOOLPREFIX=riscv64-linux-gnu-
```

注意：内核和 xv6 用户程序使用交叉工具链；`mkfs/mkfs`、`ph`、`barrier` 是在 Ubuntu 宿主机上运行的程序，所以使用本机 `gcc`。

## 5. 编译参数（第 91～129 行）

主要参数及含义：

| 参数 | 含义 |
|---|---|
| `-Wall -Werror` | 开启常见警告，并把警告当作错误 |
| `-O` | 开启优化 |
| `-fno-omit-frame-pointer` | 保留栈帧指针，便于回溯和调试 |
| `-ggdb -gdwarf-2` | 生成 GDB 可用的 DWARF 2 调试信息 |
| `-MD` | 编译时生成 `.d` 头文件依赖文件 |
| `-mcmodel=medany` | 使用适合 RISC-V 内核地址布局的代码模型 |
| `-ffreestanding` | 声明这是无标准宿主环境的独立程序 |
| `-nostdlib` | 不链接宿主机标准库 |
| `-fno-common` | 不把未初始化全局变量当作 common 符号 |
| `-fno-builtin-*` | 禁止 GCC 把 xv6 自己实现的函数替换成内建函数 |
| `-I.` | 将项目根目录加入头文件搜索路径 |
| `-fno-stack-protector` | 工具链支持时关闭栈保护，避免依赖宿主运行库 |
| `-fno-pie/-no-pie` | 工具链支持时关闭位置无关可执行文件 |

链接参数：

```make
LDFLAGS = -z max-page-size=4096
```

它把最大页面大小限制为 4096 字节，使生成文件与 xv6 的页大小设计一致。

## 6. 内核的编译与链接（第 131～144 行）

最终内核目标是：

```make
kernel/kernel: 内核对象文件 kernel/kernel.ld
```

构建过程：

1. 用 `kernel/kernel.ld` 作为链接脚本，把全部对象链接为 `kernel/kernel`。
2. 用 `objdump -S` 生成混合了源代码和汇编的 `kernel/kernel.asm`。
3. 用 `objdump -t` 提取符号，生成 `kernel/kernel.sym`。

两条模式规则分别处理 C 和汇编：

```make
kernel/%.o: kernel/%.c
kernel/%.o: kernel/%.S
```

`$@` 表示目标文件，`$<` 表示第一个依赖文件。例如构建 `kernel/proc.o` 时：

```text
$@ = kernel/proc.o
$< = kernel/proc.c
```

`tags` 目标用 Emacs 的 `etags` 为内核源码生成跳转索引。

## 7. 用户程序的编译与链接（第 146～176 行）

所有普通用户程序共享一个很小的用户态库：

```make
ULIB = user/ulib.o user/usys.o user/printf.o user/umalloc.o
```

其中：

- `ulib.o`：字符串、内存等基础函数
- `usys.o`：系统调用入口
- `printf.o`：用户态格式化输出
- `umalloc.o`：用户态内存分配

`user/usys.S` 不是手写文件，而是由 Perl 脚本 `user/usys.pl` 自动生成，再编译成 `usys.o`。

模式规则：

```make
_%: %.o $(ULIB) user/user.ld
```

会把 `user/cat.o` 链接成 `user/_cat`。下划线用于区分 xv6 内部的可执行文件与宿主环境程序名。链接后同样生成 `.asm` 和 `.sym` 辅助文件。

`forktest` 使用单独规则，只链接最少的库代码，以缩小程序体积，从而能够创建足够多的进程来测试进程表上限。

`.PRECIOUS: %.o` 阻止 make 把链式隐式规则产生的 `.o` 中间文件自动删除。

## 8. 用户程序列表和各实验条件（第 178～279 行）

`UPROGS` 决定哪些用户程序会被编译并写入 `fs.img`。基础列表包括 `cat`、`echo`、`init`、`sh`、`usertests` 等。

各实验额外加入：

| 实验 | 额外构建内容 |
|---|---|
| `syscall` | `attack`、`secret` |
| `lock` | `stats`、`kalloctest`、`bcachetest`，并扩充用户库 |
| `traps` | `call`、`bttest` |
| `lazy` | `lazytests` |
| `cow` | `cowtest` |
| `thread` | `uthread`，以及宿主机程序 `ph`、`barrier` |
| `pgtbl` | `pgtbltest` |
| `fs` | `bigfile` |
| `mmap` | `mmaptest` |
| `net` | `nettest` |
| `util` | `memdump`，以及额外文件 `findtest.sh`、`sixfive.txt` |

### 当前 util 实验的重要提醒

当前 `UPROGS` 中，util 实验条件块只自动加入了 `user/_memdump`。如果你正在完成 MIT xv6 的 util 实验，自己实现的 `sleep`、`pingpong`、`primes`、`find`、`xargs` 等程序通常还需要手工加入 `UPROGS`，否则即使源文件存在，它们也不会被装入 `fs.img`。

示意写法：

```make
UPROGS += \
	$U/_sleep\
	$U/_pingpong\
	$U/_primes\
	$U/_find\
	$U/_xargs
```

应根据实际实验要求和已有源文件添加，不要仅为了消除错误而添加尚不存在的程序。

## 9. 文件系统镜像（第 282～288 行）

`mkfs/mkfs` 是宿主机工具，由本机 GCC 编译。`fs.img` 的依赖为：

```text
mkfs/mkfs + README + UEXTRA + UPROGS
```

构建命令把这些文件写进新的 xv6 文件系统镜像：

```make
mkfs/mkfs fs.img README $(UEXTRA) $(UPROGS)
```

`newfs.img` 目标会尝试把已有 `fs.img` 移为 `fs.img.bk`。命令前的 `-` 表示移动失败也继续执行。这个目标的配方不会生成名为 `newfs.img` 的实体文件，因此 `make qemu` 时它通常每次都会执行，促使后续重建全新的 `fs.img`。

相对地，`make qemu-fs` 不依赖 `newfs.img`，因此会尽量复用现有镜像；这适合保留上一次 xv6 会话中对文件系统的修改。

第 288 行通过：

```make
-include kernel/*.d user/*.d
```

读取 GCC `-MD` 生成的依赖文件。修改头文件后，相关 `.o` 会自动重新编译。

## 10. clean 目标（第 290～296 行）

`make clean` 会删除：

- 内核、用户程序和文件系统镜像
- `.o`、`.d`、`.asm`、`.sym` 等编译产物
- 自动生成的 `user/usys.S`、`.gdbinit`
- `mkfs/mkfs` 和其他临时文件

它使用 `rm -rf`，不要把 `K`、`U` 等变量改成不受控制的绝对路径。`fs.img.bk` 不在清理列表中，所以备份镜像会保留。

## 11. QEMU 运行配置（第 298～337 行）

默认设置：

```text
QEMU        = qemu-system-riscv64
最低版本    = 7.2
内存        = 128 MiB
CPU         = 3 个（fs 实验强制为 1 个）
机器        = RISC-V virt
固件        = none
控制台      = nographic
磁盘        = fs.img，通过 virtio-blk-device 挂载
```

GDB 端口根据用户 UID 计算：

```text
GDBPORT = UID % 5000 + 25000
```

这样可降低多用户机器上的端口冲突概率。

三个主要启动目标的区别：

| 命令 | 行为 |
|---|---|
| `make qemu` | 备份旧镜像、重建新 `fs.img`，再启动 QEMU |
| `make qemu-fs` | 尽量使用现有 `fs.img`，保留镜像中的修改 |
| `make qemu-gdb` | QEMU 启动后暂停 CPU，开放 GDB stub 等待调试器 |

`make qemu-gdb` 会由 `.gdbinit.tmpl-riscv` 生成 `.gdbinit`，并把模板中的默认端口替换成实际 `GDBPORT`。另开一个终端，在项目根目录启动适配 RISC-V 的 GDB 即可连接。

`net` 实验会额外：

- 加入 E1000 网卡
- 把宿主 UDP 端口转发到客户机 2000/2001 端口
- 把网络数据包记录到 `packets.pcap`

## 12. 自动评分（第 345～360 行）

```bash
make grade
```

执行顺序是：

1. 调用 `make clean`。
2. 如果清理失败，提示检查是否仍有 xv6 实例在运行。
3. 执行与当前实验匹配的脚本 `./grade-lab-$(LAB)`。

当前 `LAB=util`，所以实际运行 `./grade-lab-util`。默认加入 `-v` 显示详细评分信息；用 `make V=@ grade` 可不加入这个详细参数。

## 13. 提交检查与打包（第 362～391 行）

`submit-check` 会检查：

1. 当前目录是否是 Git 仓库。
2. 当前分支是否与 `LAB` 同名。
3. 是否存在未提交的已跟踪文件修改。
4. 是否存在不会被打包的未跟踪文件。

当前分支为 `util`，与 `LAB=util` 一致。不过当前仓库存在未跟踪文件 `fs.img.bk`；执行提交检查时会提示确认，而 `git archive` 不会把该文件装入提交包。

```bash
make zipball
```

会先清理、执行提交检查，再用当前 `HEAD` 生成 `lab.zip`。因此只有已经提交到 Git 的内容会进入压缩包。

## 14. QEMU 版本检查（第 393～398 行）

Makefile 从 `qemu-system-riscv64 --version` 的第一行提取主、次版本号，再通过 `bc` 判断是否大于等于 7.2。

这意味着系统除了 QEMU 外还需要安装 `bc`。如果 `bc` 缺失，版本检查可能报错，即使 QEMU 本身已经安装。

## 15. 默认目标与常用命令

由于 `kernel/kernel` 是 Makefile 中出现的第一个普通目标，直接运行：

```bash
make
```

默认只保证构建内核，不等同于启动 xv6。常用命令如下：

```bash
# 进入项目
cd /home/wangxin/xv6-labs-2025

# 编译默认目标（内核）
make

# 构建全新镜像并运行
make qemu

# 复用现有文件系统镜像运行
make qemu-fs

# GDB 调试模式
make qemu-gdb

# 查看实际 GDB 端口
make print-gdbport

# 运行当前 util 实验评分
make grade

# 删除构建产物
make clean

# 检查提交状态并生成 lab.zip
make zipball
```

退出无图形界面的 QEMU，通常按：`Ctrl-a`，松开后再按 `x`。

## 16. 阅读这个 Makefile 时最值得掌握的 GNU make 知识

- `目标: 依赖`：依赖比目标新，或目标不存在时，执行配方。
- `-include 文件`：包含文件，但缺失时不终止。
- `ifeq`、`ifdef`：根据实验或功能开关进行条件构建。
- `%`：模式规则中的通配 stem。
- `$@`：当前目标。
- `$<`：第一个依赖。
- `$^`：全部依赖。
- `$*`：模式规则匹配到的 stem。
- `:=`：立即展开变量；`=` 通常在使用时再展开。
- `+=`：追加变量内容。
- 命令前的 `@`：执行时不回显命令本身。
- 命令前的 `-`：命令失败时仍继续。
- `.PHONY`：声明伪目标，避免被同名文件干扰。

## 17. 总结

这个 Makefile 的设计重点是把“交叉编译内核和用户程序—制作磁盘镜像—用 QEMU 运行—用实验脚本评分”连成一条可重复的流水线。当前项目处于 util 实验，最需要注意的是：

1. `conf/lab.mk` 和 Git 分支都已正确设置为 `util`。
2. 新增的用户程序必须加入 `UPROGS` 才会进入 `fs.img`。
3. `make qemu` 会换掉旧镜像，`make qemu-fs` 才会复用它。
4. `make grade` 会先执行 `make clean`，再调用 `grade-lab-util`。
5. `make zipball` 只打包已提交到 Git 的内容。

