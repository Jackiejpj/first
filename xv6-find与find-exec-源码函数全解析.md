# xv6 `find` 与 `find -exec`：源码函数、数据结构及输入输出全解析

> 分析对象：`/home/wangxin/xv6-labs-2025` 当前源码  
> 对应教程：`xv6-find-详细教程.md`、`xv6-find-exec-详细教程.md`  
> 说明：本文只整理和解释源码，没有修改 xv6 工程。

---

## 1. 分析范围与结论

两份教程直接涉及四类函数：

1. `find` 程序自身的 `base_name`、`run_cmd`、`find`、`main`；
2. 用户库函数 `strlen`、`strcmp`、`strcpy`、`memmove`、`printf`、`fprintf`，以及教程中作为替代方案提到的 `stat`；
3. 系统调用 `open`、`fstat`、`read`、`close`、`fork`、`exec`、`wait`、`exit`，以及打印函数间接使用的 `write`；
4. 上述系统调用在内核中的处理函数和关键下层函数，例如 `sys_open`、`fileread`、`readi`、`kfork`、`kexec`、`kwait` 等。

当前工程已经包含带简化版 `-exec` 的最终实现：

```text
find path name
find path name -exec cmd
```

其行为是：

- 三个参数时，打印每个名称匹配的路径；
- 五个参数时，为每个匹配路径创建子进程，执行 `cmd path`；
- 每执行一次命令，父进程都会 `wait(0)`，之后才继续遍历；
- 这里不支持 Linux `find -exec ... {} \;` 的完整语法。

---

## 2. 源码文件总览

| 层次 | 源码文件 | 与本程序的关系 |
|---|---|---|
| 用户程序 | `user/find.c` | `find` 的全部业务逻辑 |
| 用户声明 | `user/user.h` | 用户库和系统调用原型 |
| 用户字符串库 | `user/ulib.c` | `strlen`、`strcmp`、`strcpy`、`memmove`、`stat` |
| 用户格式化输出 | `user/printf.c` | `printf`、`fprintf`、`vprintf`、`putc` |
| 系统调用桩生成器 | `user/usys.pl` | 生成用户态系统调用汇编入口 |
| 生成的调用桩 | `user/usys.S` | 设置系统调用号、执行 `ecall` |
| 系统调用编号 | `kernel/syscall.h` | `SYS_fork`、`SYS_read`、`SYS_exec` 等编号 |
| 系统调用分派 | `kernel/syscall.c` | 从 `a7` 取调用号，调用 `sys_*` |
| 文件系统调用 | `kernel/sysfile.c` | `sys_open/read/fstat/close/exec/write` |
| 文件描述符层 | `kernel/file.c` | `filealloc`、`fileread`、`filestat`、`fileclose` |
| inode/路径层 | `kernel/fs.c` | `namei`、`dirlookup`、`readi`、`stati` 等 |
| 进程调用入口 | `kernel/sysproc.c` | `sys_fork`、`sys_wait`、`sys_exit` |
| 进程实现 | `kernel/proc.c` | `kfork`、`kwait`、`kexit` |
| 程序装载 | `kernel/exec.c` | `kexec`、`loadseg`、`flags2perm` |
| 文件系统格式 | `kernel/fs.h` | `DIRSIZ`、`struct dirent`、`struct dinode` |
| 用户可见元数据 | `kernel/stat.h` | `struct stat`、文件类型常量 |
| 内核文件对象 | `kernel/file.h` | `struct file`、`struct inode` |
| 进程对象 | `kernel/proc.h` | `struct proc`、`struct trapframe`、进程状态 |
| ELF 格式 | `kernel/elf.h` | `struct elfhdr`、`struct proghdr` |
| 系统限制 | `kernel/param.h` | `NOFILE`、`MAXARG`、`MAXPATH` 等 |
| 打开标志 | `kernel/fcntl.h` | `O_RDONLY` 等 |

---

## 3. `find` 程序的实际源码

源码位置：`/home/wangxin/xv6-labs-2025/user/find.c`

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "kernel/fs.h"
#include "kernel/fcntl.h"
#include "user/user.h"

static char* base_name(char *path){
    char* p;
    p = path + strlen(path);
    while(p > path && *(p-1)!='/')
        p--;
    return p;
}

static void run_cmd(char *cmd,char *path){
    int pid;
    char *args[3];

    pid = fork();
    if(pid < 0){
        fprintf(2,"find:fork failed\n");
        return;
    }
    if(pid == 0){
        args[0] = cmd;
        args[1] = path;
        args[2] = 0;

        exec(cmd,args);

        fprintf(2,"find:exec %s failed\n",cmd);
        exit(1);
    }
    wait(0);
}

static void find(char* path, char* target, char *cmd){
    char buf[512];
    char *p;
    int fd;
    struct dirent de;
    struct stat st;

    if((fd = open(path,O_RDONLY)) <0){
        fprintf(2,"find: cannot open %s\n",path);
        return;
    }

    if(fstat(fd,&st) < 0){
        fprintf(2,"find: cannot stst %s\n",path);
        close(fd);
        return;
    }

    if(strcmp(base_name(path),target) == 0){
        if(cmd == 0)
            printf("%s\n",path);
        else
            run_cmd(cmd,path);
    }

    if(st.type != T_DIR){
        close(fd);
        return;
    }

    if(strlen(path) +1+DIRSIZ+1 > sizeof(buf)){
        fprintf(2,"find:path too long\n");
        close(fd);
        return;
    }

    strcpy(buf,path);
    p = buf + strlen(buf);

    if(p == buf || *(p-1) != '/')
        *p++ = '/';

    while(read(fd,&de,sizeof(de)) == sizeof(de)){
        if(de.inum == 0){
            continue;
        }
        memmove(p,de.name,DIRSIZ);
        p[DIRSIZ] = '\0';
        if(strcmp(p,".") == 0 || strcmp(p,"..") == 0){
            continue;
        }
        find(buf,target,cmd);
    }
    close(fd);
}

int main(int argc, char* argv[]){
    char *cmd;
    if(argc != 3 && argc != 5){
        fprintf(2,"usage:find path name [-exec cmd]\n");
        exit(1);
    }
    if(argc == 5 && strcmp(argv[3], "-exec") != 0){
        fprintf(2, "usage: find path name [-exec cmd]\n");
        exit(1);
    }
    if(strlen(argv[2]) > DIRSIZ){
        fprintf(2,"find:name is too long\n");
        exit(1);
    }

    cmd = 0;
    if(argc == 5){
        cmd = argv[4];
    }
    find(argv[1],argv[2],cmd);
    exit(0);
}
```

### 源码现状说明

`fstat` 失败时的错误文本写成了：

```c
fprintf(2,"find: cannot stst %s\n",path);
```

这里的 `stst` 应是 `stat` 的拼写错误。它只影响提示文字，不影响控制流程或返回值。本文没有修改工程。

---

## 4. 程序自身四个函数

### 4.1 `base_name`

```c
static char *base_name(char *path);
```

| 项目 | 说明 |
|---|---|
| 输入 | `path`：以 `\0` 结束的路径字符串 |
| 输出 | 指向 `path` 内部最后一个路径分量的指针 |
| 失败值 | 无 |
| 副作用 | 无，不复制、不修改原字符串 |
| 功能 | 从字符串末尾向前找到最后一个 `/` |

示例：

```text
输入 path = "./a/c/b"
返回值指向          ^ b
返回字符串 = "b"
```

边界行为：

- `base_name("hello")` 返回原指针；
- `base_name("a/b")` 返回指向 `b` 的指针；
- `base_name("a/")` 返回指向结尾 `\0` 的指针，即空字符串；
- 因而路径末尾带 `/` 时，当前实现不会把前一个分量当作 basename。

### 4.2 `run_cmd`

```c
static void run_cmd(char *cmd, char *path);
```

| 项目 | 说明 |
|---|---|
| 输入 | `cmd`：可执行文件路径或名称；`path`：匹配到的路径 |
| 输出 | 无 |
| 副作用 | 创建子进程、在子进程执行程序、父进程等待 |
| 错误处理 | `fork` 失败则打印并返回；`exec` 失败则子进程 `exit(1)` |

它构造的参数数组为：

```c
args[0] = cmd;
args[1] = path;
args[2] = 0;
```

所以 `run_cmd("echo", "./a/b")` 的语义是：

```text
argc = 2
argv[0] = "echo"
argv[1] = "./a/b"
argv[2] = NULL
```

`exec` 成功后不会返回原来的子进程代码；只有失败才会执行其后的 `fprintf` 和 `exit(1)`。父进程使用 `wait(0)`，因此忽略子进程退出状态。

### 4.3 `find`

```c
static void find(char *path, char *target, char *cmd);
```

| 参数 | 输入含义 |
|---|---|
| `path` | 当前要处理的完整路径，也是本次递归的树根 |
| `target` | 要进行完全匹配的最后一个路径分量 |
| `cmd` | `0` 表示打印；非 `0` 表示执行 `cmd path` |

返回值为 `void`。其输出或副作用为：

- 名称匹配且 `cmd == 0`：向文件描述符 1 输出路径；
- 名称匹配且 `cmd != 0`：调用 `run_cmd`；
- 当前对象为目录：逐项读取并递归；
- 所有成功打开的 `fd` 都在正常或错误分支上关闭。

算法步骤：

```text
open(path)
  ├─失败：报错、返回
  └─成功
      ↓
fstat(fd, &st)
      ↓
比较 base_name(path) 与 target
      ↓
不是目录：close(fd)、返回
      ↓
逐个 read(struct dirent)
  ├─跳过 inum == 0
  ├─把固定长度名称复制到 buf 并补 '\0'
  ├─跳过 . 和 ..
  └─find(子路径, target, cmd)
      ↓
close(fd)
```

路径缓冲区检查：

```c
strlen(path) + 1 + DIRSIZ + 1 <= sizeof(buf)
```

四项分别是当前路径、`/`、最长目录名和字符串终止符。

### 4.4 `main`

```c
int main(int argc, char *argv[]);
```

| 调用形式 | `argc` | 参数解释 |
|---|---:|---|
| `find path name` | 3 | `argv[1]=path`，`argv[2]=name` |
| `find path name -exec cmd` | 5 | 再加 `argv[3]=-exec`，`argv[4]=cmd` |

输出和退出：

- 用法错误、选项错误或目标名超过 `DIRSIZ`：向 fd 2 报错并 `exit(1)`；
- 参数合法：调用 `find`，之后 `exit(0)`；
- 当前实现不汇总 `run_cmd` 中命令的退出状态。

---

## 5. 关键数据结构与常量

### 5.1 基础整数类型

源码：`kernel/types.h`

```c
typedef unsigned int   uint;
typedef unsigned short ushort;
typedef unsigned char  uchar;
typedef unsigned long  uint64;
typedef uint64 pde_t;
```

其中 `ushort` 用于目录项 inode 编号，`uint64` 广泛用于地址、文件大小和寄存器。

### 5.2 `struct dirent`：目录中的一条记录

源码：`kernel/fs.h`

```c
#define DIRSIZ 14

struct dirent {
  ushort inum;
  char name[DIRSIZ];
};
```

| 字段 | 含义 |
|---|---|
| `inum` | inode 编号；为 0 表示该目录槽无效 |
| `name` | 固定 14 字节的名称字段，不保证以 `\0` 结尾 |

目录本质上是连续保存 `struct dirent` 的文件，所以可以用 `read(fd, &de, sizeof(de))` 逐项读取。

### 5.3 `struct stat`：用户可见的文件元数据

源码：`kernel/stat.h`

```c
#define T_DIR     1
#define T_FILE    2
#define T_DEVICE  3

struct stat {
  int dev;
  uint ino;
  short type;
  short nlink;
  uint64 size;
};
```

| 字段 | 含义 |
|---|---|
| `dev` | 文件所在设备号 |
| `ino` | inode 编号 |
| `type` | `T_DIR`、`T_FILE` 或 `T_DEVICE` |
| `nlink` | 硬链接数 |
| `size` | 文件字节数；目录中是目录数据总字节数 |

`find` 实际只读取 `st.type`，但 `fstat` 会填充全部字段。

### 5.4 `struct file`：内核的打开文件对象

源码：`kernel/file.h`

```c
struct file {
  enum { FD_NONE, FD_PIPE, FD_INODE, FD_DEVICE } type;
  int ref;
  char readable;
  char writable;
  struct pipe *pipe;
  struct inode *ip;
  uint off;
  short major;
};
```

对 `find` 最重要的字段：

- `type`：对象是 inode、设备还是管道；
- `ref`：引用计数，`fork` 会增加它；
- `readable`：能否读取；
- `ip`：所对应的 inode；
- `off`：当前读取偏移。每次成功读取目录项后自动增加。

用户程序持有的是整数 `fd`，内核通过 `proc.ofile[fd]` 找到 `struct file *`。

### 5.5 `struct dinode` 与 `struct inode`

磁盘 inode，源码：`kernel/fs.h`

```c
struct dinode {
  short type;
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```

内存 inode，源码：`kernel/file.h`

```c
struct inode {
  uint dev;
  uint inum;
  int ref;
  struct sleeplock lock;
  int valid;
  short type;
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```

区别：`dinode` 是磁盘格式；`inode` 是内核缓存的活动对象，额外具有引用计数、锁和有效标志。

### 5.6 `struct proc`：进程状态

源码：`kernel/proc.h`

本实验相关字段为：

```c
enum procstate { UNUSED, USED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };

struct proc {
  struct spinlock lock;
  enum procstate state;
  void *chan;
  int killed;
  int xstate;
  int pid;
  struct proc *parent;
  uint64 sz;
  pagetable_t pagetable;
  struct trapframe *trapframe;
  struct file *ofile[NOFILE];
  struct inode *cwd;
  char name[16];
  // 其余字段省略
};
```

| 字段 | 在 `find -exec` 中的作用 |
|---|---|
| `state` | 子进程退出后进入 `ZOMBIE`，等待父进程回收 |
| `xstate` | 保存 `exit(status)` 的状态值 |
| `pid` | `fork` 在父进程中的返回值、`wait` 的返回值 |
| `parent` | `wait` 判断父子关系 |
| `pagetable`、`sz` | `fork` 复制地址空间，`exec` 替换地址空间 |
| `trapframe` | 保存系统调用参数和返回值寄存器 |
| `ofile` | 每进程文件描述符表；`fork` 时复制引用 |
| `cwd` | 相对路径解析的起点 |

### 5.7 `struct trapframe` 与系统调用寄存器

同样定义在 `kernel/proc.h`。最相关的寄存器是：

- `a0`～`a5`：最多六个系统调用参数；
- `a7`：系统调用编号；
- `a0`：系统调用完成后也用来保存返回值；
- `epc`：返回用户态时的程序计数器；
- `sp`：用户栈指针。

这解释了 `syscall.c` 中 `argraw(n)` 从 `trapframe->a0...a5` 取参数，以及分派完成后把结果写回 `a0`。

### 5.8 `argv` 参数数组

`argv` 不是结构体，而是以空指针结尾的指针数组：

```c
char *args[3] = { cmd, path, 0 };
```

最后的 `0` 必不可少。内核 `sys_exec` 会逐项读取指针，直到发现空指针；工程中 `MAXARG` 为 32。

### 5.9 ELF 结构

源码：`kernel/elf.h`

- `struct elfhdr`：ELF 文件头，含魔数、入口地址、程序头位置和数量；
- `struct proghdr`：一个可装载段的类型、权限、文件偏移、虚拟地址和大小；
- `ELF_MAGIC`：验证可执行文件格式；
- `ELF_PROG_LOAD`：表示该程序段需要装入内存。

它们由 `kexec` 使用，不由 `find` 直接操作。

### 5.10 相关限制与标志

| 常量 | 当前值 | 来源 | 含义 |
|---|---:|---|---|
| `DIRSIZ` | 14 | `kernel/fs.h` | 单个目录项名称字段长度 |
| `NOFILE` | 16 | `kernel/param.h` | 每个进程最多打开文件数 |
| `NFILE` | 100 | `kernel/param.h` | 全系统打开文件对象数 |
| `MAXARG` | 32 | `kernel/param.h` | `exec` 最大参数数 |
| `MAXPATH` | 128 | `kernel/param.h` | 内核复制系统调用路径时的上限 |
| `O_RDONLY` | `0x000` | `kernel/fcntl.h` | 只读打开 |

注意：`find.c` 使用 512 字节的用户态拼接缓冲区，但 `open`/`exec` 在内核入口只复制最多 `MAXPATH` 字节的路径。因此“用户缓冲区装得下”不等于“内核一定接受该长度的路径”。

---

## 6. 用户库函数：源码、输入输出与功能

这些函数的声明位于 `user/user.h`。

### 6.1 `strlen`

源码：`user/ulib.c:38`

```c
uint
strlen(const char *s)
{
  int n;
  for(n = 0; s[n]; n++)
    ;
  return n;
}
```

- 输入：以 `\0` 结尾的字符串 `s`；
- 输出：不含末尾 `\0` 的字符数；
- 功能：确定 basename 起始位置、检查目标名和路径长度；
- 前提：传入指针有效且最终存在 `\0`。

### 6.2 `strcmp`

源码：`user/ulib.c:30`

```c
int
strcmp(const char *p, const char *q)
{
  while(*p && *p == *q)
    p++, q++;
  return (uchar)*p - (uchar)*q;
}
```

- 输入：两个 C 字符串；
- 输出：0 表示相等，负数/正数表示首个不同字节的顺序；
- 功能：匹配目标名称、排除 `.`/`..`、检查 `-exec`；
- 注意：不能直接用于可能没有 `\0` 的 `de.name`。

### 6.3 `strcpy`

源码：`user/ulib.c:19`

```c
char*
strcpy(char *s, const char *t)
{
  char *os = s;
  while((*s++ = *t++) != 0)
    ;
  return os;
}
```

- 输入：目标缓冲区 `s`、源字符串 `t`；
- 输出：原始目标指针；
- 副作用：把源字符串连同 `\0` 复制到目标；
- 功能：把当前目录路径复制到 `buf`；
- 安全条件：调用者必须先保证目标空间足够。

### 6.4 `memmove`

源码：`user/ulib.c:111`

```c
void*
memmove(void *vdst, const void *vsrc, int n)
{
  char *dst = vdst;
  const char *src = vsrc;
  if (src > dst) {
    while(n-- > 0)
      *dst++ = *src++;
  } else {
    dst += n;
    src += n;
    while(n-- > 0)
      *--dst = *--src;
  }
  return vdst;
}
```

- 输入：目标地址、源地址、字节数；
- 输出：原始目标地址；
- 功能：复制固定的 `DIRSIZ` 字节目录名；
- 特点：源和目标重叠时仍然正确；
- 注意：它不会自动追加 `\0`，所以 `find` 必须执行 `p[DIRSIZ] = '\0'`。

### 6.5 `printf`、`fprintf`、`vprintf`、`putc`

源码：`user/printf.c`

```c
void printf(const char *fmt, ...);          // 固定输出到 fd 1
void fprintf(int fd, const char *fmt, ...); // 输出到指定 fd
void vprintf(int fd, const char *fmt, va_list ap);
static void putc(int fd, char c);           // 调用 write(fd, &c, 1)
```

| 函数 | 输入 | 输出/返回 | 功能 |
|---|---|---|---|
| `printf` | 格式串和可变参数 | `void`；写 fd 1 | 打印匹配路径 |
| `fprintf` | fd、格式串和可变参数 | `void`；写指定 fd | 向 fd 2 打印错误 |
| `vprintf` | fd、格式串、`va_list` | `void` | 解析 `%d/%x/%p/%c/%s` 等 |
| `putc` | fd、字符 | `void` | 通过 `write` 输出单字节 |

因此 `printf`/`fprintf` 最终也会进入 `write` 系统调用。这里没有标准 C 库的缓冲式 `stdio`。

### 6.6 `stat`：教程提到但最终代码未采用

源码：`user/ulib.c:86`

```c
int
stat(const char *n, struct stat *st)
{
  int fd;
  int r;
  fd = open(n, O_RDONLY);
  if(fd < 0)
    return -1;
  r = fstat(fd, st);
  close(fd);
  return r;
}
```

- 输入：路径和 `struct stat *`；
- 输出：成功 0，失败 -1；
- 功能：把 `open + fstat + close` 封装为一次路径查询；
- 最终 `find` 不使用它，因为遍历目录还要继续使用已打开的 fd 进行 `read`，若调用 `stat` 后再 `open` 会多做一次路径解析。

### 6.7 `sizeof` 不是函数

教程代码中的 `sizeof(buf)`、`sizeof(de)` 是 C 语言运算符，结果类型为 `size_t`；它不在 xv6 源码中拥有函数实现，也不会产生系统调用。

---

## 7. 系统调用的公共路径

`open`、`read` 等在 `user/user.h` 中只有声明，其用户态入口由 `user/usys.pl` 生成到 `user/usys.S`。每个入口的形式都是：

```asm
.global read
read:
 li a7, SYS_read
 ecall
 ret
```

统一流程为：

```text
用户 C 函数调用
  ↓
usys.S：a7 = SYS_xxx，执行 ecall
  ↓
陷阱处理进入内核
  ↓
kernel/syscall.c::syscall()
  ↓ 按 a7 查 syscalls[]
sys_xxx()
  ↓
内核核心实现
  ↓
返回值写入 trapframe->a0
  ↓
回到用户态
```

`kernel/syscall.c` 的参数辅助函数：

| 函数 | 输入 | 输出 | 功能 |
|---|---|---|---|
| `argraw(n)` | 参数序号 0～5 | 对应寄存器的 `uint64` | 从 `a0`～`a5` 取原始参数 |
| `argint(n, &i)` | 序号、整数地址 | 通过指针写出整数 | 获取整数/fd/长度参数 |
| `argaddr(n, &addr)` | 序号、地址变量 | 写出用户虚拟地址 | 获取指针参数 |
| `argstr(n, buf, max)` | 序号、内核缓冲区、上限 | 字符串长度或 -1 | 复制用户字符串到内核 |
| `fetchaddr(addr, &v)` | 用户地址 | 0/-1，并写出 64 位值 | `exec` 读取 `argv[i]` 指针 |
| `fetchstr(addr, buf, max)` | 用户地址 | 长度或 -1 | 复制以 `\0` 结尾的字符串 |

---

## 8. 文件相关系统调用

### 8.1 `open`

用户原型：

```c
int open(const char *path, int omode);
```

- 输入：路径、打开模式；本程序传入 `O_RDONLY`；
- 输出：成功返回非负 fd，失败返回 -1；
- 功能：解析路径、创建 `struct file`、把它放入当前进程 `ofile[]`。

内核调用链：

```text
open
→ sys_open                 kernel/sysfile.c:304
→ namei                    kernel/fs.c:709
→ namex
→ skipelem + dirlookup
→ filealloc                kernel/file.c:29
→ fdalloc                  kernel/sysfile.c:39
```

关键内部函数：

| 函数 | 输入 | 输出 | 功能 |
|---|---|---|---|
| `sys_open()` | 从寄存器取 `path/omode` | fd 或 -1 | 系统调用处理器 |
| `namei(path)` | 路径 | `inode *` 或 0 | 查找整个路径 |
| `namex(path, parent, name)` | 路径及模式 | inode 或 0 | 逐层解析路径；相对路径从 `cwd` 开始 |
| `skipelem(path, name)` | 剩余路径 | 下一段位置或 0 | 取一个最多 `DIRSIZ` 的路径分量 |
| `dirlookup(dp, name, poff)` | 目录 inode、名称 | inode 或 0 | 扫描目录的 `dirent` |
| `filealloc()` | 无 | `file *` 或 0 | 从全局文件表分配对象并设 `ref=1` |
| `fdalloc(f)` | `file *` | fd 或 -1 | 找当前进程空闲 `ofile[]` 槽位 |

`sys_open` 为 inode 类型的对象设置：

```c
f->type = FD_INODE;
f->off = 0;
f->ip = ip;
f->readable = 1;
f->writable = 0;
```

### 8.2 `fstat`

用户原型：

```c
int fstat(int fd, struct stat *st);
```

- 输入：打开的 fd、用户态 `struct stat` 地址；
- 输出：成功 0，失败 -1；
- 副作用：填写 `*st`。

调用链：

```text
fstat
→ sys_fstat                kernel/sysfile.c:110
→ argfd
→ filestat                 kernel/file.c:87
→ stati                    kernel/fs.c:480
→ copyout 到用户地址
```

内部函数：

| 函数 | 输入 | 输出 | 功能 |
|---|---|---|---|
| `argfd(n, pfd, pf)` | 参数序号 | 0/-1，写出 fd/file | 校验 fd 并查 `ofile[]` |
| `filestat(f, addr)` | file、用户地址 | 0/-1 | 锁 inode、调用 `stati`、复制到用户态 |
| `stati(ip, st)` | inode、内核 stat | `void` | 复制 dev/inum/type/nlink/size |

### 8.3 `read`

用户原型：

```c
int read(int fd, void *buf, int n);
```

- 输入：fd、目标缓冲区、最多读取字节数；
- 输出：实际字节数；文件末尾为 0；错误通常为 -1；
- 副作用：成功读取 inode 文件时增加 `struct file.off`。

调用链：

```text
read
→ sys_read                 kernel/sysfile.c:68
→ argfd
→ fileread                 kernel/file.c:106
→ readi                    kernel/fs.c:494
→ either_copyout
```

`fileread` 根据 `file.type` 分派管道、设备或 inode。对目录使用 `FD_INODE` 分支：

```c
ilock(f->ip);
if((r = readi(f->ip, 1, addr, f->off, n)) > 0)
  f->off += r;
iunlock(f->ip);
```

`readi(ip, user_dst, dst, off, n)` 的参数：

- `ip`：要读取的 inode；
- `user_dst=1`：目标是用户虚拟地址；
- `dst`：`&de`；
- `off`：当前文件偏移；
- `n`：`sizeof(struct dirent)`。

它按块读取 inode 数据，并复制到用户缓冲区。目录内容和普通文件内容在这一层没有本质区别。

### 8.4 `close`

用户原型：

```c
int close(int fd);
```

- 输入：fd；
- 输出：成功 0，非法 fd 返回 -1；
- 功能：清空当前进程 `ofile[fd]`，减少打开文件引用计数。

调用链：

```text
close
→ sys_close                kernel/sysfile.c:97
→ argfd
→ myproc()->ofile[fd] = 0
→ fileclose                kernel/file.c:59
→ 引用归零时 iput/pipeclose
```

`find` 递归期间，父目录 fd 在子递归返回之前保持打开，所以同时打开的描述符数量约为目录深度 `O(H)`，并受 `NOFILE=16` 限制。

### 8.5 `write`：由打印函数间接使用

用户原型：

```c
int write(int fd, const void *buf, int n);
```

- 输入：fd、源缓冲区、字节数；
- 输出：成功写入字节数，失败 -1；
- 调用链：`write → sys_write → filewrite`；
- `printf` 最终写 fd 1，`fprintf(2, ...)` 最终写 fd 2。

---

## 9. 进程与程序装载系统调用

### 9.1 `fork`

用户原型：

```c
int fork(void);
```

返回值：

| 所在进程 | 返回值 |
|---|---:|
| 创建失败的父进程 | -1 |
| 子进程 | 0 |
| 创建成功的父进程 | 子进程 pid（大于 0） |

调用链：

```text
fork
→ sys_fork                 kernel/sysproc.c:25
→ kfork                    kernel/proc.c:256
```

`kfork` 的主要工作：

1. `allocproc()` 分配 `struct proc`；
2. `uvmcopy()` 复制父进程用户内存；
3. 复制 `trapframe`；
4. 把子进程 `trapframe->a0` 设为 0；
5. 对每个已打开文件调用 `filedup` 增加引用；
6. `idup` 增加当前目录引用；
7. 设置 `parent` 和 `RUNNABLE`；
8. 父进程返回子 pid。

特别说明：父子进程拥有不同的 `ofile[]` 数组，但数组项引用相同的 `struct file`，所以文件偏移也由该共享对象保存。`find -exec` 的命令通常不会操作继承来的目录 fd，退出时会自动减少引用。

### 9.2 `exec`

用户原型：

```c
int exec(const char *path, char **argv);
```

- 输入：可执行文件路径、以空指针结束的参数数组；
- 输出：失败返回 -1；成功不会返回旧程序；
- 功能：保留进程身份和大部分内核资源，用新程序替换用户地址空间。

调用链：

```text
exec
→ sys_exec                 kernel/sysfile.c:434
→ fetchaddr/fetchstr 复制 argv
→ kexec                    kernel/exec.c:26
→ namei + readi 读取 ELF
→ loadseg
→ 构造新用户栈 argc/argv
→ 提交新页表、入口地址和栈指针
```

`sys_exec` 的输入复制：

- 用 `argstr(0, path, MAXPATH)` 取得程序路径；
- 用 `argaddr(1, &uargv)` 取得用户态参数数组地址；
- 用 `fetchaddr` 逐项取 `argv[i]` 指针；
- 用 `fetchstr` 把每个参数字符串复制到内核页；
- 参数达到空指针时结束，最多 `MAXARG` 项。

`kexec(path, argv)` 的关键步骤：

1. `namei(path)` 找可执行文件；
2. `readi` 读取 `struct elfhdr` 并校验 `ELF_MAGIC`；
3. 遍历 `struct proghdr`，为 `ELF_PROG_LOAD` 段分配内存；
4. `loadseg` 把程序段读入新页表；
5. 分配用户栈和保护页；
6. 把参数字符串及 `argv[]` 指针数组复制到新栈；
7. 设置 `trapframe->a1 = argv地址`；
8. 设置 `epc = elf.entry`、`sp = 新栈顶`；
9. 最后才用新页表替换旧页表。

成功时 `kexec` 返回 `argc`，它经系统调用返回寄存器 `a0` 成为新程序 `main(argc, argv)` 的第一个参数，而不是回到旧的 `exec` 调用点。

### 9.3 `wait`

用户原型：

```c
int wait(int *status);
```

- 输入：退出状态接收地址；可传 0 表示忽略状态；
- 输出：成功返回被回收子进程 pid；无子进程或出错返回 -1；
- 功能：等待一个直接子进程变成 `ZOMBIE`，取状态并回收资源。

调用链：

```text
wait
→ sys_wait                 kernel/sysproc.c:31
→ kwait                    kernel/proc.c:367
```

`kwait` 扫描进程表：

- 找到 `parent == 当前进程` 且状态为 `ZOMBIE` 的子进程；
- `status != 0` 时用 `copyout` 写出 `xstate`；
- 调用 `freeproc` 回收进程；
- 若子进程仍运行，父进程 `sleep`；
- 若根本没有子进程，返回 -1。

当前 `run_cmd` 调用 `wait(0)`，所以命令失败不会改变 `find` 最终的退出码。

### 9.4 `exit`

用户声明：

```c
int exit(int status) __attribute__((noreturn));
```

- 输入：退出状态；
- 输出：不返回；
- 功能：关闭文件、释放当前目录引用、唤醒父进程并进入 `ZOMBIE`。

调用链：

```text
exit
→ sys_exit                 kernel/sysproc.c:10
→ kexit                    kernel/proc.c:323
```

`kexit` 会：

1. 逐个 `fileclose` 当前进程所有 `ofile[]`；
2. `iput` 当前工作目录；
3. 把子进程转交给 `init`；
4. 唤醒可能睡在 `wait` 中的父进程；
5. 保存 `xstate=status`；
6. 设置 `state=ZOMBIE` 并进入调度器。

这也是执行命令的子进程最终释放继承目录 fd 的位置。

---

## 10. 各函数总表

### 10.1 直接出现在最终 `find.c` 中

| 函数 | 所在源码 | 输入 | 返回/输出 | 功能 |
|---|---|---|---|---|
| `base_name` | `user/find.c:7` | 路径 | 最后分量指针 | 提取比较名称 |
| `run_cmd` | `user/find.c:15` | 命令、路径 | `void` | `fork-exec-wait` |
| `find` | `user/find.c:36` | 路径、目标、命令 | `void` | 深度优先递归搜索 |
| `main` | `user/find.c:92` | `argc/argv` | 经 `exit` 结束 | 参数校验和启动搜索 |
| `strlen` | `user/ulib.c:38` | 字符串 | 长度 | 长度与边界检查 |
| `strcmp` | `user/ulib.c:30` | 两字符串 | 比较值 | 名称和选项比较 |
| `strcpy` | `user/ulib.c:19` | 目标、源 | 目标指针 | 复制路径前缀 |
| `memmove` | `user/ulib.c:111` | 目标、源、长度 | 目标指针 | 复制定长目录名 |
| `printf` | `user/printf.c:125` | 格式及参数 | `void` | 输出匹配路径到 fd 1 |
| `fprintf` | `user/printf.c:116` | fd、格式及参数 | `void` | 输出错误到 fd 2 |
| `open` | `user/usys.S:48` | 路径、模式 | fd/-1 | 打开路径 |
| `fstat` | `user/usys.S:63` | fd、stat 地址 | 0/-1 | 获取元数据 |
| `read` | `user/usys.S:23` | fd、缓冲区、长度 | 字节数/-1 | 读取目录项 |
| `close` | `user/usys.S:33` | fd | 0/-1 | 释放描述符 |
| `fork` | `user/usys.S:3` | 无 | -1/0/子 pid | 创建子进程 |
| `exec` | `user/usys.S:43` | 路径、argv | 失败 -1 | 替换用户程序 |
| `wait` | `user/usys.S:13` | 状态地址或 0 | 子 pid/-1 | 等待并回收子进程 |
| `exit` | `user/usys.S:8` | 状态码 | 不返回 | 终止进程 |

### 10.2 为上述功能服务的主要内核函数

| 函数 | 所在源码 | 主要职责 |
|---|---|---|
| `syscall` | `kernel/syscall.c:131` | 按系统调用号分派，写回返回值 |
| `argfd` | `kernel/sysfile.c:21` | fd 校验与 `struct file` 查找 |
| `fdalloc` | `kernel/sysfile.c:39` | 分配进程 fd |
| `sys_open` | `kernel/sysfile.c:304` | 处理 `open` |
| `sys_read` | `kernel/sysfile.c:68` | 处理 `read` |
| `sys_fstat` | `kernel/sysfile.c:110` | 处理 `fstat` |
| `sys_close` | `kernel/sysfile.c:97` | 处理 `close` |
| `sys_write` | `kernel/sysfile.c:82` | 处理打印产生的 `write` |
| `sys_exec` | `kernel/sysfile.c:434` | 复制 `exec` 参数并调用 `kexec` |
| `filealloc` | `kernel/file.c:29` | 分配打开文件对象 |
| `filedup` | `kernel/file.c:47` | 增加文件对象引用计数 |
| `fileclose` | `kernel/file.c:59` | 减引用，归零时真正释放 |
| `filestat` | `kernel/file.c:87` | 从 inode 生成用户 `stat` |
| `fileread` | `kernel/file.c:106` | 按 file 类型执行读取 |
| `filewrite` | `kernel/file.c:134` | 按 file 类型执行写入 |
| `stati` | `kernel/fs.c:480` | inode 字段复制到 stat |
| `readi` | `kernel/fs.c:494` | 从 inode 指定偏移读数据 |
| `dirlookup` | `kernel/fs.c:574` | 在目录项中查名称 |
| `skipelem` | `kernel/fs.c:645` | 从路径取下一个分量 |
| `namex` | `kernel/fs.c:674` | 路径逐层解析核心 |
| `namei` | `kernel/fs.c:709` | 返回完整路径对应 inode |
| `sys_fork` | `kernel/sysproc.c:25` | 调用 `kfork` |
| `kfork` | `kernel/proc.c:256` | 创建并复制进程 |
| `sys_wait` | `kernel/sysproc.c:31` | 调用 `kwait` |
| `kwait` | `kernel/proc.c:367` | 等待并回收子进程 |
| `sys_exit` | `kernel/sysproc.c:10` | 调用 `kexit` |
| `kexit` | `kernel/proc.c:323` | 结束进程并进入僵尸态 |
| `kexec` | `kernel/exec.c:26` | 装载 ELF 并替换地址空间 |
| `loadseg` | `kernel/exec.c:154` | 把 ELF 段读入新页表 |
| `flags2perm` | `kernel/exec.c:13` | ELF 权限转换为页表权限 |

---

## 11. 两条完整调用链

### 11.1 遍历一个目录项

```text
find(path, target, cmd)
  │
  ├─ open(path, O_RDONLY)
  │    └─ sys_open → namei/namex → dirlookup → filealloc/fdalloc
  │
  ├─ fstat(fd, &st)
  │    └─ sys_fstat → filestat → stati → copyout
  │
  ├─ strcmp(base_name(path), target)
  │
  ├─ st.type == T_DIR
  │
  ├─ read(fd, &de, sizeof(de))
  │    └─ sys_read → fileread → readi → copyout
  │
  ├─ memmove + 补 '\0' + 排除 . 和 ..
  │
  ├─ find(子路径, target, cmd)
  │
  └─ close(fd)
       └─ sys_close → fileclose → iput
```

### 11.2 匹配后执行命令

```text
run_cmd(cmd, path)
  │
  ├─ fork
  │    └─ sys_fork → kfork
  │         ├─复制用户地址空间和寄存器
  │         ├─复制文件引用和 cwd 引用
  │         └─子进程变为 RUNNABLE
  │
  ├─子进程：exec(cmd, {cmd, path, 0})
  │    └─ sys_exec → kexec
  │         ├─读取并验证 ELF
  │         ├─建立新页表和用户栈
  │         └─从新程序入口开始运行
  │
  ├─新程序最终 exit(status)
  │    └─ sys_exit → kexit → ZOMBIE
  │
  └─父进程：wait(0)
       └─ sys_wait → kwait → 回收子进程
```

---

## 12. 输入、输出与示例

### 12.1 普通搜索

输入：

```text
$ find . b
```

假设存在 `./a/b` 和 `./c/b`，标准输出：

```text
./a/b
./c/b
```

程序本身的退出状态为 0。

### 12.2 搜索并执行命令

输入：

```text
$ find . b -exec echo
```

其效果依次等价于：

```text
echo ./a/b
echo ./c/b
```

显示出来的路径由 `echo` 输出，不是 `find` 的 `printf` 输出。

### 12.3 错误输入

```text
$ find . b -x echo
usage: find path name [-exec cmd]
```

标准错误使用 fd 2，程序 `exit(1)`。

### 12.4 命令不存在

```text
$ find . b -exec no_such_cmd
find:exec no_such_cmd failed
```

每个匹配项都会创建子进程并尝试执行；失败的子进程退出 1，但父进程忽略状态并继续搜索。

---

## 13. 正确性、资源与边界分析

### 13.1 为什么必须跳过 `.` 和 `..`

目录中的 `.` 指回当前目录，`..` 指回父目录。若递归进入它们，搜索会形成环并不断消耗栈与文件描述符。

### 13.2 为什么目录名必须先复制并补零

`dirent.name` 是固定 14 字节字段。若名称恰好占满 14 字节，它不包含 `\0`。直接传给 `strcmp` 会越界读取，因此需要：

```c
memmove(p, de.name, DIRSIZ);
p[DIRSIZ] = '\0';
```

### 13.3 为什么不能在 `find` 父进程中直接 `exec`

成功的 `exec` 会替换当前进程地址空间，原搜索状态随之消失。因此必须先 `fork`，只让子进程 `exec`，父进程保留递归栈和打开目录。

### 13.4 局部路径缓冲区为何可传给子进程

- `fork` 后，子进程拥有自己的用户地址空间副本；
- 父进程随后立即 `wait`，不会提前覆盖下一个目录名；
- `sys_exec` 会把参数字符串复制到内核，`kexec` 再复制到新用户栈。

因此 `run_cmd(cmd, buf)` 在当前同步实现中是安全的。

### 13.5 文件描述符生命周期

每层目录递归会保持一个 fd：

```text
open → fstat/read → 递归子目录 → close
```

最大同时打开量约为目录深度。`fork` 又会临时增加这些 `struct file` 的引用计数；命令子进程退出后由 `kexit` 关闭。

### 13.6 时间和空间复杂度

设访问对象数为 `N`、匹配数为 `M`、最大目录深度为 `H`：

- 遍历时间：`O(N)`，不计文件系统缓存和路径比较常数；
- `-exec` 额外创建 `M` 个子进程并装载 `M` 次程序；
- 递归用户栈：`O(H)`，每层至少含 `buf[512]`；
- 同时打开目录描述符：`O(H)`；
- 因逐个 `wait`，同一时刻最多有一个由 `find` 创建且尚未回收的命令子进程。

### 13.7 当前实现的几个边界

1. 只支持一个没有额外参数的命令；
2. 不支持 `{}`、`;` 或 shell 语法；
3. `exec` 不经过 shell，`"echo hello"` 会被当作一个文件名；
4. 不汇总命令退出状态；
5. 尾随 `/` 的起始路径 basename 为空；
6. 用户路径缓冲区为 512，但内核 `MAXPATH` 为 128；
7. 极深目录可能超过用户栈或 `NOFILE`；
8. `read` 返回不足一个完整 `dirent` 时循环直接结束，当前代码不另报读取错误。

---

## 14. 最终理解要点

`find` 看似只是递归程序，实际把 xv6 的多层抽象完整串了起来：

```text
目录项名称
  ↕ struct dirent
inode 元数据
  ↕ struct stat / struct inode
打开文件状态与偏移
  ↕ struct file
用户整数描述符
  ↕ proc.ofile[]
系统调用
  ↕ usys.S / syscall / sys_*
进程创建与替换
  ↕ fork / exec / wait / exit
ELF 程序和用户栈
```

普通 `find` 的核心是“目录也是文件，目录内容是 `dirent` 数组”；`find -exec` 的核心是“父进程保留遍历状态，子进程通过 `exec` 执行命令，父进程通过 `wait` 同步并回收”。

---

## 15. 源码核对索引

以下位置均基于当前 `/home/wangxin/xv6-labs-2025` 工作树：

```text
user/find.c             7   base_name
user/find.c            15   run_cmd
user/find.c            36   find
user/find.c            92   main
user/ulib.c            19   strcpy
user/ulib.c            30   strcmp
user/ulib.c            38   strlen
user/ulib.c            86   stat
user/ulib.c           111   memmove
user/printf.c           9   putc
user/printf.c          51   vprintf
user/printf.c         116   fprintf
user/printf.c         125   printf
kernel/syscall.c       11   fetchaddr
kernel/syscall.c       24   fetchstr
kernel/syscall.c       33   argraw
kernel/syscall.c       56   argint
kernel/syscall.c       65   argaddr
kernel/syscall.c       74   argstr
kernel/syscall.c      131   syscall
kernel/sysfile.c       21   argfd
kernel/sysfile.c       39   fdalloc
kernel/sysfile.c       68   sys_read
kernel/sysfile.c       82   sys_write
kernel/sysfile.c       97   sys_close
kernel/sysfile.c      110   sys_fstat
kernel/sysfile.c      304   sys_open
kernel/sysfile.c      434   sys_exec
kernel/file.c          29   filealloc
kernel/file.c          47   filedup
kernel/file.c          59   fileclose
kernel/file.c          87   filestat
kernel/file.c         106   fileread
kernel/file.c         134   filewrite
kernel/fs.c           480   stati
kernel/fs.c           494   readi
kernel/fs.c           566   namecmp
kernel/fs.c           574   dirlookup
kernel/fs.c           645   skipelem
kernel/fs.c           674   namex
kernel/fs.c           709   namei
kernel/sysproc.c       10   sys_exit
kernel/sysproc.c       25   sys_fork
kernel/sysproc.c       31   sys_wait
kernel/proc.c         256   kfork
kernel/proc.c         323   kexit
kernel/proc.c         367   kwait
kernel/exec.c          13   flags2perm
kernel/exec.c          26   kexec
kernel/exec.c         154   loadseg
```

