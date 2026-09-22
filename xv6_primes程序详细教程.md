# xv6 `primes` 程序详细教程：使用管道与进程实现 McIlroy 素数筛

> 源码目录：`/home/wangxin/xv6-labs-2025`  
> 实验目标：使用 `pipe`、`fork`、`read`、`write`、`close`、`wait` 和 `exit`，找出 2～40（含端点）之间的素数。  
> 本文只提供操作教程，不直接修改 xv6 源码。

## 1. 实验目标与预期输出

在 xv6 shell 中执行：

```sh
primes
```

预期输出：

```text
prime 2
prime 3
prime 5
prime 7
prime 11
prime 13
prime 17
prime 19
prime 23
prime 29
prime 31
prime 37
```

本实验不使用一个进程中的普通数组筛法，而是建立一条进程流水线：

```text
生成器 → 过滤 2 → 过滤 3 → 过滤 5 → 过滤 7 → …… → 过滤 37
```

每个筛选进程：

1. 从上游管道读取第一个整数；
2. 把这个整数当作本进程负责的素数并打印；
3. 创建下游管道和下游进程；
4. 继续读取上游整数；
5. 丢弃能被当前素数整除的整数；
6. 把其余整数写入下游管道；
7. 上游结束后关闭下游写端；
8. 等待直接子进程退出，然后自己退出。

这就是 McIlroy 用并发进程和管道表达的素数筛。

## 2. 先理解算法，而不是先写代码

主进程向第一条管道写入：

```text
2 3 4 5 6 7 8 9 10 ... 40
```

第一个筛选进程读到的第一个数是 `2`，所以它负责过滤 2 的倍数：

```text
输入：2 3 4 5 6 7 8 9 10 11 ... 40
素数：2
输出：3 5 7 9 11 13 15 17 ... 39
```

下一个进程读到的第一个数是 `3`，所以它负责过滤 3 的倍数：

```text
输入：3 5 7 9 11 13 15 17 19 21 23 25 27 29 31 33 35 37 39
素数：3
输出：5 7 11 13 17 19 23 25 29 31 35 37
```

再下一个进程读到 `5`：

```text
输入：5 7 11 13 17 19 23 25 29 31 35 37
素数：5
输出：7 11 13 17 19 23 29 31 37
```

依此类推，最终输出所有素数。

### 2.1 为什么每个进程读到的第一个数一定是素数

以某个进程读到的第一个数 `p` 为例。这个数已经经过前面所有素数进程的过滤，因此它不能被任何小于它的素数整除。

如果 `p` 是合数，那么它必然存在一个小于 `p` 的素因子。这个素因子对应的上游进程早已把 `p` 过滤掉，与“当前进程读到了 p”矛盾。因此，当前进程读到的第一个数一定是素数。

这是一种归纳式保证：

```text
第一个数 2 是素数
  ↓
过滤掉所有 2 的倍数
  ↓
剩余序列的第一个数 3 是素数
  ↓
过滤掉所有 3 的倍数
  ↓
剩余序列的第一个数 5 是素数
  ↓
……
```

## 3. 当前仓库需要手动修改的两个位置

进入源码目录：

```sh
cd /home/wangxin/xv6-labs-2025
```

需要手动完成：

1. 新建 `user/primes.c`；
2. 在 `Makefile` 的 `UPROGS` 中加入 `$U/_primes`。

不需要添加新的系统调用。本实验使用的接口已经在 `user/user.h` 中声明：

```c
int fork(void);
int exit(int) __attribute__((noreturn));
int wait(int*);
int pipe(int*);
int write(int, const void*, int);
int read(int, void*, int);
int close(int);
```

## 4. 推荐实现：结构清楚且适合实验提交

新建文件：

```sh
vim user/primes.c
```

写入以下完整代码：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

// 输出错误消息并终止当前进程。
static void
fail(const char *message)
{
  fprintf(2, "primes: %s\n", message);
  exit(1);
}

// 从 input_fd 接收一串整数，筛掉当前素数的倍数，
// 再通过新管道把其他整数传递给下一层筛选进程。
static void
sieve(int input_fd)
{
  int prime;
  int number;
  int next_pipe[2];
  int pid;
  int n;

  // 用循环让“下游子进程”进入下一层筛选，避免 C 函数直接递归。
  // 进程链仍然由每一层的 fork 建立。
  for(;;){
    // 上游关闭写端且管道中已没有数据时，read 返回 0。
    // 这表示整个筛选链已经结束。
    n = read(input_fd, &prime, sizeof(prime));
    if(n == 0){
      close(input_fd);
      exit(0);
    }
    if(n != (int)sizeof(prime))
      fail("cannot read prime");

    printf("prime %d\n", prime);

    if(pipe(next_pipe) < 0)
      fail("pipe failed");

    pid = fork();
    if(pid < 0)
      fail("fork failed");

    if(pid == 0){
      // 子进程只读取本层新建管道的读端。
      // 它不再需要父进程的上游读端，也不需要新管道写端。
      close(input_fd);
      close(next_pipe[1]);
      input_fd = next_pipe[0];
      continue;
    }

    // 当前筛选进程只向新管道写，不从新管道读。
    close(next_pipe[0]);

    while((n = read(input_fd, &number, sizeof(number))) > 0){
      if(n != (int)sizeof(number))
        fail("incomplete integer from pipe");

      if(number % prime != 0){
        if(write(next_pipe[1], &number, sizeof(number)) != (int)sizeof(number))
          fail("write failed");
      }
    }

    if(n < 0)
      fail("read failed");

    close(input_fd);

    // 必须先关闭写端，再 wait。
    // 关闭最后一个写端后，下游 read 才能返回 0 并退出。
    close(next_pipe[1]);
    wait(0);
    exit(0);
  }
}

int
main(int argc, char *argv[])
{
  int first_pipe[2];
  int pid;
  int number;

  if(argc != 1){
    fprintf(2, "usage: primes\n");
    exit(1);
  }

  if(pipe(first_pipe) < 0)
    fail("pipe failed");

  pid = fork();
  if(pid < 0)
    fail("fork failed");

  if(pid == 0){
    // 第一个筛选进程只读取第一条管道。
    close(first_pipe[1]);
    sieve(first_pipe[0]);
    exit(0);
  }

  // 生成器进程只写第一条管道。
  close(first_pipe[0]);

  for(number = 2; number <= 40; number++){
    if(write(first_pipe[1], &number, sizeof(number)) != (int)sizeof(number))
      fail("write failed");
  }

  // 表示不会再产生数据，使第一个筛选进程最终能看到 EOF。
  close(first_pipe[1]);

  wait(0);
  exit(0);
}
```

这份实现比最小版本多了错误检查，但仍保持 McIlroy 筛法的核心结构清晰。

## 5. 修改 Makefile

打开：

```sh
vim Makefile
```

当前仓库的 `UPROGS` 末尾已经包含 `_rwcopy` 和 `_sleep`：

```make
	$U/_rwcopy\
	$U/_sleep
```

把它改为：

```make
	$U/_rwcopy\
	$U/_sleep\
	$U/_primes
```

注意：

- `_sleep` 后面必须增加续行反斜杠 `\`；
- `_primes` 前面的下划线属于 xv6 的构建命名约定；
- 最后一项可以不写反斜杠；
- 不要删除已有的 `_rwcopy` 和 `_sleep`；
- 保持 Makefile 原有 Tab 缩进和排版。

构建系统会把：

```text
user/primes.c
      ↓ 编译
user/primes.o
      ↓ 与用户库链接
user/_primes
      ↓ mkfs 写入文件系统镜像
fs.img 中的 primes 命令
```

## 6. 编译与运行

### 6.1 定向编译

先只编译该用户程序：

```sh
cd /home/wangxin/xv6-labs-2025
make user/_primes
```

成功后通常会生成：

```text
user/primes.o
user/_primes
user/primes.asm
user/primes.sym
```

### 6.2 启动 xv6

```sh
make qemu
```

在 xv6 shell 中执行：

```sh
primes
```

核对完整输出：

```text
prime 2
prime 3
prime 5
prime 7
prime 11
prime 13
prime 17
prime 19
prime 23
prime 29
prime 31
prime 37
```

还应确认命令返回 shell，而不是打印完成后卡住：

```sh
primes
echo returned
```

最后一行应出现：

```text
returned
```

退出 QEMU：先按 `Ctrl-a`，松开后再按 `x`。

### 6.3 关于 `make grade`

当前仓库的定制 `grade-lab-util` 没有搜索到 `primes` 测试项。因此：

```sh
make grade
```

即使全部通过，也不能证明 `primes` 正确。应以手动输出、能否正常返回、文件描述符关闭逻辑和进程链分析作为本题验证依据。如果老师另有评分脚本，以老师的脚本为准。

## 7. 主进程逐行分析

### 7.1 参数检查

```c
if(argc != 1){
  fprintf(2, "usage: primes\n");
  exit(1);
}
```

本程序固定筛选 2～40，不需要用户提供参数。`argc` 应为 1，`argv[0]` 是程序名 `primes`。

### 7.2 创建第一条管道

```c
int first_pipe[2];
pipe(first_pipe);
```

约定：

```text
first_pipe[0]：读端
first_pipe[1]：写端
```

不要把下标含义写反。

### 7.3 fork 后文件描述符会被复制

```c
pid = fork();
```

`fork` 后，父子进程都暂时拥有：

| 进程 | `first_pipe[0]` | `first_pipe[1]` |
|---|---:|---:|
| 父进程（生成器） | 打开 | 打开 |
| 子进程（第一个筛选器） | 打开 | 打开 |

必须立即关闭不用的端点：

| 进程 | 保留 | 关闭 | 原因 |
|---|---|---|---|
| 父进程 | `first_pipe[1]` | `first_pipe[0]` | 只产生数据 |
| 子进程 | `first_pipe[0]` | `first_pipe[1]` | 只消费数据 |

### 7.4 为什么写的是二进制整数

```c
write(first_pipe[1], &number, sizeof(number));
```

这里不是写字符串 `"10"`，而是写一个 `int` 的原始字节。接收端也必须用相同类型和大小：

```c
read(input_fd, &number, sizeof(number));
```

发送端和接收端的数据协议是：每 4 字节表示一个整数（在该构建环境中 `sizeof(int)` 通常为 4）。用 `sizeof(number)` 而不是硬编码 `4` 更清楚。

### 7.5 为什么范围条件是 `<= 40`

题目要求 2～40，含 2 和 40：

```c
for(number = 2; number <= 40; number++)
```

虽然 40 不是素数，但仍应发送给流水线，由负责素数 2 的进程过滤。这样实现的是通用筛法，而不是主进程预先判断数字。

### 7.6 主进程为何必须关闭写端

```c
close(first_pipe[1]);
```

关闭写端具有协议意义：它表示“数据流结束”。只有所有指向该管道写端的文件引用都关闭，而且缓冲区已读空，下游的 `read` 才返回 0。

如果忘记关闭，筛选进程会一直等待更多数字，整个程序无法退出。

### 7.7 为什么最后要 `wait(0)`

```c
wait(0);
```

主进程等待第一个筛选进程。第一个筛选进程又等待第二个筛选进程，以此类推，形成逐层回收：

```text
生成器 wait P2
P2 wait P3
P3 wait P5
P5 wait P7
...
```

这避免产生未回收的僵尸子进程，也确保 shell 在整条筛选链结束后才重新显示提示符。

## 8. `sieve()` 进程链函数分析

### 8.1 使用循环进入下一层，但每一层仍是独立进程

早期 xv6 教程常在子进程分支中直接写：

```c
sieve(next_pipe[0]);
```

算法上没有问题，因为调用发生在 `fork()` 产生的子进程中；但是当前工具链启用了 `-Werror`，较新的 GCC 会把它诊断为 `-Winfinite-recursion`，导致编译失败。

本教程改用：

```c
input_fd = next_pipe[0];
continue;
```

子进程更新自己的输入描述符，然后进入循环下一轮。每一层筛选器仍然运行在独立进程里：

```text
进程 P2 运行 sieve，prime=2
进程 P3 运行 sieve，prime=3
进程 P5 运行 sieve，prime=5
进程 P7 运行 sieve，prime=7
...
```

它们具有独立用户地址空间和独立的局部变量。连接它们的不是共享数组，而是内核管道。

### 8.2 读取第一个数

```c
n = read(input_fd, &prime, sizeof(prime));
```

可能出现三种结果：

| 返回值 | 含义 | 处理方式 |
|---:|---|---|
| `sizeof(prime)` | 成功读取一个整数 | 打印并建立下一层 |
| `0` | 上游写端全部关闭，且数据已读完 | 本层直接退出 |
| `< 0` | 读取错误 | 输出错误并退出 |

推荐代码还把其他正数视作“不完整整数”，因为管道协议规定每条消息应是完整 `int`。

### 8.3 打印素数的时机

```c
printf("prime %d\n", prime);
```

必须先成功读取第一个数再打印。若一进入 `sieve` 就创建下一层，会无谓产生进程；若没有数据仍打印，则会使用未初始化变量。

### 8.4 当前层创建下游

```c
pipe(next_pipe);
pid = fork();
```

此时 fork 之前，当前进程持有：

```text
input_fd      上游管道读端
next_pipe[0]  下游管道读端
next_pipe[1]  下游管道写端
```

fork 后父子双方都继承三者，所以必须分别关闭。

当前筛选进程需要：

```text
input_fd      从上游读取
next_pipe[1]  向下游写入
```

下游子进程需要：

```text
next_pipe[0]  从本层读取
```

所有权表：

| 文件描述符 | 当前筛选进程 | 下游子进程 |
|---|---|---|
| `input_fd` | 保留 | 关闭 |
| `next_pipe[0]` | 关闭 | 保留 |
| `next_pipe[1]` | 保留 | 关闭 |

### 8.5 过滤规则

```c
if(number % prime != 0)
  write(next_pipe[1], &number, sizeof(number));
```

- 余数为 0：是当前素数的倍数，丢弃；
- 余数不为 0：无法被当前素数整除，传给下游。

例如当前进程 `prime == 5`：

```text
7  → 7 % 5 != 0 → 传递
11 → 11 % 5 != 0 → 传递
25 → 25 % 5 == 0 → 丢弃
35 → 35 % 5 == 0 → 丢弃
37 → 37 % 5 != 0 → 传递
```

### 8.6 为什么必须先 close 再 wait

正确顺序：

```c
close(next_pipe[1]);
wait(0);
```

错误顺序：

```c
wait(0);
close(next_pipe[1]);
```

错误顺序会导致死锁：

```text
父进程：等待子进程退出
子进程：read 等待父进程关闭写端或继续写数据
父进程：只有 wait 返回后才关闭写端
```

两边互相等待，永远无法继续。

正确顺序表达：

```text
父进程关闭写端，发出 EOF
        ↓
子进程 read 返回 0
        ↓
子进程退出
        ↓
父进程 wait 返回
```

## 9. 完整进程链与数据流

对 2～40，最终打印的素数有：

```text
2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37
```

逻辑进程链为：

```text
Generator
  │ 2..40
  ▼
P2  ──过滤 2 的倍数──▶
  ▼
P3  ──过滤 3 的倍数──▶
  ▼
P5  ──过滤 5 的倍数──▶
  ▼
P7  ──过滤 7 的倍数──▶
  ▼
P11 ──过滤 11 的倍数─▶
  ▼
P13
  ▼
P17
  ▼
P19
  ▼
P23
  ▼
P29
  ▼
P31
  ▼
P37
  ▼
终止进程读取 EOF 后退出
```

推荐实现中，每个打印素数的进程都会预先建立一个下游进程。因此 P37 后还有一个不打印任何内容的终止进程；它第一次 `read` 就得到 0，然后退出。这不会影响算法结果，也是经典 xv6 primes 写法的常见结构。

### 9.1 为什么输出顺序稳定

虽然各进程并发运行，素数输出仍按升序出现。P3 必须先从 P2 收到数字 3，才可能打印 3；P2 在传递后续数据前已经打印 2。同理，每个下游进程只能在上游输出自己的素数并传来数据后启动有效筛选。

这种由管道数据依赖建立的先后关系，使输出具有自然顺序，而不需要额外排序。

## 10. xv6 内核中的管道语义

当前 `kernel/pipe.c` 定义：

```c
#define PIPESIZE 512

struct pipe {
  struct spinlock lock;
  char data[PIPESIZE];
  uint nread;
  uint nwrite;
  int readopen;
  int writeopen;
};
```

### 10.1 空管道不一定等于 EOF

`piperead()` 的等待条件是：

```c
while(pi->nread == pi->nwrite && pi->writeopen){
  sleep(&pi->nread, &pi->lock);
}
```

如果缓冲区为空但仍有写端打开，读进程进入睡眠，等待未来数据。

只有满足：

```text
缓冲区为空
并且
writeopen == 0
```

`read` 才会返回 0。因此，EOF 是“没有数据且不会再有数据”，不是简单的“当前暂时没数据”。

### 10.2 管道满时写进程会阻塞

`pipewrite()` 检查：

```c
if(pi->nwrite == pi->nread + PIPESIZE){
  wakeup(&pi->nread);
  sleep(&pi->nwrite, &pi->lock);
}
```

当 512 字节缓冲区写满时，写进程睡眠，直到读进程消费数据并唤醒它。这提供了背压：快速上游不会无限制占用内存，而会被慢速下游自然限制。

### 10.3 管道是字节流

内核管道不理解“整数”概念，只保存字节。整数边界由用户程序约定：

```text
发送端：每次写 sizeof(int) 字节
接收端：每次按 sizeof(int) 字节读取
```

因为本实验所有进程都来自同一个程序、运行在同一体系结构中，所以整数大小和字节序一致。

## 11. fork 为什么会影响 EOF

`kernel/proc.c` 的 `kfork()` 会对父进程每个已打开文件执行：

```c
np->ofile[i] = filedup(p->ofile[i]);
```

`filedup()` 增加文件对象引用计数。`close(fd)` 只会减少当前引用；只有所有写端引用都被关闭后，管道的 `writeopen` 才最终变为 0。

例如，生成器创建管道后 fork：

```text
生成器持有写端引用 ─┐
                     ├─ 同一个管道写端
子进程持有写端引用 ─┘
```

即使生成器关闭写端，只要子进程忘记关闭其继承的写端，内核仍认为管道可能继续写入。子进程自己在读这个管道时就永远看不到 EOF。

因此：

```c
if(pid == 0){
  close(first_pipe[1]);
  sieve(first_pipe[0]);
}
```

不是资源洁癖，而是保证程序正确终止的必要操作。

## 12. 更严谨的增强版：处理短读和短写

在当前 xv6 的这类小整数管道场景中，基础版通常会一次读写完整 `int`。如果希望把字节流协议写得更健壮，可以封装 `read_int()` 和 `write_int()`，循环到完整传输一个整数。

完整增强版 `user/primes.c`：

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

#define FIRST_NUMBER 2
#define LAST_NUMBER  40

enum read_result {
  READ_ERROR = -1,
  READ_END = 0,
  READ_VALUE = 1
};

static void
report_error(const char *message)
{
  fprintf(2, "primes: %s\n", message);
}

// 返回 READ_VALUE、READ_END 或 READ_ERROR。
static int
read_int(int fd, int *value)
{
  int received;
  int n;
  int size;
  char *bytes;

  received = 0;
  size = sizeof(*value);
  bytes = (char *)value;

  while(received < size){
    n = read(fd, bytes + received, size - received);

    if(n < 0)
      return READ_ERROR;

    if(n == 0){
      if(received == 0)
        return READ_END;

      // 在一个 int 的中间遇到 EOF，说明数据协议损坏。
      return READ_ERROR;
    }

    received += n;
  }

  return READ_VALUE;
}

// 成功返回 0，失败返回 -1。
static int
write_int(int fd, int value)
{
  int sent;
  int n;
  int size;
  char *bytes;

  sent = 0;
  size = sizeof(value);
  bytes = (char *)&value;

  while(sent < size){
    n = write(fd, bytes + sent, size - sent);
    if(n <= 0)
      return -1;
    sent += n;
  }

  return 0;
}

static void
sieve(int input_fd)
{
  int prime;
  int number;
  int next_pipe[2];
  int pid;
  int result;
  int status;
  int failed;

  for(;;){
    result = read_int(input_fd, &prime);
    if(result == READ_END){
      close(input_fd);
      exit(0);
    }
    if(result == READ_ERROR){
      close(input_fd);
      report_error("cannot read prime");
      exit(1);
    }

    printf("prime %d\n", prime);

    if(pipe(next_pipe) < 0){
      close(input_fd);
      report_error("pipe failed");
      exit(1);
    }

    pid = fork();
    if(pid < 0){
      close(input_fd);
      close(next_pipe[0]);
      close(next_pipe[1]);
      report_error("fork failed");
      exit(1);
    }

    if(pid == 0){
      close(input_fd);
      close(next_pipe[1]);
      input_fd = next_pipe[0];
      continue;
    }

    close(next_pipe[0]);
    failed = 0;

    while((result = read_int(input_fd, &number)) == READ_VALUE){
      if(number % prime != 0){
        if(write_int(next_pipe[1], number) < 0){
          report_error("write failed");
          failed = 1;
          break;
        }
      }
    }

    if(result == READ_ERROR){
      report_error("read failed");
      failed = 1;
    }

    close(input_fd);
    close(next_pipe[1]);

    if(wait(&status) < 0){
      report_error("wait failed");
      failed = 1;
    } else if(status != 0){
      failed = 1;
    }

    if(failed)
      exit(1);
    exit(0);
  }
}

int
main(int argc, char *argv[])
{
  int first_pipe[2];
  int pid;
  int number;
  int status;
  int failed;

  if(argc != 1){
    fprintf(2, "usage: primes\n");
    exit(1);
  }

  if(pipe(first_pipe) < 0){
    report_error("pipe failed");
    exit(1);
  }

  pid = fork();
  if(pid < 0){
    close(first_pipe[0]);
    close(first_pipe[1]);
    report_error("fork failed");
    exit(1);
  }

  if(pid == 0){
    close(first_pipe[1]);
    sieve(first_pipe[0]);
    exit(0);
  }

  close(first_pipe[0]);
  failed = 0;

  for(number = FIRST_NUMBER; number <= LAST_NUMBER; number++){
    if(write_int(first_pipe[1], number) < 0){
      report_error("write failed");
      failed = 1;
      break;
    }
  }

  close(first_pipe[1]);

  if(wait(&status) < 0){
    report_error("wait failed");
    failed = 1;
  } else if(status != 0){
    failed = 1;
  }

  if(failed)
    exit(1);
  exit(0);
}
```

### 12.1 增强版增加了什么

- 用 `FIRST_NUMBER` 和 `LAST_NUMBER` 表达边界；
- `read_int` 明确区分正常整数、EOF 和错误；
- `write_int` 处理可能的短写；
- fork 失败时主动关闭已创建的描述符；
- 用 `wait(&status)` 获取并传播下游失败状态；
- 每层保证先关闭管道，再等待子进程；
- 下游子进程通过更新 `input_fd` 并 `continue` 进入下一层，避免新版 GCC 的无限递归告警。

### 12.2 基础版与增强版如何选择

| 版本 | 优点 | 适用场景 |
|---|---|---|
| 第 4 节推荐版 | 简洁、核心结构直观 | 普通课程实验提交 |
| 第 12 节增强版 | 错误处理和数据协议更完整 | 老师允许更复杂实现、实验报告展示 |

如果评分代码重视经典结构，推荐优先提交第 4 节版本；增强版主要用于深入理解系统编程中的异常路径。

## 13. 并发行为分析

### 13.1 流水线会并发运行

生成器不需要等全部数字写完才启动筛选。可能的执行交错为：

```text
生成器写 2
P2 读到 2，打印 prime 2
生成器写 3
P2 把 3 传给 P3
P3 打印 prime 3
生成器继续写 4、5、6……
P2、P3 同时处理不同阶段的数据
```

进程的具体调度顺序不确定，但管道保持字节流顺序，所以数字不会在同一条管道中乱序。

### 13.2 阻塞就是同步机制

本程序没有用户态锁，也没有忙等循环。同步来自管道：

- 管道为空但仍有写端：`read` 阻塞；
- 管道已满：`write` 阻塞；
- 上游关闭全部写端：下游 `read` 返回 0；
- 父进程执行 `wait`：等待直接子进程退出。

这四种行为共同构成进程流水线的同步协议。

### 13.3 进程数量

2～40 共有 12 个素数：

```text
2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37
```

经典实现通常包含：

- 1 个生成器主进程；
- 12 个打印素数的筛选进程；
- 1 个读到最终 EOF 后退出的终止进程。

总计最多约 14 个本程序进程并存，再加 shell、init 等系统进程。当前非 FS 实验配置的 `NPROC` 是 64，因此 2～40 不会触及进程表上限。

### 13.4 每个进程的文件描述符数量

当前 `NOFILE` 是 16。每个筛选进程在创建下一层前后只需要少量描述符：标准输入、输出、错误，以及一条上游读端和一条下游管道的端点。

及时关闭不用的端点不仅决定 EOF，也可防止长进程链耗尽每进程或全局文件表资源。

## 14. 常见错误与死锁排查

### 错误 0：`infinite recursion detected [-Werror=infinite-recursion]`

如果把下游子进程写成：

```c
if(pid == 0){
  close(input_fd);
  close(next_pipe[1]);
  sieve(next_pipe[0]);
  exit(0);
}
```

较新的 GCC 会发现 `sieve()` 的直接自调用，并报：

```text
error: infinite recursion detected [-Werror=infinite-recursion]
```

因为 xv6 使用 `-Werror`，警告会导致构建失败。不要关闭 `-Werror`，也不要给编译器增加忽略该警告的选项；把子进程分支改成循环进入下一层：

```c
if(pid == 0){
  close(input_fd);
  close(next_pipe[1]);
  input_fd = next_pipe[0];
  continue;
}
```

同时确保 `sieve()` 的主体位于：

```c
for(;;){
  // 每一层的读取、pipe、fork 和过滤逻辑
}
```

这不会把多个筛选层合并成一个进程：执行 `continue` 的只有刚刚 fork 出来的子进程，父进程仍负责当前素数并最终退出。因此进程链模型不变，只是消除了 C 语言层面的直接递归。

### 错误 1：把管道端点写反

错误：

```c
write(p[0], &n, sizeof(n));
read(p[1], &n, sizeof(n));
```

正确：

```c
read(p[0], &n, sizeof(n));
write(p[1], &n, sizeof(n));
```

记忆方式：下标 0 对应输入，1 对应输出。

### 错误 2：在主进程中忘记关闭读端

```c
close(first_pipe[0]);
```

虽然这通常不直接阻止 EOF，但会泄漏描述符并破坏清晰的端点所有权。

### 错误 3：第一个子进程忘记关闭写端

这是最典型的卡死原因：

```c
if(pid == 0){
  // 缺少 close(first_pipe[1]);
  sieve(first_pipe[0]);
}
```

子进程仍持有它正在读取的管道写端，因此即使生成器关闭写端，内核仍认为未来可能有数据，`read` 不返回 0。

### 错误 4：下游子进程没有关闭 `next_pipe[1]`

下游进程会继承新管道写端。如果不关闭，它读取同一管道时同样无法收到 EOF。

### 错误 5：父进程先 wait 后 close

```c
wait(0);
close(next_pipe[1]);
```

会形成父等子、子等 EOF 的循环等待。必须交换顺序。

### 错误 6：把整数当字符串传输

发送端若写：

```c
write(fd, "10", 2);
```

而接收端用 `int` 读取，协议不一致，得到的数值不是 10。两端必须统一使用二进制 `int`，或者都使用文本解析；本实验应使用二进制整数。

### 错误 7：每个整数都 fork 一个进程

题目要求每个进程负责过滤一个素数，而不是每个候选数创建一个进程。fork 应发生在确定本层素数并准备下游筛选器之后。

### 错误 8：只创建一个子进程完成所有过滤

单进程数组筛法可以找出素数，但没有形成 McIlroy 进程链，未满足并发模型要求。

### 错误 9：范围写成 `< 40`

题目是 2～40，应写：

```c
number <= 40
```

40 虽然会被过滤，但它仍属于输入范围。

### 错误 10：忘记把程序加入 `UPROGS`

若 QEMU 中显示：

```text
exec primes failed
```

通常不是 C 代码问题，而是 `Makefile` 没有 `$U/_primes`，程序没有进入 `fs.img`。

### 错误 11：在宿主 Linux 上直接运行 `user/_primes`

它是 RISC-V xv6 用户程序，不能当作普通宿主机程序执行。应运行 `make qemu`，再在 xv6 shell 输入 `primes`。

### 错误 12：遗漏 `wait`

程序可能仍能打印正确结果，但父进程会提前退出，子进程可能被重新托管，输出与 shell 提示符可能交错，也无法体现规范的子进程回收。

## 15. 调试方法

### 15.1 用阶段日志检查数据流

可临时在筛选函数中加入：

```c
fprintf(2, "pid %d filters prime %d\n", getpid(), prime);
```

在写入下游前加入：

```c
fprintf(2, "pid %d passes %d\n", getpid(), number);
```

这样能观察哪个进程负责哪个素数，以及每个数如何向后流动。调试完成后删除日志，以免污染要求的标准输出。

### 15.2 用 Ctrl-P 查看进程链

程序对 2～40 执行很快，不容易手动捕获。调试时可以临时扩大上界或在适当位置调用 `pause`，然后按 `Ctrl-P` 查看进程表。

正式提交前必须恢复上界 40，并删除人为延迟。

### 15.3 用 GDB 在系统调用处断点

启动：

```sh
make qemu-gdb
```

查询端口：

```sh
make print-gdbport
```

另一个终端使用 RISC-V GDB 连接后，可设置：

```gdb
break sys_pipe
break sys_fork
break sys_read
break sys_write
break sys_close
continue
```

然后在 xv6 中执行 `primes`，观察系统调用链。断点很多时输出会非常频繁，可以一次只保留一两个断点。

### 15.4 卡死时的检查顺序

如果素数都打印出来但程序不返回，优先检查：

1. 生成器是否关闭 `first_pipe[1]`；
2. 第一个子进程是否关闭继承的 `first_pipe[1]`；
3. 每个下游子进程是否关闭 `next_pipe[1]`；
4. 每个父筛选器是否在 `wait` 前关闭 `next_pipe[1]`；
5. 是否有某个错误分支仍保留写端并进入等待。

“结果全打印了但卡住”几乎总是某个写端引用没有关闭。

## 16. 可直接写入实验报告的内容

### 16.1 实验目的

使用 xv6 的管道和进程系统调用实现 McIlroy 素数筛，理解 `fork` 后文件描述符继承、管道阻塞读写、EOF 传播、迭代式进程链及父子进程回收。

### 16.2 总体设计

主进程作为数据生成器，把 2～40 的整数依次写入第一条管道。第一个筛选进程从管道读取第一个整数 2，将其输出为素数，然后创建新管道和子进程。当前进程继续从上游读取数据，过滤掉 2 的倍数，把其余数据写给子进程。子进程更新输入管道描述符并进入下一轮循环，以相同方式处理收到的数据，形成一条逐层创建的进程流水线。

当上游完成写入后关闭管道写端，下游在消费完缓冲区后 `read` 返回 0。每层筛选器收到 EOF 后关闭自己的下游写端，使 EOF 逐层传播到链尾。父进程在关闭写端后调用 `wait` 回收直接子进程，最终整条进程链有序退出。

### 16.3 核心系统调用

| 系统调用 | 作用 |
|---|---|
| `pipe(fd)` | 创建读端 `fd[0]` 和写端 `fd[1]` |
| `fork()` | 复制当前进程及其文件描述符，创建下一层筛选器 |
| `read()` | 从上游读取二进制整数；EOF 时返回 0 |
| `write()` | 把未被当前素数整除的整数写给下游 |
| `close()` | 释放不用的端点，并在最后一个写端关闭时传播 EOF |
| `wait()` | 等待并回收直接子进程 |
| `exit()` | 结束当前筛选进程并返回状态 |

### 16.4 进程与管道关系

```text
生成器 --pipe--> 筛 2 --pipe--> 筛 3 --pipe--> 筛 5 --pipe--> ...
```

每个筛选进程同时扮演：

- 上游管道的消费者；
- 下游管道的生产者；
- 下一层筛选进程的父进程。

### 16.5 EOF 终止协议

管道为空时，若仍存在写端，`read` 会阻塞；只有最后一个写端关闭且缓冲区为空时，`read` 才返回 0。因此每个进程必须关闭所有不用的写端，父筛选器必须在 `wait` 前关闭下游写端。EOF 会从生成器开始沿进程链逐层传播，最终使全部进程退出。

### 16.6 实验结果

程序输出 2～40 范围内全部素数：

```text
2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37
```

输出顺序递增，执行结束后能正常返回 xv6 shell。实验验证了管道既用于传输数据，也通过阻塞与 EOF 提供进程同步。

### 16.7 实验结论

McIlroy 筛法把传统筛选算法映射为 Unix 进程流水线。每个进程只负责一种局部规则，即过滤一个素数的倍数；复杂的整体计算通过多个简单进程及管道连接自然形成。正确关闭文件描述符是该程序的关键，因为 fork 会复制管道引用，任何遗留写端都可能阻止 EOF 并导致死锁。

## 17. 最终自检清单

- [ ] 已创建 `user/primes.c`；
- [ ] 包含 `kernel/types.h`、`kernel/stat.h` 和 `user/user.h`；
- [ ] 主进程使用 `pipe` 和 `fork` 创建第一个筛选器；
- [ ] 主进程发送 2～40，循环条件为 `<= 40`；
- [ ] 管道传输的是完整二进制 `int`；
- [ ] 每层把收到的第一个数打印为 `prime N`；
- [ ] 每层过滤 `number % prime == 0` 的数；
- [ ] 每层使用新管道和 fork 创建下一层；
- [ ] 所有进程都关闭不用的读端和写端；
- [ ] 所有父进程都在 `wait` 前关闭下游写端；
- [ ] 上游 EOF 能逐层传播；
- [ ] 每个父进程调用 `wait` 回收直接子进程；
- [ ] `Makefile` 已加入 `$U/_primes`；
- [ ] 原有 `_rwcopy`、`_sleep` 均保留；
- [ ] `make user/_primes` 编译成功；
- [ ] QEMU 中输出 12 个正确素数；
- [ ] 输出完成后能够返回 shell，不会卡住；
- [ ] 能解释为什么忘记关闭写端会导致 `read` 永远不返回 0。

## 18. 最终提交建议

一般实验提交建议使用第 4 节的推荐实现。它包含必要的错误检查，同时没有让辅助代码掩盖进程链的核心结构。若老师鼓励更完整的系统编程错误处理，可以采用第 12 节增强版。

本题真正的重点不是判断一个数是否为素数，而是建立并正确终止如下并发结构：

```text
pipe → fork → 关闭无用端点 → read → 过滤 → write
                                      ↓
                       close 写端 → EOF → wait → exit
```

只要能清楚说明每次 fork 后谁持有哪些管道端点、最后一个写端何时关闭、下游为何能读到 EOF，就真正掌握了这道实验。
