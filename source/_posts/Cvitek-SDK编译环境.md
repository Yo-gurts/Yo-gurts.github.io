---
title: Cvitek SDK 编译环境
date: 2024-10-06 14:02:04
updated: 2024-10-07 11:25:33
tags:
  - Cvitek
  - Linux
categories: Cvitek
keywords:
description:
top_img: transparent
---

## SDK 下载

利用 sophpi 中的脚本下载 SDK 到当前目录。

```bash
git clone -b sg200x-evb git@github.com:sophgo/sophpi.git
```

sophpi 中的脚本是将需要的仓库拉取到当前目录，为了区分，专门新建 v4.1.x 目录：

```bash
mkdir v4.1.x && cd v4.1.x

../sophpi/scripts/repo_clone.sh --gitclone ../sophpi/scripts/subtree.xml
```

如需使用双系统快启分支，则：

```bash
mkdir v4.2.x && cd v4.2.x

../sophpi/scripts/repo_clone.sh --gitclone ../sophpi/scripts/subtree_cv18xx-v4.2.x.xml
```

> 🙃可能遇到的问题：
> 1. 没有权限拉取仓库。
>    sophpi 的脚本中是用的 git clone ssh@xxx 走的 ssh，需要配置好 ssh key，否则会报错。
>    可以替换 xml 中的 git@github.com:sophgo/ 为 https://github.com/sophgo/

## v4.1.x 编译环境配置

> 下面以 ubuntu 22.04 为例。

```bash
sudo apt install -y pkg-config build-essential ninja-build automake autoconf libtool wget curl git gcc libssl-dev bc slib squashfs-tools android-sdk-libsparse-utils android-sdk-ext4-utils jq cmake python3-distutils tclsh scons parallel ssh-client tree python3-dev python3-pip device-tree-compiler libssl-dev ssh cpio squashfs-tools fakeroot libncurses5 flex bison
```

> 🙃可能遇到的问题：
>
> 1. 找不到软件包 android-sdk-ext4-utils
>
>     ```bash
>     sudo apt install -y android-sdk-ext4-utils
>     ```
>     ubuntu 20.04 可能找不到该软件包
>
>     ```bash
>     ❯ sudo apt-get install -y android-sdk-ext4-utils
>
>     正在读取软件包列表... 完成
>     正在分析软件包的依赖关系树... 完成
>     正在读取状态信息... 完成
>     E: 无法定位软件包 android-sdk-ext4-utils
>     ```
>     解决方法如下：
>
>     wget http://mirrors.kernel.org/ubuntu/pool/universe/a/android-platform-system-extras/android-sdk-ext4-utils_8.1.0+r23-2_amd64.deb
>
>     ```bash
>     sudo apt install android-libcutils android-libext4-utils android-libselinux android-libsepol
>
>     sudo dpkg -i android-sdk-ext4-utils_8.1.0+r23-2_amd64.deb
>     ```
>
> 2. 在执行 source build/cvisetup.sh 时出现以下情况：
>
>     ```bash
>     /proc/self/fd/17:9: athena2_board_sel: assignment to invalid subscript range
>     ```
>     （其中 17:9 可能是其他数字）
>
>     cvisetup.sh 脚本未执行成功。执行 defconfig cv1812h_wevb_0007a_emmc_huashan 结果如下：
>
>     ```bash
>     ❯ defconfig cv1812h_wevb_0007a_emmc
>      Run  function
>     _call_kconfig_script:11: no such file or directory: /mnt/e/wsl/v4.1.0.3/v4.1.0.3_source/build/scripts/.py
>     ```
>
>     原因：使用的 shell 并非 bash。且出 invalid subcript range（下标范围问题）可能是使用 zsh 执行 shell 脚本导致的。zsh 数组的下标索从 1 开始，与 bash 不同（bash 从 0 开始）。
>     更多讨论请参考：https://unix.stackexchange.com/questions/562561/bash-script-throws-assignment-to-invalid-subscript-range-when-running-from-zsh
>
>     解决方法：终端键入 bash，在 bash 环境下执行source build/cvisetup.sh
>
> 3. 找不到 python。
>
>     /bin/sh: 1: python: not found
>
>     ```bash
>     sudo ln -s /usr/bin/python3 /usr/bin/python
>     ```
>
> 4. **python 库缺少**
>
>     ```bash
>     /home/yogurt/Documents/sophgo/v4.1.x/cvi_mpi/modules/isp/common/toolJsonGenerator
>     Updated 0 paths from the index
>     Traceback (most recent call last):
>       File "/home/yogurt/Documents/sophgo/v4.1.x/cvi_mpi/modules/isp/common/toolJsonGenerator/hFile2json.py", line 5, in <module>
>         from jinja2 import Template
>     ModuleNotFoundError: No module named 'jinja2'
>     Updated 1 path from the index
>     cat: pqtool_definition.json: No such file or directory
>     the json file is a invalid, error is : Expecting value: line 1 column 1 (char 0)!!!
>     make[3]: *** [Makefile:11: all] Error 255
>     make[3]: Leaving directory '/home/yogurt/Documents/sophgo/v4.1.x/cvi_mpi/modules/isp/cv181x'
>     make[2]: *** [Makefile:8: all] Error 1
>     make[2]: Leaving directory '/home/yogurt/Documents/sophgo/v4.1.x/cvi_mpi/modules/isp'
>     make[1]: *** [Makefile:18: all] Error 1
>     make[1]: Leaving directory '/home/yogurt/Documents/sophgo/v4.1.x/cvi_mpi/modules'
>     make: *** [Makefile:35: module] Error 2
>      build middleware failed !!
>     ```
>     解决方法：执行 `pip install jinja2`

## v4.2.x/Alios 编译

> 下面以 ubuntu 22.04 为例，其他版本可能会有差异。
>
> v4.2.x 在 v4.1.x 的基础上，增加了小核 alios 的编译环境配置。

```bash
pip install yoctools -i https://pypi.tuna.tsinghua.edu.cn/simple/
```
