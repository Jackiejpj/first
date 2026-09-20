# xv6 Makefile 内核链接规则详细解析

## 1. 原始代码

```make
$K/kernel: $(OBJS) $(OBJS_KCSAN) $K/kernel.ld
	$(LD) $(LDFLAGS) -T $K/kernel.ld -o $K/kernel $(OBJS) $(OBJS_KCSAN)
	$(OBJDUMP) -S $K/kernel > $K/kernel.asm
	$(OBJDUMP) -t $K/kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $K/kernel.sym
```

这段规则负责完成三件事：

1. 把已经编译好的内核对象文件链接为最终内核 `kernel/kernel`。
2. 生成便于查看源码与 RISC-V 指令对应关系的 `kernel/kernel.asm`。
3. 提取内核符号及地址，生成精简的 `kernel/kernel.sym`。

整体流程如下：

```text
kernel/*.o + kernel/kernel.ld
              │
              ▼
        RISC-V 链接器
              │
              ▼
        kernel/kernel
          │         │
          │         └── objdump -t ── sed ──► kernel/kernel.sym
          │
          └──────────── objdump -S ─────────► kernel/kernel.asm
```

## 2. 第一行：声明目标与依赖

```make
$K/kernel: $(OBJS) $(OBJS_KCSAN) $K/kernel.ld
```

GNU make 规则的基本形式是：

```make
目标: 依赖项
	生成目标所需的命令
```

因此，这一行的含义是：为了得到目标文件 `kernel/kernel`，make 必须先准备好全部内核对象文件以及链接脚本。

### 2.1 `$K/kernel` 是什么

Makefile 前面定义了：

```make
K=kernel
```

`K` 是单字符变量，因此 `$K` 是 `$(K)` 的简写。于是：

```make
$K/kernel
```

展开为：

```text
kernel/kernel
```

为了避免歧义，现代 Makefile 通常更倾向写成：

```make
$(K)/kernel
```

两种写法在这里效果相同。

`kernel/kernel` 是最终的 RISC-V ELF 内核文件，之后会通过 QEMU 的以下参数加载：

```text
-kernel kernel/kernel
```

### 2.2 `$(OBJS)` 是什么

`OBJS` 保存 xv6 内核主体的对象文件，例如：

```text
kernel/entry.o
kernel/kalloc.o
kernel/main.o
kernel/vm.o
kernel/proc.o
kernel/trap.o
kernel/syscall.o
kernel/fs.o
kernel/virtio_disk.o
...
```

这些 `.o` 文件分别由对应的 `.c` 或 `.S` 文件编译得到。例如：

```text
kernel/proc.c  ──编译──► kernel/proc.o
kernel/swtch.S ──汇编──► kernel/swtch.o
```

对象文件中已经包含机器代码，但其中的函数和变量通常还没有获得最终运行地址，跨文件的符号引用也尚未全部解析。因此还需要链接器把它们组合成一个完整内核。

### 2.3 `$(OBJS_KCSAN)` 是什么

该变量的基础内容包括：

```text
kernel/start.o
kernel/console.o
kernel/printf.o
kernel/uart.o
kernel/spinlock.o
```

虽然变量名带有 `KCSAN`，这些对象在普通构建中也始终会被链接。只有在定义 `KCSAN` 时，Makefile 才会额外加入：

```text
kernel/kcsan.o
```

KCSAN 是用于发现内核并发数据竞争问题的检测机制。

### 2.4 `$K/kernel.ld` 是什么

它展开为：

```text
kernel/kernel.ld
```

这是内核链接脚本，负责规定：

- 内核入口点
- 内核装载和运行的地址
- `.text` 代码段的位置
- `.rodata` 只读数据段的位置
- `.data` 已初始化数据段的位置
- `.bss` 未初始化数据段的位置
- 各段的边界和对齐方式
- 内核需要使用的特殊链接器符号

可以把各个 `.o` 文件理解为还未确定最终位置的“零件”，而 `kernel.ld` 就是安排这些零件在内核地址空间中如何摆放的“布局图”。

### 2.5 make 什么时候执行这条规则

出现以下任一情况时，make 会执行后面的三条命令：

- `kernel/kernel` 不存在。
- `$(OBJS)` 中任意对象文件比 `kernel/kernel` 更新。
- `$(OBJS_KCSAN)` 中任意对象文件比 `kernel/kernel` 更新。
- `kernel/kernel.ld` 比 `kernel/kernel` 更新。

因此，修改一个内核 `.c` 文件后，make 会先重编译对应 `.o`，再重新链接整个内核。修改链接脚本后，即使所有 `.o` 都没有变化，也会重新链接。

## 3. 第二行：链接最终内核

```make
	$(LD) $(LDFLAGS) -T $K/kernel.ld -o $K/kernel $(OBJS) $(OBJS_KCSAN)
```

行首必须是制表符 Tab，而不是普通空格。传统 GNU make 使用 Tab 判断这一行是构建命令。

变量展开后，命令大致类似：

```bash
riscv64-linux-gnu-ld \
  -z max-page-size=4096 \
  -T kernel/kernel.ld \
  -o kernel/kernel \
  kernel/entry.o kernel/kalloc.o kernel/main.o ... \
  kernel/start.o kernel/console.o kernel/printf.o kernel/uart.o kernel/spinlock.o
```

### 3.1 `$(LD)`：RISC-V 链接器

Makefile 中定义：

```make
LD = $(TOOLPREFIX)ld
```

假设自动检测到的工具链前缀为：

```text
riscv64-linux-gnu-
```

那么 `$(LD)` 就是：

```text
riscv64-linux-gnu-ld
```

这是交叉链接器。它运行在当前 Ubuntu x86-64 系统中，但输出的是能够在 RISC-V CPU 上运行的程序。

这里直接调用 `ld`，而不是让 `gcc` 间接执行链接，因为 xv6 是独立运行的操作系统内核，不应该自动链接 Ubuntu 的启动代码、C 标准库或其他宿主环境组件。

### 3.2 `$(LDFLAGS)`：链接选项

Makefile 中定义：

```make
LDFLAGS = -z max-page-size=4096
```

因此实际传给链接器的参数是：

```text
-z max-page-size=4096
```

它把 ELF 程序段允许使用的最大页面大小设置为 4096 字节，也就是 4 KiB，与 xv6 的页大小保持一致，避免链接器按更大的默认页尺寸安排程序段。

### 3.3 `-T kernel/kernel.ld`

`-T` 用于指定自定义链接脚本：

```text
-T kernel/kernel.ld
```

如果没有指定该脚本，链接器会采用工具链自带的默认用户程序布局，这不符合内核的地址空间设计。

### 3.4 `-o kernel/kernel`

`-o` 指定输出文件：

```text
-o kernel/kernel
```

这个文件不是磁盘镜像，也不是源码文件，而是带有代码、数据、符号和调试信息的 RISC-V ELF 可执行文件。

### 3.5 对象文件链接时发生了什么

链接器主要执行以下工作：

1. 合并各个 `.o` 文件中的代码段和数据段。
2. 根据 `kernel.ld` 安排各段的最终地址。
3. 解析跨文件的函数和全局变量引用。
4. 应用重定位信息，把尚未确定的地址填入机器指令或数据。
5. 确定内核入口点。
6. 输出完整的 `kernel/kernel` ELF 文件。

例如，`main.o` 中可能调用定义在 `proc.o` 中的 `procinit()`。编译 `main.c` 时，编译器只记录这是一个尚待解析的外部符号；链接时，链接器找到 `procinit` 的定义和最终地址，再把调用指令修正到正确位置。

因此，这一行执行的是：

```text
多个尚未完全确定地址的对象文件
                  │
                  ▼
        解析符号、分配地址、重定位
                  │
                  ▼
          可由 QEMU 加载的内核
```

## 4. 第三行：生成源码混合反汇编

```make
	$(OBJDUMP) -S $K/kernel > $K/kernel.asm
```

展开后大致为：

```bash
riscv64-linux-gnu-objdump -S kernel/kernel > kernel/kernel.asm
```

### 4.1 `$(OBJDUMP)`

Makefile 中定义：

```make
OBJDUMP = $(TOOLPREFIX)objdump
```

它必须与生成内核时所用的 RISC-V 工具链匹配。例如：

```text
riscv64-linux-gnu-objdump
```

`objdump` 可以检查 ELF 文件中的程序段、符号表、机器指令和其他结构。

### 4.2 `-S` 选项

大写 `-S` 要求 `objdump` 在反汇编机器指令时，尽可能同时显示对应的源代码。

Makefile 的编译选项包含：

```text
-ggdb -gdwarf-2
```

所以 `kernel/kernel` 中带有 DWARF 调试信息，`objdump` 能够把 C 源码与生成的 RISC-V 指令对应起来。

输出可能类似：

```asm
void
main()
{
    80000000: 1141        addi sp,sp,-16
    80000002: e406        sd   ra,8(sp)

  consoleinit();
    80000004: 00001097    auipc ra,0x1
    80000008: ...         jalr  ...
}
```

它特别适合分析：

- C 语句如何变成 RISC-V 指令
- 函数入口如何建立栈帧
- 参数和返回值如何传递
- 函数调用最终跳转到什么地址
- 上下文切换、中断和系统调用的汇编过程
- 某个内核异常地址属于哪个函数

### 4.3 输出重定向

```bash
> kernel/kernel.asm
```

`>` 是 Shell 的输出重定向符号，把 `objdump` 原本显示在终端的内容写入 `kernel/kernel.asm`。如果该文件已经存在，其旧内容会被覆盖。

`kernel.asm` 只是分析和调试辅助文件，不是 QEMU 启动 xv6 所必需的运行文件。

## 5. 第四行：生成精简符号表

```make
	$(OBJDUMP) -t $K/kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $K/kernel.sym
```

变量展开并经过 make 处理后，大致等价于：

```bash
riscv64-linux-gnu-objdump -t kernel/kernel \
  | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$/d' \
  > kernel/kernel.sym
```

这一行由四个环节组成：

```text
kernel/kernel
      │
      ▼
objdump -t 输出完整符号表
      │
      ▼
管道交给 sed
      │
      ▼
删除标题、精简字段、删除空行
      │
      ▼
kernel/kernel.sym
```

### 5.1 `objdump -t`：输出符号表

小写 `-t` 表示显示 ELF 符号表。原始输出可能类似：

```text
kernel/kernel:     file format elf64-littleriscv

SYMBOL TABLE:
0000000080000000 l    d  .text  0000000000000000 .text
0000000080000000 g       .text  0000000000000000 _entry
0000000080000030 g       .text  0000000000000080 start
0000000080000100 g       .text  00000000000000a0 main
```

一条符号记录可能包含：

- 符号地址
- 符号是局部还是全局
- 符号类型
- 所属段
- 符号大小
- 符号名称

符号可能是函数、全局变量、汇编标签、段边界或链接脚本定义的特殊位置。

### 5.2 管道 `|`

```bash
objdump ... | sed ...
```

管道把左侧 `objdump` 的标准输出直接连接到右侧 `sed` 的标准输入。这样不需要先创建一个完整的临时符号表文件。

### 5.3 sed 第一条命令

```sed
1,/SYMBOL TABLE/d
```

它表示：从输入的第一行开始，一直删除到第一次匹配 `SYMBOL TABLE` 的行，并且也删除匹配行本身。

所以以下标题会被去掉：

```text
kernel/kernel: file format elf64-littleriscv
SYMBOL TABLE:
```

处理后只剩真正的符号记录。

### 5.4 sed 第二条命令

```sed
s/ .* / /
```

这是替换命令。基本形式为：

```text
s/匹配内容/替换内容/
```

其中 ` .* ` 表示：从一个空格开始，经过任意数量字符，一直匹配到后面的空格。由于 `.*` 默认是贪婪匹配，它通常会延伸到该行最后一个空格。

例如原始记录：

```text
0000000080000100 g       .text  00000000000000a0 main
```

中间的属性字段会被一个空格替换，得到：

```text
0000000080000100 main
```

最终只保留最重要的两个字段：

```text
符号地址 符号名称
```

### 5.5 sed 第三条命令以及 `$$`

Makefile 中写的是：

```sed
/^$$/d
```

但 sed 最终收到的是：

```sed
/^$/d
```

原因是 `$` 对 make 有特殊含义。要把一个字面量 `$` 传递给 Shell，Makefile 中必须写成 `$$`。

处理层次如下：

```text
Makefile 源码：/^$$/d
             │
             │ make 将 $$ 转换成 $
             ▼
sed 实际收到：/^$/d
```

在正则表达式中：

- `^` 表示行首
- `$` 表示行尾

因此 `^$` 只能匹配没有任何字符的空行，末尾的 `d` 表示删除该行。

### 5.6 写入 `kernel/kernel.sym`

```bash
> kernel/kernel.sym
```

经过 sed 清理的内容最终写入符号文件。例如：

```text
0000000080000000 _entry
0000000080000030 start
0000000080000100 main
0000000080001234 procinit
```

这个文件可以快速回答：

- 某个函数位于什么地址
- 某个异常地址附近有哪些符号
- 内核函数的排列顺序
- 汇编中的地址对应哪个函数或全局符号

## 6. 三个输出文件的区别

| 文件 | 文件性质 | 主要用途 |
|---|---|---|
| `kernel/kernel` | RISC-V ELF 内核 | 由 QEMU 实际加载和运行 |
| `kernel/kernel.asm` | 源码与汇编混合的文本文件 | 阅读指令、分析底层实现和调试 |
| `kernel/kernel.sym` | 地址和符号名构成的文本文件 | 地址到函数/变量名称的快速映射 |

`kernel/kernel` 是核心构建产物；`.asm` 和 `.sym` 是根据它生成的辅助分析文件。

## 7. 一个容易忽略的 make 细节

规则中只有 `kernel/kernel` 被声明为目标：

```make
kernel/kernel: ...
```

但命令还额外生成了：

```text
kernel/kernel.asm
kernel/kernel.sym
```

make 并不知道这两个文件也是独立目标。比如只删除：

```text
kernel/kernel.asm
```

如果 `kernel/kernel` 仍然存在并且比所有依赖都新，重新执行 `make` 时，这条规则可能不会运行，因此 `.asm` 不会自动恢复。

要强制重新生成，可以让内核重新链接，例如先删除 `kernel/kernel`，然后重新执行 make。更严格的 Makefile 也可以使用多目标规则或分别声明 `.asm` 和 `.sym` 的生成规则。

## 8. 这段规则不负责什么

这段规则不直接负责：

- 把 `.c` 文件编译为 `.o`
- 把 `.S` 汇编文件编译为 `.o`
- 构建用户程序
- 创建 `fs.img`
- 启动 QEMU

它依赖其他规则先生成所有 `.o`，然后完成内核链接及辅助文件生成。后续的 `qemu` 目标才会使用生成的内核启动虚拟机。

## 9. 完整构建关系

```text
kernel/*.c ── RISC-V GCC ──► kernel/*.o ──┐
                                           │
kernel/*.S ── RISC-V GCC ──► kernel/*.o ──┼── RISC-V LD + kernel.ld
                                           │             │
kernel/kernel.ld ──────────────────────────┘             ▼
                                                 kernel/kernel
                                                   │       │
                                     objdump -S ───┘       └── objdump -t
                                          │                       │
                                          ▼                       ▼
                                  kernel/kernel.asm          sed 清理
                                                                  │
                                                                  ▼
                                                          kernel/kernel.sym
```

## 10. 一句话总结

这段 Makefile 代码将 xv6 的全部内核对象文件按照 `kernel.ld` 规定的内存布局链接成 QEMU 可加载的 RISC-V ELF 内核，同时生成用于分析机器指令的反汇编文件和用于查询地址的精简符号表。

---

# 附录：xv6 `fs.img` 文件系统镜像生成规则

## 一、原始代码

在 `/home/wangxin/xv6-labs-2025/Makefile` 中有以下规则：

```make
fs.img: mkfs/mkfs README $(UEXTRA) $(UPROGS)
	mkfs/mkfs fs.img README $(UEXTRA) $(UPROGS)
```

这是一条标准的 Makefile 构建规则，其通用格式为：

```make
目标文件: 依赖文件
	生成目标文件时执行的命令
```

对应到这里：

- 目标文件是 `fs.img`。
- 依赖文件是 `mkfs/mkfs`、`README`、`$(UEXTRA)` 和 `$(UPROGS)`。
- 第二行是创建 `fs.img` 时实际执行的命令。

## 二、第一行：声明目标及其依赖

```make
fs.img: mkfs/mkfs README $(UEXTRA) $(UPROGS)
```

冒号左侧的 `fs.img` 是构建目标，冒号右侧是生成它所需要的全部文件。

### 1. `fs.img`

`fs.img` 是 xv6 使用的文件系统磁盘镜像。它不是普通目录，而是一个二进制文件，其中包含 xv6 文件系统所需的数据结构，例如：

- 超级块；
- inode 区域；
- 空闲块位图；
- 日志区域；
- 根目录；
- 用户程序和普通文件的内容。

启动 xv6 时，QEMU 会把这个文件作为磁盘提供给 xv6。xv6 内核中的文件系统代码会按照自己的磁盘布局读取它。

### 2. `mkfs/mkfs`

`mkfs/mkfs` 是创建 xv6 文件系统镜像的工具。这里的 `mkfs` 不是 xv6 内部运行的程序，而是在宿主 Linux 系统中运行的程序。

Makefile 中还定义了它的生成规则：

```make
mkfs/mkfs: mkfs/mkfs.c $K/fs.h $K/param.h
	gcc $(XCFLAGS) -Werror -Wall -I. -o mkfs/mkfs mkfs/mkfs.c
```

这表示宿主系统使用自己的 `gcc` 编译 `mkfs/mkfs.c`，生成可直接在 Linux 中执行的 `mkfs/mkfs`。如果该工具尚未生成，`make` 会先编译它，再生成 `fs.img`。

### 3. `README`

`README` 是一个普通文件。它既是 `fs.img` 的依赖，也是稍后传给 `mkfs/mkfs` 的输入文件，因此会被复制到 xv6 文件系统的根目录中。

### 4. `$(UEXTRA)`

`$(UEXTRA)` 是 Makefile 变量，保存某些实验需要额外写入文件系统的文件。它最初为空：

```make
UEXTRA=
```

在特定实验中，Makefile 会向它追加文件。例如 `util` 实验包含：

```make
ifeq ($(LAB),util)
	UEXTRA += user/findtest.sh
	UEXTRA += user/sixfive.txt
	UPROGS += $U/_memdump
endif
```

当 `LAB=util` 时，变量近似展开为：

```text
user/findtest.sh user/sixfive.txt
```

这两个文件随后会被写入 `fs.img`。

### 5. `$(UPROGS)`

`$(UPROGS)` 保存所有要放进 xv6 文件系统的用户程序，例如：

```make
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	...
```

如果变量 `U` 的值为 `user`，展开后就会得到：

```text
user/_cat user/_echo user/_forktest user/_grep ...
```

这些文件不是源代码，而是已经编译、链接完成的 xv6 用户程序。把 `$(UPROGS)` 写在依赖列表中还有一个重要作用：`make` 必须先生成所有用户程序，之后才能创建包含它们的文件系统镜像。

## 三、第二行：真正创建文件系统镜像

```make
	mkfs/mkfs fs.img README $(UEXTRA) $(UPROGS)
```

变量展开后，命令大致类似：

```bash
mkfs/mkfs fs.img README user/findtest.sh user/sixfive.txt \
  user/_cat user/_echo user/_forktest user/_grep user/_init user/_ls
```

各参数的含义是：

- `mkfs/mkfs`：要运行的宿主工具；
- 第一个参数 `fs.img`：要创建的镜像文件；
- 后续参数：需要复制进镜像的文件。

在 `mkfs/mkfs.c` 中，镜像文件通过类似下面的代码打开：

```c
fsfd = open(argv[1], O_RDWR | O_CREAT | O_TRUNC, 0666);
```

其中：

- `O_CREAT` 表示文件不存在时创建它；
- `O_TRUNC` 表示文件已存在时先清空它；
- `O_RDWR` 表示以可读写方式打开。

因此，每当这条命令重新执行时，都会重新生成一个新的 `fs.img`，而不是简单地向旧镜像末尾追加内容。

## 四、`mkfs/mkfs` 的主要工作流程

执行命令后，`mkfs/mkfs` 大致完成以下工作：

1. 创建或清空 `fs.img`。
2. 计算日志、inode、位图和数据块各自占用的空间。
3. 写入文件系统超级块。
4. 创建根目录 inode。
5. 在根目录中加入 `.` 和 `..`。
6. 依次打开命令行给出的每个输入文件。
7. 为每个文件分配 inode 和数据块。
8. 将文件内容复制到 `fs.img`。
9. 在根目录中创建相应的目录项。
10. 更新空闲块位图。

最终生成的镜像已经包含启动 xv6 用户空间所需要的程序和文件。

## 五、为什么用户程序的文件名前面带 `_`

宿主系统中的程序文件名是：

```text
user/_cat
user/_ls
user/_rm
```

但进入 xv6 后看到的名称是：

```text
cat
ls
rm
```

`mkfs/mkfs.c` 中包含去掉前导下划线的逻辑：

```c
if (shortname[0] == '_')
  shortname += 1;
```

使用下划线是为了防止宿主 Linux 将 xv6 用户程序和 Linux 自带的 `cat`、`ls`、`rm` 等命令混淆。构建阶段保留 `_`，写入 xv6 文件系统时再将其去掉。

程序路径中的 `user/` 也不会出现在 xv6 文件系统里。`mkfs` 会先去掉该路径前缀：

```c
if (strncmp(argv[i], "user/", 5) == 0)
  shortname = argv[i] + 5;
```

因此 `user/_cat` 最终会作为根目录中的 `cat` 写入镜像。

## 六、Make 如何决定是否执行这条命令

执行 `make` 时，Make 会比较目标与依赖的状态。

以下任一情况发生时，都需要重新生成 `fs.img`：

- `fs.img` 不存在；
- `mkfs/mkfs` 比 `fs.img` 更新；
- `README` 比 `fs.img` 更新；
- `$(UEXTRA)` 中的任意文件比 `fs.img` 更新；
- `$(UPROGS)` 中的任意用户程序比 `fs.img` 更新。

例如，修改并重新编译 `user/cat.c` 后，`user/_cat` 的修改时间会更新。因为它属于 `fs.img` 的依赖，下一次执行 `make` 时便会重新运行 `mkfs/mkfs`，把新版 `cat` 写入镜像。

如果所有依赖都没有变化，而且 `fs.img` 已存在并比依赖更新，Make 就不会重复执行创建命令。

## 七、为什么两行中都出现了相同的文件

乍看之下，依赖列表和命令参数似乎重复：

```make
fs.img: mkfs/mkfs README $(UEXTRA) $(UPROGS)
	mkfs/mkfs fs.img README $(UEXTRA) $(UPROGS)
```

实际上两处承担不同职责：

- 第一行告诉 Make **何时需要重新构建**，以及构建前必须准备哪些文件；
- 第二行告诉 shell **具体怎样构建**，并把这些文件作为参数传给 `mkfs/mkfs`。

如果文件只出现在第二行而没有出现在依赖列表中，文件修改后 Make 可能无法发现 `fs.img` 已过期。如果文件只出现在依赖列表中而没有传给 `mkfs/mkfs`，它虽然会触发重建，却不会被复制进镜像。

## 八、命令行开头必须使用 Tab

Makefile 中的命令行通常必须以 Tab 字符开头：

```make
fs.img: ...
	mkfs/mkfs ...
```

这里展示的 `\t` 表示一个真正的 Tab。若错误地使用普通空格，Make 通常会报告：

```text
missing separator
```

这是 Makefile 最常见的语法问题之一。

## 九、完整的构建关系

该规则体现的构建流程可以概括为：

```text
用户程序源代码
      │
      ▼
编译并链接出 user/_cat、user/_ls 等文件
      │
      ├──────────────┐
      │              │
mkfs/mkfs.c      README、UEXTRA
      │              │
      ▼              │
编译出 mkfs/mkfs     │
      │              │
      └──────┬───────┘
             ▼
mkfs/mkfs fs.img README $(UEXTRA) $(UPROGS)
             │
             ▼
          fs.img
             │
             ▼
     QEMU 将其作为 xv6 磁盘
```

## 十、一句话总结

这条 Makefile 规则的含义是：先确保文件系统创建工具、普通文件、实验额外文件以及全部 xv6 用户程序都已经准备好，然后运行宿主工具 `mkfs/mkfs`，重新创建 `fs.img`，并将这些文件写入 xv6 文件系统的根目录，供 xv6 启动后使用。
