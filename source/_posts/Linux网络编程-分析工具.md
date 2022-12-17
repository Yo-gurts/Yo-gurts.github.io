---
title: Linux网络编程-分析工具
top_img: transparent
date: 2022-07-17 19:32:32
updated: 2022-07-17 19:32:32
tags:
  - Linux
categories: Linux
keywords:
description: 网络分析工具汇总
---

| 名称         | 功能                                                         |
| :----------- | :----------------------------------------------------------- |
| **dstat**    | 查看各种系统资源的统计信息，可保存到文件！                   |
| **iftop**    | display bandwidth usage on an interface by host. 可指定TCP/UDP端口 |
| **ip**       | show / manipulate routing, network devices, interfaces and tunnels |
| **ip link**  |                                                              |
| **ip route** |                                                              |
| **iptables** | administration tool for IPv4/IPv6 packet filtering and NAT   |
| **lsof**     | list open files. 查看端口被谁占用                            |
| **nc**       | arbitrary TCP and UDP connections and listens.               |
| **netstat**  | 打印网络连接、路由表、连接的数据统计、伪装连接以及广播域成员 |
| **ss**       | another utility to investigate sockets.                      |
| **tcpdump**  | 抓包工具                                                     |
| **telnet**   | user interface to the TELNET protocol.                       |
| **wrk**      | http 压测工具                                                |
| **top**      |                                                              |

## dstat

> - [dstat命令详解](https://www.cnblogs.com/zh-dream/p/12081455.html)

## iftop

> - [iftop命令详解](https://www.cnblogs.com/yinzhengjie/p/6223467.html)

## ip

## iptables

> - [iptables 中文文档](https://wangchujiang.com/linux-command/c/iptables.html)
> - [iptables - Linux man page](https://linux.die.net/man/8/iptables)
> - [Ubuntu iptables配置](https://blog.51cto.com/yangzhiming/1982814)

**注意：规则的次序非常关键，`谁的规则越严格，应该放的越靠前`，而检查规则的时候，是按照从上往下的方式进行检查的。**

1、`Ubuntu` 默认有装`iptables`，可通过`dpkg -l`或`which iptables`确认

2、`Ubuntu` 默认没有`iptables`配置文件，需通过`iptables-save > /etc/network/iptables.up.rules`生成

3、`iptables`配置文件路径及文件名建议为`/etc/network/iptables.up.rules`，因为执行`iptables-apply`默认指向该文件，也可以通过-w参数指定文件

4、`Ubuntu` 没有重启`iptables`的命令，执行`iptables-apply`生效

5、`Ubuntu iptables`默认重启服务器后清空，需在`/etc/network/interfaces`里写入`pre-up iptables-restore < /etc/network/iptables.up.rules`才会开机生效

## lsof

> - [lsof 文档](https://wangchujiang.com/linux-command/c/lsof.html)

## nc

> - [nc工具使用](https://www.cnblogs.com/zhaijiahui/p/9028402.html#autoid-3-2-3)

## ss

> - [ss命令详解](https://www.cnblogs.com/machangwei-8/p/10352986.html)

## tcpdump

![img](../images/Linux%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B-%E5%88%86%E6%9E%90%E5%B7%A5%E5%85%B7/20200628111325.png)

| 命令                              | 说明                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| `tcpdump -n`                      | 不把IP转化成域名，直接显示 ，**若不加，使用`ctrl+c`停止时会很慢** |
| `tcpdump -nn`                     | 不把协议和端口号转化成名字，速度会更快                       |
| `tcpdump host www.baidu.com`      | 抓取某个网站                                                 |
| `tcpdump host 192.168.10.100`     | 基于**IP地址**过滤                                           |
| `tcpdump host src 192.168.10.100` | 基于**源IP地址**过滤                                         |
| `tcpdump ip dst 192.168.10.100`   | 基于**目的IP地址**过滤                                       |
| `tcpdump net 192.168.10.0/24`     | 基于**网段**进行过滤                                         |
| `tcpdump tcp port 8088`           | 基于**端口**进行过滤                                         |
| `tcpdump -nn tcp src port 10000`  | 基于传输层协议+端口过滤                                      |
| `tcpdump port 80 or port 8088`    | 同时指定两个端口                                             |
| `tcpdump portrange 8000-8080`     | 指定一个端口范围                                             |
| `tcpdump icmp`                    | 基于协议进行过滤`ip ip6 arp icmp rarp`                       |
| `tcpdump ip proto 6`              | 基本IP协议的版本进行过滤                                     |
| `tcpdump -i eth0`                 | 过滤指定网卡的数据包                                         |
| `tcpdump -e`                      | 每行的打印输出中将包括数据包的**数据链路层头部信息**         |
| `tcpdump ether host src [mac]`    | 根据 mac 地址进行过滤                                        |
| `tcpdump -c 100`                  | **捕获100个包就退出**                                        |
| `tcpdump -A`                      | **以ASCII码方式显示**每一个数据包(不显示链路层头部信息)      |
| `tcpdump -w icmp.pcap`            | 抓到的包数据生成到文件中，以`.pcap`为后缀                    |
| `tcpdump -r icmp.pcap`            | 从文件中读取包数据                                           |

- **组合过滤**
  - and：所有的条件都需要满足，也可以表示为 `&&`
  - or：只要有一个条件满足就可以，也可以表示为 `||`
  - not：取反，也可以使用 `!`

  ```bash
  tcpdump -nn 'ether src 04:42:1a:ea:da:ad && ip src 192.168.0.3 and udp src port 10000'
  ```

- **控制详细内容的输出**
  - `-v`：产生详细的输出. 比如包的TTL，id标识，数据包长度，以及IP包的一些选项。同时它还会打开一些附加的包完整性检测，比如对IP或ICMP包头部的校验和。
  - `-vv`：产生比-v更详细的输出. 比如NFS回应包中的附加域将会被打印, SMB数据包也会被完全解码。（摘自网络，目前我还未使用过）
  - `-vvv`：产生比-vv更详细的输出。比如 telent 时所使用的SB, SE 选项将会被打印, 如果telnet同时使用的是图形界面，其相应的图形选项将会以16进制的方式打印出来（摘自网络，目前我还未使用过）
- **控制时间的显示**
  - `-t`：在每行的输出中不输出时间
  - `-tt`：在每行的输出中会输出时间戳
  - `-ttt`：输出每两行打印的时间间隔(以毫秒为单位)
  - `-tttt`：在每行打印的时间戳之前添加日期的打印（此种选项，输出的时间最直观）
- **显示数据包的头部**
  - `-x`：以16进制的形式打印每个包的头部数据（但不包括数据链路层的头部）
  - `-xx`：以16进制的形式打印每个包的头部数据（包括数据链路层的头部）
  - `-X`：以16进制和 ASCII码形式打印出每个包的数据(但不包括连接层的头部)，这在分析一些新协议的数据包很方便。
  - `-XX`：以16进制和 ASCII码形式打印出每个包的数据(包括连接层的头部)，这在分析一些新协议的数据包很方便。

> - [推荐先看这个：全网最详细的 tcpdump 使用指南](https://www.cnblogs.com/wongbingming/p/13212306.html)
> - [tcpdump简明教程](https://github.com/mylxsw/growing-up/blob/master/doc/tcpdump%E7%AE%80%E6%98%8E%E6%95%99%E7%A8%8B.md)
> - [tcpdump 示例教程](https://colobu.com/2019/07/16/a-tcpdump-tutorial-with-examples/)

## netstat

打印网络连接、路由表、连接的数据统计、伪装连接以及广播域成员。

> [Netstat 备忘清单](https://wangchujiang.com/reference/docs/netstat.html)

| 命令            | 说明                                                 |
| --------------- | --------------------------------------------------- |
| `netstat -a`    | 列出 `tcp`, `udp` 和 `unix` 协议下所有套接字的所有连接    |
| `netstat -t`    | 只列出 `TCP` 协议的连接                                |
| `netstat -u`    | 只列出 `UDP` 协议的连接                                |
| `netstat -ant`  | 禁用反向域名解析（不显示主机名），加快查询速度              |
| `netstat -tnl`  | 只列出监听中的连接，`-l`                               |
| `netstat -nlpt` | 获取进程名、进程号以及用户 ID，`-p` 进程信息              |
| `netstat -ltpe` | 使用 `-ep` 选项可以同时查看进程名和用户名。               |
| `netstat -s`    | 打印统计数据                                          |
| `netstat -rn`   | 显示内核路由信息                                       |
| `netstat -i`    | 打印网络接口                                          |
| `netstat -ct`   | 使用 `netstat` 的 `-c` 选项持续输出信息                 |
| `netstat -g`    | 显示多播组信息                                         |

## telnet

## wrk

## top

Linux 的 top 命令用于实时显示系统中**各个进程**的 CPU 和内存使用情况。

> [top命令详解](https://www.cnblogs.com/ggjucheng/archive/2012/01/08/2316399.html)

top 能查看的信息有：

## 相关资料

- [netstat 的10个基本用法](https://linux.cn/article-2434-1.html)
- [ss 的10个基本用法](https://www.binarytides.com/linux-ss-command/)
- [如何在Linux下统计高速网络中的流量](https://linux.cn/article-2493-1.html)
- [Linux下路由配置](https://blog.csdn.net/zqixiao_09/article/details/79165925)
