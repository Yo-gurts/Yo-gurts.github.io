---
title: Linux系统编程-信号
top_img: transparent
date: 2022-05-02 21:49:35
updated: 2022-05-02 21:49:35
tags:
  - Linux
categories: Linux
keywords:
description: Linux 编程中的信号
---

## 信号

1. 信号是软件层面上的“中断”。一旦信号产生，无论程序执行到什么位置，必须立即停止运行，处理信号，处理结束，再继续执行后续指令。
2. 所有信号的产生及处理全部都是由【内核】完成的。
3. 简单、不能携带大量信息、满足条件才发送。

### 信号的生命周期

1. 产生：
2. 未决：产生与递达之间状态。
3. 递达：产生并且送达到进程。直接被内核处理掉。
4. 处理：执行默认处理动作、忽略、捕捉（自定义）

阻塞信号集（信号屏蔽字）：进程控制块PCB中的变量（位图），用来记录信号的屏蔽状态。被屏蔽的信号，在解除屏蔽前，一直处于未决态。

未决信号集：进程控制块PCB中的变量（位图），用来记录信号的处理状态。
  - 信号产生时，未决信号集中对应位立刻翻转为1，表示信号处于未决状态。信号被处理后对应位翻转回0，这一过程往往非常短暂。
  - 信号产生后，由于某些原因（主要是阻塞）不能递达，一直处理未决状态。
  - 设置阻塞后，同一信号多次产生，也只能记录一次（也只会被处理一次）。

```c
struct task_struct {
    /* Signal handlers: */
    struct signal_struct    *signal;
    struct sighand_struct   *sighand;
    sigset_t                blocked;
    sigset_t                real_blocked;

    /* Restored if set_restore_sigmask() was used: */
    sigset_t                saved_sigmask; // 信号屏蔽字
    struct sigpending       pending;    // 未决信号集
    unsigned long           sas_ss_sp;
    size_t                  sas_ss_size;
    unsigned int            sas_ss_flags;
}
```

### 产生信号

1. 按键产生，如：`ctrl+c, ctrl+z, ctrl+\`
2. 系统调用产生，如：`kill, raise, abort`
3. 软件条件产生，如：定时器`alarm`
4. 硬件异常产生，如：非法访问内存（段错误）、除0（浮点数例外）、内存对齐出错（总线错误）
5. 命令产生， 如：`kill -9`

### 信号四要素

**编号、名称、事件、默认处理动作**

总计有64个信号（可通过`kill -l`查看信号名称及编号），`man 7 signal`也可查看信号。前32个为常规信号，后32个为实时信号。

```
Term   终止进程
Ign    忽略信号
Core   终止进程并产生core文件
Stop   暂停进程
Cont   如果当前进程被暂停，让进程继续执行

名称       编号      动作      事件
Signal   x86/ARM   Action   Comment
────────────────────────────────────────────────────────────────
SIGHUP       1      Term    Hangup detected on controlling terminal
SIGINT       2      Term    Interrupt from keyboard
SIGQUIT      3      Core    Quit from keyboard
SIGILL       4      Core    Illegal Instruction
SIGTRAP      5      Core    Trace/breakpoint trap
SIGABRT      6      Core    Abort signal from abort(3)
SIGIOT       6      Core    IOT trap. A synonym for SIGABRT
SIGBUS       7      Core    Bus error (bad memory access)
SIGFPE       8      Core    Floating-point exception
SIGKILL      9      Term    Kill signal
SIGUSR1     10      Term    User-defined signal 1
SIGSEGV     11      Core    Invalid memory reference
SIGUSR2     12      Term    User-defined signal 2
SIGPIPE     13      Term    Broken pipe: write to pipe with no
SIGALRM     14      Term    Timer signal from alarm(2)
SIGTERM     15      Term    Termination signal
SIGSTKFLT   16      Term    Stack fault on coprocessor (unused)
SIGCHLD     17      Ign     Child stopped or terminated
SIGCONT     18      Cont    Continue if stopped
SIGSTOP     19      Stop    Stop process
SIGTSTP     20      Stop    Stop typed at terminal see also seccomp(2)
SIGTTIN     21      Stop    Terminal input for background process
SIGTTOU     22      Stop    Terminal output for background process
SIGURG      23      Ign     Urgent condition on socket (4.2BSD)
SIGXCPU     24      Core    CPU time limit exceeded (4.2BSD);
SIGXFSZ     25      Core    File size limit exceeded (4.2BSD);
SIGVTALRM   26      Term    Virtual alarm clock (4.2BSD)
SIGPROF     27      Term    Profiling timer expired
SIGWINCH    28      Ign     Window resize signal (4.3BSD, Sun)
SIGIO       29      Term    I/O now possible (4.2BSD)
SIGPWR      30      Term    Power failure (System V)
SIGSYS      31      Core    Bad system call (SVr4);
SIGUNUSED   31      Core    Synonymous with SIGSYS
```

`SIGKILL, SIGSTOP` 不能被捕捉、阻塞、忽略。

重点是下面的信号：

```
SIGHUP       1   Term   当用户退出shell时，由该shell启动的所有进程都会收到该信号。
SIGINT       2   Term   用户按下 ctrl+c
SIGQUIT      3   Core   用户按下 ctrl+\
SIGBUS       7   Core   非法访问内存地址，包括内存对齐出错
SIGFPE       8   Core   浮点数错误、溢出、除数为0等所有运算错误
SIGKILL      9   Term   Kill信号，不能被捕捉、阻塞、忽略
SIGUSR1     10   Term   用户定义的信号1
SIGSEGV     11   Core   进行了无效的内存访问
SIGUSR2     12   Term   用户定义的信号2
SIGPIPE     13   Term   向一个没有读端口的管道写数据
SIGALRM     14   Term   定时器超时
SIGTERM     15   Term   结束信号，kill缺省发送此信号，与KILLSIG不同，可以阻塞、忽略
SIGCHLD     17   Ign    子进程状态发生变化时，默认忽略此信号
```

## kill 函数

向指定进程发送指定的信号。

```c
#include <sys/types.h>
#include <signal.h>

int kill(pid_t pid, int sig);

// example
if (fork() == 0) {
    printf("pid:%d, ppid:%d\n", getpid(), getppid());
    kill(getppid(), SIGBUS); // 向父进程发送SIGBUS信号
} else {
    printf("parent\n");
    sleep(1);
}
```

- sig：信号编号，不推荐直接使用数字，应使用宏名。
- pid：进程编号
  - pid > 0：指定进程编号
  - pid = 0：发给与kill调用者属于同一进程组的 所有进程
  - pid = -1：发给进程有权限发送的 所有进程
  - pid < -1：发给进程组编号为abs(pid)的进程组中的 所有进程
- 返回值：成功返回0，失败返回-1

进程组: 每个进程都属于一个进程组，进程组是一个或多个进程集合，他们相互关联，共同完成一个实体任务，每个进程组都有一个进程组长，默认进程组ID 与进程组长ID 相同。

权限保护: super 用户(root)可以发送信号给任意用户，普通用户是不能向系统用户发送信号的。 kill -9 (root 用户的pid) 是不可以的。同样，普通用户也不能向其他普通用户发送信号，终止其进程。 只能向自己创建的进程发送信号。普通用户基本规则是: **发送者实际或有效用户ID == 接收者实际或有效用户ID**

## alarm 函数

设置定时器（闹钟），指定的 seconds 后，内核给进程发送 SIGALRM 信号，默认动作为终止进程。

**每个进程有且只有一个定时器**。

```c
#include <unistd.h>

unsigned int alarm(unsigned int seconds);
```

- seconds：设置定时器超时时间，单位为秒
- 返回值：返回之前设置的定时器剩余的时间，之前没有设置返回0。无失败！

计时与进程状态无关，自然计时法。

## setitimer 函数

与 alarm 功能类似，精度为微秒，可以实现周期定时。

```c
#include <sys/time.h>

int getitimer(int which, struct itimerval *curr_value);
int setitimer(int which, const struct itimerval *new_value,
                struct itimerval *old_value);

struct itimerval {
    struct timeval it_interval; /* 周期定时 */
    struct timeval it_value;    /* 下一次时间 */
};
struct timeval {
    time_t      tv_sec;         /* seconds */
    suseconds_t tv_usec;        /* microseconds */
};

// example 第一次两秒后执行，后续每隔五秒执行一次
void myfunc(int signo) {
    printf("hello\n");
}

int main() {
    signal(SIGALRM, myfunc); // 注册信号的处理函数
    struct itimerval new_value;
    new_value.it_interval.tv_sec = 5;
    new_value.it_interval.tv_usec = 0;
    new_value.it_value.tv_sec = 2;
    new_value.it_value.tv_usec = 0;

    setitimer(ITIMER_REAL, &new_value, NULL);
    while(1);
    return 0;
}
```

- which：定时器类型
  - ITIMER_REAL：真实时间（自然计时），发送信号 SIGALRM
  - ITIMER_VIRTUAL：虚拟空间计时（用户空间），发送信号 SIGVTALRM。进程占用CPU时间
  - ITIMER_PROF：运行时计时（用户+内核），发送信号 SIGPROF。进程占用CPU时间及系统调用时间
- new_value：设置新的定时器值，如果为 NULL，则不设置新的定时器值，只返回旧的定时器值
- old_value：返回旧的定时器值，如果为 NULL，则不返回旧的定时器值
- 返回值：成功返回0，失败返回-1

`time` 命令查看程序执行时间，real总时间，user用户空间时间，sys系统空间时间。

## raise 函数


## abort 函数


## 信号集操作函数

信号集`sigset_t`虽然是位图，但一般不允许直接对PCB中的信号集进行操作，而是先自定义一个信号集，然后操作这个信号集，最后再把信号集通过提供的函数与PCB中的信号集进行运算（与、或、覆盖等）。

```c
#include <signal.h>

sigset_t set;
int sigemptyset(sigset_t *set);     // 全部清零
int sigfillset(sigset_t *set);      // 全部置1
int sigaddset(sigset_t *set, int signum); // 将 signum 置1
int sigdelset(sigset_t *set, int signum); // 将 signum 置0
int sigismember(const sigset_t *set, int signum); // 判断 signum 是否为1
```

## sigprocmask 函数

```c
#include <signal.h>

int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
```

- how：操作方式
  - SIG_BLOCK：设置屏蔽，`mask = mask | set`
  - SIG_UNBLOCK：解除屏蔽，`mask = mask & ~set`
  - SIG_SETMASK：覆盖，不推荐使用，`mask = set`
- set：自定义的传入信号集
- oldset：返回旧的信号集
- 返回值：成功返回0，失败返回-1

## sigpending 函数

读取当前进程的未决信号集。

```c
#include <signal.h>

int sigpending(sigset_t *set);

// example
sigset_t set;
sigpending(&set);
```

- set：传出参数，返回未决信号集
- 返回值：成功返回0，失败返回-1

```c
void print_set(sigset_t *set) {
    for (int i=1; i < 33; i++) {
        if (sigismember(set, i))
            printf("has : %d\n", i);
    }
}

int main() {

    sigset_t pend, in;
    sigemptyset(&in);
    sigaddset(&in, SIGINT);
    sigprocmask(SIG_BLOCK, &in, NULL);

    while(1) {
        sigpending(&pend);
        print_set(&pend);
        sleep(2);
    }
	return 0;
}
```

## signal 函数

注册一个信号捕捉函数，当收到信号时，调用该函数。

```c
#include <signal.h>

typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);
```

- signum：要注册的信号编号
- handler：信号处理函数，注意这里是函数指针，函数的参数是信号编号
- 返回值：成功返回之前的信号处理函数，失败返回SIG_ERR，并设置errno

## sigaction 函数

同 signal 函数，注册一个信号捕捉函数

```c
#include <signal.h>

int sigaction(int signum, const struct sigaction *act,
                struct sigaction *oldact);
struct sigaction {
    void     (*sa_handler)(int);   // 信号处理函数
    void     (*sa_sigaction)(int, siginfo_t *, void *); // 一般不用
    sigset_t   sa_mask; // 信号屏蔽字，只作用于信号捕捉函数执行期间！
    int        sa_flags; // =0 时，信号捕捉函数执行期间屏蔽当前信号
    void     (*sa_restorer)(void);  // 废弃
};

// example
void print_set(int signo) {
    printf("catch you : %d\n", signo);
}
int main() {
    struct sigaction act, oldact;
    act.sa_handler = print_set;
    sigemptyset(&act.sa_mask);
    act.sa_flags = 0;
    sigaction(SIGINT, &act, &oldact);

    while(1);
	return 0;
}
```

### 信号捕捉特性

1. 进程正常运行时，默认 PCB 中有一个信号屏蔽宇，假定为 X，它决定了进程自动屏蔽哪些信号。当注册了某个信号捕捉函数，捕捉到该信号以后，要调用该函数。而该函数有可能执行很长时间，在这期间所屏蔽的信号不由X来指定。而是用 sa_mask 来指定。调用完信号处理函数，再恢复为 X。
2. XXX 信号捕捉函数执行期间，XXX 信号自动被屏蔽（sa_flags=0）。
3. 阻塞的常规信号不支持排队，产生多次只记录一次。(后 32 个实时信号支持排队)
4. XXX 信号捕捉函数执行期间，若信号 B 未被屏蔽，当信号 B 递达时，会从当前的执行函数跳转到 B 信号的处理函数。（信号就像中断，来了就打断当前操作）。

### sa_flags

SA_RESTART：当一个系统调用被信号中断时，信号处理结束是否还要继续执行系统调用。

## SIGCHLD 信号

子进程状态发生变化时，父进程会收到 SIGCHLD 信号。

- 子进程终止时。
- 子进程接收到 SIGSTOP 信号停止时。
- 子进程处在停止态，接受到SIGCONT 后唤醒时

借助 SIGCHILD 信号回收子进程。

```c
#include <fcntl.h>
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <time.h>
#include <unistd.h>


void catch_chld(int signo) {
    pid_t wpid;
    while((wpid = waitpid(-1, NULL, 0)) != -1) {
        printf("catch child id: %d\n", wpid);
    }
}

int main() {
    pid_t pid;
    sigset_t set;
    // 先阻塞信号，防止父进程还未注册处理函数，子进程就结束了
    sigaddset(&set, SIGCHLD);
    sigprocmask(SIG_BLOCK, &set, NULL);

    int i;
    for (i = 0; i < 15; i++)
        if ((pid = fork()) == 0)
            break;

    if (i == 15) {
        struct sigaction act;
        act.sa_handler = catch_chld;
        sigemptyset(&act.sa_mask);
        act.sa_flags = 0;
        sigaction(SIGCHLD, &act, NULL);
        // 解除阻塞
        // sigdelset(&set, SIGCHLD);
        sigprocmask(SIG_UNBLOCK, &set, NULL);

        printf("i'm parent, pid = %d\n", getpid());
        while(1);
    } else {
        printf("i'm child, pid = %d\n", getpid());
    }

	return 0;
}
```


## 参考资料

- [PCB 中信号相关内容 L913~L923](https://elixir.bootlin.com/linux/v5.4.190/source/include/linux/sched.h#L913)