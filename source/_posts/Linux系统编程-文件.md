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

![](https://xzchsia.github.io/img/in-post/linux-hard-soft-link/linux-soft-hard-link-diff.png)

冷知识：`.`、`..`目录也是通过硬链接创建的。

## open 函数

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);

// example
int fd = open("/etc/passwd", O_RDWR|O_CREAT|O_EXCL, 0664);
```

- pathname：文件路径
- flags：指定文件的打开方式，`O_RDONLY, O_WRONLY, or O_RDWR`
    - O_RDONLY：只读
    - O_WRONLY：只写
    - O_RDWR：读写
    - O_APPEND：若文件有内容，在文件末尾追加
    - O_TRUNC：若文件存在则清空
    - O_CREAT：若文件不存在则创建
    - O_EXCL：若文件存在则报错，配合O_CREAT使用
- mode：flags 指定了 O_CREAT 时才可使用指定文件权限`mode=0664`，文件权限 = mode & ~umask。
- 返回值：文件描述符，若失败则返回-1并设置errno

### umask

umask值用于设置用户在创建文件时的默认权限，当我们在系统中创建目录或文件时，目录或文件所具有的**默认权限就是由umask值决定的**。

对于root用户，系统默认的umask值是0022；对于普通用户，系统默认的umask值是0002。执行umask命令可以查看当前用户的umask值。

umask值一共有4组数字，其中第1组数字用于定义特殊权限，我们一般不予考虑，与一般权限有关的是后3组数字。

默认情况下，对于目录，用户所能拥有的最大权限是777；对于文件，用户所能拥有的最大权限是目录的最大权限去掉执行权限，即666。因为x执行权限对于目录是必须的，没有执行权限就无法进入目录，而对于文件则不必默认赋予x执行权限。

对于root用户，他的umask值是022。当root用户创建目录时，默认的权限就是用最大权限777去掉相应位置的umask值权限，即对于所有者不必去掉任何权限，对于所属组要去掉w权限，对于其他用户也要去掉w权限，所以目录的默认权限就是755；当root用户创建文件时，默认的权限则是用最大权限666去掉相应位置的umask值，即文件的默认权限是644。

## 错误处理

系统调用的函数出错时，通常会设置变量`errno`，可以通过 `strerror(errno)`来查看报错数字的含义。

```c
#include <error.h>

printf("error: %s \n", strerror(errno))
```

## close 函数

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

- fd：文件描述符
- buf：存放读取数据的缓冲区
- count：要读取的字节数
- 返回值：
  - `>0`，读取的字节数；
  - =0，读到文件末尾；
  - -1，读取失败，错误码存储在errno中；

## write 函数

向文件写入数据，返回写入的字节数。

```c
#include <unistd.h>

ssize_t write(int fd, const void *buf, size_t count);

// example
char buf[100] = "hello world";
int fd = open("/etc/file", O_RDWR);
ssize_t n = write(fd, buf, strlen(buf));
```

- fd：文件描述符
- buf：要写入的数据
- count：要写入的字节数
- 返回值：
  - `>0`，写入的字节数；
  - -1，写入失败，错误码存储在errno中；

### 缓冲区

每调用一次write，就会进行一次内核态和用户态的切换，如果频繁调用且每次写入的数量不大，可以使用缓冲区，提高性能。

库函数中的fputc()和fputs()函数，就使用了缓冲区（默认大小4096byte），以提高性能。

**所以系统函数并不是一定比库函数牛逼，能使用库函数的地方就使用库函数。**

## lseek 函数

设置文件指针的位置，用于文件的读写。

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

- fd：文件描述符
- offset：偏移量，将读写指针从 whence 指定位置向后偏移 offset 个单位
- whence：起始偏移位置，可以是 SEEK_SET（文件开头），SEEK_CUR（当前位置），SEEK_END（文件末尾）
- 返回值：文件指针的新位置，若失败则返回-1并设置errno

应用场景：
  1. 使用 lseek 获取文件大小
  2. 使用 lseek 拓展文件大小，但要想使文件大小真正拓展，必须进行写操作。也可使用 truncate 函数，直接拓展文件

## stat 函数

获取文件属性，从 inode 结构体中获取。

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>

int stat(const char *pathname, struct stat *statbuf);
```

- pathname：文件路径
- statbuf：存储文件属性的结构体，inode 结构体中的属性信息都会被存储到 statbuf 中
- 返回值：
  - 0，成功；
  - -1，失败，错误码存储在errno中；

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
  - 0，成功；
  - -1，失败，错误码存储在errno中；

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

- oldfd：旧文件描述符
- 返回值：
  - 新的文件描述符；
  - -1，失败，错误码存储在errno中；

## dup2 函数

与dup函数类似，只不过它是将文件描述符复制到指定编号的文件描述符。复制前它会自动调用 close(newfd)。

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
