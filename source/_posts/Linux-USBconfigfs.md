---
title: Linux USB gadget_configfs
top_img: transparent
date: 2024-09-22 21:44:30
updated: 2024-09-22 21:44:34
tags:
  - Linux
  - USB
  - configfs
categories: USB
keywords:
description:
---

## Linux USB gadget 通过 configfs 配置

- [ ] 先阅读 [Linux configfs](Linux-configfs.md)，再看此内容

usb 只有在作为从设备时，才会用到 configfs 去配置功能。

更新日期：2013年4月25日

## 概述

USB Linux Gadget 是一种设备，它具有一个 UDC（USB 设备控制器），并且可以连接到 USB 主机，通过添加功能扩展主机，如串行端口或大容量存储功能。

```plaintext
A USB Linux Gadget is a device which has a UDC (USB Device Controller)
and can be connected to a USB Host to extend it with additional functions
like a serial port or a mass storage capability.
```

从主机的角度来看，**一个 Gadget 是一组配置的集合**，**每个配置包含若干接口，而从 Gadget 的角度看，这些接口被称为功能（function），每个功能代表一个如串行连接或 SCSI 磁盘的设备**。

```plaintext
A gadget is seen by its host as a set of configurations, each of which
contains a number of interfaces which, from the gadget's perspective,
are known as functions, each function representing e.g. a serial
connection or a SCSI disk.
```

Linux 提供了许多功能供 Gadget 使用。

```plaintext
Linux provides a number of functions for gadgets to use.
```

**创建一个 Gadget 意味着决定将有哪些配置，以及每个配置将提供哪些功能**。

```plaintext
Creating a gadget means deciding what configurations there will be and which
functions each configuration will provide.
```

Configfs（请参见 `Documentation/filesystems/configfs.rst`）非常适合用于向内核传达上述决策。本文档讨论了如何实现这一目标。

```plaintext
Configfs (please see `Documentation/filesystems/configfs.rst`) lends itself
nicely for the purpose of telling the kernel about the above mentioned decision.
This document is about how to do it.
```

它还描述了 Gadget 中的 configfs 集成设计。

```plaintext
It also describes how configfs integration into gadget is designed.
```

## 要求

为了使其工作，configfs 必须可用，因此 .config 中的 `CONFIGFS_FS` 必须为 'y' 或 'm'。截至本文写作时，`USB_LIBCOMPOSITE` 选择了 `CONFIGFS_FS`。

```plaintext
In order for this to work configfs must be available, so CONFIGFS_FS must be 'y'
or 'm' in .config. As of this writing USB_LIBCOMPOSITE selects CONFIGFS_FS.
```

## 用法

（描述第一个通过 configfs 提供的功能的原始帖子可以在这里看到：http://www.spinics.net/lists/linux-usb/msg76388.html）

```plaintext
(The original post describing the first function made available through configfs
can be seen here: http://www.spinics.net/lists/linux-usb/msg76388.html)
```

```bash
$ modprobe libcomposite
$ mount none $CONFIGFS_HOME -t configfs
```

其中 CONFIGFS_HOME 是 configfs 的挂载点。

```plaintext
where CONFIGFS_HOME is the mount point for configfs
```

### 1. 创建 Gadgets

为每个要创建的 Gadget 创建相应的目录：

```bash
$ mkdir $CONFIGFS_HOME/usb_gadget/<gadget name>
```

```plaintext
For each gadget to be created its corresponding directory must be created:
```

例如：

```bash
$ mkdir $CONFIGFS_HOME/usb_gadget/g1
```

进入对应的目录：

```bash
$ cd $CONFIGFS_HOME/usb_gadget/g1
```

每个 Gadget 需要指定其供应商 ID（VID）和产品 ID（PID）：

```bash
$ echo <VID> > idVendor
$ echo <PID> > idProduct
```

```plaintext
Each gadget needs to have its vendor id <VID> and product id <PID> specified:
```

一个 Gadget 还需要其序列号、制造商和产品字符串。为了存储它们，需要为每种语言创建一个字符串子目录，例如：

```plaintext
A gadget also needs its serial number, manufacturer and product strings. In order
to have a place to store them, a strings subdirectory must be created for each language,
e.g.:
```

```bash
$ mkdir strings/0x409
```

> [usb language IDS](https://github.com/brookebasile/USB-langids/blob/master/USB%20Lang%20IDs%20-%20Language%20Identifiers.csv) or http://www.baiheee.com/Documents/090518/090518112619/USB_LANGIDs.pdf
>
> ```
> 0x0404	Chinese (Taiwan)
> 0x0804	Chinese (PRC)
> 0x0c04	Chinese (Hong Kong SAR, PRC)
> 0x1004	Chinese (Singapore)
> 0x1404	Chinese (Macau SAR)
> 0x0409	English (United States)
> 0x0809	English (United Kingdom)
> ```

然后可以指定字符串：

```plaintext
Then the strings can be specified:
```

```bash
$ echo <serial number> > strings/0x409/serialnumber
$ echo <manufacturer> > strings/0x409/manufacturer
$ echo <product> > strings/0x409/product
```

### 2. 创建配置

每个 Gadget 将包含若干配置，需要创建相应的目录：

```bash
$ mkdir configs/<name>.<number>
```

```plaintext
Each gadget will consist of a number of configurations, their corresponding directories must be created:
```

其中 `<name>` 可以是文件系统中合法的任意字符串，`<number>` 是配置的编号，例如：

```bash
$ mkdir configs/c.1
```

```plaintext
where <name> can be any string which is legal in a filesystem and the <number> is the configuration's number, e.g.:
```

每个配置也需要其字符串，因此需要为每种语言创建一个子目录，例如：

```bash
$ mkdir configs/c.1/strings/0x409
```

```plaintext
Each configuration also needs its strings, so a subdirectory must be created for
each language, e.g.:
```

然后可以指定配置字符串：

```bash
$ echo <configuration> > configs/c.1/strings/0x409/configuration
```

```plaintext
Then the configuration string can be specified:
```

还可以为配置设置一些属性，例如：

```bash
$ echo 120 > configs/c.1/MaxPower
```

```plaintext
Some attributes can also be set for a configuration, e.g.:
```

### 3. 创建功能

Gadget 将提供一些功能，为每个功能创建相应的目录：

```bash
$ mkdir functions/<name>.<instance name>
```

```plaintext
The gadget will provide some functions, for each function its corresponding
directory must be created:
```

**其中 `<name>` 对应于允许的功能名称之一**，实例名称是文件系统中允许的任意字符串，例如：

```bash
$ mkdir functions/ncm.usb0 # usb_f_ncm.ko gets loaded with request_module()
```

```plaintext
where <name> corresponds to one of allowed function names and instance name is
an arbitrary string allowed in a filesystem, e.g.:
```

每个功能提供其特定的一组属性，具有只读或读写访问权限。在适用的情况下，需要适当写入它们。请参阅 `Documentation/ABI/*/configfs-usb-gadget*` 了解更多信息。

```plaintext
Each function provides its specific set of attributes, with either read-only or
read-write access. Where applicable they need to be written to as appropriate.
Please refer to Documentation/ABI/*/configfs-usb-gadget* for more information.
```

> 💥💥💥💥💥💥 [Documentation/ABI/*/configfs-usb-gadget](https://elixir.bootlin.com/linux/v5.10.186/source/Documentation/ABI/testing) **这里有对每个文件的详细介绍**。

### 4. 将功能与配置关联

此时创建了若干 Gadget，每个 Gadget 都指定了若干配置，并且有若干可用功能。剩下的就是指定哪个功能在哪个配置中可用（同一功能可以在多个配置中使用）。这可以通过创建符号链接来实现：

```bash
$ ln -s functions/<name>.<instance name> configs/<name>.<number>
```

```plaintext
At this moment a number of gadgets is created, each of which has a number of configurations specified and a number of functions available. What remains is specifying which function is available in which configuration (the same function can be used in multiple configurations). This is achieved with creating symbolic links:
```

例如：

```bash
$ ln -s functions/ncm.usb0 configs/c.1
```

### 5. 启用 Gadget

以上所有步骤的目的是将 Gadget 组成配置和功能。

```plaintext
All the above steps serve the purpose of composing the gadget of configurations and functions.
```

一个示例目录结构可能如下所示：

```plaintext
An example directory structure might look like this:
```

```plaintext
.
./strings
./strings/0x409
./strings/0x409/serialnumber
./strings/0x409/product
./strings/0x409/manufacturer
./configs
./configs/c.1
./configs/c.1/ncm.usb0 -> ../../../../usb_gadget/g1/functions/ncm.usb0
./configs/c.1/strings
./configs/c.1/strings/0x409
./configs/c.1/strings/0x409/configuration
./configs/c.1/bmAttributes
./configs/c.1/MaxPower
./functions
./functions/ncm.usb0
./functions/ncm.usb0/ifname
./functions/ncm.usb0/qmult
./functions/ncm.usb0/host_addr
./functions/ncm.usb0/dev_addr
./UDC
./bcdUSB
./bcdDevice
./idProduct
./idVendor
./bMaxPacketSize0
./bDeviceProtocol
./bDeviceSubClass
./bDeviceClass
```

这样的 Gadget 必须最终启用，以便 USB 主机可以枚举它。

```plaintext
Such a gadget must be finally enabled so that the USB host can enumerate it.
```

为了启用 Gadget，必须将其绑定到 UDC（USB 设备控制器）：

```bash
$ echo <udc name> > UDC
```

```plaintext
In order to enable the gadget it must be bound to a UDC (USB Device Controller):
```

其中 `<udc name>` 是在 `/sys/class/udc/*` 中找到的一个名称，例如：

```plaintext
where <udc name> is one of those found in /sys/class/udc/* e.g.:
```

```bash
$ echo s3c-hsotg > UDC
```

### 6. 禁用 Gadget

```bash
$ echo "" > UDC
```

```plaintext
$ echo "" > UDC
```

### 7. 清理

从配置中删除功能：

```plaintext
Remove functions from configurations:
```

```bash
$ rm configs/<config name>.<number>/<function>
```

其中 `<config name>.<number>` 指定配置，`<function>` 是要从配置中删除的功能的符号链接，例如：

```plaintext
where <config name>.<number> specify the configuration and <function> is a
symlink to a function being removed from the configuration, e.g.:
```

```bash
$ rm configs/c.1/ncm.usb0
```

删除配置中的字符串目录：

```plaintext
Remove strings directories in configurations:
```

```bash
$ rmdir configs/<config name>.<number>/strings/<lang>
```

例如：

```bash
$ rmdir configs/c.1/strings/0x409
```

然后删除配置：

```plaintext
and remove the configurations:
```

```bash
$ rmdir configs/<config name>.<number>
```

例如：

```bash
$ rmdir configs/c.1
```

删除功能（功能模块不会被卸载）：

```plaintext
Remove functions (function modules are not unloaded, though):
```

```bash
$ rmdir functions/<name>.<instance name>
```

例如：

```bash
$ rmdir functions/ncm.usb0
```

删除 Gadget 中的字符串目录：

```plaintext
Remove strings directories in the gadget:
```

```bash
$ rmdir strings/<lang>
```

例如：

```bash
$ rmdir strings/0x409
```

最后删除 Gadget：

```plaintext
and finally remove the gadget:
```

```bash
$ cd ..
$ rmdir <gadget name>
```

例如：

```bash
$ rmdir g1
```

## 实现设计

> [linux configfs](Linux-configfs.md)

下面展示了 configfs 的工作原理。💥💥💥💥**在 configfs 中，有项目`item`和组`group`，均表示为目录。项目和组的区别在于组可以包含其他组**💥💥💥。下图仅显示了一个项目。项目和组都可以有属性，这些属性表示为文件。用户可以创建和删除目录，但不能删除文件，这些文件可以是只读或读写的，具体取决于它们表示的内容。

```plaintext
Below the idea of how configfs works is presented. In configfs there are items and groups,
both represented as directories. The difference between an item and a group is
that a group can contain other groups. In the picture below only an item is shown.
Both items and groups can have attributes, which are represented as files.
The user can create and remove directories, but cannot remove files, which can
be read-only or read-write, depending on what they represent.
```

configfs 文件系统部分操作 config_items/groups 和 configfs_attributes，它们是通用且对所有配置元素类型相同的。然而，它们嵌入在特定用途的更大结构中。下图中有一个 "cs" 包含一个 config_item，还有一个 "sa" 包含一个 configfs_attribute。

```plaintext
The filesystem part of configfs operates on config_items/groups and
configfs_attributes which are generic and of the same type for all configured elements.
However, they are embedded in usage-specific larger structures. In the picture
below there is a "cs" which contains a config_item and an "sa"
which contains a configfs_attribute.
```

文件系统视图如下所示：

```plaintext
The filesystem view would be like this:
```

```plaintext
  ./
  ./cs        (directory)
     |
     +--sa    (file)
     |
     .
     .
     .
```

每当用户读取/写入 "sa" 文件时，会调用一个函数，该函数接收一个 struct config_item 和一个 struct configfs_attribute。在该函数中，使用已知的 container_of 技术检索 "cs" 和 "sa"，并调用适当的 sa 的函数（show 或 store），并传递 "cs" 和一个字符缓冲区。"show" 用于显示文件内容（将数据从 cs 复制到缓冲区），而 "store" 用于修改文件内容（将数据从缓冲区复制到 cs），但具体做什么取决于这两个函数的实现者。

```plaintext
Whenever a user reads/writes the "sa" file, a function is called which accepts
a struct config_item and a struct configfs_attribute. In the said function the
"cs" and "sa" are retrieved using the well known container_of technique and an
appropriate sa's function (show or store) is called and passed the "cs" and a
character buffer. The "show" is for displaying the file's contents (copy data
from the cs to the buffer), while the "store" is for modifying the file's
contents (copy data from the buffer to the cs), but it is up to the implementer
of the two functions to decide what they actually do.
```

```plaintext
  typedef struct configured_structure cs;
  typedef struct specific_attribute sa;

                                         sa
                         +----------------------------------+
          cs             |  (*show)(cs *, buffer);          |
  +-----------------+    |  (*store)(cs *, buffer, length); |
  |                 |    |                                  |
  | +-------------+ |    |       +------------------+       |
  | | struct      |-|----|------>|struct            |       |
  | | config_item | |    |       |configfs_attribute|       |
  | +-------------+ |    |       +------------------+       |
  |                 |    +----------------------------------+
  | data to be set  |                .
  |                 |                .
  +-----------------+                .
```

文件名由配置项/组设计者决定，而目录通常可以随意命名。一个组可以自动创建其默认子组。

```plaintext
The file names are decided by the config item/group designer, while the directories
 in general can be named at will. A group can have a number of its default
 sub-groups created automatically.
```

有关 configfs 的更多信息，请参见 `Documentation/filesystems/configfs.rst`。

```plaintext
For more information on configfs please see `Documentation/filesystems/configfs.rst`.
```

上述概念在 USB gadgets 中的应用如下：

```plaintext
The concepts described above translate to USB gadgets like this:
```

1. 一个 Gadget 有其配置组，具有一些属性（如 idVendor、idProduct 等）和默认子组（configs、functions、strings）。写入这些属性会将信息存储在适当的位置。在 configs、functions 和 strings 子组中，用户可以创建它们的子组来表示给定语言的配置、功能和字符串组。

```plaintext
1. A gadget has its config group, which has some attributes (idVendor, idProduct etc)
 and default sub-groups (configs, functions, strings). Writing to the attributes
 causes the information to be stored in appropriate locations. In the configs,
 functions and strings sub-groups a user can create their sub-groups to represent
 configurations, functions, and groups of strings in a given language.
```

2. 用户创建配置和功能，在配置中创建指向功能的符号链接。当 Gadget 的 UDC 属性被写入时，这些信息将被使用，这意味着将 Gadget 绑定到 UDC。drivers/usb/gadget/configfs.c 中的代码遍历所有配置，在每个配置中遍历所有功能并绑定它们。这样整个 Gadget 就被绑定了。

```plaintext
2. The user creates configurations and functions, in the configurations creates
symbolic links to functions. This information is used when the gadget's UDC
attribute is written to, which means binding the gadget to the UDC. The code in
drivers/usb/gadget/configfs.c iterates over all configurations, and in each
configuration it iterates over all functions and binds them.
This way the whole gadget is bound.
```

3. drivers/usb/gadget/configfs.c 文件包含以下代码：

- Gadget 的 config_group
- Gadget 的默认组（configs、functions、strings）
- 将功能与配置关联（符号链接）

```plaintext
3. The file drivers/usb/gadget/configfs.c contains code for

- gadget's config_group
- gadget's default groups (configs, functions, strings)
- associating functions with configurations (symlinks)
```

4. 每个 USB 功能自然有其希望配置的视图，因此在 functions 实现文件 drivers/usb/gadget/f_*.c 中定义了特定功能的 config_groups。

```plaintext
4. Each USB function naturally has its own view of what it wants configured,
so config_groups for particular functions are defined in the functions
implementation files drivers/usb/gadget/f_*.c.
```

5. 功能代码的编写方式是使用 usb_get_function_instance()，该函数反过来调用 request_module。因此，只要 modprobe 工作，特定功能的模块就会自动加载。请注意，反之则不然：在 Gadget 被禁用和拆除后，模块仍然保持加载状态。

```plaintext
5. Function's code is written in such a way that it uses usb_get_function_instance(),
which, in turn, calls request_module. So, provided that modprobe works,
modules for particular functions are loaded automatically. Please note that
the converse is not true: after a gadget is disabled and torn down,
the modules remain loaded.
```

## useful links

一些开源仓库基于上面介绍的 usb configfs，USB Gadget API 等，搞了些 cool things.

> copy from https://www.reddit.com/r/linux/comments/1annx0u/a_comprehensive_list_of_all_configfs_functionfs/?rdt=34774
>
> **A comprehensive list of all ConfigFS, FunctionFS, USB Gadget API, etc. tools and libraries on Github(*)**
>
> Ok, so I'm writing a comprehensive list for all libraries, tools, modules, and scripts for any linux distro or kernel that I can find on Github, so I can learn more about how to make use of it.
>
> ### What is this?
>
> For those who don't know what any of the above APIs are, they are the underlying APIs that power things like the MTP protocol that is associated with your mobile electronics, such as smartphone and digital cameras. The Media Transfer Protocol (MTP) is the protocol that allows host devices to *pretend* to be peripheral storage devices to other hosts. This is what makes it possible for your cell phone or digital camera to show up as a storage device on your PC, even though they are technically small computers.
>
> You can read more about these APIs from the kernel docs:
>
> - [USB Gadget API for Linux — The Linux Kernel documentation](https://docs.kernel.org/driver-api/usb/gadget.html?highlight=gadgetfs)
>
> ### Why is this cool?
>
> The really cool thing about these kernel APIs is that they can work at the USB protocol layer. Which means *any* USB device can be emulated by a linux machine any way you want. You are not limited to just the MTP protocol.
>
> The reason I am exploring these tools is that I want to see exactly what kind of things I can do with this API. Something that has interested me about this API since I came across it was whether or not it could support hosting a *bootable* mass storage device. This could allow me to do something like providing Ventoy (or Medicat) over the Gadget API. I was thinking that I could build some kind of stack with this that would allow me to install an operating system dynamically on the host using live ISO, then once the OS has been installed provision it via Ansible over RNDIS without ever having to reboot the device doing the install. Basically, it would provide a method for a fully unattended bare metal install that could fit in your pocket.
>
> ### What's this about kernels (plural)?
>
> Note that earlier, in the first paragraph, I said any linux distro as well as any *kernel.* Since implementations of this API typically require USB OTG or USB Dual Role hardware, the kernel that is likely to have the most stable tools/libraries is going to be the Android kernel. This is something to keep in mind when discussing these APIs.
>
> ### (TL;DR) The list
>
> (*) - Most of these libraries are on Github, but it is worth-while to at least point out the Android Code Search
>
> - (*) [Android Code Search](https://cs.android.com/)
> - [linux-usb-gadgets](https://github.com/linux-usb-gadgets) organization
>     - ✨✨✨ [libusbgx](https://github.com/linux-usb-gadgets/libusbgx)
>         - Seemingly really well rounded C library for linux for encapsulating the ConfigFS API. It also includes some bash tools for automatically creating and monitoring USB Gadgets
>     - [gt (Gadget-tool)](https://github.com/linux-usb-gadgets/gt)
>         - A command line tool for setting USB gadgets with ConfigFS
>     - [ptp-gadget](https://github.com/linux-usb-gadgets/ptp-gadget)
>         - An implementation of the PTP protocol (the predecessor for MTP) for Linux
>     - (old org/repo) [libusbg/libusbg](https://github.com/libusbg/libusbg)
>         - the original version of libusbg
> - [litui/gadgeteer](https://github.com/litui/gadgeteer)
>     - A JSON daemon for controlling ConfigFS
> - [luka177/mmc-to-usb](https://github.com/luka177/mmc-to-usb)
>     - Allows you to control an emmc storage device from another PC, including allowing you to partition it.
> - [tylerwhall/configfs-regulator](https://github.com/tylerwhall/configfs-regulator)
>     - A ConfigFS-based Voltage Regulator Driver
> - [kopasiak/simple_usb_chat](https://github.com/kopasiak/simple_usb_chat)
>     - A simple chat program over FunctionFS
> - [jghubert/FunctionFSTester](https://github.com/jghubert/FunctionFSTester)
>     - A repository for testing FunctionFS features on both the remote host and peripheral device. Have been ignoring most of these kinds of repos, but added this one, since it included tests to be run on the remote host.
> - [vladrobot/Raspberry-GadgetFS](https://github.com/vladrobot/Raspberry-GadgetFS/tree/master)
>     - A wrapper for GadgetFS tailored to the raspberry pi that provides HID spoofing as well as USB mass storage via Python
> - PTP/MTP protocol
>     - linux-usb-gadgets/ptp-gadget (see above)
>     - ✨✨ [viveris/uMTP-Responder](https://github.com/viveris/uMTP-Responder)
>         - An implementation of a MTP responder for Linux. README has a decent list of Single Board Computers with OTG/Dual Role USB ports
> - *Controller Emulators/Adapters*
>     - [Poohl/gadfetfs-Switch-Fightstick](https://github.com/Poohl/gadfetfs-Switch-Fightstick)
>         - Wii Remote Emulator/Adapter for the Nintendo Switch
>     - [dogtopus/ffsds4](https://github.com/dogtopus/ffsds4)
>         - A PS4 Controller Emulator over FunctionFS
>     - [Frederic98/GadgetDeck](https://github.com/Frederic98/GadgetDeck)
>         - A controller emulator using a steam deck as a gadget device over ConfigFS
> - *Android USB Tools*
>     - [tejado/android-usb-gadget](https://github.com/tejado/android-usb-gadget)
>         - Rooted Android tool used to spoof what your phone appears as to your computer. Really good tool for visually showing you what the ConfigFS/FunctionFS is capable of.
>     - [tejado/Authorizer](https://github.com/tejado/Authorizer)
>         - Secure Password Store that can autotype passwords over FunctionFS
> - *Hacking, Sniffing, and Security:*
>     - [threadexio/anducky](https://github.com/threadexio/anducky)
>         - An implementation of Hak5's Rubber Ducky on Android
>     - [ehabhussein/AutoGadgetFS](https://github.com/ehabhussein/AutoGadgetFS)
>         - Very comprehensive USB security, sniffing, and analyzing tool that can be used to perform USB man-in-the-middle attacks built on ConfigFS
>     - [greatscottgadgets/Facedancer](https://github.com/greatscottgadgets/Facedancer)
>         - Another USB man-in-the-middle-ware built on GadgetFS
>         - [usb-tools/USBProxy-legacy](https://github.com/usb-tools/USBProxy-legacy) (The original)
>     - [Iskuri/Usb-Over-TCP](https://github.com/Iskuri/Usb-Over-TCP)
>         - A USB over TCP implementation built on GadgetFS
> - *Simple HID Spoofing Scripts:*
>     - [linkjumper/configfs](https://github.com/linkjumper/configfs)
>         - A simple shell script for creating a HID device via ConfigFS
>     - [qlyoung/keyboard-gadget](https://github.com/qlyoung/keyboard-gadget)
>         - Another simple shell script for creating a HID device via ConfigFS
>     - [karacanil/linux-mouse-gadget](https://github.com/karacanil/linux-mouse-gadget)
>         - A small mouse spoofer
> - *Non-HID Spoofing Tools:*
>     - [oandrew/ipod-gadget](https://github.com/oandrew/ipod-gadget)
>         - A tool to spoof your raspberry pi or beaglebone black as an iPod for streaming audio and use with iPod accessories.
>     - [du33169/usb-gadget-utils](https://github.com/du33169/usb-gadget-utils)
>         - A small, but comprehensive HID spoofing kit. Includes a toolkit for extracting and parsing HID data from real hardware for use with ConfigFS.
> - *ConfigFS Device Tree Overlay Patches (not yet part of the linux kernel):*
>     - [ikwzm/dtbocfg](https://github.com/ikwzm/dtbocfg)
>         - A custom implementation of a "ConfigFS overlay interface." This is not part of the Linux Kernel, but independently developed "patch"
>     - [Xilinx/linux-xlnx](https://github.com/Xilinx/linux-xlnx)
>         - A similar solution, but as an actually patched kernel
>     - [digitalloggers/linux-custom-gpio-patches](https://github.com/digitalloggers/linux-custom-gpio-patches)
>         - A ConfigFS Device Tree Overlay patch specifically tailored to Raspberry Pi GPIO headers
> - ✨✨ *Programming Libraries for ConfigFS/FunctionFS:*
>     - libusbg/libusbgx (see above)
>         - ConfigFS for C
>     - [ueno/libusb-gadget](https://github.com/ueno/libusb-gadget)
>         - GadgetFS for C
>     - [xstevens/usbg.rs](https://github.com/xstevens/usbg.rs)
>         - ConfigFS for Rust
>     - [R030t1/linux-usb-functionfs-sys](https://github.com/R030t1/linux-usb-functionfs-sys)
>         - Crude FunctionFS for Rust
>     - [AlinaNova21/node-gadget](https://github.com/AlinaNova21/node-gadget)
>         - ConfigFS for Node.JS
>     - [mgalka/pygadget](https://github.com/mgalka/pygadget)
>         - ConfigFS for Python
>     - [vpelletier/python-functionfs](https://github.com/vpelletier/python-functionfs)
>         - FunctionFS for Python
>     - [google/go-configfs-tsm](https://github.com/google/go-configfs-tsm)
>         - ConfigFS (and other things) for Go - NOTE that this one is written by Google
>
> This list is mostly all inclusive:
>
> - It contains the majority of repositories with noteworthy ConfigFS/FunctionFS features.
> - Most of the ignored repositories were ones that either:
>     - had little documentation/were definitively unmaintained
>     - were just some playground or test repos for learning/future development

## gadget configfs 源码分析

> https://elixir.bootlin.com/linux/v5.10.186/source/drivers/usb/gadget/configfs.c
> [Linux-configfs#示例)](Linux-configfs#示例)

### 顶层 subsystem

初始化的时候注册了一个名为 `usb_gadget` 的 subsystem。所以，insmod 该模块之后，会得到 `/configfs/usb_gadget`。

```c
static struct configfs_subsystem gadget_subsys = {
	.su_group = {
		.cg_item = {
			.ci_namebuf = "usb_gadget",
			.ci_type = &gadgets_type,
		},
	},
	.su_mutex = __MUTEX_INITIALIZER(gadget_subsys.su_mutex),
};

static int __init gadget_cfs_init(void)
{
	int ret;

	config_group_init(&gadget_subsys.su_group);

	ret = configfs_register_subsystem(&gadget_subsys);
	return ret;
}
module_init(gadget_cfs_init);

static void __exit gadget_cfs_exit(void)
{
	configfs_unregister_subsystem(&gadget_subsys);
}
module_exit(gadget_cfs_exit);
```

该 `gadgets_type` 中没有 `.ct_attrs`，所以没有任何属性，即 `/configfs/usb_gadget` 目录为空。

```c
static const struct config_item_type gadgets_type = {
	.ct_group_ops   = &gadgets_ops,
	.ct_owner       = THIS_MODULE,
};
```

`gadgets_ops` 指定了 `.make_group`，所以 `mkdir` 会创建子项。

```
static struct configfs_group_operations gadgets_ops = {
	.make_group     = &gadgets_make,
	.drop_item      = &gadgets_drop,
};
```

比如 `mkdir cvitek` 会创建 `/configfs/usb_gadget/cvitek`，并同时为 cvitek 自动创建 `functions`, `configs`, `strings` 和 `os_desc` 4 个子项。

```c
static struct config_group *gadgets_make(
		struct config_group *group,
		const char *name)
{
	struct gadget_info *gi;

	gi = kzalloc(sizeof(*gi), GFP_KERNEL);
	if (!gi)
		return ERR_PTR(-ENOMEM);

	// 创建一个 `/configfs/usb_gadget/cvitek`
	config_group_init_type_name(&gi->group, name, &gadget_root_type);

	config_group_init_type_name(&gi->functions_group, "functions",
			&functions_type);
	configfs_add_default_group(&gi->functions_group, &gi->group);

	config_group_init_type_name(&gi->configs_group, "configs",
			&config_desc_type);
	configfs_add_default_group(&gi->configs_group, &gi->group);

	config_group_init_type_name(&gi->strings_group, "strings",
			&gadget_strings_strings_type);
	configfs_add_default_group(&gi->strings_group, &gi->group);

	config_group_init_type_name(&gi->os_desc_group, "os_desc",
			&os_desc_type);
	configfs_add_default_group(&gi->os_desc_group, &gi->group);

	gi->composite.bind = configfs_do_nothing;
	gi->composite.unbind = configfs_do_nothing;
	gi->composite.suspend = NULL;
	gi->composite.resume = NULL;
	gi->composite.max_speed = USB_SPEED_SUPER;

	spin_lock_init(&gi->spinlock);
	mutex_init(&gi->lock);
	INIT_LIST_HEAD(&gi->string_list);
	INIT_LIST_HEAD(&gi->available_func);

	composite_init_dev(&gi->cdev);
	gi->cdev.desc.bLength = USB_DT_DEVICE_SIZE;
	gi->cdev.desc.bDescriptorType = USB_DT_DEVICE;
	gi->cdev.desc.bcdDevice = cpu_to_le16(get_default_bcdDevice());

	gi->composite.gadget_driver = configfs_driver_template;

	gi->composite.gadget_driver.function = kstrdup(name, GFP_KERNEL);
	gi->composite.name = gi->composite.gadget_driver.function;

	if (!gi->composite.gadget_driver.function)
		goto err;

	return &gi->group;
err:
	kfree(gi);
	return ERR_PTR(-ENOMEM);
}
```

### gadget_root_type

```c
static const struct config_item_type gadget_root_type = {
	.ct_item_ops	= &gadget_root_item_ops,
	.ct_attrs	= gadget_root_attrs,
	.ct_owner	= THIS_MODULE,
};

static struct configfs_item_operations gadget_root_item_ops = {
	.release                = gadget_info_attr_release,
};
```

没有定义 `.make_item/.make_group`，所以无法在 `/configfs/usb_gadget/cvitek` 目录中通过 `mkdir` 创建子项。

它有 10 个可读写的属性。

```c
CONFIGFS_ATTR(gadget_dev_desc_, bDeviceClass);
CONFIGFS_ATTR(gadget_dev_desc_, bDeviceSubClass);
CONFIGFS_ATTR(gadget_dev_desc_, bDeviceProtocol);
CONFIGFS_ATTR(gadget_dev_desc_, bMaxPacketSize0);
CONFIGFS_ATTR(gadget_dev_desc_, idVendor);
CONFIGFS_ATTR(gadget_dev_desc_, idProduct);
CONFIGFS_ATTR(gadget_dev_desc_, bcdDevice);
CONFIGFS_ATTR(gadget_dev_desc_, bcdUSB);
CONFIGFS_ATTR(gadget_dev_desc_, UDC);
CONFIGFS_ATTR(gadget_dev_desc_, max_speed);

static struct configfs_attribute *gadget_root_attrs[] = {
	&gadget_dev_desc_attr_bDeviceClass,
	&gadget_dev_desc_attr_bDeviceSubClass,
	&gadget_dev_desc_attr_bDeviceProtocol,
	&gadget_dev_desc_attr_bMaxPacketSize0,
	&gadget_dev_desc_attr_idVendor,
	&gadget_dev_desc_attr_idProduct,
	&gadget_dev_desc_attr_bcdDevice,
	&gadget_dev_desc_attr_bcdUSB,
	&gadget_dev_desc_attr_UDC,
	&gadget_dev_desc_attr_max_speed,
	NULL,
};
```

### functions_type

```c
static const struct config_item_type functions_type = {
	.ct_group_ops   = &functions_ops,
	.ct_owner       = THIS_MODULE,
};

static struct configfs_group_operations functions_ops = {
	.make_group     = &function_make,
	.drop_item      = &function_drop,
};
```

> 💥💥💥💥在 configfs 中，有项目item和组group，均表示为目录。项目和组的区别在于组可以包含其他组💥💥💥

没有属性，所以一开始 `/configfs/usb_gadget/cvitek/functions` 目录为空。但定义了 `make_group`，所以可以创建组。

## 参考资料

- [Linux-configfs.md](Linux-configfs.md)
- [Linux-USBmtp.md](Linux-USBmtp.md)
- https://elixir.bootlin.com/linux/v5.10.186/source/Documentation/ABI/testing
- https://elixir.bootlin.com/linux/v5.10.186/source/Documentation/ABI/testing/configfs-usb-gadget
- http://www.spinics.net/lists/linux-usb/msg76388.html
- https://www.kernel.org/doc/html/v6.1/usb/gadget-testing.html
- https://www.kernel.org/doc/html/v6.1/usb/raw-gadget.html
- https://www.kernel.org/doc/html/v6.1/usb/functionfs.html
- [usb gadget configfs 原理](https://wowothink.com/6a85234b/)
- [usb gadget configfs 验证](https://wowothink.com/a64c6a27/)
- [usb gadget configfs 翻译](https://wowothink.com/440dc3d9/)
- [Android usb gadget类型](https://wowothink.com/ffcaead/)
- [ACM&ECM&NCM&EEM&RNDIS&RmNet介绍](https://wowothink.com/588ebc22/)
- [Linux USB Test Mode](https://wowothink.com/488cc6d0/)
- [USB2.0理论传输速度和实际传输速度](https://wowothink.com/69b1c932/)
- [Linux驱动中配置支持特定USB HUB](https://wowothink.com/68203a0/)
- [配置USB Host和USB Device full-speed工作](https://wowothink.com/892013c5/)
- [Linux测试U盘读写速度](https://wowothink.com/e0007988/)
