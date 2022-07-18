---
title: OpenVSwitch-测试拓扑
top_img: transparent
date: 2022-07-18 10:57:20
updated: 2022-07-18 10:57:20
tags:
  - OpenVSwitch
  - DPDK
categories: OpenVSwitch
keywords:
description: 只搭建拓扑，但相关配置参数不一定最优
---

|                            topo1                             |
| :----------------------------------------------------------: |
| ![](../images/OpenVSwitch-%E6%B5%8B%E8%AF%95%E6%8B%93%E6%89%91/PKTGEN-OVS-DPDK-TESTPMD.png) |
|                          **topo2**                           |
| ![ovs-ovs](../images/OpenVSwitch-%E6%B5%8B%E8%AF%95%E6%8B%93%E6%89%91/QEMU-OVS-QEMU.png) |
|                          **topo3**                           |
| ![ovs-ovs](../images/OpenVSwitch-%E6%B5%8B%E8%AF%95%E6%8B%93%E6%89%91/ovs-ovs.png) |
|                          **topo4**                           |
| ![ovs-ovs-ovs](../images/OpenVSwitch-%E6%B5%8B%E8%AF%95%E6%8B%93%E6%89%91/ovs-ovs-ovs.png) |
|                          **topo5**                           |
| ![ovs-ovs-ovs2](../images/OpenVSwitch-%E6%B5%8B%E8%AF%95%E6%8B%93%E6%89%91/ovs-ovs-ovs2.png) |

## topo1

![](../images/OpenVSwitch-%E6%B5%8B%E8%AF%95%E6%8B%93%E6%89%91/PKTGEN-OVS-DPDK-TESTPMD.png)

```bash
## 删除之前的配置文件
rm /usr/local/etc/openvswitch/*
rm /usr/local/var/run/openvswitch/*
rm /usr/local/var/log/openvswitch/*

ovs-ctl --no-ovs-vswitchd start --system-id=random
ovs-vsctl --no-wait set Open_vSwitch . other_config:pmd-cpu-mask=0x02
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-socket-mem="1024"
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true
ovs-ctl --no-ovsdb-server --db-sock="$DB_SOCK" start

## 添加 bridge
ovs-vsctl add-br br0 -- set bridge br0 datapath_type=netdev
ovs-vsctl add-port br0 vhost-user1 \
    -- set Interface vhost-user1 type=dpdkvhostuser ofport_request=1
ovs-vsctl add-port br0 vhost-user2 \
    -- set Interface vhost-user2 type=dpdkvhostuser ofport_request=2
```

```bash
# pktgen
docker run -it --rm --privileged --name=app-pktgen \
    -v /dev/hugepages:/dev/hugepages \
    -v /usr/local/var/run/openvswitch:/var/run/openvswitch \
    pktgen20:latest /bin/bash

pktgen -c 0x19 --main-lcore 3 -n 1 --socket-mem 1024 --file-prefix pktgen --no-pci  \
--vdev 'net_virtio_user1,mac=00:00:00:00:00:01,path=/var/run/openvswitch/vhost-user1' \
-- -T -P -m "0.0"
```

```bash
# testpmd
docker run -it --rm --privileged --name=app-testpmd \
    -v /dev/hugepages:/dev/hugepages \
    -v /usr/local/var/run/openvswitch:/var/run/openvswitch \
    pktgen20:latest /bin/bash

dpdk-testpmd -c 0xE0 -n 1 --socket-mem 1024 --file-prefix testpmd --no-pci \
--vdev 'net_virtio_user2,mac=00:00:00:00:00:02,path=/var/run/openvswitch/vhost-user2' \
-- -i -a --coremask=0xc0 --forward-mode=rxonly
```

## topo2

![ovs-ovs](../images/OpenVSwitch-%E6%B5%8B%E8%AF%95%E6%8B%93%E6%89%91/QEMU-OVS-QEMU.png)

```bash
## 删除之前的配置文件
rm /usr/local/etc/openvswitch/*
rm /usr/local/var/run/openvswitch/*
rm /usr/local/var/log/openvswitch/*

ovs-ctl --no-ovs-vswitchd start --system-id=random
ovs-vsctl --no-wait set Open_vSwitch . other_config:pmd-cpu-mask=0x02
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-socket-mem="1024"
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true
ovs-ctl --no-ovsdb-server --db-sock="$DB_SOCK" start

## 添加 bridge
ovs-vsctl add-br br0 -- set bridge br0 datapath_type=netdev
ovs-vsctl add-port br0 vhost-user1 \
    -- set Interface vhost-user1 type=dpdkvhostuser ofport_request=1
ovs-vsctl add-port br0 vhost-user2 \
    -- set Interface vhost-user2 type=dpdkvhostuser ofport_request=2
```

使用QEMU虚拟机

```bash
# 基于vhost-user1, vhost-user2创建虚拟机
qemu-system-x86_64 -enable-kvm -m 2048 -smp 4 \
    -chardev socket,id=char0,path=/usr/local/var/run/openvswitch/vhost-user1 \
    -netdev type=vhost-user,id=mynet1,chardev=char0,vhostforce \
    -device virtio-net-pci,netdev=mynet1,mac=52:54:00:02:d9:01 \
    -object memory-backend-file,id=mem,size=2048M,mem-path=/dev/hugepages,share=on \
    -numa node,memdev=mem -mem-prealloc \
    -net user,hostfwd=tcp::10021-:22 -net nic \
    ./qemu-vm1.img

qemu-system-x86_64 -enable-kvm -m 2048 -smp 4 \
    -chardev socket,id=char0,path=/usr/local/var/run/openvswitch/vhost-user2 \
    -netdev type=vhost-user,id=mynet1,chardev=char0,vhostforce \
    -device virtio-net-pci,netdev=mynet1,mac=52:54:00:02:d9:02 \
    -object memory-backend-file,id=mem,size=2048M,mem-path=/dev/hugepages,share=on \
    -numa node,memdev=mem -mem-prealloc \
    -net user,hostfwd=tcp::10022-:22 -net nic \
    ./qemu-vm2.img &
```

## topo3

![ovs-ovs](../images/OpenVSwitch-%E6%B5%8B%E8%AF%95%E6%8B%93%E6%89%91/ovs-ovs.png)

**拓扑搭建**：

```bash
sudo dpdk-hugepages.py -p 1G --setup 4G
sudo modprobe openvswitch

## 创建容器
sudo docker create -it --name=s1 --privileged -v /dev/hugepages:/dev/hugepages -v /root/run:/root/run ovs-dpdk:2.16.0
sudo docker create -it --name=s2 --privileged -v /dev/hugepages:/dev/hugepages -v /root/run:/root/run ovs-dpdk:2.16.0
sudo docker start s1
sudo docker start s2

## 添加 veth-peer
sudo ip link add p1_2 type veth peer name p2_1 > /dev/null
sudo ip link set dev p1_2 name p1_2 netns $(sudo docker inspect -f '{{.State.Pid}}' s1) up
sudo ip link set dev p2_1 name p2_1 netns $(sudo docker inspect -f '{{.State.Pid}}' s2) up
```

**启动OVS**：

```bash
################################ s1 ####################################
docker exec -it s1 bash
ovs-ctl --no-ovs-vswitchd start --system-id=1001
ovs-vsctl --no-wait set Open_vSwitch . other_config:pmd-cpu-mask=0x02
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-socket-mem="1024"
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true
ovs-ctl --no-ovsdb-server --db-sock="$DB_SOCK" start

## 添加 bridge
ovs-vsctl add-br s1 -- set bridge s1 datapath_type=netdev
ifconfig s1 192.168.1.12 netmask 255.255.255.0 up
ovs-vsctl add-port s1 p1_2 -- set Interface p1_2 ofport_request=12
# ovs-vsctl add-port s1 vhost-user1 -- set Interface vhost-user1 type=dpdkvhostuser ofport_request=2

################################ s2 ####################################
docker exec -it s2 bash
ovs-ctl --no-ovs-vswitchd start --system-id=1002
ovs-vsctl --no-wait set Open_vSwitch . other_config:pmd-cpu-mask=0x08
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-socket-mem="1024"
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true
ovs-ctl --no-ovsdb-server --db-sock="$DB_SOCK" start

## 添加 bridge
ovs-vsctl add-br s2 -- set bridge s2 datapath_type=netdev
ifconfig s2 192.168.1.21 netmask 255.255.255.0 up
ovs-vsctl add-port s2 p2_1 -- set Interface p2_1 ofport_request=21
# ovs-vsctl add-port s2 vhost-user1 -- set Interface vhost-user1 type=dpdkvhostuser ofport_request=2
```

## topo4

![ovs-ovs-ovs](../images/OpenVSwitch-%E6%B5%8B%E8%AF%95%E6%8B%93%E6%89%91/ovs-ovs-ovs.png)

**拓扑搭建**：

```bash
sdpdk-hugepages.py -p 1G --setup 6G
sudo modprobe openvswitch

## 创建容器
sudo docker create -it --name=s1 --privileged -v /dev/hugepages:/dev/hugepages -v /root/run:/root/run ovs-dpdk:2.16.0
sudo docker create -it --name=s2 --privileged -v /dev/hugepages:/dev/hugepages -v /root/run:/root/run ovs-dpdk:2.16.0
sudo docker create -it --name=s3 --privileged -v /dev/hugepages:/dev/hugepages -v /root/run:/root/run ovs-dpdk:2.16.0
sudo docker start s1 s2 s3

## 添加 veth-peer
sudo ip link add s1_2 type veth peer name s2_1 > /dev/null
sudo ip link add s3_2 type veth peer name s2_3 > /dev/null
sudo ip link add s1_3 type veth peer name s3_1 > /dev/null
sudo ip link set dev s1_2 name s1_2 netns $(sudo docker inspect -f '{{.State.Pid}}' s1) up
sudo ip link set dev s1_3 name s1_3 netns $(sudo docker inspect -f '{{.State.Pid}}' s1) up
sudo ip link set dev s2_1 name s2_1 netns $(sudo docker inspect -f '{{.State.Pid}}' s2) up
sudo ip link set dev s2_3 name s2_3 netns $(sudo docker inspect -f '{{.State.Pid}}' s2) up
sudo ip link set dev s3_1 name s3_1 netns $(sudo docker inspect -f '{{.State.Pid}}' s3) up
sudo ip link set dev s3_2 name s3_2 netns $(sudo docker inspect -f '{{.State.Pid}}' s3) up
```

**启动OVS**：

```bash
################################ s1 ####################################
docker exec -it s1 bash
ovs-ctl --no-ovs-vswitchd start --system-id=1001
ovs-vsctl --no-wait set Open_vSwitch . other_config:pmd-cpu-mask=0x02
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-socket-mem="1024"
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true
ovs-ctl --no-ovsdb-server --db-sock="$DB_SOCK" start

## 添加 bridge
ovs-vsctl add-br s1 -- set bridge s1 datapath_type=netdev
ovs-vsctl set bridge s1 other_config:hwaddr="00:00:00:00:10:01"
ifconfig s1 192.168.10.1 netmask 255.255.255.0 up
ovs-vsctl add-port s1 s1_2 -- set Interface s1_2 ofport_request=12
ovs-vsctl add-port s1 s1_3 -- set Interface s1_3 ofport_request=13
# ovs-vsctl add-port s1 vhost-user1 -- set Interface vhost-user1 type=dpdkvhostuser ofport_request=2

################################ s2 ####################################
docker exec -it s2 bash
ovs-ctl --no-ovs-vswitchd start --system-id=1002
ovs-vsctl --no-wait set Open_vSwitch . other_config:pmd-cpu-mask=0x08
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-socket-mem="1024"
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true
ovs-ctl --no-ovsdb-server --db-sock="$DB_SOCK" start

## 添加 bridge
ovs-vsctl add-br s2 -- set bridge s2 datapath_type=netdev
ovs-vsctl set bridge s2 other_config:hwaddr="00:00:00:00:10:02"
ifconfig s2 192.168.10.2 netmask 255.255.255.0 up
ovs-vsctl add-port s2 s2_1 -- set Interface s2_1 ofport_request=21
ovs-vsctl add-port s2 s2_3 -- set Interface s2_3 ofport_request=23
# ovs-vsctl add-port s2 vhost-user1 -- set Interface vhost-user1 type=dpdkvhostuser ofport_request=2

################################ s3 ####################################
docker exec -it s3 bash
ovs-ctl --no-ovs-vswitchd start --system-id=1002
ovs-vsctl --no-wait set Open_vSwitch . other_config:pmd-cpu-mask=0x20
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-socket-mem="1024"
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true
ovs-ctl --no-ovsdb-server --db-sock="$DB_SOCK" start

## 添加 bridge
ovs-vsctl add-br s3 -- set bridge s3 datapath_type=netdev
ovs-vsctl set bridge s3 other_config:hwaddr="00:00:00:00:10:03"
ifconfig s3 192.168.10.3 netmask 255.255.255.0 up
ovs-vsctl add-port s3 s3_1 -- set Interface s3_1 ofport_request=31
ovs-vsctl add-port s3 s3_2 -- set Interface s3_2 ofport_request=32
# ovs-vsctl add-port s3vhost-user1 -- set Interface vhost-user1 type=dpdkvhostuser ofport_request=2
```

**转发流表**：1<->2 ，2<->3 ，1<->3

```bash
# s1
ovs-ofctl add-flow s1 ip,nw_dst=192.168.10.1,actions=output:LOCAL
ovs-ofctl add-flow s1 ip,nw_dst=192.168.10.2,actions=output:s1_2
ovs-ofctl add-flow s1 ip,nw_dst=192.168.10.3,actions=output:s1_3
# s2
ovs-ofctl add-flow s2 ip,nw_dst=192.168.10.2,actions=output:LOCAL
ovs-ofctl add-flow s2 ip,nw_dst=192.168.10.1,actions=output:s2_1
ovs-ofctl add-flow s2 ip,nw_dst=192.168.10.3,actions=output:s2_3
# s3
ovs-ofctl add-flow s3 ip,nw_dst=192.168.10.3,actions=output:LOCAL
ovs-ofctl add-flow s3 ip,nw_dst=192.168.10.1,actions=output:s3_1
ovs-ofctl add-flow s3 ip,nw_dst=192.168.10.2,actions=output:s3_2
# arp
arp -s 192.168.10.1 00:00:00:00:10:01
arp -s 192.168.10.2 00:00:00:00:10:02
arp -s 192.168.10.3 00:00:00:00:10:03
```

## topo5

**转发流表2**：1<->2<->3 (1与3不直连)

![ovs-ovs-ovs2](../images/OpenVSwitch-%E6%B5%8B%E8%AF%95%E6%8B%93%E6%89%91/ovs-ovs-ovs2.png)

```bash
# s1
ovs-ofctl add-flow s1 ip,nw_dst=192.168.10.1,actions=output:LOCAL
ovs-ofctl add-flow s1 ip,nw_dst=192.168.10.2,actions=output:s1_2
ovs-ofctl add-flow s1 ip,nw_dst=192.168.10.3,actions=output:s1_2
# s2
ovs-ofctl add-flow s2 ip,nw_dst=192.168.10.2,actions=output:LOCAL
ovs-ofctl add-flow s2 ip,nw_dst=192.168.10.1,actions=output:s2_1
ovs-ofctl add-flow s2 ip,nw_dst=192.168.10.3,actions=output:s2_3
# s3
ovs-ofctl add-flow s3 ip,nw_dst=192.168.10.3,actions=output:LOCAL
ovs-ofctl add-flow s3 ip,nw_dst=192.168.10.1,actions=output:s3_2
ovs-ofctl add-flow s3 ip,nw_dst=192.168.10.2,actions=output:s3_2
# arp
arp -s 192.168.10.1 00:00:00:00:10:01
arp -s 192.168.10.2 00:00:00:00:10:02
arp -s 192.168.10.3 00:00:00:00:10:03
```

