---
title: OpenVSwitch-DPDK-区分服务
top_img: transparent
date: 2022-06-01 18:52:24
updated: 2022-06-01 18:52:24
tags:
    - OpenVSwitch
    - DPDK
categories: OpenVSwitch
keywords:
description:
---

## IP 区分服务 TOS

![network_protocol-IPv4](../images/OpenVSwitch-DPDK-%E5%8C%BA%E5%88%86%E6%9C%8D%E5%8A%A1/network_protocol-IPv4.drawio.png)

由 8 比特构成，用来表明服务质量。每一位的具体含义如下表所示：

| 比特  | 含义       |
| :-----: | :---------- |
| 0 1 2 | 优先度     |
| 3     | 最低延迟   |
| 4     | 最大吞吐   |
| 5     | 最大可靠性 |
| 6     | 最小代价   |
| 3~6   | 最大安全   |
| 7     | 未定义     |

这个值通常由应用指定。而且现在也鼓励这种结合应用的特性设定 TOS 的方法。然而在目前，几乎所有的网络都无视这些字段。这不仅仅是因为在符合质量要求的情况下按其要求发送本身的功能实现起来十分困难，还因为若不符合质量要求就可能会产生不公平的现象。因此，实现 TOS 控制变得极其复杂。这也导致TOS 整个互联网几乎就没有被投入使用。不过已有人提出将 TOS 字段本身再划分为 DSCP 和 ECN 两个字段的建议。

## sk_buff->priority

Linux 内核在收到数据包后，经过内核协议栈处理，将包的信息提取并记录[在 sk_buff 结构](https://elixir.bootlin.com/linux/v4.16.18/source/include/linux/skbuff.h#L800)中。

```c
__u32           priority;
```

可以看到，`sk_buff`中的 priority 并非是 8 位，而是 32 位！这是因为 priority 与 IP 包头中的 TOS 并没有直接的对应关系！

```c
skb->priority = rt_tos2priority(val)
...
static inline char rt_tos2priority(u8 tos)
{
    return ip_tos2prio[IPTOS_TOS(tos)>>1];
}
...
#define IPTOS_TOS_MASK      0x1E
#define IPTOS_TOS(tos)      ((tos)&IPTOS_TOS_MASK)
const __u8 ip_tos2prio[16]={0,0,0,0,2,2,2,2,6,6,6,4,4,4,4};
```

也就是说：

```c
skb->priority = ip_tos2prio[(tos & 0x1E) >> 1];
```

`skb->priority`字段的主要用途就是区分发送队列，**需要注意，该字段并非`IP`头中的 TOS，修改该字段也不会改变 TOS 的值！**

linux 内核中的流量控制就是基于`skb->priority`来区分队列的，但往往也不是直接用`skb->priority`作为队列号，而是再进行一次映射，不同的排队规则Qdisc映射关系可能不一，pfifo_fast 的实现如下：

```c
int band = prio2band[skb->priority & TC_PRIO_MAX];
#define TC_PRIO_MAX         15
static const u8 prio2band[TC_PRIO_MAX + 1] = {
    1, 2, 2, 2, 1, 2, 0, 0 , 1, 1, 1, 1, 1, 1, 1, 1
};
```

OVS 添加流表时可以指定 `actions=set_queue:3`，就会将 `skb->priority` 修改为指定的数值，用于后续的流量控制！

## dp_packet->md->priority

在 OVS 基于 DPDK 的用户态数据通路中，数据帧经过 DPDK 用户态协议栈解析后用 `rte_mbuf` 结构表示，再经过封装为 `dp_packet` 结构后，才作为数据包的结构体。同样，`dp_packet` 中也有 `priority`字段。

```bash
struct dp_packet {
...
    struct pkt_metadata {
        uint32_t skb_priority;
        ...
    }
}
```

但在 PMD 轮询处理中，出于性能考虑，该字段并没有进行有效初始化！！而是简单地初始化为0！具体需要看 PMD 的处理流程，但结论就是这样！下面只展示关键过程：

```c
1. 从指定队列读取一批数据包
netdev_rxq_recv(rxq->rx, &batch, qlen_p);	// dpif-netdev.c
    1.1 从 vhost 端口的队列读取数据包，
    netdev_dpdk_vhost_rxq_recv(rxq, batch)	// netdev-dpdk.c
        1.1.1 传入batch，读入对应的 mbuf，注意后续并没有对 dp_packet metadata 进行初始化
        rte_vhost_dequeue_burst(vid, qid, dev->dpdk_mp->mp,
                                    (struct rte_mbuf **) batch->packets,
                                    NETDEV_MAX_BURST);
2. 处理数据包（流表匹配、转发），注意传入参数 md_is_valid = false;
dp_netdev_input__(pmd, packets, false, port_no);	// dpif-netdev.c
    2.1 EMC/SMC 流表匹配，注意传入参数 md_is_valid = false;
    /* 该函数的注释中也做了说明，md_is_valid = false 时，metadata 并未初始化，在该函数中只简单地
     * 初始化了 in_port 端口号为 port_no（入端口号），并没有对 skb_priority 进行初始化
     * For performance reasons a caller may choose not to initialize the metadata
     * in 'packets_'.  If 'md_is_valid' is false, the metadata in 'packets'
     * is not valid and must be initialized by this function using 'port_no'.
     * If 'md_is_valid' is true, the metadata is already valid and 'port_no'
     * will be ignored.
     */
    dfc_processing(pmd, packets, keys, missed_keys, batches, &n_batches,
                   flow_map, &n_flows, index_map, md_is_valid, port_no);
        2.1.1 初始化 metadata 的入端口号
        pkt_metadata_init(&packet->md, port_no)     // packets.h
        {
            /* This is called for every packet in userspace datapath and affects
             * performance if all the metadata is initialized. Hence, fields should
             * only be zeroed out when necessary.
             *
             * Initialize only till ct_state. Once the ct_state is zeroed out rest
             * of ct fields will not be looked at unless ct_state != 0.
             */
            memset(md, 0, offsetof(struct pkt_metadata, ct_orig_tuple_ipv6));

            /* It can be expensive to zero out all of the tunnel metadata. However,
             * we can just zero out ip_dst and the rest of the data will never be
             * looked at. */
            md->tunnel.ip_dst = 0;
            md->tunnel.ipv6_dst = in6addr_any;
            md->in_port.odp_port = port;
            md->orig_in_port = port;
            md->conn = NULL;
        }
```

因此，要使用 IP 包头中的 TOS 字段，只能自己添加用 TOS 初始化 `skb_priority` 的实现代码。

虽然上面的步骤中没有涉及到解析数据包的头部信息（L2/L3/L4），但要进行流表匹配，又必须要有这些信息，但这里只将这些信息提取到了`miniflow`结构中！

```c
2.1.2 提取信息到 miniflow 结构中！该函数就相当于协议栈，完成了对数据包 header 的解析
miniflow_extract(packet, &key->mf);		// dpif-netdev.c
```

只需要在该函数中解析 `ip_tos` 后赋值给 `md->skb_priority` 即可！

同样，后续还可能存在其他操作会修改 `md->skb_priority` 字段，但并不会改变数据包中 `TOS` 的值，**如果需要修改 `TOS` ，可参考函数`packet_set_ipv4() lib/packets.c`，会涉及到重新计算校验和等。**

## 相关资料

- 《图解TCP/IP》
- [skb->priority, ip tos, ping -Q的关系](https://stackoverflow.com/questions/19810839/skb-priority-and-iptos-and-ping-q)
