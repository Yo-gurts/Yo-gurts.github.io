---
title: Linux sysfs
top_img: transparent
date: 2024-09-24 22:37:59
updated: 2024-09-24 22:38:03
tags:
  - Linux
  - sysfs
categories: Linux
keywords:
description:
---

> chatgpt translate from https://www.kernel.org/doc/Documentation/filesystems/sysfs.txt

### sysfs - 内核对象导出的文件系统

Patrick Mochel <mochel@osdl.org>

Mike Murphy <mamurph@cs.clemson.edu>

### 什么是 sysfs:

**sysfs 是一个基于内存的文件系统，最初基于 ramfs。它提供了一种将内核数据结构、其属性及其之间的链接导出到用户空间的方法**。

sysfs 本质上与 kobject 基础结构紧密相关。请阅读 `Documentation/core-api/kobject.rst` 以获取有关 kobject 接口的更多信息。

```plaintext
sysfs is a ram-based filesystem initially based on ramfs. It provides
a means to export kernel data structures, their attributes, and the
linkages between them to userspace.

sysfs is tied inherently to the kobject infrastructure. Please read
Documentation/core-api/kobject.rst for more information concerning the kobject
interface.
```

### 使用 sysfs

如果定义了 `CONFIG_SYSFS`，sysfs 总是被编译进去。你可以通过以下命令访问它：

```plaintext
sysfs is always compiled in if CONFIG_SYSFS is defined. You can access
it by doing::

    mount -t sysfs sysfs /sys
```

### 目录创建

每当一个 kobject 注册到系统中时，会在 sysfs 中为其创建一个目录。该目录是在 kobject 的父对象的子目录中创建的，这向用户空间表达了内部对象的层次结构。sysfs 中的顶级目录代表对象层次结构的共同祖先；即对象所属的子系统。

```plaintext
For every kobject that is registered with the system, a directory is
created for it in sysfs. That directory is created as a subdirectory
of the kobject's parent, expressing internal object hierarchies to
userspace. Top-level directories in sysfs represent the common
ancestors of object hierarchies; i.e. the subsystems the objects
belong to.
```

### 属性

属性可以以文件系统中的常规文件形式导出给 kobject。sysfs 将文件 I/O 操作转发给为这些属性定义的方法，从而提供读写内核属性的方法。

```plaintext
Attributes can be exported for kobjects in the form of regular files in
the filesystem. Sysfs forwards file I/O operations to methods defined
for the attributes, providing a means to read and write kernel
attributes.
```

属性应该是 ASCII 文本文件，最好每个文件只包含一个值。尽管如此，包含多个同类型值的数组也是可以接受的。

```plaintext
Attributes should be ASCII text files, preferably with only one value
per file. It is noted that it may not be efficient to contain only one
value per file, so it is socially acceptable to express an array of
values of the same type.
```

混合类型、表达多行数据以及进行花哨的数据格式化是被强烈反对的。这些做法可能会让你在公开场合受到羞辱，并且你的代码可能会在没有通知的情况下被重写。

```plaintext
Mixing types, expressing multiple lines of data, and doing fancy
formatting of data is heavily frowned upon. Doing these things may get
you publicly humiliated and your code rewritten without notice.
```

一个属性定义非常简单：

```c
An attribute definition is simply::

    struct attribute {
	    char                    * name;
	    struct module		*owner;
	    umode_t                 mode;
    };

    int sysfs_create_file(struct kobject * kobj, const struct attribute * attr);
    void sysfs_remove_file(struct kobject * kobj, const struct attribute * attr);
```

一个裸属性不包含读取或写入属性值的方法。子系统被鼓励为特定对象类型定义自己的属性结构和包装函数，以添加和删除属性。

```plaintext
A bare attribute contains no means to read or write the value of the
attribute. Subsystems are encouraged to define their own attribute
structure and wrapper functions for adding and removing attributes for
a specific object type.
```

例如，驱动模型定义了 `struct device_attribute` 如下：

```c
For example, the driver model defines struct device_attribute like::

    struct device_attribute {
	    struct attribute	attr;
	    ssize_t (*show)(struct device *dev, struct device_attribute *attr,
			    char *buf);
	    ssize_t (*store)(struct device *dev, struct device_attribute *attr,
			    const char *buf, size_t count);
    };

    int device_create_file(struct device *, const struct device_attribute *);
    void device_remove_file(struct device *, const struct device_attribute *);
```

它还定义了用于定义设备属性的辅助函数：

```c
It also defines this helper for defining device attributes::

    #define DEVICE_ATTR(_name, _mode, _show, _store) \
    struct device_attribute dev_attr_##_name = __ATTR(_name, _mode, _show, _store)
```

例如，声明：

```c
For example, declaring::

    static DEVICE_ATTR(foo, S_IWUSR | S_IRUGO, show_foo, store_foo);
```

等同于：

```c
is equivalent to doing::

    static struct device_attribute dev_attr_foo = {
	    .attr = {
		    .name = "foo",
		    .mode = S_IWUSR | S_IRUGO,
	    },
	    .show = show_foo,
	    .store = store_foo,
    };
```

在 `include/linux/kernel.h` 中提到“OTHER_WRITABLE? 通常被认为是一个坏主意。”所以尝试为所有人设置一个可写的 sysfs 文件将失败，并会恢复为只读模式。

```c
Note as stated in include/linux/kernel.h "OTHER_WRITABLE?  Generally
considered a bad idea." so trying to set a sysfs file writable for
everyone will fail reverting to RO mode for "Others".
```

对于常见情况，`sysfs.h` 提供了便捷的宏来简化属性定义，使代码更简洁和易读。上述情况可以简化为：

```c
For the common cases sysfs.h provides convenience macros to make
defining attributes easier as well as making code more concise and
readable. The above case could be shortened to:

static struct device_attribute dev_attr_foo = __ATTR_RW(foo);
```

可用来定义包装函数的辅助函数列表如下：

```plaintext
the list of helpers available to define your wrapper function is:

__ATTR_RO(name):
		 assumes default name_show and mode 0444
__ATTR_WO(name):
		 assumes a name_store only and is restricted to mode
                 0200 that is root write access only.
__ATTR_RO_MODE(name, mode):
	         fore more restrictive RO access currently
                 only use case is the EFI System Resource Table
                 (see drivers/firmware/efi/esrt.c)
__ATTR_RW(name):
	         assumes default name_show, name_store and setting
                 mode to 0644.
__ATTR_NULL:
	         which sets the name to NULL and is used as end of list
                 indicator (see: kernel/workqueue.c)
```

### 子系统特定的回调

当子系统定义一个新的属性类型时，它必须实现一组 sysfs 操作，用于将读写调用转发给属性所有者的 `show` 和 `store` 方法：

```c
Subsystem-Specific Callbacks
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When a subsystem defines a new attribute type, it must implement a
set of sysfs operations for forwarding read and write calls to the
show and store methods of the attribute owners::

    struct sysfs_ops {
	    ssize_t (*show)(struct kobject *, struct attribute *, char *);
	    ssize_t (*store)(struct kobject *, struct attribute *, const char *, size_t);
    };
```

[子系统应该已经定义了一个 `struct kobj_type` 作为此类型的描述符，`sysfs_ops` 指针存储在其中。有关更多信息，请参阅 kobject 文档。]

```plaintext
[ Subsystems should have already defined a struct kobj_type as a
descriptor for this type, which is where the sysfs_ops pointer is
stored. See the kobject documentation for more information. ]
```

当文件被读取或写入时，sysfs 调用适当的方法。该方法然后将通用的 `struct kobject` 和 `struct attribute` 指针转换为适当的指针类型，并调用相关的方法。

```plaintext
When a file is read or written, sysfs calls the appropriate method
for the type. The method then translates the generic struct kobject
and struct attribute pointers to the appropriate pointer types, and
calls the associated methods.
```

举个例子：

```c
To illustrate::

    #define to_dev_attr(_attr) container_of(_attr, struct device_attribute, attr)

    static ssize_t dev_attr_show(struct kobject *kobj, struct attribute *attr,
				char *buf)
    {
	    struct device_attribute *dev_attr = to_dev_attr(attr);
	    struct device *dev = kobj_to_dev(kobj);
	    ssize_t ret = -EIO;

	    if (dev_attr->show)
		    ret = dev_attr->show(dev, dev_attr, buf);
	    if (ret >= (ssize_t)PAGE_SIZE) {
		    printk("dev_attr_show: %pS returned bad count\n",
				    dev_attr->show);
	    }
	    return ret;
    }
```

### 读取/写入属性数据

要读取或写入属性，必须在声明属性时指定 `show()` 或 `store()` 方法。方法类型应该像为设备属性定义的那样简单：

```c
Reading/Writing Attribute Data
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To read or write attributes, show() or store() methods must be
specified when declaring the attribute. The method types should be as
simple as those defined for device attributes::

    ssize_t (*show)(struct device *dev, struct device_attribute *attr, char *buf);
    ssize_t (*store)(struct device *dev, struct device_attribute *attr,
		    const char *buf, size_t count);
```

换句话说，它们应该只接收一个对象、一个属性和一个缓冲区作为参数。

```plaintext
IOW, they should take only an object, an attribute, and a buffer as parameters.
```

sysfs 分配一个大小为 `PAGE_SIZE` 的缓冲区并将其传递给该方法。sysfs 将对每次读取或写入调用该方法一次。这迫使方法实现遵循以下行为：

```plaintext
sysfs allocates a buffer of size (PAGE_SIZE) and passes it to the
method. Sysfs will call the method exactly once for each read or
write. This forces the following behavior on the method
implementations:
```

- 在 `read(2)` 调用时，`show()` 方法应该填满整个缓冲区。请记住，一个属性应该只导出一个值或类似值的数组，因此这不应该太昂贵。

```plaintext
- On read(2), the show() method should fill the entire buffer.
  Recall that an attribute should only be exporting one value, or an
  array of similar values, so this shouldn't be that expensive.
```

- 这允许用户空间任意地对整个文件进行部分读取和前向查找。如果用户空间回到零位置或使用偏移量为 `0` 的 `pread(2)`，`show()` 方法将再次被调用，重新填充缓冲区。

```plaintext
  This allows userspace to do partial reads and forward seeks
  arbitrarily over the entire file at will. If userspace seeks back to
  zero or does a pread(2) with an offset of '0' the show() method will
  be called again, rearmed, to fill the buffer.
```

- 在 `write(2)` 调用时，sysfs 期望在第一次写入时传递整个缓冲区。然后 sysfs 将整个缓冲区传递给 `store()` 方法。在存储时，数据后会添加一个终止空字符。这使得像 `sysfs_streq()` 这样的函数可以安全使用。

```plaintext
- On write(2), sysfs expects the entire buffer to be passed during the
  first write. Sysfs then passes the entire buffer to the store() method.
  A terminating null is added after the data on stores. This makes
  functions like sysfs_streq() safe to use.
```

- 在写入 sysfs 文件时，用户空间进程应该首先读取整个文件，修改它希望更改的值，然后将整个缓冲区写回。

```plaintext
  When writing sysfs files, userspace processes should first read the
  entire file, modify the values it wishes to change, then write the
  entire buffer back.
```

- 属性方法实现应该在读取和写入值时操作相同的缓冲区。

```plaintext
  Attribute method implementations should operate on an identical
  buffer when reading and writing values.
```

其他注意事项：

```plaintext
Other notes:
```

- **写入会导致 `show()` 方法重新激活，无论当前文件位置如何**。

```plaintext
- Writing causes the show() method to be rearmed regardless of current
  file position.
```

> 在正常连续 `read` 读取的时候，`show()` 方法的调用行为取决于读取的方式和文件位置。
>
> 在 sysfs 中，每次 `read` 操作都会调用 `show()` 方法来填充缓冲区。具体行为如下：
>
> 1. **首次读取**：当你第一次读取 sysfs 文件时，`show()` 方法会被调用，填充缓冲区。
> 2. **连续读取**：如果读取操作没有改变文件偏移量（即连续读取），那么 `show()` 方法不会被再次调用。缓冲区中的数据会被直接返回，直到缓冲区的数据被读取完毕。
> 3. **重新读取**：如果文件偏移量被改变（例如，用户空间程序通过 `lseek` 调用将文件指针移动到文件开头），再次读取时，`show()` 方法会被重新调用，重新填充缓冲区。
>

- 缓冲区的长度总是 `PAGE_SIZE` 字节。在 i386 上，这个值是 4096。

```plaintext
- The buffer will always be PAGE_SIZE bytes in length. On i386, this
  is 4096.
```

- `show()` 方法应返回填充到缓冲区中的字节数。

```plaintext
- show() methods should return the number of bytes printed into the
  buffer.
```

- `show()` 方法在格式化返回给用户空间的值时应仅使用 `sysfs_emit()` 或 `sysfs_emit_at()`。

```plaintext
- show() should only use sysfs_emit() or sysfs_emit_at() when formatting
  the value to be returned to user space.
```

- `store()` 方法应返回缓冲区中使用的字节数。如果整个缓冲区都已使用，只返回 `count` 参数。

```plaintext
- store() should return the number of bytes used from the buffer. If the
  entire buffer has been used, just return the count argument.
```

- `show()` 或 `store()` 方法总是可以返回错误。如果有一个错误值传入，请务必返回一个错误。

```plaintext
- show() or store() can always return errors. If a bad value comes
  through, be sure to return an error.
```

- 传递给方法的对象将通过 sysfs 引用计数其嵌入对象固定在内存中。然而，该对象代表的物理实体（例如设备）可能不存在。如果必要，请确保有一种方法来检查这一点。

```plaintext
- The object passed to the methods will be pinned in memory via sysfs
  referencing counting its embedded object. However, the physical
  entity (e.g. device) the object represents may not be present. Be
  sure to have a way to check this, if necessary.
```

一个非常简单（且天真的）设备属性实现如下：

```c
A very simple (and naive) implementation of a device attribute is::

    static ssize_t show_name(struct device *dev, struct device_attribute *attr,
			    char *buf)
    {
	    return scnprintf(buf, PAGE_SIZE, "%s\n", dev->name);
    }

    static ssize_t store_name(struct device *dev, struct device_attribute *attr,
			    const char *buf, size_t count)
    {
	    snprintf(dev->name, sizeof(dev->name), "%.*s",
		    (int)min(count, sizeof(dev->name) - 1), buf);
	    return count;
    }

    static DEVICE_ATTR(name, S_IRUGO, show_name, store_name);
```

（注意，真实实现不允许用户空间设置设备的名称。）

```plaintext
(Note that the real implementation doesn't allow userspace to set the
name for a device.)
```

### 顶级目录布局

sysfs 目录结构暴露了内核数据结构的关系。

```plaintext
Top Level Directory Layout
~~~~~~~~~~~~~~~~~~~~~~~~~~

The sysfs directory arrangement exposes the relationship of kernel
data structures.
```

顶级 sysfs 目录如下所示：

```plaintext
The top level sysfs directory looks like::

    block/
    bus/
    class/
    dev/
    devices/
    firmware/
    net/
    fs/
```

`devices/` 包含设备树的文件系统表示。它直接映射到内核设备树，这是一个 `struct device` 的层次结构。

```plaintext
devices/ contains a filesystem representation of the device tree. It maps
directly to the internal kernel device tree, which is a hierarchy of
struct device.
```

`bus/` 包含各种总线类型的扁平目录布局。每个总线的目录包含两个子目录：

```plaintext
bus/ contains flat directory layout of the various bus types in the
kernel. Each bus's directory contains two subdirectories::

	devices/
	drivers/
```

`devices/` 包含系统中发现的每个设备的符号链接，指向设备在根目录下的目录。

```plaintext
devices/ contains symlinks for each device discovered in the system
that point to the device's directory under root/.
```

`drivers/` 包含每个设备驱动程序的目录，这些驱动程序是为该总线上的设备加载的（这假设驱动程序不会跨多个总线类型）。

```plaintext
drivers/ contains a directory for each device driver that is loaded
for devices on that particular bus (this assumes that drivers do not
span multiple bus types).
```

`fs/` 包含一些文件系统的目录。目前，每个希望导出属性的文件系统必须在 `fs/` 下创建自己的层次结构（参见 `./fuse.txt` 了解示例）。

```plaintext
fs/ contains a directory for some filesystems.  Currently each
filesystem wanting to export attributes must create its own hierarchy
below fs/ (see ./fuse.txt for an example).
```

`dev/` 包含两个目录 `char/` 和 `block/`。在这两个目录中，有名为 `<major>:<minor>` 的符号链接。这些符号链接指向给定设备的 sysfs 目录。`/sys/dev` 提供了一种从 `stat(2)` 操作的结果快速查找设备的 sysfs 接口的方法。

```plaintext
dev/ contains two directories char/ and block/. Inside these two
directories there are symlinks named <major>:<minor>.  These symlinks
point to the sysfs directory for the given device.  /sys/dev provides a
quick way to lookup the sysfs interface for a device from the result of
a stat(2) operation.
```

更多有关驱动模型特定功能的信息可以在 `Documentation/driver-api/driver-model/` 中找到。

```plaintext
More information can driver-model specific features can be found in
Documentation/driver-api/driver-model/.
```

### 当前接口

以下接口层目前存在于 sysfs 中：

```plaintext
Current Interfaces
~~~~~~~~~~~~~~~~~~

The following interface layers currently exist in sysfs:
```

#### 设备（`include/linux/device.h`）

结构体：

```c
devices (include/linux/device.h)
--------------------------------
Structure::

    struct device_attribute {
	    struct attribute	attr;
	    ssize_t (*show)(struct device *dev, struct device_attribute *attr,
			    char *buf);
	    ssize_t (*store)(struct device *dev, struct device_attribute *attr,
			    const char *buf, size_t count);
    };
```

声明：

```c
Declaring::

    DEVICE_ATTR(_name, _mode, _show, _store);
```

创建/删除：

```c
Creation/Removal::

    int device_create_file(struct device *dev, const struct device_attribute * attr);
    void device_remove_file(struct device *dev, const struct device_attribute * attr);
```

#### 总线驱动程序（`include/linux/device.h`）

结构体：

```c
bus drivers (include/linux/device.h)
------------------------------------
Structure::

    struct bus_attribute {
	    struct attribute        attr;
	    ssize_t (*show)(struct bus_type *, char * buf);
	    ssize_t (*store)(struct bus_type *, const char * buf, size_t count);
    };
```

声明：

```c
Declaring::

    static BUS_ATTR_RW(name);
    static BUS_ATTR_RO(name);
    static BUS_ATTR_WO(name);
```

创建/删除：

```c
Creation/Removal::

    int bus_create_file(struct bus_type *, struct bus_attribute *);
    void bus_remove_file(struct bus_type *, struct bus_attribute *);
```

#### 设备驱动程序（`include/linux/device.h`）

结构体：

```c
device drivers (include/linux/device.h)
---------------------------------------

Structure::

    struct driver_attribute {
	    struct attribute        attr;
	    ssize_t (*show)(struct device_driver *, char * buf);
	    ssize_t (*store)(struct device_driver *, const char * buf,
			    size_t count);
    };
```

声明：

```c
Declaring::

    DRIVER_ATTR_RO(_name)
    DRIVER_ATTR_RW(_name)
```

创建/删除：

```c
Creation/Removal::

    int driver_create_file(struct device_driver *, const struct driver_attribute *);
    void driver_remove_file(struct device_driver *, const struct driver_attribute *);
```

### 文档

sysfs 目录结构和每个目录中的属性定义了内核与用户空间之间的 ABI。对于任何 ABI 来说，确保其稳定性和正确文档化是非常重要的。所有新的 sysfs 属性必须在 `Documentation/ABI` 中进行文档化。有关更多信息，请参阅 `Documentation/ABI/README`。

```plaintext
Documentation
~~~~~~~~~~~~~

The sysfs directory structure and the attributes in each directory define an
ABI between the kernel and user space. As for any ABI, it is important that
this ABI is stable and properly documented. All new sysfs attributes must be
documented in Documentation/ABI. See also Documentation/ABI/README for more
information.
```
