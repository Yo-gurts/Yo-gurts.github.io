---
title: 解析命令行选项
top_img: transparent
date: 2022-10-05 23:03:51
updated: 2022-10-05 23:03:51
tags:
  - c
  - Linux
categories: c
keywords:
description: 解析命令行选项的库函数
---

支持短选项和长选项！

- **短选项**：形如`-a`，`-v`，以`-`加单个字符组成。`-`叫做选项标识符，选项标识符后面可以紧跟一个字符，这个字符叫做**选项字符**。选项字符后面可以**紧跟**一个或多个字符，这些字符叫做**选项参数**。如`-lpthread`选项字符为`l`，参数为`pthread`。
- **长选项**：形如`--all`，`--version`，以`--`加一个单词组成，用`=`跟参数。如`--data=format`选项为`data`，参数为`format`。

```bash
-lpthread -d format
--data=format
```

## 短选项解析 `getopt()`

```c
#include <unistd.h>

int getopt(int argc, char * const argv[],
           const char *optstring);

extern char *optarg;
extern int optind, opterr, optopt;
```

- `argc, argv`为`main`函数参数中的`argc, argv`，代表命令行参数！
- `optstring`为
- `optind`为`argv`中下一个待处理的选项下标。默认值为1，当然如果要重新解析`argv`，可以手动重置其为1。

## 长命令

```c
#include <getopt.h>
#include <unistd.h>

static void
parse_client_options(int argc, char *argv[])
{
    enum {
        OPT_SRCIP = 256,
        OPT_DSTIP,
        OPT_SRCPORT,
        OPT_DSTPORT
    };

    // : 选项后面必须有参数
    static char *short_options = "hp:t:s:n:i:b:";
    static struct option long_options[] = {
        {"help",      no_argument,       NULL, 'h'},
        {"srcip",     required_argument, NULL, OPT_SRCIP },
        {"dstip",     required_argument, NULL, OPT_DSTIP },
        {"srcport",   required_argument, NULL, OPT_SRCPORT },
        {"dstport",   required_argument, NULL, OPT_DSTPORT },
        {"proto",     required_argument, NULL, 'p'},
        {"tos",       required_argument, NULL, 't'},
        {"pktsize",   required_argument, NULL, 's'},
        {"interval",  required_argument, NULL, 'i'},
        {"bandwidth", required_argument, NULL, 'b'},
        {"pktnums",   required_argument, NULL, 'n'},
        {0,         0,                 0,  0 }
    };

    int opt;
    while((opt = getopt_long_only(
                 argc, argv, short_options, long_options, NULL)) != -1) {

        switch(opt) {
            case 'h': client_help_info(); exit(EXIT_SUCCESS);
            case 'p':
                pkts.proto = atoi(optarg);
                if (pkts.proto != TCP && pkts.proto != UDP) {
                    printf("参数错误: %d，只能为TCP(6)/UDP(17)\n", pkts.proto);
                    exit(EXIT_FAILURE);
                }
                break;
            case 't': pkts.tos = atoi(optarg); break;
            case 's':
                pkts.pktsize = atoi(optarg);
                if (pkts.pktsize < 64 || pkts.pktsize > 1500) {
                    printf("数据包大小应该在[64, 1500]的范围内\n");
                    exit(EXIT_FAILURE);
                }
                break;
            case 'n': pkts.pktnums = atoi(optarg); break;
            case 'i': pkts.interval = atoi(optarg); break;
            case 'b': pkts.bandwidth = atoi(optarg); break;
            case OPT_SRCIP: strncpy(pkts.srcip, optarg, 16); break;
            case OPT_DSTIP: strncpy(pkts.dstip, optarg, 16); break;
            case OPT_SRCPORT: pkts.srcport = atoi(optarg); break;
            case OPT_DSTPORT: pkts.dstport = atoi(optarg); break;
            default:
                client_help_info();
                exit(EXIT_FAILURE);
        }
    }
}
```



## 短命令 only

```c
/*
 * 解析命令行参数处理函数
 */
void parse_args(int argc, char **argv) {
    int opt;

    mdelay = 1000000;      // 默认刷新率为1秒
    x1 = 0;                // 默认区域为全屏
    x2 = vinfo.xres - 1;
    y1 = 0;
    y2 = vinfo.yres - 1;

    while ((opt = getopt(argc, argv, "hrsd:m:x:y:")) != -1) {
        switch (opt) {
            case 'r': // 全屏刷新标志
                rect = true;
                break;
            case 's':
                show_fps = true;
                break;
            case 'd': // 帧间隔 ms
                mdelay = atoi(optarg) * 1000;
                break;
            case 'm':
                mode = atoi(optarg);
                if (mode > 1 || mode < 0) {
                    printf("mode must be 0 or 1\n\n");
                    help_info();
                    exit(EXIT_FAILURE);
                }
                break;
            case 'x': // 指定x区域
                sscanf(optarg, "%d:%d", &x1, &x2);
                break;
            case 'y': // 指定y区域
                sscanf(optarg, "%d:%d", &y1, &y2);
                printf("y1, y2: %d, %d\n", y1, y2);
                break;
            case 'h': // FALLTHROUGH
            default: /* '?' */
                help_info();
                exit(EXIT_FAILURE);
        }
    }
}
```





## 其他资料

- [getopt例子](https://www.cnblogs.com/liwei0526vip/p/4873111.html)
- [getopt系列函数](https://blog.csdn.net/liao20081228/article/details/76557548)
- [getopt](https://blog.csdn.net/Mculover666/article/details/106646339)
