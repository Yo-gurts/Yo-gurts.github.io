---
title: Linux USB mtp
top_img: transparent
date: 2024-09-17 20:02:15
updated: 2024-09-17 20:02:18
tags:
  - Linux
  - USB
categories: Linux
keywords:
description:
---

## USB mtp 介绍

https://en.wikipedia.org/wiki/Media_Transfer_Protocol

> from wiki:
> MTP是一种高水平的文件传输协议，与通用串行总线大容量存储等通用存储协议相反。这意味着MTP客户端（计算机）看不到构成文件系统的数据结构的字节块数组，而是用文件和文件夹来表示MTP设备。这使得MTP设备可以参与高水平的操作（如更新其元信息索引），同时将文件系统的完整性掌握在自己手中。特别是，掉线传输（例如过早拔掉USB电缆）不会损坏设备文件系统。[6]MTP的非通用性会影响计算机操作系统如何向其他程序和用户呈现MTP设备。
> 根据其规范，MTP的
> - 主要目的是促进具有瞬态连接的媒体设备之间的通信。[5]
> - 第二个目的是启用对连接设备的命令和控制。[5]电池供电的移动设备可以通过MTP报告其电池电量水平。[6]
>
> 该协议最初是为跨USB使用而实现的，但扩展为跨TCP/IP和蓝牙使用。Windows Vista支持TCP/IP上的MTP。带有Windows Vista平台更新的Windows 7和Windows Vista也支持蓝牙上的MTP。[7]连接到MTP设备的主机称为MTP启动器，而设备本身是MTP响应器。[8]

还是 wiki 介绍得更好，其他的随便看看

https://www.cnblogs.com/skywang12345/p/3474206.html

## libmtp 库

https://github.com/libmtp/libmtp

Initiator and Responder
-----------------------

libmtp implements an MTP initiator, which means it initiate
MTP sessions with devices. The devices responding are known
as MTP responders. libmtp runs on something with a USB host
controller interface, using libusb to access the host
controller.

If you're more interested in the MTP responders, gadgets like
MP3 players, mobile phones etc, look into:
- MeeGo:s Buteo Sync:
  https://github.com/nemomobile/buteo-mtp
  https://wiki.merproject.org/wiki/Buteo/MTP
- Android has an MTP responder implementation:
  https://android.googlesource.com/platform/frameworks/base/+/master/media/jni/
- Ubuntu/Ricardo Salveti has mtp-server and libmtp-server going:
  https://code.launchpad.net/~phablet-team/mtp/trunk
  https://bazaar.launchpad.net/~phablet-team/mtp/trunk/files

明显，libmtp 是主机端用的库，

## umtprd

buildroot 编译 umtprd 工具，类似于 adbd 的作用。

```
# mtp tools
BR2_PACKAGE_UMTPRD=y
```

### configfs 配置 mtp 流程

#### 0. 切换为 device 模式

```bash
echo device >/proc/cviusb/otg_role
```

> 参考 https://wowothink.com/6a85234b/

#### 1. 挂载 configfs

注册了一个名为usb_gadget的configfs子系统。

```bash
mkdir -p /tmp/usb
mount none /tmp/usb -t configfs
```

>```
>[root@sg200x]~# tree /tmp/usb/
>/tmp/usb/
>└── usb_gadget
>
>1 directory, 0 files
>```

#### 2. 创建复合设备

`mkdir usb_gadget/cvitek` 创建名为 cvitek 的usb复合设备。

```
mkdir -p /tmp/usb/usb_gadget/cvitek
```

在 cvitek 目录下会创建很多属性，

> ```
> [root@sg200x]/tmp/usb/usb_gadget# tree .
> .
> └── cvitek
>     ├── UDC
>     ├── bDeviceClass
>     ├── bDeviceProtocol
>     ├── bDeviceSubClass
>     ├── bMaxPacketSize0
>     ├── bcdDevice
>     ├── bcdUSB
>     ├── configs
>     ├── functions
>     ├── idProduct
>     ├── idVendor
>     ├── max_speed
>     ├── os_desc
>     │   ├── b_vendor_code
>     │   ├── qw_sign
>     │   └── use
>     └── strings
>
> 5 directories, 13 files
> ```

#### 2.1 配置PID和VID

```bash
echo 0x3346 >/tmp/usb/usb_gadget/cvitek/idVendor
echo 0x1003 >/tmp/usb/usb_gadget/cvitek/idProduct
```

#### 2.2 创建并配置 strings 子目录

配置 strings 首先必须设置`language`，这里设置为`0x0409`表示使用的是`en-us`语言。

```
# Set the product information string
mkdir /tmp/usb/usb_gadget/cvitek/strings/0x409
echo "Cvitek" >/tmp/usb/usb_gadget/cvitek/strings/0x409/manufacturer
echo "USB Com Port" >/tmp/usb/usb_gadget/cvitek/strings/0x409/product
echo "0123456789" >/tmp/usb/usb_gadget/cvitek/strings/0x409/serialnumber
```

>```
>[root@sg200x]/tmp/usb/usb_gadget# tree .
>.
>└── cvitek
>    ├── UDC
>    ├── bDeviceClass
>    ├── bDeviceProtocol
>    ├── bDeviceSubClass
>    ├── bMaxPacketSize0
>    ├── bcdDevice
>    ├── bcdUSB
>    ├── configs
>    ├── functions
>    ├── idProduct
>    ├── idVendor
>    ├── max_speed
>    ├── os_desc
>    │   ├── b_vendor_code
>    │   ├── qw_sign
>    │   └── use
>    └── strings
>        └── 0x409
>            ├── manufacturer
>            ├── product
>            └── serialnumber
>
>6 directories, 16 files
>```

#### 2.3 创建configuration和字符串

创建配置 `c.1`

```bash
mkdir /tmp/usb/usb_gadget/cvitek/configs/c.1
```

> ```
> [root@sg200x]/tmp/usb/usb_gadget# tree .
> .
> └── cvitek
>     ├── UDC
>     ├── bDeviceClass
>     ├── bDeviceProtocol
>     ├── bDeviceSubClass
>     ├── bMaxPacketSize0
>     ├── bcdDevice
>     ├── bcdUSB
>     ├── configs
>     │   └── c.1
>     │       ├── MaxPower
>     │       ├── bmAttributes
>     │       └── strings
>     ├── functions
>     ├── idProduct
>     ├── idVendor
>     ├── max_speed
>     ├── os_desc
>     │   ├── b_vendor_code
>     │   ├── qw_sign
>     │   └── use
>     └── strings
>         └── 0x409
>             ├── manufacturer
>             ├── product
>             └── serialnumber
>
> 8 directories, 18 files
> ```

设置配置名。

```bash
mkdir /tmp/usb/usb_gadget/cvitek/configs/c.1/strings/0x409
echo "config1" >/tmp/usb/usb_gadget/cvitek/configs/c.1/strings/0x409/configuration
```

> ```
> [root@sg200x]/tmp/usb/usb_gadget# tree .
> .
> └── cvitek
>     ├── UDC
>     ├── bDeviceClass
>     ├── bDeviceProtocol
>     ├── bDeviceSubClass
>     ├── bMaxPacketSize0
>     ├── bcdDevice
>     ├── bcdUSB
>     ├── configs
>     │   └── c.1
>     │       ├── MaxPower
>     │       ├── bmAttributes
>     │       └── strings
>     │           └── 0x409
>     │               └── configuration
>     ├── functions
>     ├── idProduct
>     ├── idVendor
>     ├── max_speed
>     ├── os_desc
>     │   ├── b_vendor_code
>     │   ├── qw_sign
>     │   └── use
>     └── strings
>         └── 0x409
>             ├── manufacturer
>             ├── product
>             └── serialnumber
>
> 9 directories, 19 files
> ```

Set the MaxPower of USB descriptor

```
echo 120 >/tmp/usb/usb_gadget/cvitek/configs/c.1/MaxPower
```

#### 3. 创建functions

**一个USB复合设备会有多个功能，每个功能由function来表示，所谓的function就是USB设备支持的功能**。当执行`mkdir functions/mass_storage.0`的时候，会调用`function_make()`函数，调用`usb_get_function_instance()`函数传入`mass_storage`字段，将会从现有的function list中找到与之匹配的function。现有的function list是由`drivers/usb/gadget/function/f_xxx.c`中进行添加的，这就将`f_xxx.c`联系起来了。

#### 3.1 创建 mtp functions

```bash
mkdir /tmp/usb/usb_gadget/cvitek/functions/mtp.usb0
```

内核还没开启相应配置的，就会报错。`USB_CONFIGFS_F_MTP=y`

```bash
[root@sg200x]/tmp/usb/usb_gadget# mkdir /tmp/usb/usb_gadget/cvitek/functions/mtp
.usb0
mkdir: can't create directory '/tmp/usb/usb_gadget/cvitek/functions/mtp.usb0': No such file or directory
```

> ```
> [root@sg200x]/tmp/usb/usb_gadget# tree .
> .
> └── cvitek
>     ├── UDC
>     ├── bDeviceClass
>     ├── bDeviceProtocol
>     ├── bDeviceSubClass
>     ├── bMaxPacketSize0
>     ├── bcdDevice
>     ├── bcdUSB
>     ├── configs
>     │   └── c.1
>     │       ├── MaxPower
>     │       ├── bmAttributes
>     │       └── strings
>     │           └── 0x409
>     │               └── configuration
>     ├── functions
>     │   └── mtp.usb0
>     ├── idProduct
>     ├── idVendor
>     ├── max_speed
>     ├── os_desc
>     │   ├── b_vendor_code
>     │   ├── qw_sign
>     │   └── use
>     └── strings
>         └── 0x409
>             ├── manufacturer
>             ├── product
>             └── serialnumber
>
> 10 directories, 19 files
> ```

#### 4. 将func和config关联起来

执行`ln -s usb_gadget/cvitek/functions/mtp.usb0 usb_gadget/cvitek/configs/c.1`命令将新添加的`mtp`的`functions`添加到`configuration`对应的function list中，表示当前usb复合设备中新增了一个function。这时候调用的是`config_usb_cfg_link()`函数。至此，`functions`和`configuration`关联起来了。接下来要将`configuration`与特定的UDC设备连接起来。

```bash
ln -s /tmp/usb/usb_gadget/cvitek/functions/mtp.usb0 /tmp/usb/usb_gadget/cvitek/configs/c.1
```

> ```
> [root@sg200x]/tmp/usb/usb_gadget# tree .
> .
> └── cvitek
>     ├── UDC
>     ├── bDeviceClass
>     ├── bDeviceProtocol
>     ├── bDeviceSubClass
>     ├── bMaxPacketSize0
>     ├── bcdDevice
>     ├── bcdUSB
>     ├── configs
>     │   └── c.1
>     │       ├── MaxPower
>     │       ├── bmAttributes
>     │       ├── mtp.usb0 -> ../../../../usb_gadget/cvitek/functions/mtp.usb0
>     │       └── strings
>     │           └── 0x409
>     │               └── configuration
>     ├── functions
>     │   └── mtp.usb0
>     ├── idProduct
>     ├── idVendor
>     ├── max_speed
>     ├── os_desc
>     │   ├── b_vendor_code
>     │   ├── qw_sign
>     │   └── use
>     └── strings
>         └── 0x409
>             ├── manufacturer
>             ├── product
>             └── serialnumber
>
> 11 directories, 19 files
> ```

------

#### 5. 绑定到UDC，使能gadget

执行`echo xxx > usb_gadget/g1/UDC`命令，就会调用`gadget_dev_desc_UDC_store()`函数，这里面调用`usb composite framework`中的`usb_gadget_probe_driver()`函数将gadget driver与USB Controller Driver绑定，这里的`Gadget Driver`就是与我们创建的usb复合设备对应的驱动，接下来就会走一系列的`bind`流程。

```bash
# Start the gadget driver
CVI_UDC=$(ls /sys/class/udc/ | awk '{print $1}')
echo ${CVI_UDC} >/tmp/usb/usb_gadget/cvitek/UDC
```

#### 汇总

```
# 0. 切换为 device 模式
echo device >/proc/cviusb/otg_role
# 1. 挂载 configfs
mkdir -p /tmp/usb
mount none /tmp/usb -t configfs
# 2. 创建复合设备
mkdir -p /tmp/usb/usb_gadget/cvitek
# 2.1 配置PID和VID
echo 0x3346 >/tmp/usb/usb_gadget/cvitek/idVendor
echo 0x1003 >/tmp/usb/usb_gadget/cvitek/idProduct
# 2.2 创建并配置 strings 子目录
mkdir /tmp/usb/usb_gadget/cvitek/strings/0x409
echo "Cvitek" >/tmp/usb/usb_gadget/cvitek/strings/0x409/manufacturer
echo "USB Com Port" >/tmp/usb/usb_gadget/cvitek/strings/0x409/product
echo "0123456789" >/tmp/usb/usb_gadget/cvitek/strings/0x409/serialnumber
# 2.3 创建configuration和字符串
mkdir /tmp/usb/usb_gadget/cvitek/configs/c.1
mkdir /tmp/usb/usb_gadget/cvitek/configs/c.1/strings/0x409
echo "config1" >/tmp/usb/usb_gadget/cvitek/configs/c.1/strings/0x409/configuration

echo 120 >/tmp/usb/usb_gadget/cvitek/configs/c.1/MaxPower
# 3. 创建functions
mkdir /tmp/usb/usb_gadget/cvitek/functions/mtp.usb0
# 4. 将func和config关联起来
ln -s /tmp/usb/usb_gadget/cvitek/functions/mtp.usb0 /tmp/usb/usb_gadget/cvitek/configs/c.1

##########
mkdir /dev/ffs-umtp
mount -t functionfs mtp /dev/ffs-umtp
# Start the umtprd service
umtprd &

# 5. 绑定到UDC，使能gadget
CVI_UDC=$(ls /sys/class/udc/ | awk '{print $1}')
echo ${CVI_UDC} >/tmp/usb/usb_gadget/cvitek/UDC
```



### umptrd 源码

https://github.com/viveris/uMTP-Responder/tree/master/conf

官方提供了一个配置脚本。

```bash
#!/bin/sh

# FunctionFS uMTPrd startup example/test script
# Must be launched from a writable/temporary folder.

modprobe libcomposite

mkdir cfg
mount none cfg -t configfs

mkdir cfg/usb_gadget/g1
cd cfg/usb_gadget/g1

mkdir configs/c.1

mkdir functions/ffs.mtp
# Uncomment / Change the follow line to enable another usb gadget function
#mkdir functions/acm.usb0

mkdir strings/0x409
mkdir configs/c.1/strings/0x409

echo 0x0100 > idProduct
echo 0x1D6B > idVendor

echo "01234567" > strings/0x409/serialnumber
echo "Viveris Technologies" > strings/0x409/manufacturer
echo "The Viveris Product !" > strings/0x409/product

echo "Conf 1" > configs/c.1/strings/0x409/configuration
echo 120 > configs/c.1/MaxPower

ln -s functions/ffs.mtp configs/c.1
# Uncomment / Change the follow line to enable another usb gadget function
#ln -s functions/acm.usb0 configs/c.1

mkdir /dev/ffs-umtp
mount -t functionfs mtp /dev/ffs-umtp
# Start the umtprd service
umtprd &

cd ../../..

sleep 1

# enable the usb functions
ls /sys/class/udc/ > cfg/usb_gadget/g1/UDC
```



```bash
mount -t functionfs mtp /dev/ffs-umtp
mount: mounting mtp on /dev/ffs-umtp failed: No such device
```

`CONFIG_USB_FUNCTIONFS=y` 可解决上面的问题。
