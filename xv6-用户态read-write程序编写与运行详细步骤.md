# xv6 用户态 `read/write` 程序编写与运行详细步骤

本文基于以下 xv6 仓库：

```text
/home/wangxin/xv6-labs-2025
```

目标是在 `user` 目录中添加一个名为 `rwcopy` 的用户程序。该程序调用 xv6 的 `read()` 和 `write()` 系统调用，将标准输入中的数据原样写到标准输出。

完成后可以在 xv6 Shell 中这样使用：

```text
$ echo hello xv6 | rwcopy
hello xv6
```

---

## 1. 实验目标

本实验分为三部分：

1. 在 `user` 目录下创建 `rwcopy.c`。
2. 把 `user/_rwcopy` 添加到 Makefile 的 `UPROGS` 列表。
3. 执行 `make qemu`，在 xv6 Shell 中运行并验证程序。

程序的数据流如下：

```text
标准输入 fd 0
      │
      │ read(0, buf, size)
      ▼
   内存缓冲区
      │
      │ write(1, buf, n)
      ▼
标准输出 fd 1
```

在 Unix 文件描述符约定中：

| 文件描述符 | 名称 | 默认用途 |
|---:|---|---|
| `0` | stdin | 标准输入 |
| `1` | stdout | 标准输出 |
| `2` | stderr | 标准错误 |

---

## 2. 进入项目目录

在 Ubuntu 终端执行：

```bash
cd /home/wangxin/xv6-labs-2025
```

确认当前目录：

```bash
pwd
```

预期输出：

```text
/home/wangxin/xv6-labs-2025
```

检查主要文件：

```bash
ls
```

应该可以看到：

```text
Makefile  kernel  user  mkfs  README  ...
```

---

## 3. 创建用户程序源码

使用编辑器创建文件：

```bash
nano user/rwcopy.c
```

也可以使用其他编辑器，例如：

```bash
vim user/rwcopy.c
```

将以下代码完整写入 `user/rwcopy.c`：

```c
#include "kernel/types.h"
#include "user/user.h"

int
main(void)
{
  char buf[128];
  int n;

  // fd 0 是标准输入。
  // read() 返回本次实际读取的字节数。
  while((n = read(0, buf, sizeof(buf))) > 0){
    int written = 0;

    // write() 理论上可能只写出部分数据，因此循环写入，
    // 直到本次 read() 读取的数据全部输出。
    while(written < n){
      int m = write(1, buf + written, n - written);

      // fd 1 是标准输出。
      if(m <= 0){
        char msg[] = "rwcopy: write error\n";
        write(2, msg, sizeof(msg) - 1);
        exit(1);
      }

      written += m;
    }
  }

  // read() 返回负数表示读取失败。
  if(n < 0){
    char msg[] = "rwcopy: read error\n";
    write(2, msg, sizeof(msg) - 1);
    exit(1);
  }

  // read() 返回 0 表示遇到文件结尾 EOF。
  exit(0);
}
```

如果使用 Nano，保存并退出的方法是：

```text
Ctrl-O
Enter
Ctrl-X
```

---

## 4. 逐段理解程序

### 4.1 头文件

```c
#include "kernel/types.h"
#include "user/user.h"
```

`kernel/types.h` 提供 xv6 使用的基本类型定义。

`user/user.h` 声明用户程序可以使用的系统调用和用户库函数，包括：

```c
int read(int, void*, int);
int write(int, const void*, int);
int exit(int);
```

xv6 用户程序不能直接使用 Ubuntu 的 `<stdio.h>`，因为它不链接宿主机的 glibc。用户程序必须使用 xv6 自己提供的接口。

### 4.2 缓冲区

```c
char buf[128];
```

该数组临时保存从标准输入读取的数据。每次最多读取 128 字节。

缓冲区大小并不要求一定为 128，也可以使用 64、256 或 512。缓冲区变大后，每次系统调用可以搬运更多数据，但会占用更多用户栈空间。

### 4.3 从标准输入读取

```c
n = read(0, buf, sizeof(buf));
```

三个参数分别为：

```text
0            标准输入的文件描述符
buf          保存输入数据的内存地址
sizeof(buf)  最多允许读取的字节数
```

`read()` 返回值的含义：

| 返回值 | 含义 |
|---:|---|
| `> 0` | 实际读到的字节数 |
| `0` | 遇到文件末尾 EOF |
| `< 0` | 读取失败 |

因此外层循环：

```c
while((n = read(0, buf, sizeof(buf))) > 0)
```

会持续读取，直到遇到 EOF 或错误。

### 4.4 写入标准输出

```c
write(1, buf + written, n - written);
```

参数含义为：

```text
1              标准输出文件描述符
buf + written  尚未输出部分的起始地址
n - written    尚未输出的字节数
```

虽然 xv6 控制台写入通常能一次完成，但健壮的程序不应无条件假设 `write()` 总能写出请求的全部字节。因此程序使用内层循环处理部分写入。

### 4.5 错误输出

```c
write(2, msg, sizeof(msg) - 1);
```

错误信息写向文件描述符 2，也就是标准错误。

`sizeof(msg)` 包括字符串结尾的 `\0`，而终端不需要输出这个终止字符，所以使用：

```c
sizeof(msg) - 1
```

### 4.6 正常退出

```c
exit(0);
```

状态码 0 表示程序正常结束；状态码 1 表示程序发生错误。

xv6 的 `exit()` 不会返回，因此 `main()` 无须再写 `return 0`。

---

## 5. 检查源码文件

确认文件已经创建：

```bash
ls -l user/rwcopy.c
```

查看内容：

```bash
sed -n '1,160p' user/rwcopy.c
```

还可以检查是否同时出现了 `read` 和 `write`：

```bash
grep -nE 'read\(|write\(' user/rwcopy.c
```

---

## 6. 修改 Makefile 的 `UPROGS`

打开 Makefile：

```bash
nano Makefile
```

搜索：

```text
UPROGS=
```

当前列表大致如下：

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
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_logstress\
	$U/_forphan\
	$U/_dorphan\
```

在这个列表末尾加入：

```make
	$U/_rwcopy
```

修改后应类似：

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
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_logstress\
	$U/_forphan\
	$U/_dorphan\
	$U/_rwcopy
```

注意以下几点：

1. 源文件名是 `user/rwcopy.c`，没有下划线。
2. 构建出的宿主文件名是 `user/_rwcopy`，带下划线。
3. xv6 Shell 中执行的名称是 `rwcopy`，不带下划线。
4. 列表中除最后一项外，每一项末尾都需要反斜杠 `\`。
5. 最后一项 `$U/_rwcopy` 后面不需要反斜杠。
6. 应把条目加入 `UPROGS` 列表，不能随意追加到整个 Makefile 最后一行。

Makefile 中的 `_` 用于避免宿主 Ubuntu 把 xv6 程序和宿主命令混淆。`mkfs/mkfs` 将它写入 xv6 文件系统时会去掉前导下划线。

---

## 7. 为什么只添加一个 `UPROGS` 条目就能编译

Makefile 已经提供通用用户程序链接规则：

```make
_%: %.o $(ULIB) $U/user.ld
	$(LD) $(LDFLAGS) -T $U/user.ld -o $@ $< $(ULIB)
	$(OBJDUMP) -S $@ > $*.asm
	$(OBJDUMP) -t $@ | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $*.sym
```

同时 Make 具有从 `.c` 生成 `.o` 的规则。因此依赖链为：

```text
user/rwcopy.c
      │
      │ RISC-V GCC 编译
      ▼
user/rwcopy.o
      │
      │ 与 ULIB 和 user.ld 链接
      ▼
user/_rwcopy
      │
      │ mkfs/mkfs 写入 fs.img，并去掉前导下划线
      ▼
xv6 文件系统中的 /rwcopy
```

`ULIB` 包含：

```make
ULIB = user/ulib.o user/usys.o user/printf.o user/umalloc.o
```

其中 `user/usys.o` 提供 `read()`、`write()` 和 `exit()` 等系统调用包装代码。

---

## 8. 检查 Makefile 修改

执行：

```bash
grep -n rwcopy Makefile
```

预期输出类似：

```text
198:    $U/_rwcopy
```

查看完整 `UPROGS` 上下文：

```bash
sed -n '175,205p' Makefile
```

如果使用 Git，还可以只查看自己修改的内容：

```bash
git diff -- user/rwcopy.c Makefile
```

---

## 9. 先单独编译程序

在启动 QEMU 前，可以先验证程序能否成功编译：

```bash
make user/_rwcopy
```

构建命令大致包括：

```text
riscv64-linux-gnu-gcc ... -c -o user/rwcopy.o user/rwcopy.c
riscv64-linux-gnu-ld ... -o user/_rwcopy user/rwcopy.o ...
riscv64-linux-gnu-objdump -S user/_rwcopy > user/rwcopy.asm
riscv64-linux-gnu-objdump -t user/_rwcopy > ...
```

构建成功后检查产物：

```bash
ls -l user/rwcopy.o user/_rwcopy user/rwcopy.asm user/rwcopy.sym
```

主要文件含义：

| 文件 | 含义 |
|---|---|
| `user/rwcopy.o` | 尚未最终链接的 RISC-V 对象文件 |
| `user/_rwcopy` | 最终用户程序 ELF |
| `user/rwcopy.asm` | 源码与 RISC-V 指令混合反汇编 |
| `user/rwcopy.sym` | 地址和符号信息 |

可以验证它确实是 RISC-V ELF：

```bash
file user/_rwcopy
```

---

## 10. 重新构建并启动 xv6

执行：

```bash
make qemu
```

Make 将根据依赖关系完成：

```text
编译 user/rwcopy.c
        │
        ▼
链接 user/_rwcopy
        │
        ▼
把 user/_rwcopy 加入 fs.img
        │
        ▼
启动 qemu-system-riscv64
```

构建输出中应当出现类似内容：

```text
... -c -o user/rwcopy.o user/rwcopy.c
... -o user/_rwcopy user/rwcopy.o ...
mkfs/mkfs fs.img ... user/_rwcopy ...
```

不需要每次都执行 `make clean`。Make 会比较修改时间，只重新生成发生变化的目标。

如果怀疑旧构建文件影响结果，可以执行：

```bash
make clean
make qemu
```

但 `make clean` 会删除全部构建产物，下一次构建时间会更长。

---

## 11. 等待 xv6 Shell

正常启动输出类似：

```text
xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$
```

出现 `$` 表示：

- 内核启动完成；
- 文件系统已经挂载；
- `/init` 已运行；
- `/init` 已启动 Shell；
- 可以执行用户命令。

---

## 12. 确认程序已经进入 xv6 文件系统

在 xv6 Shell 中执行：

```text
$ ls
```

应该能看到：

```text
rwcopy
```

这里的名称没有下划线。宿主构建目录中的 `user/_rwcopy` 被 `mkfs` 写入镜像时变成了 `/rwcopy`。

如果 `ls` 中没有 `rwcopy`，说明程序没有进入 `fs.img`，应检查 Makefile 的 `UPROGS` 配置。

---

## 13. 测试方法一：使用管道

这是最清楚的测试方法：

```text
$ echo hello xv6 | rwcopy
hello xv6
```

数据流为：

```text
echo
 │
 │ write(标准输出)
 ▼
管道写端
 │
 ▼
管道读端
 │
 │ 成为 rwcopy 的 fd 0
 ▼
rwcopy: read(0, ...)
 │
 │ write(1, ...)
 ▼
控制台
```

还可以测试多组数据：

```text
$ echo 123456789 | rwcopy
123456789
```

---

## 14. 测试方法二：交互输入

直接执行：

```text
$ rwcopy
```

然后输入：

```text
hello
```

可能看到：

```text
hello
hello
```

第一行是 xv6 控制台驱动对键盘输入的回显；第二行是 `rwcopy` 调用 `write()` 输出的结果。这并不表示程序读取了两次。

继续输入：

```text
xv6 test
xv6 test
```

输入结束后按：

```text
Ctrl-D
```

在 xv6 控制台中，`Ctrl-D` 表示 EOF。控制台的 `read()` 最终返回 0，外层循环结束，程序调用 `exit(0)`，Shell 提示符重新出现：

```text
$
```

---

## 15. 测试方法三：输入重定向

`README` 默认会被写入 xv6 文件系统，可以执行：

```text
$ rwcopy < README
```

程序会把 `README` 的内容输出到终端。

也可以先创建测试文件：

```text
$ echo abc > test.txt
$ rwcopy < test.txt
abc
```

还可以验证输出重定向：

```text
$ rwcopy < README > copy.txt
$ cat copy.txt
```

这时 `rwcopy` 的文件描述符并没有改变编号：

```text
fd 0 → README
fd 1 → copy.txt
```

Shell 在运行程序前完成了文件描述符重定向，因此程序本身不需要知道输入来自键盘还是文件。

---

## 16. `read()` 系统调用的内核路径

当用户程序调用：

```c
read(0, buf, sizeof(buf));
```

大致执行路径为：

```text
user/rwcopy.c
      │
      ▼
user/usys.S 中的 read 包装函数
      │
      │ a7 = SYS_read
      │ ecall
      ▼
kernel/trampoline.S:uservec
      │
      ▼
kernel/trap.c:usertrap()
      │
      ▼
kernel/syscall.c:syscall()
      │
      ▼
kernel/sysfile.c:sys_read()
      │
      ▼
fileread()
      │
      ├── 控制台输入
      ├── 普通文件
      └── 管道
```

系统调用包装函数会把系统调用号放进寄存器 `a7`，再执行 `ecall`。用户参数已经按照 RISC-V 调用约定位于 `a0`、`a1`、`a2` 等寄存器中。

CPU 进入 S-mode 后，trampoline 保存用户寄存器、切换到内核页表和进程内核栈，然后由 `usertrap()` 和 `syscall()` 分派到 `sys_read()`。

返回值最终写回 trapframe 的 `a0`，程序恢复用户态后，C 代码便得到 `read()` 的返回值。

---

## 17. `write()` 系统调用的内核路径

执行：

```c
write(1, buf, n);
```

大致路径为：

```text
user/rwcopy.c
      │
      ▼
user/usys.S:write
      │
      │ a7 = SYS_write
      │ ecall
      ▼
trampoline.S:uservec
      │
      ▼
trap.c:usertrap()
      │
      ▼
syscall.c:syscall()
      │
      ▼
sysfile.c:sys_write()
      │
      ▼
filewrite()
      │
      ├── consolewrite()
      ├── pipewrite()
      └── inode 写入
```

当文件描述符 1 指向控制台时，数据最终通过 UART 显示在运行 QEMU 的终端中。

当 Shell 使用输出重定向时，文件描述符 1 可能指向普通文件；`rwcopy` 无须修改代码，`write(1, ...)` 会自动写入该文件。

---

## 18. 为什么该程序体现了 Unix 的统一 I/O 模型

`rwcopy` 只知道两个整数：

```text
0：输入
1：输出
```

它不需要区分输入究竟来自：

- 键盘；
- 文件；
- 管道；
- 另一个程序。

也不需要区分输出究竟流向：

- 终端；
- 文件；
- 管道。

Shell 通过建立和重定向文件描述符改变数据来源与去向。程序始终使用相同的 `read()` 和 `write()` 接口，这正是 Unix“一切皆文件”和统一文件描述符接口的核心思想。

---

## 19. 常见错误与解决方法

### 19.1 `make: No rule to make target 'user/_rwcopy'`

检查源文件是否叫：

```text
user/rwcopy.c
```

错误命名包括：

```text
user/_rwcopy.c
user/rwcopy
rwcopy.c
```

正确关系是：

```text
源码：user/rwcopy.c
目标：user/_rwcopy
```

### 19.2 编译器找不到 `read` 或 `write`

确认包含：

```c
#include "user/user.h"
```

不要改用宿主系统的：

```c
#include <unistd.h>
```

xv6 用户程序使用自己的头文件和系统调用入口。

### 19.3 找不到 `uint` 等类型

确保 `user/user.h` 之前包含：

```c
#include "kernel/types.h"
```

### 19.4 Makefile 报 `missing separator`

这通常表示 Makefile 命令行使用了普通空格而不是 Tab，或者列表续行的反斜杠配置错误。

在 `UPROGS` 列表中，确认前一项具有 `\`，最后一项不需要：

```make
	$U/_dorphan\
	$U/_rwcopy
```

### 19.5 xv6 中提示 `exec rwcopy failed`

这表示 Shell 没能从文件系统找到或加载程序。检查：

```bash
grep -n rwcopy Makefile
ls -l user/_rwcopy
```

然后重新生成文件系统：

```bash
make qemu
```

### 19.6 `ls` 中没有 `rwcopy`

说明 `user/_rwcopy` 没有被写入 `fs.img`。重点检查：

- `UPROGS` 是否包含 `$U/_rwcopy`；
- 条目是否在 `fs.img` 规则之前定义；
- 是否重新运行了 `make qemu`；
- `mkfs/mkfs` 命令行中是否出现 `user/_rwcopy`。

### 19.7 程序运行后一直不退出

直接运行 `rwcopy` 时，它会持续等待标准输入。按：

```text
Ctrl-D
```

发送 EOF。

### 19.8 输入内容看起来被输出两次

第一遍来自控制台输入回显，第二遍才是程序调用 `write()` 的结果。使用管道测试可以避免这种视觉混淆：

```text
$ echo hello | rwcopy
hello
```

### 19.9 想退出 QEMU

在 `-nographic` 模式下依次按：

```text
Ctrl-A
x
```

即先按 `Ctrl-A`，松开后再按 `x`。

---

## 20. 推荐的完整操作清单

宿主 Ubuntu 中执行：

```bash
cd /home/wangxin/xv6-labs-2025
nano user/rwcopy.c
nano Makefile
grep -n rwcopy Makefile
make user/_rwcopy
make qemu
```

进入 xv6 Shell 后执行：

```text
$ ls
$ echo hello xv6 | rwcopy
hello xv6

$ rwcopy
test
test
Ctrl-D

$ rwcopy < README
```

实验成功的判断标准：

1. 宿主目录生成 `user/_rwcopy`。
2. 构建 `fs.img` 时命令行包含 `user/_rwcopy`。
3. xv6 中执行 `ls` 能看到 `rwcopy`。
4. `echo hello | rwcopy` 能输出 `hello`。
5. 交互输入时，按 `Ctrl-D` 能正常退出程序。

---

## 21. 实验总结

本实验完成的完整链路是：

```text
编写 user/rwcopy.c
        │
        ▼
在 UPROGS 中加入 user/_rwcopy
        │
        ▼
RISC-V 编译器生成 user/rwcopy.o
        │
        ▼
链接器生成 user/_rwcopy
        │
        ▼
mkfs 将程序写入 fs.img，名称变为 rwcopy
        │
        ▼
QEMU 启动 xv6
        │
        ▼
Shell exec("rwcopy")
        │
        ▼
read(0, ...) 从标准输入读取
        │
        ▼
write(1, ...) 写到标准输出
```

通过这个小程序可以同时理解：

- xv6 用户程序的目录和头文件结构；
- Makefile 的 `UPROGS` 构建机制；
- RISC-V 用户程序的编译与链接；
- `mkfs` 如何把程序加入文件系统镜像；
- 文件描述符 0、1、2 的含义；
- `read()` 和 `write()` 系统调用；
- Shell 管道与输入输出重定向；
- 用户态通过 `ecall` 进入内核态的基本路径。
