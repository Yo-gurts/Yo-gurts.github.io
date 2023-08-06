---
title: Linux系统编程-文件
top_img: transparent
date: 2022-04-24 17:51:02
updated: 2022-04-24 17:51:02
tags:
  - Linux
  - 文件
categories: Linux
keywords:
description: Linux下一切皆文件，整理处理文件的相关系统调用
---

应用程序的系统调用过程：应用程序->库函数->系统调用->驱动->硬件（磁盘、网卡等）

## 内核态和用户态

现代处理器架构一般允许 CPU 至少在两种不同状态下运行，即：用户态和核心态（有时也称之为监管态 supervisor mode）。执行硬件指令可使 CPU 在两种状态间来回切换。与之对应，**可将虚拟内存区域划分（标记）为用户空间部分或内核空间部分**。**在用户态下运行时，CPU 只能访问被标记为用户空间的内存**，试图访问属于内核空间的内存会引发硬件异常。**当运行于核心态时，CPU 既能访问用户空间内存，也能访问内核空间内存**。

仅当处理器在核心态运行时，才能执行某些特定操作。这样的例子包括：执行宕机（halt）指令去关闭系统，访问内存管理硬件，以及设备 I/O 操作的初始化等。实现者们利用这一硬件设计，将操作系统置于内核空间。这确保了用户进程既不能访问内核指令和数据结构，也无法执行不利于系统运行的操作。

## 系统调用

系统调用是受控的内核入口，借助于这一机制，进程可以请求内核以自己的名义去执行某些动作。以应用程序编程接口（API）的形式，内核提供有一系列服务供程序访问。这包括创建新进程、执行 I/O，以及为进程间通信创建管道等。

- 系统调用将处理器从用户态切换到核心态，以便 CPU 访问受到保护的内核内存。
- 系统调用的组成是固定的，每个系统调用都由一个唯一的数字来标识。（程序通过名称来标识系统调用，对这一编号方案往往一无所知。）
- 每个系统调用可辅之以一套参数，对用户空间（亦即进程的虚拟地址空间）与内核空间之间（相互）传递的信息加以规范。

在探究系统调用时会反复涉及原子操作的概念。**所有系统调用都是以原子操作方式执行的**。之所以这么说，是指内核保证了某系统调用中的所有步骤会作为独立操作而一次性加以执行，其间不会为其他进程或线程所中断。

## 文件类型

| 文件类型标识 | 文件类型 |
| ------------ | -------- |
| -            | 普通文件 |
| d            | 目录     |
| l            | 符号链接 |
| s（伪文件）  | 套接字   |
| b（伪文件）  | 块设备   |
| c（伪文件）  | 字符设备 |
| p（伪文件）  | 管道     |

伪文件不占用磁盘空间。

### 文件描述符

每一个进程对应一个PCB进程控制块（一个记录进程信息的结构体），PCB中包含了一个文件描述符数组0/1/2...1023，每一个文件描述符对应该进程打开的一个文件。

*将文件描述符数组理解为指针数组，文件描述符 fd 就是指针数组的下标！通过它得到指向对应文件的指针。*

其中，0,1,2是预定义了的，0是标准输入，1是标准输出，2是标准错误输出。

进程里面用户打开的文件的文件描述符从3开始编号。

**两个不同的文件描述符，若指向同一打开文件句柄，将共享同一文件偏移量**。因此，如果通过其中一个文件描述符来修改文件偏移量（由调用 `read()`、`write()`或 `lseek()`所致），那么从另一文件描述符中也会观察到这一变化。**无论这两个文件描述符分属于不同进程，还是同属于一个进程**，情况都是如此。

### dentry 和 inode

一个文件主要由两部分组成，dentry(目录项)和 inode，inode 本质是结构体，存储文件的属性信息，如：权限、类型、大小、时间、用户、盘快位置等。也叫做文件属性管理结构，大多数的 inode 都存储在磁盘上，少量常用、近期使用的 inode 会被缓存到内存中。

所谓的删除文件，就是删除 inode，但是数据其实还是在硬盘上，以后会覆盖掉。

### 硬链接与软链接

通过硬链接新建的文件与旧文件对应同一个 inode，只是新建了 dentry。

- 具有相同 inode 节点号的多个文件互为硬链接文件；
- 只有删除了源文件和所有对应的硬链接文件，文件实体才会被删除；
- 硬链接文件是文件的另一个入口；

软链接则类型于windows上的快捷方式，删除软链接文件完全不影响原文件。

```bash
ln oldfile.txt newfile.txt # 创建硬链接
ln -s /etc/oldfile.txt newfile.txt # 软链接文件创建时最好用绝对路径，或者文件移动位置后会失效
```

![img](../images/Linux%E7%B3%BB%E7%BB%9F%E7%BC%96%E7%A8%8B-%E6%96%87%E4%BB%B6/linux-soft-hard-link-diff.png)

冷知识：`.`、`..`目录也是通过硬链接创建的。**但不支持用户为目录添加硬连接！**

不能为目录创建硬链接，从而避免出现令诸多系统程序陷于混乱的链接环路。使用绑定挂载（bind mount）可以获得与为目录创建硬链接相似的效果。

## open 函数

打开文件并返回文件描述符，其返回值为进程未用文件描述符中数值最小者！

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);

// example
int fd = open("/etc/passwd", O_RDWR|O_CREAT|O_EXCL, 0664);
```

- `pathname`：文件路径
- `flags`：指定文件的打开方式，`O_RDONLY, O_WRONLY, or O_RDWR`
  - `O_RDONLY`：只读
  - `O_WRONLY`：只写
  - `O_RDWR`：读写。注意 `O_RDWR` 并不等同于 `O_RDONLY | O_WRONLY`，后者（或组合）属于逻辑错误。
    早期的 UNIX 实现中使用数字 0、1、2 表示上述三个打开方式，为了与早期系统兼容，采用了同样的方式！
  - `O_APPEND`：若文件有内容，在文件末尾追加
  - `O_TRUNC`：若文件存在则清空
  - `O_CREAT`：若文件不存在则创建
  - `O_EXCL`：若文件存在则报错，配合 `O_CREAT` 使用
- `mode`：flags 指定了 `O_CREAT` 时才可使用，指定文件权限 `mode=0664`(八进制数)，文件权限 `= mode & ~umask`。
- 返回值：文件描述符，若失败则返回-1并设置errno，其返回值为进程未用文件描述符中数值最小者！

### umask

umask值用于设置用户在创建文件时的默认权限，当我们在系统中创建目录或文件时，目录或文件所具有的**默认权限就是由umask值决定的**。

对于root用户，系统默认的umask值是0022；对于普通用户，系统默认的umask值是0002。执行umask命令可以查看当前用户的umask值。

umask值一共有4组数字，其中第1组数字用于定义特殊权限，我们一般不予考虑，与一般权限有关的是后3组数字。

默认情况下，对于目录，用户所能拥有的最大权限是777；对于文件，用户所能拥有的最大权限是目录的最大权限去掉执行权限，即666。因为x执行权限对于目录是必须的，没有执行权限就无法进入目录，而对于文件则不必默认赋予x执行权限。

对于root用户，他的umask值是022。当root用户创建目录时，默认的权限就是用最大权限777去掉相应位置的umask值权限，即对于所有者不必去掉任何权限，对于所属组要去掉w权限，对于其他用户也要去掉w权限，所以目录的默认权限就是755；当root用户创建文件时，默认的权限则是用最大权限666去掉相应位置的umask值，即文件的默认权限是644。

## 错误处理

系统调用的函数出错时，通常会设置变量`errno`，可以通过库函数 `strerror(errno)`和`perror(char *msg)`来查看报错数字的含义。

```c
#include <stdio.h>
void perror(const char *s);

#include <string.h>
char *strerror(int errnum);

fd = open(pathname，flags，mode);
if (fd == -1) {
    perror("open");
    // printf("error: %s \n", strerror(errno));
    exit(EXIT_FATLURE);
}
```

函数 `perror()`会打印出其 msg 参数所指向的字符串，紧跟一条与当前 errno 值相对应的消息。

函数 `strerror()`会针对其 errno 参数中所给定的错误号，返回相应的错误字符串。

## close 函数

`close()`系统调用关闭一个打开的文件描述符，并将其释放回调用进程，供该进程继续使用。当一进程终止时，将自动关闭其已打开的所有文件描述符。

```c
#include <unistd.h>

int close(int fd);
```

## read 函数

从文件读取数据，返回读取的字节数。

```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t count);

// example
char buf[100];
int fd = open("/etc/file", O_RDONLY);
ssize_t n = read(fd, buf, 100);
```

- `fd`：文件描述符
- `buf`：用来存放输入数据的内存缓冲区地址。缓冲区至少应有 count 个字节。
- `count`：指定最多能读取的字节数，size_t 数据类型属于无符号整数类型。
- 返回值：ssize_t 数据类型属于有符号的整数类型
  - `>0`，读取的字节数；
  - `=0`，读到文件末尾（EOF）；
  - `-1`，读取失败，错误码存储在errno中；

## write 函数

向文件写入数据，返回实际写入的字节数。

```c
#include <unistd.h>

ssize_t write(int fd, const void *buf, size_t count);

// example
char buf[100] = "hello world";
int fd = open("/etc/file", O_RDWR);
ssize_t n = write(fd, buf, strlen(buf));
```

- fd：文件描述符
- buf：要写入的数据的内存地址
- count：欲从 buffer 写入文件的数据字节数
- 返回值：
  - `>0`，写入的字节数；
  - -1，写入失败，错误码存储在errno中；

### 缓冲区

每调用一次`write()`，就会进行一次内核态和用户态的切换，如果频繁调用且每次写入的数量不大，可以使用缓冲区，提高性能。

库函数中的`fputc()`和`fputs()`函数，就使用了缓冲区（默认大小4096byte），以提高性能。

**所以系统函数并不是一定比库函数牛逼，能使用库函数的地方就使用库函数。**

注意，将日志写入到文件时，对缓冲区的处理与输出到终端不一样。**当输出到文件时，只有当缓冲区满的时候，才回真正输出缓冲区的内容，并清空缓冲区**。当输出到屏幕时，*除了缓冲区满外，遇到’\n’会自动清空缓冲区，另外读入内容也会清空缓冲区*。当写入到文件时，往往需要显示调用 `fflush()`;

## lseek 函数

对于每个打开的文件，系统内核会记录其文件偏移量，有时也将文件偏移量称为读写偏移量或指针。文件偏移量是指执行下一个 `read()`或 `write()`操作的文件起始位置，会以相对于文件头部起始点的文件当前位置来表示。文件第一个字节的偏移量为 0。

```c
#include <sys/types.h>
#include <unistd.h>

off_t lseek(int fd, off_t offset, int whence);

// example
char msg[100] = "hello world";
int fd = open("/etc/file", O_RDWR);
ssize_t n = write(fd, msg, strlen(msg));
lseek(fd, 100, SEEK_END); // 拓展文件大小 100 byte
write(fd, "\0", 1); // 写入一个空字符，不进行操作就无法拓展文件大小
lseek(fd, 0, SEEK_SET); // 设置文件指针到文件开头
read(fd, msg, n);
```

- `fd`：文件描述符
- `offset`：偏移量，将读写指针从 whence 指定位置向后偏移 offset 个单位
- `whence`：起始偏移位置，可以是 `SEEK_SET`（文件开头），`SEEK_CUR`（当前位置），`SEEK_END`（文件末尾）
- 返回值：文件指针的新位置，若失败则返回-1并设置errno

获取文件偏移量的当前位置：`curr = lseek(fd，0，SEEK_CUR);`

应用场景：

  1. 使用 `lseek` 获取文件大小
  2. 使用 `lseek` 拓展文件大小，但要想使文件大小真正拓展，必须进行写操作。也可使用 `truncate` 函数，直接拓展文件

## truncate/ftruncate 函数

设置文件为指定大小，若文件大小大于指定值时，会截断到指定大小。小于时，会在文件末尾添加空字符。

文件不存在，或文件没有写权限，返回 -1。

```c
#include <unistd.h>
#include <sys/types.h>

int truncate(const char *path, off_t length);
int ftruncate(int fd, off_t length);
```

- `path`：文件路径
- `length`：指定文件大小，若文件当前长度大于参数 length，调用将丢弃超出部分，若小于参数 length，调用将在文件尾部添加一系列空字节或是一个文件空洞。
- 返回值：
  - 0，成功；
  - -1，失败，错误码存储在errno中；
- `fd`：要修改的文件描述符。该系统调用不会修改文件偏移量！

编译时使用`-std=c99`会报警告，可以使用`-std=gnu99`来兼容。

## fcntl 函数

`fcntl()`系统调用对一个打开的文件描述符执行一系列控制操作。

```c
#include <unistd.h>
#include <fcntl.h>

int fcntl(int fd, int cmd, ... /* arg */ );
```

- `cmd` 参数所支持的操作范围很广。
- 第三个参数以省略号来表示，这意味着可以将其设置为不同的类型，或者加以省略。内核会依据 cmd 参数（如果有的话）的值来确定该参数的数据类型。

## pread/pwrite 函数

系统调用 `pread()` 和 `pwrite()` 完成与 `read()` 和 `write()` 相类似的工作，**只是前两者会在 `offset` 参数所指定的位置**进行文件 I/O 操作，而非始于文件的当前偏移量处，且**它们不会改变文件的当前偏移量**。

```c
#include <unistd.h>

ssize_t pread(int fd, void *buf, size_t count, off_t offset);
ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);

// pread(O)调用等同于将如下调用纳入同一原子操作:
off_t orig;

orig = lseek(fd，0，SEEK_CUR);   /# Save CUITITent offset *#/
lseek(fd，offset，SEEK_SET);
s = read(fd，buf，len);
lseek(fd，orig，SEEK_SET);       /# Restore original file offset *#/
```

- `offset`: 在指定的位置进行文件 I/O 操作

该函数在多线程下有用武之地，当调用 `pread()` 或 `pwrite()` 时，多个线程可同时对同一文件描述符执行 I/O 操作，且不会因其他线程修改文件偏移量而受到影响。

## stat 函数

获取文件属性，从 inode 结构体中获取。

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>

int stat(const char *pathname, struct stat *statbuf);

int fstat(int fd, struct stat *statbuf);
int lstat(const char *pathname, struct stat *statbuf);
```

- pathname：文件路径
- statbuf：存储文件属性的结构体，inode 结构体中的属性信息都会被存储到 statbuf 中
- 返回值：
  - 0，成功；
  - -1，失败，错误码存储在errno中；

`fstat`与`stat`等价，只是传入参数为文件描述符。

`lstat`与`stat`等价，只是传入的文件类型为**符号链接**时，返回的文件信息是符号链接本身的信息。

```c
struct stat {
    dev_t     st_dev;         /* ID of device containing file */
    ino_t     st_ino;         /* Inode number */
    mode_t    st_mode;        /* File type and mode */
    nlink_t   st_nlink;       /* Number of hard links */
    uid_t     st_uid;         /* User ID of owner */
    gid_t     st_gid;         /* Group ID of owner */
    dev_t     st_rdev;        /* Device ID (if special file) */
    off_t     st_size;        /* Total size, in bytes */
    blksize_t st_blksize;     /* Block size for filesystem I/O */
    blkcnt_t  st_blocks;      /* Number of 512B blocks allocated */

    struct timespec st_atim;  /* Time of last access */
    struct timespec st_mtim;  /* Time of last modification */
    struct timespec st_ctim;  /* Time of last status change */
};
```

查看文件类型：

```c
switch (sb.st_mode & S_IFMT) {
    case S_IFBLK:  printf("block device\n");            break;
    case S_IFCHR:  printf("character device\n");        break;
    case S_IFDIR:  printf("directory\n");               break;
    case S_IFIFO:  printf("FIFO/pipe\n");               break;
    case S_IFLNK:  printf("symlink\n");                 break;
    case S_IFREG:  printf("regular file\n");            break;
    case S_IFSOCK: printf("socket\n");                  break;
    default:       printf("unknown?\n");                break;
}

// or
if (S_ISDIR(sb.st_mode)) {
    printf("directory\n");
} else if (S_ISREG(sb.st_mode)) {
    printf("regular file\n");
}
```

## link 函数

硬链接数就是 dentry 数目，link 就是用来创建硬链接的。

```c
#include <unistd.h>

int link(const char *oldpath, const char *newpath);
```

- oldpath：旧文件路径
- newpath：新文件路径
- 返回值：
  - 0，成功；
  - -1，失败，错误码存储在errno中；

## unlink 函数

删除一个文件的目录项 dentry，使硬链接数-1。

清除文件时，如果文件的硬链接数到 0 了，没有 dentry 对应，但该文件仍不会马上被释放，要等到所有打开文件的进程关闭该文件，系统才会挑时间将该文件释放掉。

```c
#include <unistd.h>

int unlink(const char *pathname);
```

- pathname：文件路径
- 返回值：
  - `0`，成功；
  - `-1`，失败，错误码存储在errno中；

## dup 函数

复制文件描述符到未使用的最小编号的描述符中。

复制后，新旧描述符都可使用，并且共享文件指针偏移和文件状态标志等。

```c
#include <unistd.h>

int dup(int oldfd);

// example
int fd = open("./test.txt", O_RDWR|O_CREAT);
int newfd = dup(fd);
printf("fd: %d, newfd: %d\n", fd, newfd); // fd: 3, newfd: 4
```

- `oldfd`：旧文件描述符
- 返回值：
  - 新的文件描述符；
  - -1，失败，错误码存储在errno中；

## dup2 函数

与dup函数类似，只不过它是将文件描述符复制到指定编号的文件描述符。复制前它会自动调用 `close(newfd)`。

同样的，复制后新旧描述符都可使用，并且共享文件指针偏移和文件状态标志等。

```c
#include <unistd.h>

int dup2(int oldfd, int newfd);

// example
int fd = open("./test.txt", O_RDWR|O_CREAT);
int newfd = dup2(fd, STDOUT_FILENO); // 将标准输出等重定向到文件
```

- oldfd：旧文件描述符
- newfd：新文件描述符
- 返回值：
  - 新的文件描述符；
  - -1，失败，错误码存储在errno中；

让 newfd 指向 oldfd，也就是说无论写newfd还是oldfd，都会写到oldfd。

## ioctl 函数

可用于向内核传递任意类型的数据。也是一种内核态和用户态通信的方式。

```c
#include <sys/ioctl.h>

int ioctl(int fd, unsigned long request, ...);

// 内核中的对应函数原型为
long module_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
```

- `fd`：与设备驱动通信时，该`fd`通常表示`/dev/`目录下的某个文件
- `request`/`cmd`：表示操作类型，也用于判断如何处理后续参数
- `arg`：在内核中 `unsigned long` 和 `void *` 是等价的，所以这里既可以表示用户态传入的指针变量，也可以表示普通的整形变量

``` c
typedef struct _attrs {
    int a;
    int b;
} attrs;

long scull_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int err = 0, tmp;
	attrs kdata;

    switch (cmd) {
        case IOCTL_INT:
            tmp = arg;
            break;
        case IOCTL_POINTER:
            copy_from_user(&kdata, (attrs *)arg, sizeof(attrs));
            break;
    }
}
```

上面例子中的 `IOCTL_INT` `IOCTL_POINTER` 由各个模块自行设计。

## 相关资料

- [**Linux系统调用列表**](https://blog.51cto.com/u_3078781/3287065)
- 《LINUX 系统编程手册》
