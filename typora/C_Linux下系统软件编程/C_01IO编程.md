# `IO`编程

[TOC]

在 Linux 系统中，**一切皆文件**，但文件有不同的类型。

---

## 1.📋 七种主要文件类型

| 类型                        | 标识符 | IO            | 说明                       | 常见示例                            |
| --------------------------- | ------ | ------------- | -------------------------- | ----------------------------------- |
| **普通文件**`File`          | `-`    | 标准IO/文件IO | 存储用户数据、程序代码等   | `.c`, `.txt`, `/bin/ls`, 可执行文件 |
| **目录文件**`Directory`     | `d`    | 目录IO/文件IO | 包含其他文件的索引列表     | `/home`, `/etc`, `/dev`             |
| **符号链接**`Link`          | `l`    | 链接IO/文件IO | 指向其他文件的路径快捷方式 | `ln -s /target link_name`           |
| **字符设备文件**`Character` | `c`    | 文件IO        | 按**字符流**访问的硬件设备 | `/dev/tty`, `/dev/null`, 串口       |
| **块设备文件**`Block `      | `b`    | 文件IO        | 按数据块访问的存储设备     | `/dev/sda`, `/dev/mmcblk0`          |
| **管道文件(`FIFO`)**        | `p`    | 文件IO        | 进程间通信的单向数据通道   | `mkfifo mypipe`                     |
| **套接字文件**`Socket`      | `s`    | 文件IO        | 用于网络或本地进程通信     | `/var/run/docker.sock`              |

---

🔍 如何查看文件类型

```bash
# 方法1：ls -l 查看首字符
$ ls -l /dev/sda /dev/tty /tmp /bin/ls
brw-rw---- 1 root disk 8, 0 Jan 1 10:00 /dev/sda      # b: 块设备
crw-rw-rw- 1 root tty  5, 0 Jan 1 10:00 /dev/tty      # c: 字符设备
drwxrwxrwt 1 root root 4096 Jan 1 10:00 /tmp          # d: 目录
-rwxr-xr-x 1 root root 142KB Jan 1 10:00 /bin/ls      # -: 普通文件

# 方法2：file 命令识别文件内容类型
$ file /bin/ls
/bin/ls: ELF 64-bit LSB executable, x86-64...

# 方法3：stat 命令查看详细信息
$ stat /dev/null
  File: /dev/null
  Size: 0          Blocks: 0    IO Block: 4096   character special file
```

---

## 2.📁 各类型详解

### 1️⃣ 普通文件（Regular File）
- 最常见的文件类型
- 包含文本、二进制、脚本、归档等
- 可通过 `read()/write()` 系统调用访问

### 2️⃣ 目录文件（Directory）
- 本质是文件名与 inode 的映射表
- 只能由内核修改，用户通过 `mkdir/rm` 等命令间接操作
- `.` 表示当前目录，`..` 表示父目录

### 3️⃣ 符号链接（Symbolic Link）
```bash
$ ln -s /etc/hostname myhost  # 创建软链接
$ ls -l myhost
lrwxrwxrwx 1 user user 15 Jan 1 10:00 myhost -> /etc/hostname
```
- 存储的是**目标路径字符串**
- 目标删除后链接失效（悬空链接）
- 与硬链接区别：硬链接直接指向 inode，不能跨文件系统，不能链接目录

### 4️⃣ 字符设备（Character Device）
- 面向**流式**访问，无缓冲，逐字节传输
- 常用于：串口、终端、键盘等
- 驱动实现 `read/write` 接口，无随机访问能力

### 5️⃣ 块设备（Block Device）
- 面向**块**访问（通常 `512B/4KB`），有缓冲机制
- 常用于：硬盘、SD 卡、U 盘等存储设备
- 支持随机读写，文件系统构建在其上

### 6️⃣ 管道文件（FIFO）
```bash
$ mkfifo mypipe
# 终端1: 写入
$ echo "hello" > mypipe
# 终端2: 读取  
$ cat < mypipe
hello
```
- 内核缓冲区实现，数据读完即消失
- 用于**相关进程**间的单向通信

### 7️⃣ 套接字文件（Socket）
- 支持本地（UNIX Domain）或网络通信
- 类型：`SOCK_STREAM`（TCP）、`SOCK_DGRAM`（UDP）
- 常见于：`/var/run/docker.sock`, `MySQL` `socket `等

---

## 3.📊 标准IO/文件IO区别对比表

| 特性           | 标准 IO (Standard I/O)                | 文件 IO (File I/O)                       |
| :------------- | :------------------------------------ | :--------------------------------------- |
| **所属层级**   | C 标准库 (`glibc`, `stdio.h`)         | 系统调用 (`POSIX` `unistd.h`, `fcntl.h`) |
| **操作对象**   | `FILE * stream` (流指针)              | `int` (文件描述符 `fd`)                  |
| **缓冲机制**   | **用户空间缓冲** (可配置)             | **无用户缓冲** (依赖内核缓冲)            |
| **可移植性**   | 高 (ANSI C, 支持 Windows)             | 中 (`POSIX 标准`, 主要 Unix/Linux)       |
| **主要函数**   | `fopen`, `fread`, `fprintf`, `fclose` | `open`, `read`, `write`, `close`         |
| **返回值检查** | 返回 `NULL` 或 负数                   | 返回 `-1` 表示错误                       |

---

### 1️⃣ 标准 IO (Standard I/O)

#### 1.原理

![image-20260328164806361](https://cdn.jsdelivr.net/gh/bingoteng/Photos/Typora/20260718_01-51_816f057c746fa8c8722b8bd506501740.png)

- **标准库函数**（标准 IO）是对系统调用的**封装**。
  - 它在用户态维护了一个**缓冲区**，减少系统调用（陷入内核）的次数，从而提高效率。
  - 方便移植

#### 2.❗流的缓冲模式
通过 `setvbuf()` 可设置三种缓冲模式：
1.  **全缓冲 (Fully Buffered)**: **缓冲区满刷新**，缓冲区大小（`8KB == 8192B` ）
    - 缓冲区满刷新
    - `fclose`刷新/程序结束
    - `fflush`刷新
      - 场景：（默认用于磁盘文件）

2.  **行缓冲 (Line Buffered)**: **遇到 `\n` 刷新**，缓冲区大小（`1KB == 1024B`）
    - 缓冲区满刷新
    - `fclose`刷新/程序结束
    - `fflush`刷新
    - 遇到`\n`刷新
      - 场景：（默认用于 `stdin/stdout` 终端）。

3.  **无缓冲 (Unbuffered)**: **立即刷新**，缓冲区大小（0）
    - （默认用于 `stderr` 错误输出）。


#### 3.标准IO函数

> [!IMPORTANT]
>
> 具体用法后续再学习

```c
man 3 fopen
    
#include <stdio.h> 
    
fopen、fclose		
fputc、fgetc		
fputs、fgets		
fprintf、fscanf
snprintf、sscanf
fwrite、fread
fseek  
fflush
setbuf  
    
fputc(ch, stdout) == putchar(ch)  
printf("hello world!\n");
fprintf(stdout, "hello world!\n")
```

```c
#include <stdio.h>

FILE *fopen(const char *pathname, const char *mode);
```

- **`pathname`**: 文件路径（字符串）。
- **`mode`**: 打开模式（决定读写权限、创建行为、截断行为等）。
- **返回值**:
  - **成功**: 返回 `FILE *` 流指针。
  - **失败**: 返回 `NULL`，并设置 `errno`（需检查 `perror` 或 `strerror`）。

---

 Mode 模式详解表

| 模式字符串 | 读/写 | 文件不存在 |   文件已存在   |     指针初始位置     |       典型用途       |
| :--------: | :---: | :--------: | :------------: | :------------------: | :------------------: |
| **`"r"`**  | 只读  |   ❌ 报错   |     ✅ 打开     |       文件开头       |  读取配置、日志分析  |
| **`"w"`**  | 只写  |   ✅ 创建   | ⚠️ **清空内容** |       文件开头       | 生成新报告、覆盖保存 |
| **`"a"`**  | 只写  |   ✅ 创建   |     ✅ 追加     |     **文件末尾**     |  日志记录、数据追加  |
| **`"r+"`** | 读写  |   ❌ 报错   |     ✅ 打开     |       文件开头       |   修改现有文件内容   |
| **`"w+"`** | 读写  |   ✅ 创建   | ⚠️ **清空内容** |       文件开头       |  临时文件、读写缓存  |
| **`"a+"`** | 读写  |   ✅ 创建   |     ✅ 追加     | 读：开头<br>写：末尾 |    日志检索与追加    |
| **`"x"`**  | 只写  |   ✅ 创建   |   ❌ **报错**   |       文件开头       |  安全创建（防覆盖）  |

> 💡 **修饰符**:
> - **`b`**: 二进制模式（如 `"rb"`, `"wb"`）。Linux 下可省略，Windows 下必须。
> - **`+`**: 更新模式（同时支持读和写）。
> - **`e`**: 设置 `FD_CLOEXEC` 标志（GNU/POSIX 扩展，防止 `exec` 后泄露 fd）。

---

### 2️⃣ 文件 IO (File I/O / System Calls)

#### 1.原理
直接通过 **系统调用** 与内核交互。每次 `read/write` 都会导致用户态到内核态的切换

操作对象：文件描述符 (`fd`)

- 是一个非负整数（`0: stdin, 1: stdout, 2: stderr`）。
- 本质是进程文件描述符表中的索引，指向内核中的 **打开文件表**。

#### 2.优缺点

- 优点

  - **控制力强**：可设置 `O_NONBLOCK` (非阻塞), `O_SYNC` (同步写入), `O_APPEND` 等。

  - **无用户缓冲**：数据直接交给内核，崩溃时数据丢失风险较小（依赖内核缓冲）。

  - **通用性**：可操作所有文件类型（包括设备文件 `/dev/*`），支持 `ioctl`, `mmap`。

  - **性能**：在大数据块读写时，避免了两层缓冲的拷贝开销。


- ❌ 缺点

  - **移植性差**

  - **无格式化功能**：需配合 `sprintf` 使用。

  - **系统调用开销**：频繁小数据读写性能较差（需手动缓冲）。


#### 3.文件IO函数

```c
man 2 open

#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>  
#include <unistd.h>  
    
open/close
read/write
lseek
ioctrl
chmod
mmap/munmap
```

```c
// 原型
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode); 
// 注意：只有当 flags 包含 O_CREAT 时，才需要第三个参数 mode
```

- **`pathname`**: 文件路径（绝对或相对）。
- **`flags`**: 打开方式标志位（位或运算 `|` 组合）。
- **`mode`**: 文件权限（仅创建新文件时有效）。
- **返回值**:
  - **成功**: 返回非负整数（文件描述符 fd），通常是当前进程未使用的最小整数。
  - **失败**: 返回 `-1`，并设置 `errno` 全局变量。

---

#### Flags 参数详解

`flags` 由三类标志位通过 **按位或 (`|`)** 组合而成：**访问模式**、**创建标志**、**状态标志**。

##### A. 访问模式 (Access Mode) - 必选
这三者是互斥的，必须指定其中一个。

| 标志           | 说明     | 对应操作             |
| :------------- | :------- | :------------------- |
| **`O_RDONLY`** | 只读打开 | 只能 `read`          |
| **`O_WRONLY`** | 只写打开 | 只能 `write`         |
| **`O_RDWR`**   | 读写打开 | 可 `read` 和 `write` |

##### B. 创建与控制标志 (Creation & Control Flags) - 可选

| 标志          | 说明             | 注意事项                                                     |
| :------------ | :--------------- | :----------------------------------------------------------- |
| **`O_CREAT`** | 文件不存在则创建 | **必须配合 `mode` 参数**；若文件已存在则直接打开             |
| **`O_TRUNC`** | 截断文件         | 若文件存在且可写，**清空内容**（长度变为 0）。**危险操作**   |
| **`O_EXCL`**  | 独占创建         | 通常与 `O_CREAT` 连用 (`O_CREAT \| O_EXCL`)。若文件已存在，`open` 失败。用于**原子创建锁文件** |

##### C. 状态标志 (Status Flags) - 可选

| 标志             | 说明           | 应用场景                                                     |
| :--------------- | :------------- | :----------------------------------------------------------- |
| **`O_APPEND`**   | 追加写入       | 每次 `write` 前自动将指针移到文件尾。**原子操作**，适合日志  |
| **`O_ASYNC`**    | 异步通知       | 文件可读/写时发送 `SIGIO` 信号（较少用）                     |
| **`O_NONBLOCK`** | 非阻塞模式     | `open/read/write` 不会挂起。适合 **FIFO、Socket、串口**      |
| **`O_SYNC`**     | 同步写         | `write` 等待数据**和元数据**物理写入磁盘。性能低，数据最安全 |
| **`O_DSYNC`**    | 数据同步写     | `write` 仅等待**数据**物理写入。比 `O_SYNC` 稍快             |
| **`O_CLOEXEC`**  | 关闭子进程继承 | 设置 `FD_CLOEXEC` 标志。`exec` 新程序时自动关闭 fd，**防止 fd 泄漏** |

---

####  Mode 参数详解 (权限控制)

`mode` 仅在 `flags` 包含 **`O_CREAT`** 时生效。它定义了新文件的权限。

##### A. 权限表示 (八进制)
Linux 权限分为三组：**所有者 (User)**、**组 (Group)**、**其他人 (Other)**。

| 权限         | 八进制值 | 二进制 | 说明                   |
| :----------- | :------- | :----- | :--------------------- |
| **r (读)**   | 4        | 100    | 读取内容               |
| **w (写)**   | 2        | 010    | 修改内容               |
| **x (执行)** | 1        | 001    | 作为程序运行或进入目录 |

**常见组合示例：**
- `0644` (`rw-r--r--`): 所有者读写，其他人只读（**普通文件默认**）
- `0755` (`rwxr-xr-x`): 所有者全权，其他人读执行（**脚本/程序默认**）
- `0600` (`rw-------`): 仅所有者读写（**密钥/配置文件**）
- `0777` (`rwxrwxrwx`): 所有人全权（**不安全，不推荐**）

---

### 🔄 两者关系与转换

1. 标准 IO:
   - 库函数，对文件IO的封装
   - 有缓冲的IO
2. 文件IO
   - 系统调用，Linux内核提供的函数接口
   - 无缓冲的IO

![image-20260328172907989](https://cdn.jsdelivr.net/gh/bingoteng/Photos/Typora/20260718_01-51_8d8865637eefe65c21188f4d4815bf1f.png)

```c
// 获取 FILE* 对应的文件描述符
int fd = fileno(fp); 

// 将文件描述符转换为 FILE* (注意：关闭 FILE* 会关闭 fd)
FILE *fp = fdopen(fd, "r");
```

> ⚠️ **黄金法则**：不要在同一文件流上混用标准 IO 和文件 IO 函数，除非你非常清楚缓冲区的刷新状态！

------

### 3️⃣目录IO

| 函数            | 原型                                            | 说明                 | 头文件         |
| :-------------- | :---------------------------------------------- | :------------------- | :------------- |
| **`mkdir`**     | `int mkdir(const char *pathname, mode_t mode);` | 创建目录             | `<sys/stat.h>` |
| **`rmdir`**     | `int rmdir(const char *pathname);`              | 删除**空**目录       | `<unistd.h>`   |
| **`opendir`**   | `DIR *opendir(const char *name);`               | 打开目录流           | `<dirent.h>`   |
| **`closedir`**  | `int closedir(DIR *dirp);`                      | 关闭目录流           | `<dirent.h>`   |
| **`readdir`**   | `struct dirent *readdir(DIR *dirp);`            | 读取目录项 (指针)    | `<dirent.h>`   |
| **`chdir`**     | `int chdir(const char *path);`                  | 改变当前工作目录     | `<unistd.h>`   |
| **`getcwd`**    | `char *getcwd(char *buf, size_t size);`         | 获取当前工作目录     | `<unistd.h>`   |
| **`rewinddir`** | `void rewinddir(DIR *dirp);`                    | 重置目录流指针到开头 | `<dirent.h>`   |

```c
man 3 opendir
#include <sys/stat.h>   
#include <unistd.h>
#include <dirent.h>
    
struct dirent {
    ino_t          d_ino;       //  inode 号
    off_t          d_off;       //  指向下一个目录项的偏移
    unsigned short d_reclen;    //  记录长度
    unsigned char  d_type;      //  文件类型 (DT_REG, DT_DIR 等)
    char           d_name[256]; //  文件名 (null 终止字符串)
};
```

