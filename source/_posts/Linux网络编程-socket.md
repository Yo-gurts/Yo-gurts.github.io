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

在网络编程中，read 函数的返回值需要仔细区分：

1. `>0` 实际读到的字节数；
2. `=0` 已经读到结尾，说明**对端socket已关闭**（重点！！）；
3. `-1` 应该进下一步判断 `errno` 的值：
   - `EAGAIN/ENOULDBLOCK` 设置了非阻塞方式读，没有数据到达
   - `EINTR` 慢速系统调用被 中断
   - 其他情况，异常

## 相关资料

- [listen 和 accpet 函数说明](https://www.bilibili.com/video/BV1iJ411S7UA?p=18&spm_id_from=pageDriver)
- [TCP 三次握手与四次挥手](https://xiaolincoding.com/network/3_tcp/tcp_interview.html)
