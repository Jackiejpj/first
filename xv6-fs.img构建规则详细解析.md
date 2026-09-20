# xv6 `fs.img` 构建规则详细解析

> 本文解析的是 xv6 Makefile 中生成文件系统镜像 `fs.img` 的规则。虽然文件名使用了“内核链接规则”，但这段代码本身并不负责链接 xv6 内核；它负责把用户程序和普通文件写入 xv6 文件系统镜像。

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
