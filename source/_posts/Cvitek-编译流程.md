---
title: Cvitek-v4.2.0 编译流程
date: 2024-10-09 22:25:10
updated: 2024-10-09 22:25:10
tags:
  - fsbl
  - opensbi
  - uboot
  - Linux
categories: Cvitek
keywords:
description:
top_img: transparent
---

## 编译流程说明

```bash
# 1. source build/envsetup_soc.sh（设置环境变量）
yogurt@s:v4.2.x$ source build/envsetup_soc.sh
  -------------------------------------------------------------------------------------------------------
    Usage:
    (1) menuconfig - Use menu to configure your board.
        ex: $ menuconfig

    (2) defconfig $CHIP_ARCH - List EVB boards($BOARD) by CHIP_ARCH.
       ** cv181x ** -> ['cv181x', 'cv1823a', 'cv1821a', 'cv1820a', 'cv1811h', 'cv1811ha', 'cv1812ha', 'cv1811c', 'cv1810c', 'cv1810ca', 'cv1812h', 'cv1812cp', 'cv1813h', 'cv1813ha']
       ** cv180x ** -> ['cv180x', 'cv1800b', 'cv1800c', 'cv1801b', 'cv1801c', 'cv180zb']
        ex: $ defconfig cv183x

    (3) defconfig $BOARD - Choose EVB board settings.
        ex: $ defconfig cv1835_wevb_0002a
        ex: $ defconfig cv1826_wevb_0005a_spinand
        ex: $ defconfig cv181x_fpga_c906
  -------------------------------------------------------------------------------------------------------

# 2. 使用 defconfig 选定板卡配置
yogurt@s:v4.2.x$ defconfig cv1811c_wevb_0006a_emmc
 Run defconfig function
Loaded configuration '/home/yogurt/Documents/sophgo/v4.2.x/build/boards/cv181x/cv1811c_wevb_0006a_emmc/cv1811c_wevb_0006a_emmc_defconfig'
Configuration saved to '.config'
Loaded configuration '.config'
Minimal configuration saved to '/home/yogurt/Documents/sophgo/v4.2.x/build/.defconfig'
~/Documents/sophgo/v4.2.x/build ~/Documents/sophgo/v4.2.x
~/Documents/sophgo/v4.2.x

====== Environment Variables =======

  PROJECT: cv1811c_wevb_0006a_emmc, DDR_CFG=ddr3_1866_x16
  CHIP_ARCH: CV181X, DEBUG=0
  SDK VERSION: musl_riscv64, RPC=0
  ATF options: ATF_KEY_SEL=default, BL32=1
  Linux source folder:linux_5.10, Uboot source folder: u-boot-2021.10
  CROSS_COMPILE_PREFIX: riscv64-unknown-linux-musl-
  ENABLE_BOOTLOGO: 0
  Flash layout xml: /home/yogurt/Documents/sophgo/v4.2.x/build/boards/cv181x/cv1811c_wevb_0006a_emmc/partition/partition_emmc.xml
  Support EMMC large part size: n
  Sensor tuning bin: gcore_gc4653
  Output path: /home/yogurt/Documents/sophgo/v4.2.x/install/soc_cv1811c_wevb_0006a_emmc

# 3. 运行 build_all 开始编译
yogurt@s:v4.2.x$ build_all
```

1. 🐾设置环境变量：主要是提供相关 `shell` 函数，比如 `defconfig`, `menuconfig`, `build_kernel`, `build_uboot`, `build_all` 等，给用户手动调用。
2. 🐾选定板卡配置：不同板卡会有不同的配置，包含：
   1. 🌱 设备树
   2. 🌱 linux kernel 配置
   3. 🌱 uboot 配置
   4. 🌱 flash 分区表
   5. 🌱 内存布局
   6. 🌱 其他配置
3. 🐾开始编译：可以运行 `build_all` 编译完整 SDK，也可以运行 `build_uboot`/`build_kernel` 等编译单个模块。

📌 对正常启动系统来说，需要编译得到的文件有：

- `fip.bin/fip_spl.bin`：`Bootloader`，包含 `fsbl`, `opensbi`(riscv only), `u-boot`；
- `boot.{flash_type}`：`Linux` 内核；
- `rootfs.{flash_type}`：大核用的文件系统；
- `yoc.bin`：小核 Alios 的固件。

| 文件夹           | 编译命令                                          | 生成文件                                             |
| ---------------- | ------------------------------------------------- | ---------------------------------------------------- |
| `fsbl`           | `build_uboot`                                     | `bl2.bin` -> `fip.bin`                               |
| `opensbi`        | `build_uboot`                                     | `fw_dynamic.bin`  + `ddr_param.bin` -> `fip.bin`     |
| `u-boot-2021.10` | `build_uboot`                                     | `u-boot-raw_spl.bin` / `u-boot-raw.bin` -> `fip.bin` |
| `cvi_alios`      | `build_uboot` 或 `build_alios` 单独编译           | `yoc.bin`  -> `fip.bin`                              |
| `linux_5.10`     | `build_kernel(arm)`/`build_opensbi_kernel(riscv)` | `boot.xxx` 内核                                      |
| `ramdisk`        | `pack_rootfs`                                     | `rootfs.xxx` 文件系统                                |
| `ramdisk`        | `build_ramboot`                                   | `ramboot` 用做 `recovery`                            |

下面以 `cv1811c_wevb_0006a_emmc` 板卡为例，分析上诉各个文件的编译流程。

## v4.2.x 编译流程

![v4.2.x 编译流程](./images/Cvitek-编译流程/v4.2.x-编译流程.svg)
