---
title: Linux网络编程-socket
top_img: transparent
date: 2022-05-08 22:41:39
updated: 2022-05-08 22:41:39
tags:
  - Linux
  - socket
categories: Linux
keywords:
description:
---

## socket 套接字

![image-20220508224718687](../images/Linux%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B-socket/image-20220508224718687.png)

图中两个虚线框表示两个套接字，在网络通信中，套接字一定是成对出现的。一端的发送缓冲区对应对端的接收缓冲区。

一个socket只有一个文件描述符，即发送缓冲区和接收缓冲区使用同一个文件描述符。

网络套接字本质：一个文件描述符指向一个套接字（该套接字内部由内核借助两个缓冲区实现）。

## 网络地址

### 字节序转换

由于历史遗留问题，网络数据流采用大端字节序，而 pc 本地一般采用的小段字节序，所以在网络通信中，需要转换字节序。

- 小端法：（pc本地存储）高位存高地址，地位存低地址。
- 大端法：（网络存储）高位存低地址，地位存高地址。

字节序只影响不同字节间的顺序，如果只看单字节内部，大小端都一样。**不存在`00001111`变为`11110000`！**

```c
#include <arpa/inet.h>

// h: host, to, n: network, l: long, s: short
// hton也就是主机到网络的转换，ntoh是网络到主机的转换
// l, s 对应 32位和 16位数据，也就是 ipv4 地址 和 端口号
uint32_t htonl(uint32_t hostlong);
uint16_t htons(uint16_t hostshort);
uint32_t ntohl(uint32_t netlong);
uint16_t ntohs(uint16_t netshort);

// example1:
unsigned int a = 0x00 00 00 01;
htonl(a) = 0x01 00 00 00;
// example2:
unsigned int b = 0xff f0 00 00;
htonl(b) = 0x00 00 f0 ff;
```

### inet_pton 函数

将一个点分十进制的 `IP` 地址（字符串）转换为一个32位的整数。

```c
#include <arpa/inet.h>

int inet_pton(int af, const char *src, void *dst);

// example
char *ip = "192.168.1.1";
int ip_int = 0;
inet_pton(AF_INET, ip, &ip_int);
```

- af：地址族，可以是`AF_INET(ipv4)`、`AF_INET6(ipv6)`
- src：要转换的 `IP` 地址，如：`"192.168.1.1"`
- dst：转换后的网络字节序的 `IP` 地址，整数，如：`&ipv4_addr`
- 返回值：
  - 1，成功
  - 0，异常，说明 `src` 不是一个合法的ip地址
  - -1，`af` 不是一个合法的地址族

### inet_ntop 函数

将一个网络字节序的IP地址转换为一个点分十进制的IP地址（字符串）。

```c
#include <arpa/inet.h>

const char *inet_ntop(int af, const void *src,
                        char *dst, socklen_t size);

// example
struct sockaddr_in client_addr;
char ipv4_str[16];
inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, ipv4_str, sizeof(ipv4_str));
```

- af：地址族，可以是`AF_INET(ipv4)`、`AF_INET6(ipv6)`
- scr：网络字节序的IP地址
- dst：转换后的点分十进制的IP地址，缓冲区，`char buf[16]`
- size：dst 的大小
- 返回值：成功返回 dst, 失败返回 NULL

### inet_addr 函数

将一个点分十进制的IP地址（字符串）转换为一个32位的整数。

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

in_addr_t inet_addr(const char *cp);
```

- cp：要转换的IP地址，如：`192.168.10.1`
- 返回值：成功返回网络字节序的IP地址，失败返回 `INADDR_NONE(-1, 0xffffffff)`

使用这个函数可能会出现问题，因为 `INADDR_NONE` 也是一个有效的IP地址`255.255.255.255`！

### sockaddr 结构

同样是历史遗留问题，socket 相关的函数大多都是使用 `struct sockaddr` 结构，但现在使用的往往是 `struct sockaddr_in`，因此在传递参数时，要进行强制类型转换。

可在 `man 7 ip` 中查看相关说明

```c
struct sockaddr {
    sa_family_t sa_family;
    char        sa_data[14];
}

struct sockaddr_in {
    sa_family_t    sin_family; /* address family: AF_INET */
    in_port_t      sin_port;   /* 16位 网络字节序的端口号 */
    struct in_addr sin_addr;   /* internet address */
};

struct in_addr {
    uint32_t       s_addr;     /* 网络字节序的 IP地址 */
};

// example
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(80);

// 指定 ip 地址，方法一
int ipv4_addr;
inet_pton(AF_INET, "192.168.1.1", (void*)&ipv4_addr);
addr.sin_addr.s_addr = ipv4_addr;
// 指定 ip 地址，方法二
addr.sin_addr.s_addr = htonl(INADDR_ANY); // 表示取出系统中任意有效的IP 地址（二进制类型）
```

## socket 创建流程

![image-20220509201419178](../images/Linux%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B-socket/image-20220509201419178.png)

**服务器端**：

1. `socket()` 创建一个 socket（socket1）
2. `bind()` 绑定 IP + port（设置 socket1）
3. `listen()` 设置**同时**监听上限（socket1用于监听）
4. `accept()` 阻塞监听客户端连接（传入socket1以建立连接，创建socket2用于通信）
5. `read()/write()` 文件处理的系统调用函数，这里就直接像访问正常文件一样访问socket2

**客户端**：

1. `socket()` 创建一个 socket（socket3）
2. `connect()` 指定服务器的地址结构（IP+port）连接服务器（设置socket3）
3. `read()/write()`

注意，在上图的流程中，最终会有3个 socket 结构，服务器端的 socket1 仅用于监听，并不负责实际的与客户端通信，`accept()` 函数阻塞等待客户端连接，当有客户端连接时，`accept()` 函数中会新建一个 socket2 与客户端 socket3 绑定，实现客户端与服务器端的通信。

### socket 函数

创建一个套接字，并返回一个套接字描述符。

```c
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int socket(int domain, int type, int protocol);

// example
int sockfd = socket(AF_INET, SOCK_STREAM, 0);   // TCP
int sockfd = socket(AF_INET, SOCK_DGRAM, 0);    // UDP
```

- domain：地址族，常用三个为 `AF_INET(ipv4)`、`AF_INET6(ipv6)`、`AF_UNIX(本地套接字)`
- type：套接字类型，常用两个为 `SOCK_STREAM(TCP)`、`SOCK_DGRAM(UDP)`
- protocol：协议，一般设置为 0
- 返回值：成功返回套接字描述符，失败返回 -1，并设置 errno

### bind 函数

给 socket 绑定一个地址结构（`IP + port`）

```c
#include <sys/types.h>
#include <sys/socket.h>

int bind(int sockfd, const struct sockaddr *addr,
        socklen_t addrlen);

// example
struct sockaddr_in addr;
addr.sin_family = AF_INET; // 类型要与 socket 创建时的 domain 一致
addr.sin_port = htons(80);
addr.sin_addr.s_addr = htonl(INADDR_ANY);
bind(socket1, (struct sockaddr*)&addr, sizeof(addr));
```

- sockfd：套接字文件描述符
- addr：地址结构（传入参数）
- addrlen：地址结构的大小 `sizeof(addr)`
- 返回值：成功返回 0，失败返回 -1，并设置 errno

### listen 函数

设置**同时**与服务器建立连接上限数（同时进行TCP三次握手的客户端数量）

```c
#include <sys/types.h>
#include <sys/socket.h>

int listen(int sockfd, int backlog);

// example
listen(sockfd, 5);
```

- sockfd：套接字文件描述符
- backlog：上限数量，最大值为 `128`！
- 返回值：成功返回 0，失败返回 -1，并设置 errno

### accept 函数

阻塞等待客户端连接，当有客户端连接时，会新建一个 socket2 与客户端 socket3 绑定，实现客户端与服务器端的通信，返回新建的 socket2 的文件描述符。

```c
#include <sys/types.h>
#include <sys/socket.h>

int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

// example
struct sockaddr_in client_addr;
socklen_t client_addrlen = sizeof(client_addr);
int sockfd2;
again:
    sockfd2 = accept(sockfd1, (struct sockaddr *)&client_addr, &client_addrlen);
    if (sockfd2 < 0) {
        if (errno == EINTR || errno == ECONNABORTED)
            goto again;
        else {
            perror("accept error");
            exit(1);
        }
    }
```

- sockfd：传入的套接字，上面绑定了地址的 socket1
- addr：传出参数，返回成功建立连接的**客户端**的地址结构
- addrlen：传入传出参数，传入为参2 addr 的大小，传出为客户端的 addr 的实际大小
- 返回值：成功返回新建的 socket2 文件描述符，失败返回 -1，并设置 errno：
  - EINTR：被信号中断
  - ECONNABORTED：连接被拒绝

### connect 函数

客户端连接到服务器！

客户端 socket 可以不需要手动 bind，系统会自动绑定本地的地址结构！

```c
#include <sys/types.h>
#include <sys/socket.h>

int connect(int sockfd, const struct sockaddr *addr,
            socklen_t addrlen);

// example
struct sockaddr_in server_addr;
server_addr.sin_family = AF_INET;
server_addr.sin_port = htons(80);
server_addr.sin_addr.s_addr = inet_addr("192.168.1.1");
connect(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr));
```

- sockfd：套接字文件描述符
- addr：服务器的地址结构
- addrlen：地址结构的大小 `sizeof(addr)`
- 返回值：成功返囟 0，失败返囟 -1，并设置 errno

### TCP example

简单的 TCP 收发测试！

```c
// 服务器端
#include <arpa/inet.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

#define SERVER_PORT 10001

int main() {
    // 1. 创建 socket
    int socketfd1 = socket(AF_INET, SOCK_STREAM, 0);

    // 2. 绑定地址(ip, port)
    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    int res = bind(socketfd1, (struct sockaddr*)&server_addr,
                    sizeof(server_addr));

    // 3. 设置监听上限
    res = listen(socketfd1, 128);

    // 4. 阻塞等待连接
    struct sockaddr_in client_addr;
    socklen_t len = sizeof(client_addr);
    int socketfd2 = accept(socketfd1, (struct sockaddr*)&client_addr, &len);

    // 5. read 读取数据
    char buf[1500];

    read(socketfd2, buf, 1500);
    printf("recv data: %s\n", buf);

    char *data = "hello!";
    write(socketfd2, data, strlen(data));


    close(socketfd1);
    close(socketfd2);
    return 0;
}
```

```c
// 客户端
#include <arpa/inet.h>
#include <netinet/in.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

#define SERVER_PORT 10001
#define SERVER_IP "192.168.0.8"

int main() {
    // 1. 创建 socket
    int socketfd3 = socket(AF_INET, SOCK_STREAM, 0);

    // 2. 连接到服务器
    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    server_addr.sin_addr.s_addr = inet_addr(SERVER_IP);
    int res = connect(socketfd3, (struct sockaddr*)&server_addr,
                      sizeof(server_addr));

    // 3. write 写数据
    char *data = "hello world!";
    write(socketfd3, data, strlen(data));

    char buf[1500];
    read(socketfd3, buf, 1500);
    printf("recv data: %s\n", buf);

    close(socketfd3);
    return 0;
}
```

`nc` 命令也可以用来做客户端测试！

```bash
nc 192.168.0.8 10001
```

## read 函数

从指定的文件描述符上最多读取 `count` 个字节存放到缓冲区 `buf` 中，返回实际读取的字节数。

```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t count);
```

在网络编程中，read 函数的返回值需要仔细区分：

1. `>0` 实际读到的字节数；
2. `=0` 已经读到结尾，说明**对端socket已关闭**（重点！！）；
3. `-1` 应该进下一步判断 `errno` 的值：
   - `EAGAIN/ENOULDBLOCK` 设置了非阻塞方式读，没有数据到达
   - `EINTR` 慢速系统调用被 中断
   - 其他情况，异常

## write 函数

将缓冲区 `buf` 中的 `count` 个字节数据写入到 `fd` 对应的文件中，返回实际写入的字节数。

```c
#include <unistd.h>

ssize_t write(int fd, const void *buf, size_t count);
```

## recv/recvfrom 函数

从指定的 `sockfd` 上接收（读取）数据写入到缓冲区 `buf`中，`buf`的长度是`len`，返回实际接收的字节数。

```c
#include <sys/types.h>
#include <sys/socket.h>

ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                 struct sockaddr *src_addr, socklen_t *addrlen);
```

`flags` 参数可以用来指定该函数的特殊行为。常用的 `flags` 参数值有：

- `0`：不设置标志。
- `MSG_PEEK`：该标志会导致 `recv` 函数返回套接字中的数据，但不会从缓冲区中删除数据。下一次调用 `recv` 函数会再次返回相同的数据。这个标志对检查接收到的数据而不消耗它很有用。
- `MSG_WAITALL`：该标志导致 `recv` 函数阻塞，直到接收到请求的字节数或连接关闭。如果在接收到请求的字节数之前连接关闭，`recv` 函数会返回已接收的字节数。
- `MSG_OOB`：该标志导致 `recv` 函数返回带外数据，如果有可用的带外数据。带外数据是单独从正常数据流发送的数据，通常用于紧急数据。
- `MSG_DONTWAIT`：该标志等同于 `O_NONBLOCK` 文件状态标志。当设置该标志时，`recv` 函数不会阻塞，如果没有数据可用，则返回 `-1` 并带有错误代码 `EAGAIN` 或 `EWOULDBLOCK`。

这些标志可以使用位运算符 `|` 组合，例如：

```c
int n = recv(sockfd, buffer, 1024, MSG_PEEK | MSG_WAITALL);
```

## send/sendto 函数

将缓冲区`buf`中的`len`个字节的数据发送到对应的`sockfd`，返回实际发送的字节数。

```c
#include <sys/types.h>
#include <sys/socket.h>

ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
               const struct sockaddr *dest_addr, socklen_t addrlen);
```

> `send()` / `write()` 的一点区别：
>
> 首先，这两者都是系统调用，都是用来向文件描述符（包括`Socket`文件描述符）写入数据的。区别在于，`send()` 系统调用可以指定一些可选的参数，例如`flags`参数用来指定发送数据的方式，如非阻塞方式和带外方式等。`send()`函数比`write()`函数更加灵活，因此在网络编程中更常用。

## get/setsockopt 函数

用于设置套接字的一些特殊选项！

```c
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int getsockopt(int sockfd, int level, int optname,
               void *optval, socklen_t *optlen);
int setsockopt(int sockfd, int level, int optname,
               const void *optval, socklen_t optlen);
```

- `socket`：表示需要设置选项的套接字的描述符。
- `level`：表示选项的协议层，通常是 `SOL_SOCKET`，表示套接字选项。
- `optname`：表示选项的名称，根据不同的协议层，选项可能不同。
- `optval`：指向包含选项值的缓冲区的指针。
- `optlen`：缓冲区的大小。

### SO_SNDBUF/SO_RCVBUF

`SO_SNDBUF` 选项控制发送缓冲区的大小，即从应用程序发送到套接字的数据在内核中的缓冲区大小。如果应用程序发送的数据量大于该缓冲区大小，则必须等待先前发送的数据完成发送才能继续发送数据。

`SO_RCVBUF` 选项控制接收缓冲区的大小，即从网络接收到的数据在内核中的缓冲区大小。如果从网络接收的数据量大于该缓冲区大小，则新接收到的数据可能会覆盖先前未读取的数据。

通过使用 `setsockopt` 函数，可以修改发送缓冲区和接收缓冲区的大小，以适应应用程序的数据传输需求。但是，修改缓冲区大小并不能保证性能提升，因为实际情况可能受到许多因素的影响。

### SO_LINGER

```C
struct linger {
    int l_onoff;        /* Nonzero to linger on close.  */
    int l_linger;       /* Time to linger.  */
};
```

`SO_LINGER` 选项是一个套接字选项，用于控制在关闭套接字时是否等待正在发送的数据的完成。

当关闭套接字时，如果有数据正在发送，那么通常有两种处理方式：

- 如果未启用 `SO_LINGER` 选项，那么套接字会立即关闭，未发送完的数据会被丢弃。
- 如果启用了 `SO_LINGER` 选项，则套接字将等待所有正在发送的数据完成，直到指定的时间到达为止。

具体的，可以使用 `setsockopt` 函数启用 `SO_LINGER` 选项，并使用一个结构体来指定等待的时间，该结构体由两个成员：

- `l_onoff`：用于控制是否启用 `SO_LINGER` 选项。
- `l_linger`：表示等待时间，单位为秒。

如果 `l_onoff` 设置为非零值，则启用 `SO_LINGER` 选项，并使用 `l_linger` 指定的等待时间。如果 `l_onoff` 设置为零，则禁用 `SO_LINGER` 选项，关闭套接字时不等待数据的完成。

### SO_KEEPALIVE

`SO_KEEPALIVE` 选项用于控制是否启用 TCP 连接的 keep-alive 检测机制。当启用该选项时，如果两端在一段时间内没有数据交互，TCP 协议会发送 keep-alive 数据包来检测对端是否仍然可用。如果多次尝试后对端仍然无法响应，则可以断开该连接。

这个选项可以用于防止长时间空闲的连接被网络中断，从而导致不必要的等待和重试。在某些情况下，它也可以用于检测对端的不存在或故障。

### SO_REUSEADDR

`SO_REUSEADDR` 选项用于控制在同一个主机上是否允许多个套接字绑定到同一端口。

在默认情况下，如果一个套接字已经绑定到某个端口，那么其他套接字将不能再绑定到该端口。这种限制有助于防止在同一端口上的竞争冲突。

但是，有时需要多个套接字共享同一端口，例如当同一主机上有多个服务器程序运行时。在这种情况下，可以使用 `SO_REUSEADDR` 选项来允许多个套接字绑定到同一端口。

### SO_REUSEPORT

`SO_REUSEPORT` 选项是一种扩展的套接字选项，主要用于允许多个套接字绑定到同一端口，从而共享同一个端口。这个选项与 `SO_REUSEADDR` 选项类似，但是提供了更多的灵活性和更高的性能。

在默认情况下，如果一个套接字已经绑定到某个端口，那么其他套接字将不能再绑定到该端口。但是，如果多个套接字都启用了 `SO_REUSEPORT` 选项，那么多个套接字就可以同时绑定到同一端口。

这个选项对于提高网络服务的性能和可靠性非常重要，因为它可以允许多个进程或线程共享同一端口，从而利用多核处理器的优势。

使用 `setsockopt` 函数，可以在套接字创建后启用或禁用 `SO_REUSEPORT` 选项。请注意，`SO_REUSEPORT` 选项可能只在某些操作系统上可用，并且可能需要在编译套接字应用程序时启用特定的宏定义才能使用该选项。

## 相关资料

- [listen 和 accpet 函数说明](https://www.bilibili.com/video/BV1iJ411S7UA?p=18&spm_id_from=pageDriver)
- [TCP 三次握手与四次挥手](https://xiaolincoding.com/network/3_tcp/tcp_interview.html)
