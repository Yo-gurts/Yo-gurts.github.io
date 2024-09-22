---
title: Linux USB functionfs
top_img: transparent
date: 2024-09-22 22:31:05
updated: 2024-09-22 22:31:08
tags:
  - Linux
  - USB
  - configfs
categories: USB
keywords:
description:
---

> translate from https://www.kernel.org/doc/html/v6.1/usb/functionfs.html

# FunctionFS 如何工作

从内核的角度来看，它只是一个具有一些独特行为的复合功能。只有在用户空间驱动程序通过写入描述符和字符串注册后，它才能被添加到 USB 配置中（用户空间程序必须提供与内核级复合功能在添加到配置中时提供的相同信息）。

```text
From kernel point of view it is just a composite function with some
unique behaviour.  It may be added to an USB configuration only after
the user space driver has registered by writing descriptors and
strings (the user space program has to provide the same information
that kernel level composite functions provide when they are added to
the configuration).
```

这特别意味着复合初始化函数可能不会在 init 部分（即可能不会使用 __init 标签）。

```text
This in particular means that the composite initialisation functions
may not be in init section (ie. may not use the __init tag).
```

从用户空间的角度来看，它是一个文件系统，当挂载时会提供一个 "ep0" 文件。用户空间驱动程序需要向该文件写入描述符和字符串。它不需要担心端点、接口或字符串编号，而只需提供描述符，如同该功能是唯一的（端点和字符串编号从 1 开始，接口编号从 0 开始）。FunctionFS 根据需要更改它们，并处理在不同配置中编号不同的情况。

```text
From user space point of view it is a file system which when
mounted provides an "ep0" file.  User space driver need to
write descriptors and strings to that file.  It does not need
to worry about endpoints, interfaces or strings numbers but
simply provide descriptors such as if the function was the
only one (endpoints and strings numbers starting from one and
interface numbers starting from zero).  The FunctionFS changes
them as needed also handling situation when numbers differ in
different configurations.
```

当描述符和字符串被写入后，“ep#”文件会出现（每个声明的端点对应一个），用于处理单个端点上的通信。同样，FunctionFS 负责实际编号和配置更改（这意味着 "ep1" 文件可能实际上映射到（例如）端点 3（当配置更改时可能映射到端点 2））。"ep0" 用于接收事件和处理设置请求。

```text
When descriptors and strings are written "ep#" files appear
(one for each declared endpoint) which handle communication on
a single endpoint.  Again, FunctionFS takes care of the real
numbers and changing of the configuration (which means that
"ep1" file may be really mapped to (say) endpoint 3 (and when
configuration changes to (say) endpoint 2)).  "ep0" is used
for receiving events and handling setup requests.
```

当所有文件关闭时，该功能会自动禁用。

```text
When all files are closed the function disables itself.
```

我还想提到的是，FunctionFS 的设计使其可以多次挂载，因此最终一个gadget可以使用多个 FunctionFS 功能。其理念是每个 FunctionFS 实例都由挂载时使用的设备名称标识。

```text
What I also want to mention is that the FunctionFS is designed in such
a way that it is possible to mount it several times so in the end
a gadget could use several FunctionFS functions. The idea is that
each FunctionFS instance is identified by the device name used
when mounting.
```

可以想象一个gadget具有以太网、MTP 和 HID 接口，其中后两个是通过 FunctionFS 实现的。在用户空间层面，它看起来像这样：

```text
One can imagine a gadget that has an Ethernet, MTP and HID interfaces
where the last two are implemented via FunctionFS.  On user space
level it would look like this::
```

```bash
  $ insmod g_ffs.ko idVendor=<ID> iSerialNumber=<string> functions=mtp,hid
  $ mkdir /dev/ffs-mtp && mount -t functionfs mtp /dev/ffs-mtp
  $ ( cd /dev/ffs-mtp && mtp-daemon ) &
  $ mkdir /dev/ffs-hid && mount -t functionfs hid /dev/ffs-hid
  $ ( cd /dev/ffs-hid && hid-daemon ) &
```

在内核层面，gadget检查 ffs_data->dev_name 以确定它是为 MTP（“mtp”）还是 HID（“hid”）设计的 FunctionFS。

```text
On kernel level the gadget checks ffs_data->dev_name to identify
whether it's FunctionFS designed for MTP ("mtp") or HID ("hid").
```

如果没有提供 "functions" 模块参数，驱动程序只接受一个具有任意名称的功能。

```text
If no "functions" module parameters is supplied, the driver accepts
just one function with any name.
```

当提供了 "functions" 模块参数时，只有列出的功能名称会被接受。特别地，如果 "functions" 参数的值只是一个单元素列表，那么行为类似于没有 "functions" 参数的情况；然而，仅接受具有指定名称的功能。

```text
When "functions" module parameter is supplied, only functions
with listed names are accepted. In particular, if the "functions"
parameter's value is just a one-element list, then the behaviour
is similar to when there is no "functions" at all; however,
only a function with the specified name is accepted.
```

只有在所有声明的功能文件系统都已挂载并且所有功能的 USB 描述符都已写入其 ep0 时，gadget才会被注册。

```text
The gadget is registered only after all the declared function
filesystems have been mounted and USB descriptors of all functions
have been written to their ep0's.
```

相反，第一个 USB 功能关闭其端点后，gadget会被注销。

```text
Conversely, the gadget is unregistered after the first USB function
closes its endpoints.
```
