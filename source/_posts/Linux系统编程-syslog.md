---
title: Linux 系统编程-syslog
top_img: transparent
date: 2023-09-02 13:02:35
updated: 2023-09-03 00:38:35
tags:
  - Linux
  - 日志系统
categories: Linux
keywords:
description: Linux 的 syslog 日志系统
---

# syslog

这里说的 `syslog` 仅仅是`api`，`man 3 syslog` 有详细的说明。简单来说，如下图所示，应用通过 `syslog()` 这个调用将日志信息传递给 `syslogd`（我们可以称为日志收集器）。`syslogd` 可以根据配置文件（一般位于 `/etc/syslog.conf` 来决定如何处理日志信息）。比如保存到某个文件、发送到远程主机、显示到控制台等。

```asciiarmor
      app1               app2              app3 (产生日志)
       │                   │                │
       │                   │  syslog()      │ (AF_UNIX 传递日志)
       │                   │                │
       │           ┌───────▼──────┐         │
       └──────────►│   syslogd    │◄────────┘ (接收日志)
                   └──────────────┘
                          ▼
    ┌──────────────┬──syslog.conf─┬──────────────┐ (处理日志)
    │              │              │              │
    │              │              │              │
    ▼              ▼              ▼              ▼
   file          console        remote         drop
```

我们要把日志信息传递给 `syslogd` 这个守护进程，实际上就是两个进程之间的通信，这里采用的是本地套接字 `AF_UNIX`这种方式实现。可以参考 [Linux 系统编程-进程通信](./Linux 系统编程-进程通信.md)

## 基础用法

对使用来说，很简单✨，就只涉及到 3 个 `API` 调用。

> 虽然说 `syslog` 协议是可以通过网络将日志信息传递给其他网络地址，但这是 `syslogd` 这个守护进程的工作，对应用而言，它只需要将日志信息传递给 `syslogd`，后者会根据配置文件来决定如何处理这些日志信息。

```c
#include <syslog.h>

void openlog(const char *ident, int option, int facility);
void syslog(int priority, const char *format, ...);
void closelog(void);
```

- `openlog`：为当前应用和 `system logger` 建立连接！其实就是和 `syslogd` 通过本地套接字实现进程间通信。
- `syslog/vsyslog`：生成一条日志信息，它将被传递给日志收集器 `syslogd`；
- `closelog`：断开和 `system logger` 的连接。

------

### openlog

> 建立本地套接字连接。对 `musl` 这个库而言，本地套接字文件位于 `/dev/log`。
>
> ```bash
> [root@cvitek]/dev# ls -lh log
> srw-rw-rw-    1 root     root           0 Jan  1 08:00 log
> ```

```c
void openlog(const char *ident, int option, int facility);
```

- `ident`：指向的字符串被添加到每条日志信息的前面，一般指定为程序名。如果 `ident` 为 `NULL`，则使用程序名称。

    > 因为 `syslog` 服务端（一般为 `syslogd`）会接收到很多客户端的消息，所以日志前面一定要带有程序名或其他标识字符串，这样才能确定日志消息的来源。

- `option`：指定这个连接的一些特性。

    > 就像`open`打开文件时可以指定一些 `flags`，用来控制读写权限、非阻塞等等。不过这只是个比喻，与打开文件毫无关系。


- `facility`：用于指定记录消息的程序类型，或者说消息来源。这是下面调用 `syslog` 时的一个默认值，不过实际调用时仍然可以改为其他类型。

  > 与 `ident` 细分来源哪个程序不同，`facility` 是按大类划分，比如来自内核的消息，或者来自用户态的消息。通过这样划分，可以为不同的大类设置不同的输出级别（通过配置文件设置）。比如内核的日志只记录`Info` 级别，而用户态的日志则设置为 `Debug`。

`openlog()` 的使用是可选的，如果没有调用，它将在第一次调用 `syslog()` 时自动调用，在这种情况下，`ident` 将默认为 `NULL`。

------

### option 类型

- `LOG_CONS`：如果发送到`system logger`时出现错误，则直接写入系统控制台 `console`。
- `LOG_NDELAY`：立即打开连接(通常，在记录第一条消息时打开连接)。这可能是有用的，例如，如果后续的 `chroot(2)` 将使日志记录工具内部使用的路径名不可达。
- `LOG_NOWAIT`：不要等待在记录消息时可能已经创建的子进程。(GNU C 库不创建子进程，所以这个选项对 Linux 没有影响。)
- `LOG_ODELAY`：`LOG_NDELAY` 的逆函数；连接的打开被延迟，直到 `syslog()` 被调用。(这是默认值，不需要指定。)
- `LOG_PERROR`：将消息记录到 `stderr`。(POSIX.1-2001 或 POSIX.1-2008 中没有。)
- `LOG_PID`：在每条消息中包含呼叫者的 PID。

### facility 类型

- `LOG_AUTH`: security/authorization messages
- `LOG_AUTHPRIV`: security/authorization messages (private)
- `LOG_CRON`: clock daemon (cron and at)
- `LOG_DAEMON`: system daemons without separate facility value
- `LOG_FTP`: ftp daemon
- `LOG_KERN`: kernel messages (these can't be generated from user processes)
- `LOG_LOCAL0 through LOG_LOCAL7`: reserved for local use。这个可以用户自定义。
- `LOG_LPR`: line printer subsystem
- `LOG_MAIL`: mail subsystem
- `LOG_NEWS`: USENET news subsystem
- `LOG_SYSLOG`: messages generated internally by syslogd(8)
- `LOG_USER (default)`: generic user-level messages
- `LOG_UUCP`: UUCP subsystem

### level 日志级别

- `LOG_EMERG`: system is unusable
- `LOG_ALERT`: action must be taken immediately
- `LOG_CRIT`: critical conditions
- `LOG_ERR`: error conditions
- `LOG_WARNING`: warning conditions
- `LOG_NOTICE`: normal, but significant, condition
- `LOG_INFO`: informational message
- `LOG_DEBUG`: debug-level message

------

### syslog

```c
void syslog(int priority, const char *format, ...);
```

- `priority` 参数由一个 `facility` 值和一个 `level` 值一起组成。

  > 如果优先级中没有用到 `facility`，则使用 `openlog()` 设置的默认值，或者，如果之前没有 `openlog()` 调用，则使用默认值 `LOG_USER`。

- `format`：一个格式字符串，后面跟该格式所需的任何参数，除了 `%m` 将被错误消息字符串 `strerror(errno)` 取代。

  > **格式字符串不需要包含结束换行符**。

比如输出一条 `DEBUG` 级别的日志如下：

```c
syslog(LOG_USER|LOG_DEBUG, "hello world\n");
// 不需要 \n
syslog(LOG_DEBUG, "hello world, num: %d", num);
```

## 配置文件

`syslog` 配置文件用于控制 `syslogd` 守护进程如何处理日志消息。该文件通常位于 `/etc/syslog.conf` 中。

`syslog` 配置文件由一行一行的配置项组成。每一行包含一个或多个关键字组成 + `action` 组成。关键字之间使用分号`;`分割，关键字和 `action` 之间使用空格` `分割。

**action**：指定消息的处理方式。常见的 `action` 包括：

- `file`：将消息写入文件。
- `remote`：将消息发送到远程主机。
- `console`：将消息显示在控制台。
- `null`：丢弃消息。

### 配置文件示例

```
# 将所有 info 级别和内核 debug 级别的消息写入 `/var/log/messages` 文件。
*.info;kern.debug /var/log/messages

# 将所有安全相关消息写入 `/var/log/auth.log` 文件
auth.* /var/log/auth.log

# 将所有信息、警告和错误级别的邮件消息写入 `/var/log/apache/access_log` 文件
mail.info;mail.warn;mail.err /var/log/apache/access_log

# 将所有错误消息发送到 root 用户的邮箱
*.err root@localhost
```

**配置文件注意事项**

- `syslog` 配置文件是全局性的，对所有系统上的应用程序都有效。
- 可以使用 `-f` 选项指定 `syslogd` 守护进程使用的配置文件。

## syslogd

`syslogd` 守护进程参数介绍：

```bash
    -n              Run in foreground
                    这个选项让 syslogd 在前端运行，而不是在后台运行。
                    如果这个选项被使用，syslogd 不会 fork 成后台进程。
    -R HOST[:PORT]  Log to HOST:PORT (default PORT:514)
                    让 syslogd 将日志消息发送到指定的主机和端口。默认的端口是 514。
    -L              Log locally and via network (default is network only if -R)
                    同时将日志记录到本地和网络上。指定 -R 后默认只记录到网络
    -O FILE         Log to FILE (default: /var/log/messages, stdout if -)
                    让 syslogd 将日志消息写入到指定的文件。默认 /var/log/messages✨✨
                    但如果 FILE 是"-"，则 syslogd 会将日志消息写入到标准输出。
    -s SIZE         Max size (KB) before rotation (default 200KB, 0=off)
                    日志文件的最大大小（以 KB 为单位），超过该大小会新建一个日志文件。
    -b N            N rotated logs to keep (default 1, max 99, 0=purge)
                    要保留的已轮转日志文件的数量。
    -l N            Log only messages more urgent than prio N (1-8)
                    syslogd 只记录优先级高于 N 的日志消息。8 是 DEBUG 级别
    -S              Smaller output
                    syslogd 会减小 syslogd 的输出，使得日志文件更小。
    -t              Strip client-generated timestamps
                    移除由客户端生成的日志消息中的时间戳
    -f FILE         Use FILE as config (default:/etc/syslog.conf)
                    syslogd 使用指定的文件作为配置文件。默认文件是 /etc/syslog.conf
```

Output File ⬇️（必须要加引号 `"-"` 才能到标准输出）

```bash
$ syslogd -n -l 8 -O "-"
Sep  3 00:28:38 panda syslog.info syslogd started: BusyBox v1.30.1
```

Smaller Output 演示结果：

```bash
Sep  2 23:50:01 panda user.debug syslog-test[2438]: hello world
Sep  2 23:50:01 panda user.debug syslog-test[2438]: hello world
Sep  2 23:50:01 panda user.debug syslog-test[2438]: hello 123
Sep  2 23:50:39 panda syslog.info syslogd exiting
# 加上 -S 后 ⬇️
Sep  3 00:20:37 syslogd started: BusyBox v1.30.1
Sep  3 00:20:47 syslog-test[2693]: hello world
Sep  3 00:20:47 syslog-test[2693]: hello world
Sep  3 00:20:47 syslog-test[2693]: hello 123
```

## 封装

使用 `syslog` 时需要指定 `facility|level` 比较麻烦，在 [`middleware/v2/include/cvi_debug.h`](https://github.com/sophgo/cvi_mmf_sdk/blob/v4.1.0/middleware/v2/include/cvi_debug.h) 对 `syslog` 通过宏函数进行了一层封装☠️，在日志消息的基础上添加了“文件名、行号、函数名”，本意是好的，但是名字太长了！`TRACE` 和 `DBG` 毫无意义！

```c
CVI_TRACE_LOG(CVI_DBG_INFO, "LT9611_MIPI_Input_Digtal: lt9611 set mipi port = 2\n");
```

相比之下，`openvswitch` 的命名更加简洁：

```c
VLOG_DBG("xxx");
VLOG_INFO("XXXX");
VLOG_ERR("XXX");
```

参照此方式，加个 `CVI_` 前缀也比较简洁啊！`CVI_VLOG_ERR("XXX");`

## 相关资料

- [rfc 5424 -- The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [musl-syslog.h](https://elixir.bootlin.com/musl/latest/source/include/syslog.h#L62)
- [musl-syslog.c](https://elixir.bootlin.com/musl/latest/source/src/misc/syslog.c#L138)
- [Linux 系统编程-进程通信](./Linux 系统编程-进程通信.md)
- [`middleware/v2/include/cvi_debug.h`](https://github.com/sophgo/cvi_mmf_sdk/blob/v4.1.0/middleware/v2/include/cvi_debug.h)

# 简单宏日志

```c
#include <stdio.h>
#include <string.h>

#define MLOG_LEVEL_NONE      0  // 无日志输出
#define MLOG_LEVEL_ERROR     1  // 错误日志
#define MLOG_LEVEL_WARN      2  // 警告日志
#define MLOG_LEVEL_INFO      3  // 信息日志
#define MLOG_LEVEL_DEBUG     4  // 调试日志

#define NONE                 "\e[0m"
#define RED                  "\e[0;31m"
#define YELLOW               "\e[1;33m"
#define BLUE                 "\e[0;34m"

// 设置当前日志级别
#define MLOG_LEVEL MLOG_LEVEL_DEBUG
#define __FILENAME__ (strrchr(__FILE__, '/') ? strrchr(__FILE__, '/') + 1 : __FILE__)

#define MLOG(level, level_str, fmt, ...)                                               \
    do {                                                                    \
        if (level <= MLOG_LEVEL) {                                          \
            printf(level_str " [%s:%d %s] " fmt, __FILENAME__, __LINE__, __func__, ##__VA_ARGS__); \
        }                                                                   \
    } while (0)

/* 只对日志级别加上颜色 */
// #define MLOG_ERROR(fmt, ...) MLOG(MLOG_LEVEL_ERROR, RED "[ERR]" NONE, fmt, ##__VA_ARGS__)
// #define MLOG_WARN(fmt, ...)  MLOG(MLOG_LEVEL_WARN, YELLOW "[WARN]" NONE, fmt, ##__VA_ARGS__)
// #define MLOG_INFO(fmt, ...)  MLOG(MLOG_LEVEL_INFO, BLUE "[INFO]" NONE, fmt, ##__VA_ARGS__)
// #define MLOG_DEBUG(fmt, ...) MLOG(MLOG_LEVEL_DEBUG, "[DBG]", fmt, ##__VA_ARGS__)

/* 所有日志行都上颜色 */
#define MLOG_ERROR(fmt, ...) MLOG(MLOG_LEVEL_ERROR, RED "[ERR]", fmt NONE , ##__VA_ARGS__)
#define MLOG_WARN(fmt, ...)  MLOG(MLOG_LEVEL_WARN, YELLOW "[WARN]", fmt NONE, ##__VA_ARGS__)
#define MLOG_INFO(fmt, ...)  MLOG(MLOG_LEVEL_INFO, BLUE "[INFO]", fmt NONE, ##__VA_ARGS__)
#define MLOG_DEBUG(fmt, ...) MLOG(MLOG_LEVEL_DEBUG, "[DBG]", fmt, ##__VA_ARGS__)


int main() {
    MLOG_ERROR("This is an error message\n");
    MLOG_WARN("This is a warning message\n");
    MLOG_INFO("This is an info message\n");
    MLOG_DEBUG("This is a debug message\n");

    return 0;
}
```

> 为什么下面的这个 `"\e[0;31m"` 能决定输出的颜色呢？
> 这段代码是使用 ANSI 转义码来控制终端文本的颜色和样式。在 ANSI 转义码中，`\e`用于表示ASCII码中的Escape字符（十进制为27），它在终端中告诉终端模拟器去执行一些特定的操作，比如改变颜色或者样式。在 ANSI 转义码中，数字部分对应不同的颜色和样式。例如，`0;31` 表示红色文字（`0`是样式代码，`31`是颜色代码）。同样地，`1;33` 表示亮黄色文字，`0;34` 表示蓝色文字。
>
> 样式代码和颜色代码分别是啥？
>
> ChatGPT
>
> 在 ANSI 转义码中，用于控制文本样式和颜色的代码分别是样式码和颜色码。
>
> 样式码:
>
> - `0`：重置所有文本属性，恢复默认状态。
> - `1`：粗体或者高亮（取决于终端的实现）。
> - `2`：暗色（并非所有终端都支持）。
> - `3`：斜体（并非所有终端都支持）。
> - `4`：下划线。
> - `5`：闪烁（并非所有终端都支持）。
> - `7`：反转显示，即前景色变背景色，背景色变前景色。
> - `8`：隐藏（不可见）。
>
> 颜色码（前景色）：
>
> - `30`：黑色
> - `31`：红色
> - `32`：绿色
> - `33`：黄色
> - `34`：蓝色
> - `35`：洋红色（品红）
> - `36`：青色
> - `37`：白色
>
> 这些代码可以通过组合使用来产生不同的效果。例如，`"\e[1;33m"`表示亮黄色文本（`1`代表高亮，`33`代表黄色）；`"\e[0;31m"`表示普通红色文本（`0`代表默认样式，`31`代表红色）。
>
> 请注意，并非所有终端都支持所有的样式码和颜色码，所以在不同的终端中可能会有不同的效果。

或者更简单一点，不区分日志级别。可以用在简单库的 `.c` 文件中用于输出错误，警告等日志。

```c
/* LOG MACRO */
#define NONE   "\e[0m"
#define RED    "\e[1;31m"
#define YELLOW "\e[1;33m"
#define BLUE   "\e[0;34m"

#define __FILENAME__ \
	(strrchr(__FILE__, '/') ? strrchr(__FILE__, '/') + 1 : __FILE__)
#define MLOG(level_str, fmt, ...)                                       \
	do {                                                            \
		printf(level_str " [%s:%d %s] " NONE fmt, __FILENAME__, \
		       __LINE__, __func__, ##__VA_ARGS__);              \
	} while (0)

#define MLOG_ERROR(fmt, ...) MLOG(RED "[ERR]", fmt, ##__VA_ARGS__)
#define MLOG_WARN(fmt, ...)  MLOG(YELLOW "[WARN]", fmt, ##__VA_ARGS__)
#define MLOG_INFO(fmt, ...)  MLOG(BLUE "[INFO]", fmt, ##__VA_ARGS__)
#define MLOG_DEBUG(fmt, ...) MLOG("[DBG]", fmt, ##__VA_ARGS__)
```

## 注意事项

`printf` 本身是一个**不可重入函数**，不能在中断、信号处理函数中使用。所以，这里的 `mlog` 同样不能在这些地方使用。

> 因为printf函数内部调用了malloc函数，而malloc之前会利用静态mutex对其进行lock。设想这样一种情况，某线程在调用printf（内部执行至对静态mutex加锁操作）时中断，而在信号处理函数中同样调用了printf，则会产生死锁。
