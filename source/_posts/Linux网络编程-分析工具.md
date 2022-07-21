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

| 名称    | 功能                                                         |
| :------ | :----------------------------------------------------------- |
| tcpdump | 抓包工具                                                     |
| lsof    | 查看端口被谁占用                                             |
| netstat | 打印网络连接、路由表、连接的数据统计、伪装连接以及广播域成员      |
| iftop   | 监控端口网速                                                 |
| telnet  |                                                              |
| nc      |                                                              |
| ss      | `socket statistics`                                          |
| wrk     |                                                              |

## wireshark

## tcpdump

## lsof

## netstat

打印网络连接、路由表、连接的数据统计、伪装连接以及广播域成员。

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

## iftop

## telnet

## nc

## ss

## wrk

## 相关资料

- [全网最详细的 tcpdump 使用指南](https://www.cnblogs.com/wongbingming/p/13212306.html)
- [tcpdump简明教程](https://github.com/mylxsw/growing-up/blob/master/doc/tcpdump%E7%AE%80%E6%98%8E%E6%95%99%E7%A8%8B.md)
- [netstat 的10个基本用法](https://linux.cn/article-2434-1.html)
- [ss 的10个基本用法](https://www.binarytides.com/linux-ss-command/)
- [如何在Linux下统计高速网络中的流量](https://linux.cn/article-2493-1.html)
