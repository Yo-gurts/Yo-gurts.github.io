---
title: OpenVSwitch-控制器ONOS
top_img: transparent
date: 2022-06-03 11:13:30
updated: 2022-06-03 11:13:30
tags:
  - OpenVSwitch
categories: OpenVSwitch
keywords:
description: OVS 连接到控制器 ONOS，以及控制器的使用
---

## 安装启动

直接使用 `docker` 安装最方便！

```bash
docker pull onosproject/onos
```

启动，做端口映射，不需要的可以去掉

```bash
docker run -t -d -p 8181:8181 -p 8101:8101 -p 6653:6653 -p 6640:6640 --name onos onosproject/onos
```

- 8181 - REST API 和 GUI
- 8101 - ONOS CLI
- 9876 - 集群内通信
- 6653 - Openflow 通信，OVS 连接控制器使用此端口（流表相关操作）
- 6640 - OVSDB，OVS 网桥、端口相关的配置

启用应用，用于控制OVS时，打开下面的应用，可通过`UI` 界面操作，也可在容器中用命令行方式启用。

```bash
docker exec -it onos bash
cd bin
./onos-app localhost activate org.onosproject.openflow-base
./onos-app localhost activate org.onosproject.ofagent
./onos-app localhost activate org.onosproject.fwd
./onos-app localhost activate org.onosproject.drivers.ovsdb
./onos-app localhost activate org.onosproject.proxyarp
./onos-app localhost activate org.onosproject.lldpprovider
```

`lldpprovider` 是用于拓扑发现的，启用该应用后，在 `UI/topology` 可以看到节点是否有链路直连。

## 编译 ONOS 镜像

[onos](https://github.com/Yo-gurts/onos/blob/master/Dockerfile)仓库中已有写好了的`Dockerfile`，将整个项目`clone`到本地后，即可直接编译镜像。该`Dockerfile`不是从网上下载源码，而是从当前目录`COPY`，所以修改源码后直接编译就行。

但是 `ONOS` 编译过程需要从`github`或一些外网地址下载相关编译工具，导致编译缓慢或者直接失败。需要配置代理使用：

```bash
docker build --network=host --build-arg http_proxy=http://127.0.0.1:8889 --build-arg https_proxy=http://127.0.0.1:8889  -t onos .
```

上面的方式在某些情况下已经够用了，但可能卡在`onos-gui-npm-install`，需要进一步处理。

- 一种方式是修改 `Dockerfile`，增加 `--action_env` 指定代理。

```Dockerfile
RUN cat WORKSPACE-docker >> WORKSPACE && bazelisk build onos \
+    --action_env=https_proxy=http://127.0.0.1:8889 \
     --jobs ${JOBS} \
     --verbose_failures \
     --java_runtime_version=dockerjdk_11
```

- 另一种方式是修改`web/gui/BUILD`。

```bash
cd onos/web/gui
sudo vi BUILD
#修改 onos-gui-npm-install
在$$NPM $$NPM_ARGS install后面加上
--registry https://registry.npm.taobao.org

#修改 onos-gui-npm-build
在$$ROOT/$$NPM $$NPM_ARGS run build --no-cache后加上
--registry https://registry.npm.taobao.org

cd onos/web/gui2-fw-lib
sudo vi BUILD
#修改 onos-gui2-fw-npm-install
在npm $$NPM_ARGS install后面加上
--registry https://registry.npm.taobao.org
```

也可以尝试两种方法都用。此外，也可以先将`bazelisk build onos`之前的编译为一个镜像（免去安装编译工具的步骤），启动容器，再慢慢编译，通过`docker commit`将完成编译的容器保存为镜像，再参考`Dockerfile`进行二阶段的编译过程（此法最可靠）。

## 常用页面

1. UI 页面地址：[http://localhost:8181/onos/ui/](http://localhost:8181/onos/ui/) 用户名：onos，密码：rocks
2. RESTFUL 接口文档：[http://localhost:8181/onos/v1/docs/](http://localhost:8181/onos/v1/docs/)
3. 流表组成说明：[https://wiki.onosproject.org/display/ONOS/Flow+Rules](https://wiki.onosproject.org/display/ONOS/Flow+Rules)
4. ONOS wiki，也就是官方文档：[https://wiki.onosproject.org](https://wiki.onosproject.org)

## OVS 连接控制器

1. 要指定OPENFLOW 协议为 `OPENFLOW13,OPENFLOW10`，**仅指定为`OPENFLOW13`时，连接控制器后将无法使用`ofctl`查看流表**。
2. 将`ovs bridge` 连接到控制器后的 `deviceId` 为`bridge`对应的`datapath-id`，可以先指定该值，再连接控制器。下发流表时会用到。

```bash
ovs-vsctl add-br s1 -- set bridge s1 datapath_type=netdev
ovs-vsctl set bridge s1 other_config:datapath-id=0x0000000000000012
ifconfig s1 192.168.1.12 netmask 255.255.255.0 up

ovs-vsctl set bridge s1 protocols=OpenFlow13,OpenFlow10
ovs-vsctl set-controller s1 tcp:172.17.0.2:6653
```

## Example

```bash
## 创建容器
sudo docker create --rm -it --name=s1 --privileged -v /dev/hugepages:/dev/hugepages -v /root/run:/root/run ovs-dpdk
sudo docker create --rm -it --name=s2 --privileged -v /dev/hugepages:/dev/hugepages -v /root/run:/root/run ovs-dpdk
sudo docker start s1
sudo docker start s2

## 添加 veth-peer，连接两容器
sudo ip link add p1_2 type veth peer name p2_1 > /dev/null
sudo ip link set dev p1_2 name p1_2 netns $(sudo docker inspect -f '{{.State.Pid}}' s1) up
sudo ip link set dev p2_1 name p2_1 netns $(sudo docker inspect -f '{{.State.Pid}}' s2) up
docker exec -it s1 tc qdisc add dev p1_2 root netem delay 1
docker exec -it s2 tc qdisc add dev p2_1 root netem delay 1

############################### s1 中启动OVS ##################################
ovs-ctl --no-ovs-vswitchd start --system-id=1001
ovs-vsctl --no-wait set Open_vSwitch . other_config:pmd-cpu-mask=0x4
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-socket-mem="1024"
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true
ovs-ctl --no-ovsdb-server --db-sock="$DB_SOCK" start

ovs-vsctl add-br s1 -- set bridge s1 datapath_type=netdev
ovs-vsctl set bridge s1 other_config:datapath-id=0x0000000000000012
ovs-vsctl add-port s1 p1_2 -- set Interface p1_2 ofport_request=12
ifconfig s1 192.168.1.12 netmask 255.255.255.0 up

ovs-vsctl set bridge s1 protocols=OpenFlow13,OpenFlow10
ovs-vsctl set-controller s1 tcp:172.17.0.2:6653

############################### s2 中启动OVS ##################################
ovs-ctl --no-ovs-vswitchd start --system-id=1002
ovs-vsctl --no-wait set Open_vSwitch . other_config:pmd-cpu-mask=0x40
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-socket-mem="1024"
ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true
ovs-ctl --no-ovsdb-server --db-sock="$DB_SOCK" start

ovs-vsctl add-br s2 -- set bridge s2 datapath_type=netdev
ovs-vsctl set bridge s1 other_config:datapath-id=0x0000000000000021
ovs-vsctl add-port s2 p2_1 -- set Interface p2_1 ofport_request=21
ifconfig s2 192.168.1.21 netmask 255.255.255.0 up

ovs-vsctl set bridge s2 protocols=OpenFlow13,OpenFlow10
ovs-vsctl set-controller s2 tcp:172.17.0.2:6653
```

## 下发流表

下发的`ARP`流表只能匹配`MAC`地址，但一般不需要手动下发。网络拓扑比较小时，也可以通过`arp -s {ip} {mac}`手动配置。

**限制**：匹配的以太类型不为IP时，无法匹配目的IP或源IP！即ARP和MPLS类型的，只能和MAC地址一起作为匹配域。

## 参考资料

- [onos 官网预编译的下载页面](https://wiki.onosproject.org/display/ONOS/Downloads)
- [onos github](https://github.com/opennetworkinglab/onos)
- [「ONOS x Mininet」从0开始搭建环境](https://chentingz.github.io/2019/10/28/%E3%80%8CONOS%20x%20Mininet%E3%80%8D%E4%BB%8E0%E5%BC%80%E5%A7%8B%E6%90%AD%E5%BB%BA%E7%8E%AF%E5%A2%83/#0x05-%E6%B5%8B%E8%AF%95%E7%8E%AF%E5%A2%83)
- [ONOS介绍](https://feisky.gitbooks.io/sdn/content/sdn/onos.html)
- [ONOS架构分析](http://developer.huawei.com/ict/cn/site-sdn-onos/article/onos-paradigm)
- [ONOS白皮书上篇之ONOS简介](http://www.sdnlab.com/6371.html)
- [ONOS白皮书中篇之ONOS架构](http://www.sdnlab.com/6800.html)
- [ONOS编译过程的问题](https://zhuanlan.zhihu.com/p/387616511?utm_id=0)
