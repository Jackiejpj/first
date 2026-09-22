# xv6 `sleep` 用户程序实现教程

> 源码目录：`/home/wangxin/xv6-labs-2025`  
> 实验分支：`util`  
> 目标：实现命令 `sleep ticks`，让当前用户进程暂停指定数量的时钟 tick。

## 1. 先明确本仓库的接口差异

很多旧版 xv6 教程会写：

```c
sleep(atoi(argv[1]));
```

但是在当前 `xv6-labs-2025` 源码中，用户态系统调用已经命名为：

```c
int pause(int);
```

可以在 `user/user.h` 中看到声明，在 `user/usys.pl` 中看到系统调用桩：

```c
// user/user.h
int pause(int);
```

```perl
# user/usys.pl
entry("pause");
```

对应的内核处理函数是 `kernel/sysproc.c` 中的 `sys_pause()`。因此，本实验创建的命令名仍然是 `sleep`，但程序内部应调用 `pause()`：

```text
用户输入 sleep 10
        ↓
执行 user/_sleep
        ↓
sleep.c 调用 pause(10)
        ↓
进入内核 sys_pause()
```

## 2. 需要手动完成的两个修改

进入 xv6 源码目录：

```sh
cd /home/wangxin/xv6-labs-2025
```

你需要：

1. 新建 `user/sleep.c`；
2. 在 `Makefile` 的 `UPROGS` 中加入 `$U/_sleep`。

不需要新增系统调用，因为 `pause` 的用户接口、系统调用号和内核实现均已存在。

## 3. 新建 `user/sleep.c`

执行：

```sh
vim user/sleep.c
```

写入以下完整代码：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int ticks;

  if(argc != 2){
    fprintf(2, "usage: sleep ticks\n");
    exit(1);
  }

  ticks = atoi(argv[1]);
  pause(ticks);

  exit(0);
}
```

保存并退出。

### 3.1 逐行理解

```c
#include "kernel/types.h"
```

引入 xv6 的基本类型定义，例如 `uint`、`uint64`。xv6 用户程序通常都包含它。

```c
#include "kernel/stat.h"
```

引入文件状态结构定义。虽然本程序没有直接使用 `struct stat`，但这是 xv6 示例用户程序的常见头文件组合。

```c
#include "user/user.h"
```

引入用户态函数和系统调用声明。本程序使用的 `fprintf`、`exit`、`atoi` 和 `pause` 都在这里声明。

```c
int main(int argc, char *argv[])
```

- `argc` 是命令行参数总数；
- `argv` 是参数字符串数组；
- 输入 `sleep 10` 时，`argc == 2`；
- `argv[0]` 是 `"sleep"`；
- `argv[1]` 是 `"10"`。

```c
if(argc != 2){
  fprintf(2, "usage: sleep ticks\n");
  exit(1);
}
```

程序要求恰好提供一个 ticks 参数。若参数数量错误，则向文件描述符 2（标准错误）输出用法，并以非零状态退出。

```c
ticks = atoi(argv[1]);
```

命令行参数本质上是字符串，`atoi` 将数字字符串转换成 `int`。例如 `"10"` 转换成整数 `10`。

当前仓库 `user/ulib.c` 中的 `atoi` 核心逻辑是：

```c
int
atoi(const char *s)
{
  int n;

  n = 0;
  while('0' <= *s && *s <= '9')
    n = n*10 + *s++ - '0';
  return n;
}
```

它只从开头连续读取数字，不提供完整的非法输入检查：

- `atoi("10")` 得到 `10`；
- `atoi("12abc")` 得到 `12`；
- `atoi("abc")` 得到 `0`；
- `atoi("-1")` 也得到 `0`，因为第一个字符不是数字。

本实验的核心是命令行解析与系统调用，采用上述基础实现即可满足实验要求。

```c
pause(ticks);
```

进入 xv6 内核，请求暂停指定数量的 tick。在当前源码中，一个 tick 大约是 0.1 秒，因此 `sleep 10` 通常约暂停 1 秒。这里是教学环境中的近似值，不应把 tick 当成通用的固定时间单位。

```c
exit(0);
```

暂停结束后正常退出。xv6 用户程序不能像普通宿主机 C 程序那样简单依赖 `return 0`；遵循仓库风格显式调用 `exit(0)`。

## 4. 修改 `Makefile`

仅创建 `user/sleep.c` 还不够。xv6 需要把程序编译后放进文件系统镜像 `fs.img`，这样 shell 才能执行它。

打开文件：

```sh
vim Makefile
```

找到 `UPROGS=\` 列表。你当前的源码已经在列表末尾加入了 `_rwcopy`，原内容类似：

```make
	$U/_forphan\
	$U/_dorphan\
	$U/_rwcopy
```

把它改为：

```make
	$U/_forphan\
	$U/_dorphan\
	$U/_rwcopy\
	$U/_sleep
```

注意：

- `_rwcopy` 后面现在要加续行反斜杠 `\`；
- `_sleep` 前面的下划线是 xv6 构建系统的命名约定；
- 最后一项 `_sleep` 后面可以不写反斜杠；
- Makefile 命令或列表缩进通常使用 Tab，不要随意破坏已有缩进；
- 不要删除现有的 `_rwcopy`，它是当前工作区已经存在的修改。

构建规则会把 `user/sleep.c` 编译、链接为 `user/_sleep`，之后 `mkfs` 再把它写入 `fs.img`。进入 xv6 后，shell 使用的命令名是 `sleep`，而不是 `_sleep`。

## 5. 编译检查

### 5.1 先做快速的定向编译

```sh
cd /home/wangxin/xv6-labs-2025
make user/_sleep
```

如果成功，应生成：

```text
user/_sleep
user/sleep.o
user/sleep.asm
user/sleep.sym
```

若看到 `pause` 未声明，通常说明误漏了：

```c
#include "user/user.h"
```

若链接时报找不到 `pause`，应检查当前源码的 `user/usys.pl` 是否有 `entry("pause")`。本仓库已经提供，无需修改。

### 5.2 构建并启动 xv6

```sh
make qemu
```

看到 xv6 shell 的 `$` 提示符后测试：

```sh
sleep 10
echo OK
```

预期现象：输入 `sleep 10` 后，shell 大约等待 1 秒才再次显示提示符；随后 `echo OK` 输出：

```text
OK
```

测试缺少参数：

```sh
sleep
```

预期输出：

```text
usage: sleep ticks
```

也可以测试零 tick：

```sh
sleep 0
echo returned
```

它应几乎立即返回。

退出 QEMU 的常用按键是先按 `Ctrl-a`，松开后再按 `x`。

### 5.3 运行实验评分

当前 `conf/lab.mk` 已设置：

```make
LAB=util
```

因此可以运行：

```sh
make grade
```

与 `sleep` 相关的评分点主要检查：

1. 执行 `sleep` 时确实存在这个程序，而不是 shell 报 `exec sleep failed`；
2. 无参数执行能退出，不会卡住；
3. `sleep 10` 确实进入内核的 `sys_pause`；
4. 暂停结束后命令可以正常返回。

`make grade` 会先执行 `make clean`，所以生成文件被清除属于正常现象，源码文件不会被删掉。

## 6. 从用户命令到系统调用的完整路径

当 shell 执行：

```sh
sleep 10
```

调用路径如下：

```text
user/sleep.c: main(argc, argv)
  │
  ├─ argv[1] 是字符串 "10"
  ├─ atoi(argv[1]) 得到整数 10
  └─ pause(10)
       │
       ▼
user/usys.pl 生成的系统调用桩
  │  把 SYS_pause 放入 a7，执行 ecall
  ▼
kernel/trap.c: usertrap()
  │  识别来自用户态的系统调用
  ▼
kernel/syscall.c: syscall()
  │  通过 SYS_pause 查表
  ▼
kernel/sysproc.c: sys_pause()
  │
  ├─ 读取参数 n
  ├─ 记录开始时的 ticks
  ├─ 条件未满足时 sleep(&ticks, &tickslock)
  └─ ticks - ticks0 >= n 后返回
```

### 6.1 系统调用号与分发表

`kernel/syscall.h` 定义：

```c
#define SYS_pause  13
```

`kernel/syscall.c` 声明并登记内核函数：

```c
extern uint64 sys_pause(void);
```

```c
[SYS_pause]   sys_pause,
```

因此，用户态的 `pause()` 最终会分发到 `sys_pause()`。

## 7. `sys_pause()` 如何按 ticks 暂停

当前 `kernel/sysproc.c` 中的实现为：

```c
uint64
sys_pause(void)
{
  int n;
  uint ticks0;

  argint(0, &n);
  if(n < 0)
    n = 0;
  acquire(&tickslock);
  ticks0 = ticks;
  while(ticks - ticks0 < n){
    if(killed(myproc())){
      release(&tickslock);
      return -1;
    }
    sleep(&ticks, &tickslock);
  }
  release(&tickslock);
  return 0;
}
```

关键过程如下。

### 7.1 读取用户参数

```c
argint(0, &n);
```

读取系统调用的第 0 个参数，即用户程序传入的 `ticks`。

### 7.2 记录起始 tick

```c
acquire(&tickslock);
ticks0 = ticks;
```

全局变量 `ticks` 表示系统启动以来发生的时钟 tick 数。读取或修改它时使用 `tickslock`，防止并发访问造成竞态。

### 7.3 循环检查是否到期

```c
while(ticks - ticks0 < n)
```

不是等待 `ticks` 到某个绝对值，而是计算已经过去多少 tick。这种无符号减法写法也能较自然地处理计数器回绕。

### 7.4 阻塞而不是忙等

```c
sleep(&ticks, &tickslock);
```

这里的 `sleep` 是内核函数，不是正在编写的用户命令，也不是用户态系统调用。它会让当前进程进入 `SLEEPING` 状态并让出 CPU，而不是一直占用 CPU 反复检查时间。

睡眠通道是 `&ticks`。之后时钟中断也使用 `wakeup(&ticks)`，两者通道地址相同，所以等待时钟的进程能被唤醒。

注意锁的配合：内核 `sleep(chan, lk)` 会在正确持有进程锁的情况下释放 `tickslock`；被唤醒后，它会重新取得 `tickslock` 再返回。这样可以避免“检查条件后、真正睡眠前恰好发生唤醒”导致的丢失唤醒问题。

## 8. `SLEEPING → RUNNABLE` 如何由时钟中断驱动

### 8.1 进入 `SLEEPING`

`kernel/proc.c` 的内核 `sleep()` 会执行：

```c
p->chan = chan;
p->state = SLEEPING;
sched();
```

含义是：

1. 记录当前进程正在等待的通道；
2. 把状态设为 `SLEEPING`；
3. 调用 `sched()` 切换回调度器，让出 CPU。

此时该进程不在可运行队列中，不会消耗 CPU 忙等。

### 8.2 时钟中断增加 `ticks`

`kernel/trap.c` 的 `clockintr()` 执行：

```c
void
clockintr()
{
  if(cpuid() == 0){
    acquire(&tickslock);
    ticks++;
    wakeup(&ticks);
    release(&tickslock);
  }

  w_stimecmp(r_time() + 1000000);
}
```

CPU 0 每次处理时钟中断时：

1. 获取 `tickslock`；
2. 执行 `ticks++`；
3. 调用 `wakeup(&ticks)`；
4. 释放锁；
5. 设置下一次时钟中断。

### 8.3 `wakeup()` 改变进程状态

`kernel/proc.c` 的核心代码是：

```c
if(p->state == SLEEPING && p->chan == chan) {
  p->state = RUNNABLE;
}
```

因此，等待 `&ticks` 通道的进程会发生：

```text
SLEEPING --时钟中断调用 wakeup(&ticks)--> RUNNABLE
```

这里的“唤醒”不等于立刻开始执行。它只是变成 `RUNNABLE`，表示有资格被调度器选中。

### 8.4 调度器重新运行该进程

调度器扫描进程表并找到：

```c
if(p->state == RUNNABLE) {
  p->state = RUNNING;
  swtch(&c->context, &p->context);
}
```

所以后续状态是：

```text
RUNNABLE --被调度器选中--> RUNNING
```

进程从内核 `sleep()` 返回，重新检查：

```c
ticks - ticks0 < n
```

- 若时间还没到，它会再次进入 `SLEEPING`；
- 若已经经过了 `n` 个 tick，`sys_pause()` 返回用户态；
- 用户程序随后执行 `exit(0)`。

完整状态变化可概括为：

```text
RUNNING
   │ 调用内核 sleep()
   ▼
SLEEPING
   │ 时钟中断：ticks++，wakeup(&ticks)
   ▼
RUNNABLE
   │ 调度器选中
   ▼
RUNNING
   │ 未到期则再次睡眠；到期则返回用户态
   ▼
退出程序
```

## 9. 为什么使用 `while` 而不是 `if`

每次时钟中断都会执行 `wakeup(&ticks)`，因此进程可能每经过一个 tick 就被唤醒一次。例如 `pause(10)` 并不是睡一次就必然刚好经过 10 tick，而可能经历多轮：

```text
睡眠 → 第 1 tick 唤醒 → 条件不满足 → 再睡眠
睡眠 → 第 2 tick 唤醒 → 条件不满足 → 再睡眠
...
睡眠 → 第 10 tick 唤醒 → 条件满足 → 返回
```

所以必须在循环中重新检查条件。即使存在无关唤醒或调度延迟，程序仍以 `ticks - ticks0 >= n` 作为真正结束条件。

## 10. 常见错误及排查

### 错误 1：写成 `sleep(ticks)`

症状可能是用户态编译时提示隐式声明，或与内核函数概念混淆。

原因：本仓库用户系统调用名是 `pause`。

正确写法：

```c
pause(ticks);
```

### 错误 2：忘记检查 `argc`

如果直接访问 `argv[1]`，无参数执行 `sleep` 时可能访问无效指针，程序行为错误。

正确做法：

```c
if(argc != 2){
  fprintf(2, "usage: sleep ticks\n");
  exit(1);
}
```

### 错误 3：直接把 `argv[1]` 传给 `pause`

`argv[1]` 是 `char *`，而 `pause` 需要 `int`。

错误写法：

```c
pause(argv[1]);
```

正确写法：

```c
pause(atoi(argv[1]));
```

### 错误 4：只新增源码，不修改 `UPROGS`

QEMU 中会出现：

```text
exec sleep failed
```

因为程序没有被加入 xv6 文件系统镜像。

### 错误 5：Makefile 续行符遗漏

加入 `_sleep` 时，前一项 `_rwcopy` 必须以 `\` 续行，否则 `_sleep` 不属于 `UPROGS` 变量。

### 错误 6：修改了内核系统调用表

本仓库已经完整提供 `pause`：

- `user/user.h` 有声明；
- `user/usys.pl` 有系统调用桩；
- `kernel/syscall.h` 有系统调用号；
- `kernel/syscall.c` 有分发表项；
- `kernel/sysproc.c` 有 `sys_pause()`。

因此本任务只需新增用户程序并登记到 `UPROGS`，不要重复添加系统调用。

### 错误 7：宿主机上执行 `./user/_sleep`

`user/_sleep` 是 RISC-V xv6 用户程序，不能作为普通 Linux x86-64 程序运行。应进入 `make qemu` 启动的 xv6 环境后输入 `sleep 10`。

## 11. 最终自检清单

完成后逐项确认：

- [ ] 已创建 `user/sleep.c`；
- [ ] 包含 `kernel/types.h`、`kernel/stat.h`、`user/user.h`；
- [ ] 使用 `argc != 2` 检查参数数量；
- [ ] 使用 `atoi(argv[1])` 完成字符串到整数转换；
- [ ] 调用的是 `pause(ticks)`，不是旧版教程中的用户态 `sleep(ticks)`；
- [ ] 末尾调用 `exit(0)`；
- [ ] `Makefile` 的 `UPROGS` 已加入 `$U/_sleep`；
- [ ] 原有 `$U/_rwcopy` 被保留并添加了续行反斜杠；
- [ ] `make user/_sleep` 编译成功；
- [ ] QEMU 中 `sleep 10` 能暂停后返回；
- [ ] 无参数执行能显示用法并退出；
- [ ] `make grade` 中 sleep 相关测试通过。

## 12. 最终应提交的核心改动

`user/sleep.c`：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int ticks;

  if(argc != 2){
    fprintf(2, "usage: sleep ticks\n");
    exit(1);
  }

  ticks = atoi(argv[1]);
  pause(ticks);

  exit(0);
}
```

`Makefile` 的 `UPROGS` 列表末尾：

```make
	$U/_forphan\
	$U/_dorphan\
	$U/_rwcopy\
	$U/_sleep
```

这两个修改就完成了 `sleep` 用户程序。核心知识点是：用户态用 `argc/argv` 接收参数，用 `atoi` 转换 ticks，再通过 `pause` 系统调用进入内核；内核进程在等待时为 `SLEEPING`，时钟中断递增 `ticks` 并调用 `wakeup(&ticks)` 将其改为 `RUNNABLE`，最后由调度器再次运行。

---

## 13. 增强版实现：严格校验输入并检查系统调用返回值

前面的基础版最贴近 xv6 util 实验的最小要求。如果希望程序更完整，可以将 `user/sleep.c` 写成下面的增强版。它仍然使用实验要求中的 `argc/argv`、`atoi` 和 `pause`，但额外处理：

- 空字符串；
- 非数字字符；
- 负数；
- 超过有符号 `int` 最大值的 ticks；
- `pause()` 返回失败。

完整代码如下：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

#define INT_MAX_VALUE 2147483647

// 检查字符串是否表示一个合法的、可放入 int 的非负整数。
// 成功返回 0，并通过 result 返回转换后的数值；失败返回 -1。
static int
parse_ticks(const char *text, int *result)
{
  const char *p;
  uint64 value;

  if(text == 0 || text[0] == '\0')
    return -1;

  value = 0;
  for(p = text; *p != '\0'; p++){
    if(*p < '0' || *p > '9')
      return -1;

    value = value * 10 + (*p - '0');
    if(value > INT_MAX_VALUE)
      return -1;
  }

  // 输入已确认合法且不会溢出，再使用实验要求的 atoi 完成转换。
  *result = atoi(text);
  return 0;
}

int
main(int argc, char *argv[])
{
  int ticks;

  if(argc != 2){
    fprintf(2, "usage: sleep ticks\n");
    exit(1);
  }

  if(parse_ticks(argv[1], &ticks) < 0){
    fprintf(2, "sleep: ticks must be a non-negative integer\n");
    exit(1);
  }

  if(pause(ticks) < 0){
    fprintf(2, "sleep: pause failed\n");
    exit(1);
  }

  exit(0);
}
```

### 13.1 为什么增强版先检查，再调用 `atoi`

xv6 的 `atoi` 很简单，只负责读取开头连续的数字：

```text
"15"       → 15
"15abc"    → 15
"abc"      → 0
"-5"       → 0
```

如果只调用 `atoi`，程序无法区分“用户合法输入了 0”和“用户输入非法字符串后被转换为 0”。增强版先逐字符验证，再调用 `atoi`，既保留实验考察点，也避免接受含糊输入。

### 13.2 为什么用 `uint64 value` 做溢出检查

最终系统调用参数是 `int`，最大值为 `2147483647`。如果直接让 `int` 不断执行：

```c
n = n * 10 + digit;
```

超大输入可能在检查之前就发生有符号整数溢出。先用范围更大的 `uint64` 累加，并在每一步与 `INT_MAX_VALUE` 比较，可以在调用 `atoi` 前确定输入不会令其溢出。

由于程序每处理一个数字就立即检查 `value > INT_MAX_VALUE`，一旦超过 32 位有符号整数范围就返回，不会继续处理任意长度的数字串。

### 13.3 增强版的测试用例

进入 xv6 后依次测试：

```sh
sleep
sleep 0
sleep 5
sleep abc
sleep 12abc
sleep -1
sleep 2147483648
sleep 1 2
```

预期结果：

| 命令 | 预期行为 |
|---|---|
| `sleep` | 输出用法并以失败状态退出 |
| `sleep 0` | 合法，立即返回 |
| `sleep 5` | 暂停约 5 个 tick |
| `sleep abc` | 拒绝，提示 ticks 必须为非负整数 |
| `sleep 12abc` | 拒绝，不接受部分数字 |
| `sleep -1` | 拒绝负数 |
| `sleep 2147483648` | 拒绝超过 `int` 范围的输入 |
| `sleep 1 2` | 输出用法，拒绝多余参数 |

增强逻辑不会影响评分脚本的正常用例 `sleep 10`，但如果课程只要求极简版本，也可以使用第 3 节的基础实现。

## 14. RISC-V 层面的系统调用过程

理解 `pause(ticks)` 不能只停留在 C 函数调用。它最终通过 RISC-V 的 `ecall` 指令从用户态进入内核态。

### 14.1 `usys.pl` 生成系统调用桩

`user/usys.pl` 中的：

```perl
entry("pause");
```

会生成大致如下的汇编代码：

```asm
.global pause
pause:
 li a7, SYS_pause
 ecall
 ret
```

各寄存器的作用：

- `a0`：保存第一个函数参数，也就是 `ticks`；
- `a7`：保存系统调用号 `SYS_pause`；
- `ecall`：触发从 U-mode 到 S-mode 的异常；
- 系统调用返回后，`a0` 保存返回值。

调用前可抽象为：

```text
a0 = ticks
a7 = SYS_pause = 13
执行 ecall
```

### 14.2 `usertrap()` 识别系统调用

用户态执行 `ecall` 后，处理器进入 `kernel/trap.c` 的 `usertrap()`。RISC-V 用 `scause == 8` 表示来自用户态的环境调用：

```c
if(r_scause() == 8){
  if(killed(p))
    kexit(-1);

  p->trapframe->epc += 4;
  intr_on();
  syscall();
}
```

`epc += 4` 很关键。`epc` 原本指向触发异常的 `ecall` 指令；如果不前移 4 字节，返回用户态后会再次执行同一个 `ecall`，形成无限系统调用循环。

### 14.3 `syscall()` 如何查表分发

系统调用入口从进程 trapframe 的 `a7` 取出编号：

```text
编号 13
  ↓
SYS_pause
  ↓
syscalls[SYS_pause]
  ↓
sys_pause()
```

用户传入的第一个参数仍保存在 trapframe 的 `a0`。`sys_pause()` 使用：

```c
argint(0, &n);
```

把它读取为内核中的 `int n`。

### 14.4 返回值如何回到用户程序

内核将 `sys_pause()` 的返回值写回 trapframe 的 `a0`。恢复用户寄存器后，用户态系统调用桩执行 `ret`，于是 C 表达式：

```c
pause(ticks)
```

得到 `0` 或 `-1`。

## 15. 锁、睡眠通道与“丢失唤醒”问题

`sys_pause()` 中最值得分析的不是循环本身，而是：

```c
acquire(&tickslock);
...
sleep(&ticks, &tickslock);
```

### 15.1 如果没有正确加锁会怎样

设想一个错误实现：

```c
if(ticks - ticks0 < n)
  sleep(&ticks, 0);
```

可能出现以下时间线：

```text
进程 A                         时钟中断
------                         --------
检查发现时间未到
                               ticks++
                               wakeup(&ticks)
开始进入睡眠
```

时钟中断的唤醒发生在进程 A 真正标记为 `SLEEPING` 之前，因此 `wakeup` 找不到 A。随后 A 才睡下，这次唤醒就永久丢失了。若之后没有新的事件，它可能长时间无法运行。

### 15.2 xv6 如何避免丢失唤醒

`sys_pause()` 先持有 `tickslock`。进入内核 `sleep(chan, lk)` 后：

```c
acquire(&p->lock);
release(lk);
p->chan = chan;
p->state = SLEEPING;
sched();
```

关键点是：

1. 进程在释放条件锁 `tickslock` 前，先获取自己的 `p->lock`；
2. `wakeup()` 检查某进程时也必须获取该进程的 `p->lock`；
3. 因此，时钟中断不可能在“条件检查完毕”和“进程状态设为 SLEEPING”之间悄悄完成一次无法被观察到的唤醒。

这个锁交接建立了同步关系：

```text
持有 tickslock 检查条件
          ↓
sleep() 获取 p->lock
          ↓
释放 tickslock
          ↓
设为 SLEEPING
          ↓
wakeup() 获取 p->lock 后才能检查并唤醒
```

### 15.3 为什么唤醒后还要重新获取 `tickslock`

内核 `sleep()` 返回前执行：

```c
release(&p->lock);
acquire(lk);
```

因此回到 `sys_pause()` 的 `while` 条件时，当前进程再次持有 `tickslock`，可以安全读取 `ticks`。这保持了循环不变量：每次检查 `ticks - ticks0 < n` 时，都持有保护 `ticks` 的锁。

## 16. 多核环境中的 ticks 维护

当前 Makefile 默认：

```make
CPUS := 3
```

也就是说 QEMU 通常以三个虚拟 CPU 运行 xv6。但 `clockintr()` 中有：

```c
if(cpuid() == 0){
  acquire(&tickslock);
  ticks++;
  wakeup(&ticks);
  release(&tickslock);
}
```

只有 CPU 0 更新全局 `ticks` 并执行唤醒。这可以避免每个 CPU 都在各自的时钟中断中递增同一个计数器，否则三个 CPU 可能让 ticks 以预期速度的数倍增长。

其他 CPU 仍会设置自己的下一次定时器，并可能因为时钟中断进行调度，但全局时间基准由 CPU 0 维护。

`tickslock` 仍然不可省略，原因包括：

- 用户进程可能在任意 CPU 上执行 `sys_pause()` 或 `sys_uptime()`；
- CPU 0 的时钟中断会并发修改 `ticks`；
- 多个进程可能同时等待 `&ticks`；
- 锁还参与防止条件检查与睡眠之间的丢失唤醒。

## 17. ticks 回绕与无符号减法

`ticks` 和 `ticks0` 的类型都是 `uint`。`sys_pause()` 使用：

```c
ticks - ticks0 < n
```

而不是：

```c
ticks < ticks0 + n
```

第一种写法对无符号计数器回绕更稳健。例如使用一个简化的 8 位计数器：

```text
ticks0 = 250
经过 10 tick 后 ticks = 4
```

按模 256 的无符号运算：

```text
4 - 250 = 10
```

所以仍能得到正确的经过时间。相比之下，直接计算绝对截止值更容易在加法回绕后出现错误比较。

这里有一个边界假设：等待时长不应大到跨越超过无符号计数范围的一半或与有符号转换产生歧义。本实验接收 `int n`，实际测试的 ticks 很小，不会触及该问题。

## 18. 更深入的调试方法

### 18.1 使用评分脚本的断点思路

仓库的 `grade-lab-util` 对 `sleep 10` 使用 `sys_pause` 断点。这说明评分并非只观察“看起来停了一会儿”，还验证程序确实调用了要求的系统调用。

因此，以下错误替代方案不会满足实验目的：

```c
// 错误：用户态忙等，未调用 pause
for(volatile int i = 0; i < 100000000; i++)
  ;
```

它会占用 CPU，暂停时间依赖处理速度，也不会触发 `sys_pause`。

### 18.2 手动使用 GDB 观察 `sys_pause`

终端一启动等待调试的 xv6：

```sh
cd /home/wangxin/xv6-labs-2025
make qemu-gdb
```

另一个终端连接 RISC-V GDB。具体可执行文件名取决于本机工具链，常见命令为：

```sh
riscv64-unknown-elf-gdb kernel/kernel
```

进入 GDB 后：

```gdb
target remote localhost:PORT
break sys_pause
continue
```

`PORT` 可用下面的命令查询：

```sh
make print-gdbport
```

回到 xv6 控制台输入：

```sh
sleep 10
```

断点命中后，可以查看：

```gdb
list
print ticks
print n
print *myproc()
```

刚进入 `sys_pause()` 时，`n` 可能尚未完成 `argint` 赋值；可以用 `next` 执行到对应代码之后再打印。

### 18.3 使用 Ctrl-P 查看进程状态

xv6 控制台的 `Ctrl-P` 会调用 `procdump()` 打印进程表。为了更容易观察 `sleep` 的阻塞状态，可运行较长时间：

```sh
sleep 100
```

在等待期间按 `Ctrl-P`，可能看到 `sleep` 进程显示为：

```text
... sleep  sleep
```

其中一处 `sleep` 是状态文本，一处是进程名。具体输出格式由 `procdump()` 决定。由于进程会在每个时钟 tick 被唤醒并再次检查条件，采样时也可能短暂看到 `runble` 或 `run`，这属于正常竞态现象。

### 18.4 临时内核日志法

若课程允许调试，可临时在 `sys_pause()` 中加入：

```c
printf("pid=%d start=%d now=%d target=%d\n",
       myproc()->pid, ticks0, ticks, n);
```

或者在循环被唤醒后打印当前 ticks。完成观察后应删除调试输出，因为频繁的内核打印会改变时序、污染评分输出，并显著降低系统运行速度。

## 19. 实验结果如何分析

可以用 `uptime` 系统调用编写一个临时测试程序，测量 `pause` 前后的 tick 差值：

```c
int before;
int after;

before = uptime();
pause(10);
after = uptime();
printf("elapsed ticks: %d\n", after - before);
```

理论上输出应满足：

```text
elapsed ticks >= 10
```

不应强制要求严格等于 10，原因是：

- 唤醒只把进程设为 `RUNNABLE`；
- 进程还要等待调度器选择；
- 系统中可能有其他可运行进程；
- 多核调度和控制台输出也会带来延迟。

这体现了 `pause(10)` 的语义是“至少等待 10 个 tick 后才有机会返回”，并不是实时系统意义上的精确截止时间保证。

不要把上述测量输出直接加进正式 `sleep.c`，否则会改变命令的标准输出，并可能影响自动评分。它更适合作为独立的临时实验程序。

## 20. 可直接写入实验报告的设计说明

### 20.1 实验目的

实现 xv6 用户命令 `sleep ticks`，掌握用户程序的命令行参数解析、用户态库函数 `atoi`、系统调用接口 `pause`，并理解进程阻塞、时钟中断、唤醒和重新调度的过程。

### 20.2 设计思路

程序首先检查 `argc`，确保用户只提供一个暂停时长参数。随后检查 `argv[1]` 是否为合法的非负十进制整数，并使用 `atoi` 转换为 `int`。转换成功后调用 `pause(ticks)` 进入内核阻塞等待，系统调用返回后通过 `exit(0)` 正常结束。

内核 `sys_pause()` 保存初始 `ticks`，在经过时间不足时调用内核 `sleep(&ticks, &tickslock)`。该操作将当前进程状态设为 `SLEEPING` 并切换到调度器。CPU 0 的时钟中断处理函数递增全局 `ticks`，随后调用 `wakeup(&ticks)`，把对应等待通道上的进程从 `SLEEPING` 修改为 `RUNNABLE`。调度器选中该进程后，它重新检查经过的 ticks；达到目标后系统调用返回。

### 20.3 关键数据结构与状态

| 对象 | 作用 |
|---|---|
| `argc/argv` | 接收命令名和 ticks 字符串 |
| `atoi` | 将合法数字字符串转换为整数 |
| `ticks` | 全局时钟中断计数器 |
| `tickslock` | 保护 ticks 并同步条件检查与睡眠 |
| `p->chan` | 记录进程等待的睡眠通道，本实验为 `&ticks` |
| `p->state` | 保存 `RUNNING`、`SLEEPING`、`RUNNABLE` 等状态 |
| `SYS_pause` | 用户态到内核态的系统调用编号 |

### 20.4 核心状态转换

```text
用户进程 RUNNING
  → 调用 pause
  → sys_pause 检查时间未到
  → 内核 sleep 将其置为 SLEEPING
  → 时钟中断 ticks++
  → wakeup 将其置为 RUNNABLE
  → scheduler 将其置为 RUNNING
  → 再次检查时间
  → 达到指定 ticks 后返回用户态
```

### 20.5 实验结论

本实验中的暂停不是用户态忙等，而是借助内核阻塞机制释放 CPU。时钟中断只负责推进时间并使等待进程重新具备运行资格，真正何时继续运行取决于调度器。因此，`pause(n)` 保证至少等待指定数量的 ticks，但不保证在第 n 个 tick 到来时立即返回。该实现同时展示了条件锁、睡眠通道和循环检查在避免丢失唤醒及处理重复唤醒方面的作用。

## 21. 选择基础版还是增强版

如果目标是严格完成 util 实验并保持代码最短，使用第 3 节的基础版即可：

```c
ticks = atoi(argv[1]);
pause(ticks);
```

如果老师允许更完整的错误处理，建议使用第 13 节增强版。两种实现的核心系统调用路径完全相同，增强版只在进入内核前增加用户输入验证。

无论选择哪一版，最重要的判断标准都是：

1. `sleep 10` 必须调用 `pause(10)`；
2. 不得用用户态循环模拟等待；
3. 程序必须加入 `UPROGS`；
4. 无参数执行必须安全退出；
5. 能解释 `SLEEPING → RUNNABLE → RUNNING` 的来源，而不是只会写三行用户代码。
