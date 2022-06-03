---
title: OpenVSwitch-编译及启动
top_img: transparent
date: 2022-05-12 19:24:45
updated: 2022-05-12 19:24:45
tags:
  - OpenVSwitch
  - DPDK
  - Docker
categories: OpenVSwitch
keywords:
description: OVS几种编译方式、生成Docker镜像、启动OVS等
---

## 基于内核的OVS

首先验证内核版本`uname -r`与你下载的OVS版本是否匹配（必须），目前 ovs-2.16 最高只支持内核5.8！

[linux内核与ovs版本匹配关系、ovs版本与DPDK版本匹配关系](https://docs.openvswitch.org/en/latest/faq/releases/)

以 `OVS-2.16.0` 为例，编译过程如下：

```bash
# 获取源码
wget https://github.com/openvswitch/ovs/archive/refs/tags/v2.16.0.zip && unzip v2.16.0.zip

# 安装依赖包
sudo apt install build-essential fakeroot
dpkg-checkbuilddeps	# 检查依赖并手动安装缺少的模块
#####################################################

cd ovs-2.16.0
./boot.sh
./configure --with-linux=/lib/modules/$(uname -r)/build
make -j     # -j 默认让所有核同时进行编译，也可以指定编译核数

sudo make install   # 安装ovs
sudo insmod datapath/linux/openvswitch.ko # 安装内核模块，报错处理见下文
sudo make modules_install
sudo modprobe openvswitch

# 报错处理，一般是缺失依赖模块，通过modprobe安装就好
modinfo ./datapath/linux/openvswitch.ko | grep depends # 查看缺少依赖模块
sudo modprobe nf_conntrack
sudo modprobe nf_nat

# 启动工具 ovs-ctl
echo 'export PATH=$PATH:/usr/local/share/openvswitch/scripts' | tee -a /root/.bashrc \
echo 'export DB_SOCK=/usr/local/var/run/openvswitch/db.sock' | tee -a /root/.bashrc
```

[安装内核模块时报错的解决办法](https://blog.csdn.net/u010710985/article/details/119980125)

## 基于DPDK的OVS

**下面的DPDK编译方式都是基于meson，从DPDK 19版本开始才能使用meson进行编译！**

同样注意OVS的版本与DPDK版本是否匹配，ovs-2.16 适配了DPDK-20.11.5，DPDK的版本`20.11.n`一定要选最新版，这种更新一般是修复了一些 bug。

### 编译安装DPDK

```bash
# 获取源码
wget https://fast.dpdk.org/rel/dpdk-20.11.5.tar.xz

# 编译工具 meson ninja，从v19开始使用meson进行DPDK的编译！
sudo pip3 install meson ninja
sudo apt install libnuma-dev libpcap-dev libfdt-dev

#####################################################

cd dpdk-stable-20.11.3
# 编译 使用`| tee file`是将日志显示在终端的同时，保存到文件中
meson build	| tee ../meson.build
ninja -C build
# 安装库 将DPDK的相关库文件复制到根目录下的相关引用目录
sudo ninja -C build install
# 更新缓存 (usually run when a new library is installed)
sudo ldconfig
# 测试一下 libdpdk 是否安装成功
pkg-config --modversion libdpdk

# 卸载 dpdk
sudo ninja uninstall
```

### 编译安装 OVS-DPDK

```bash
# get ovs v2.16.0
wget https://github.com/openvswitch/ovs/archive/refs/tags/v2.16.0.zip
# 安装依赖包
sudo apt install build-essential fakeroot
dpkg-checkbuilddeps	# 检查依赖并手动安装缺少的模块
#####################################################

cd ovs-2.16.0
./boot.sh
./configure --with-dpdk=static CFLAGS="-Ofast -msse4.2 -mpopcnt"
make -j     # 多线程编译

# 单元测试（可以跳过）
make check TESTSUITEFLAGS=-j8
sudo make install

# 启动工具 ovs-ctl
echo 'export PATH=$PATH:/usr/local/share/openvswitch/scripts' | tee -a /root/.bashrc \
echo 'export DB_SOCK=/usr/local/var/run/openvswitch/db.sock' | tee -a /root/.bashrc
```

## 生成Docker镜像

**前提：主机上先安装好OVS！**

注意：**docker与宿主机是共享内核的！**，即使是在 docker 中使用 OVS，无论是基于内核的还是基于 DPDK 的，都会用到内核模块`openvswitch`，只有在主机上安装 OVS 后才带有这个模块，并且每次关机、重启后，都需要重新加载`modprobe`该模块，否则在docker中启动 OVS 时会报下面的错误：

```bash
root@72a4b64ba608:~# ovs-ctl start
 * /usr/local/etc/openvswitch/conf.db does not exist
 * Creating empty database /usr/local/etc/openvswitch/conf.db
nice: cannot set niceness: Permission denied
 * Starting ovsdb-server
 * system ID not configured, please use --system-id
 * Configuring Open vSwitch system IDs
/usr/local/share/openvswitch/scripts/ovs-kmod-ctl: 112: modprobe: not found	############# !!!! ################
 * Inserting openvswitch module
/usr/local/share/openvswitch/scripts/ovs-kmod-ctl: 112: rmmod: not found
 * removing bridge module
```

上述原因是，启动 OVS 时要判断内核中是否有 openvswitch 模块，如果没有，就需要加载（通过命令 modprobe），但 docker 中并没有 modprobe 命令，因此报错，解决办法就是在主机上先加载该模块到内核（前提是主机上安装过OVS）：

```bash
sudo modprobe openvswitch
```

### 基于内核的 Dockerfile

```Dockerfile
FROM ubuntu:20.04 as build
ARG ovs_version=2.16.3

RUN apt-get update \
  && DEBIAN_FRONTEND=noninteractive apt-get -y install build-essential fakeroot cmake wget unzip \
  graphviz autoconf automake debhelper dh-autoreconf libssl-dev libtool python3-all python3-sphinx \
  python3-twisted python3-zope.interface libunbound-dev libunwind-dev

WORKDIR /opt

# 下载编译 OVS
RUN wget https://codeload.github.com/openvswitch/ovs/zip/refs/tags/v${ovs_version} -O ovs-${ovs_version}.zip \
    && unzip ovs-${ovs_version}.zip

RUN cd ovs-${ovs_version} \
    && ./boot.sh \
    && ./configure --with-linux=/lib/modules/$(uname -r)/build \
    && make -j \
    && make install

#####################################################
FROM ubuntu:20.04

# 复制编译后的文件
COPY --from=build /usr/local /usr/local/

RUN apt-get update \
  && apt-get -y --no-install-recommends install python3-all iproute2 net-tools \
  && ldconfig \
  && echo 'export PATH=$PATH:/usr/local/share/openvswitch/scripts' | tee -a /root/.bashrc \
  && echo 'export DB_SOCK=/usr/local/var/run/openvswitch/db.sock' | tee -a /root/.bashrc
```

### 基于 DPDK 的 Dockerfile

```Dockerfile
FROM ubuntu:20.04 as build
ARG dpdk_version=20.11.5
ARG ovs_version=2.16.3

RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get -y install build-essential python3-pip liblua5.3-dev \
    cmake wget unzip libnuma-dev pciutils libpcap-dev libelf-dev linux-headers-generic \
    && pip3 install meson ninja pyelftools

WORKDIR /opt

# 下载编译 DPDK
RUN wget -q https://fast.dpdk.org/rel/dpdk-${dpdk_version}.tar.xz \
    && tar xf dpdk-${dpdk_version}.tar.xz

RUN cd dpdk-stable-${dpdk_version} \
    && meson build \
    && ninja -C build \
    && ninja -C build install \
    && cp -r build/lib /usr/local/

# 下载编译 OVS
RUN wget https://codeload.github.com/openvswitch/ovs/zip/refs/tags/v${ovs_version} -O ovs-${ovs_version}.zip \
	&& unzip ovs-${ovs_version}.zip

RUN cd ovs-${ovs_version} \
    && ./boot.sh \
    && ./configure --with-dpdk=static CFLAGS="-Ofast -msse4.2 -mpopcnt" \
    && make -j \
    && make install

#####################################################
FROM ubuntu:20.04

# 复制相关文件
COPY --from=build /usr/local /usr/local/

RUN apt-get update \
    && apt-get -y --no-install-recommends install liblua5.3 libnuma-dev pciutils \
    libpcap-dev python3 iproute2 net-tools iputils-ping \
    && ldconfig \
    && echo 'export PATH=$PATH:/usr/local/share/openvswitch/scripts' | tee -a /root/.bashrc \
    && echo 'export DB_SOCK=/usr/local/var/run/openvswitch/db.sock' | tee -a /root/.bashrc
```

### 自定义 OVS-DPDK 的 Dockerfile

如果对OVS，DPDK有任何修改，需要将源代码拷贝到容器中进行编译，准备一个文件夹，并准备以下文件：

```bash
Dockerfile
dpdk-stable-20.11.5 # DPDK的源码
ovs-2.16.0          # OVS的源码
```

`Dockerfile` 文件中的内容如下：

```Dockerfile
FROM ubuntu:20.04 as build
ARG dpdk_version=20.11.5
ARG ovs_version=2.16.3

RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get -y install build-essential python3-pip liblua5.3-dev \
    cmake wget libnuma-dev pciutils libpcap-dev libelf-dev linux-headers-generic \
    && pip3 install meson ninja pyelftools

WORKDIR /opt

# 将本地文件复制到docker中
COPY . .

# 编译 DPDK
RUN cd dpdk-stable-${dpdk_version} \
    && meson build \
    && ninja -C build \
    && ninja -C build install \
    && cp -r build/lib /usr/local/

# 编译 OVS
RUN cd ovs-${ovs_version} \
    && sh boot.sh \
    && ./configure --with-dpdk=static CFLAGS="-Ofast -msse4.2 -mpopcnt" \
    && make -j \
    && make install

#####################################################
FROM ubuntu:20.04

# 复制编译后的文件
COPY --from=build /usr/local /usr/local/

RUN apt-get update \
    && apt-get -y --no-install-recommends install liblua5.3 libnuma-dev pciutils \
    libpcap-dev python3 iproute2 net-tools iputils-ping \
    && ldconfig \
    && echo 'export PATH=$PATH:/usr/local/share/openvswitch/scripts' | tee -a /root/.bashrc \
    && echo 'export DB_SOCK=/usr/local/var/run/openvswitch/db.sock' | tee -a /root/.bashrc
```

### Pktgen 和 testpmd 的 Dockerfile

`pktgen` 用来生成、发送数据包，`testpmd` 用来接收数据包，两者都是针对DPDK的。

```dockerfile
FROM ubuntu:20.04 as build
# 指定使用的DPDK, pktgen 版本
ARG dpdk_version=20.11.5
ARG pktgen_version=20.11.3

RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get -y install build-essential python3-pip liblua5.3-dev \
    cmake wget libnuma-dev pciutils libpcap-dev libelf-dev linux-headers-generic a\
    && pip3 install meson ninja pyelftools

WORKDIR /opt

# 下载编译 DPDK
RUN wget -q https://fast.dpdk.org/rel/dpdk-${dpdk_version}.tar.xz \
    && tar xf dpdk-${dpdk_version}.tar.xz

RUN cd dpdk-stable-${dpdk_version} \
    && meson build \
    && ninja -C build \
    && ninja -C build install \
    && cp -r build/lib /usr/local/

# patch to make pktgen compile on arm. Got tip from
# https://medium.com/codex/nvidia-mellanox-bluefield-2-smartnic-dpdk-rig-for-dive-part-ii-change-mode-of-operation-a994f0f0e543
RUN sed -i 's/#  error Platform.*//' /usr/local/include/rte_spinlock.h
RUN sed -i 's/#  error Platform.*//' /usr/local/include/rte_atomic_32.h

# downlaod and unpack pktgen
RUN wget -q https://git.dpdk.org/apps/pktgen-dpdk/snapshot/pktgen-dpdk-pktgen-${pktgen_version}.tar.gz \
    && tar xf pktgen-dpdk-pktgen-${pktgen_version}.tar.gz

# building pktgen
RUN cd pktgen-dpdk-pktgen-${pktgen_version} \
    && tools/pktgen-build.sh clean \
    && tools/pktgen-build.sh buildlua \
    && cp -r usr/local /usr/ \
    && mkdir -p /usr/local/share/lua/5.3/ \
    && cp Pktgen.lua /usr/local/share/lua/5.3/

#####################################################
FROM ubuntu:20.04

COPY --from=build /usr/local /usr/local/

RUN apt-get update \
    && apt-get -y --no-install-recommends install liblua5.3 libnuma-dev pciutils libpcap-dev python3 iproute2 \
    && ldconfig
```

生成镜像：`docker build -t pktgen:20.11.3 .`

## OVS 启动及配置

基于内核的数据通路时，OVS启动需要配置的内容不多！但基于OVS-DPDK的数据通路，涉及到的参数设置比较麻烦！

### 基于内核数据通路



### 基于 DPDK 的数据通路



## 相关资料

- [linux内核与ovs版本匹配关系、ovs版本与DPDK版本匹配关系](https://docs.openvswitch.org/en/latest/faq/releases/)
- [安装内核模块时报错的解决办法](https://blog.csdn.net/u010710985/article/details/119980125)
