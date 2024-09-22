
---
title: Linux configfs
top_img: transparent
date: 2024-09-22 10:37:42
updated: 2024-09-22 10:37:46
tags:
  - Linux
  - configfs
categories: Linux
keywords:
description:
---

> translate from https://www.kernel.org/doc/Documentation/filesystems/configfs/configfs.txt

---

## configfs - 用户空间驱动的内核对象配置

乔尔·贝克尔 <joel.becker@oracle.com>

更新日期：2005年3月31日

版权所有 (c) 2005 Oracle Corporation,
乔尔·贝克尔 <joel.becker@oracle.com>

---

## 什么是 configfs？

`configfs` 是基于内存的文件系统，它提供与 `sysfs` 相反的功能。`sysfs` 提供的是内核对象的文件系统视图，而 `configfs` 则是内核对象或 `config_items` 的文件系统管理器。

```plaintext
configfs is a ram-based filesystem that provides the converse of
sysfs's functionality. Where sysfs is a filesystem-based view of
kernel objects, configfs is a filesystem-based manager of kernel
objects, or config_items.
```

> **view VS manager** 的区别。

在 `sysfs` 中，当一个对象在内核中创建时（例如，发现一个设备），它会被注册到 `sysfs` 中。它的属性会出现在 `sysfs` 中，允许用户空间通过 `readdir(3)`/`read(2)` 读取这些属性。某些属性也许可以通过 `write(2)` 进行修改。重要的是，内核控制着这些对象的生命周期，`sysfs` 仅仅是这些操作的一个窗口。

```plaintext
With sysfs, an object is created in kernel (for example, when a device
is discovered) and it is registered with sysfs. Its attributes then
appear in sysfs, allowing userspace to read the attributes via
readdir(3)/read(2). It may allow some attributes to be modified via
write(2). The important point is that the object is created and
destroyed in kernel, the kernel controls the lifecycle of the sysfs
representation, and sysfs is merely a window on all this.
```

而 `configfs` 的 `config_item` 是通过用户空间的显式操作创建的：使用 `mkdir(2)`。它通过 `rmdir(2)` 删除。属性在 `mkdir(2)` 操作时出现，可以通过 `read(2)` 和 `write(2)` 进行读取和修改。与 `sysfs` 类似，`readdir(3)` 可以查询项目和/或属性的列表，`symlink(2)` 可以用于将项目分组。与 `sysfs` 不同的是，表示的**生命周期完全由用户空间控制**，内核模块必须对此作出响应。

```plaintext
A configfs config_item is created via an explicit userspace operation:
mkdir(2). It is destroyed via rmdir(2). The attributes appear at
mkdir(2) time, and can be read or modified via read(2) and write(2).
As with sysfs, readdir(3) queries the list of items and/or attributes.
symlink(2) can be used to group items together. Unlike sysfs, the
lifetime of the representation is completely driven by userspace. The
kernel modules backing the items must respond to this.
```

`sysfs` 和 `configfs` 可以并且应该共存于同一系统上，它们互不替代。

```plaintext
Both sysfs and configfs can and should exist together on the same
system. One is not a replacement for the other.
```

---

## 使用 configfs

`configfs` 可以作为模块编译，也可以编译到内核中。可以通过以下方式访问它：

```bash
mount -t configfs none /config
```

```plaintext
configfs can be compiled as a module or into the kernel. You can access
it by doing

	mount -t configfs none /config
```

除非加载了客户端模块，否则 `configfs` 树是空的。这些模块会将它们的项目类型注册为 `configfs` 的子系统。一旦客户端子系统被加载，它会作为 `/config` 下的一个（或多个）子目录出现。和 `sysfs` 一样，`configfs` 树始终存在，无论是否挂载到 `/config`。

```plaintext
The configfs tree will be empty unless client modules are also loaded.
These are modules that register their item types with configfs as
subsystems. Once a client subsystem is loaded, it will appear as a
subdirectory (or more than one) under /config. Like sysfs, the
configfs tree is always there, whether mounted on /config or not.
```

> 如果打开了某个子模块的 configfs 配置，在 mount configfs 之后，就能在对应的目录看到。比如 `CONFIG_USB_CONFIGFS=y` 开启之后，`/config/usb` 就会出现。

一个项目可以通过 `mkdir(2)` 创建。项目的属性也会在此时出现。可以通过 `readdir(3)` 确定属性，`read(2)` 查询默认值，`write(2)` 存储新值。不要在一个属性文件中混合多个属性。

```plaintext
An item is created via mkdir(2). The item's attributes will also
appear at this time. readdir(3) can determine what the attributes are,
read(2) can query their default values, and write(2) can store new
values. Don't mix more than one attribute in one attribute file.
```

> 比如：[Linux-USBmtp](./Linux-USBmtp.md#2-创建复合设备)

`configfs` 属性有两种类型：

1. 正常属性，与 `sysfs` 属性类似，是小型 ASCII 文本文件，最大大小为一页（`PAGE_SIZE`，在 i386 上为 4096 字节）。每个文件中最好只包含一个值，并且遵循 `sysfs` 的相同注意事项。`configfs` 期望 `write(2)` 在一次操作中写入整个缓冲区。当写入正常的 `configfs` 属性时，**用户空间进程应首先读取整个文件，修改他们想要更改的部分，然后写回整个缓冲区**。

```plaintext
There are two types of configfs attributes:

* Normal attributes, which similar to sysfs attributes, are small ASCII text
files, with a maximum size of one page (PAGE_SIZE, 4096 on i386). Preferably
only one value per file should be used, and the same caveats from sysfs apply.
Configfs expects write(2) to store the entire buffer at once. When writing to
normal configfs attributes, userspace processes should first read the entire
file, modify the portions they wish to change, and then write the entire
buffer back.
```

2. 二进制属性，与 `sysfs` 二进制属性类似，但语义上有一些细微变化。`PAGE_SIZE` 限制不适用，但整个二进制项必须适合单个内核 `vmalloc` 分配的缓冲区。来自用户空间的 `write(2)` 调用是缓冲的，属性的 `write_bin_attribute` 方法将在最终关闭时被调用，因此用户空间必须检查 `close(2)` 的返回码以验证操作是否成功完成。为了防止恶意用户导致内核内存不足，每个二进制属性都有最大缓冲区大小限制。

```plaintext
* Binary attributes, which are somewhat similar to sysfs binary attributes,
but with a few slight changes to semantics. The PAGE_SIZE limitation does not
apply, but the whole binary item must fit in single kernel vmalloc'ed buffer.
The write(2) calls from user space are buffered, and the attributes'
write_bin_attribute method will be invoked on the final close, therefore it is
imperative for user-space to check the return code of close(2) in order to
verify that the operation finished successfully.
To avoid a malicious user OOMing the kernel, there's a per-binary attribute
maximum buffer value.
```

当需要销毁一个项目时，使用 `rmdir(2)` 删除它。如果有其他项目通过 `symlink(2)` 链接到该项目，它是不能被删除的。链接可以通过 `unlink(2)` 删除。

```plaintext
When an item needs to be destroyed, remove it with rmdir(2). An
item cannot be destroyed if any other item has a link to it (via
symlink(2)). Links can be removed via unlink(2).
```

---

## 配置 FakeNBD: 一个例子

假设有一个网络块设备（NBD）驱动程序允许您访问远程块设备，称之为 FakeNBD。FakeNBD 使用 `configfs` 进行配置。显然，会有一个专门的程序供系统管理员使用来配置 FakeNBD，但该程序必须通过某种方式与驱动程序通信。这时 `configfs` 就派上用场了。

```plaintext
[Configuring FakeNBD: an Example]

Imagine there's a Network Block Device (NBD) driver that allows you to
access remote block devices. Call it FakeNBD. FakeNBD uses configfs
for its configuration. Obviously, there will be a nice program that
sysadmins use to configure FakeNBD, but somehow that program has to tell
the driver about it. Here's where configfs comes in.
```

当加载 FakeNBD 驱动程序时，它会注册到 `configfs`。使用 `readdir(3)` 可以看到这一点：

```bash
# ls /config
fakenbd
```

```plaintext
When the FakeNBD driver is loaded, it registers itself with configfs.
readdir(3) sees this just fine:

	# ls /config
	fakenbd
```

可以使用 `mkdir(2)` 创建一个 fakenbd 连接。名称是任意的，但该工具可能会使用这个名称。也许它是一个 `uuid` 或磁盘名称：

```bash
# mkdir /config/fakenbd/disk1
# ls /config/fakenbd/disk1
target device rw
```

```plaintext
A fakenbd connection can be created with mkdir(2). The name is
arbitrary, but likely the tool will make some use of the name. Perhaps
it is a uuid or a disk name:

# mkdir /config/fakenbd/disk1
# ls /config/fakenbd/disk1
target device rw
```

`target` 属性包含 FakeNBD 将连接的服务器的 IP 地址。`device` 属性是服务器上的设备。显然，`rw` 属性决定了连接是只读还是读写。

```bash
# echo 10.0.0.1 > /config/fakenbd/disk1/target
# echo /dev/sda1 > /config/fakenbd/disk1/device
# echo 1 > /config/fakenbd/disk1/rw
```

```plaintext
The target attribute contains the IP address of the server FakeNBD will
connect to. The device attribute is the device on the server.
Predictably, the rw attribute determines whether the connection is
read-only or read-write.

# echo 10.0.0.1 > /config/fakenbd/disk1/target
# echo /dev/sda1 > /config/fakenbd/disk1/device
# echo 1 > /config/fakenbd/disk1/rw
```

就是这样，仅此而已。现在，设备已配置好，并通过 shell 完成。

```plaintext
That's it. That's all there is. Now the device is configured, via the
shell no less.
```

---

## 使用 configfs 编码

```plaintext
[Coding With configfs]
```

在 `configfs` 中，每个对象都是一个 `config_item`。一个 `config_item` 反映了子系统中的一个对象。它具有与该对象的值匹配的属性。`configfs` 处理该对象及其属性的文件系统表示形式，允许子系统忽略除基本 `show/store` 交互以外的所有操作。

```plaintext
Every object in configfs is a config_item. A config_item reflects an
object in the subsystem. It has attributes that match values on that
object. configfs handles the filesystem representation of that object
and its attributes, allowing the subsystem to ignore all but the
basic show/store interaction.
```

项目是在 `config_group` 中创建和销毁的。`group` 是一组具有相同属性和操作的项目。项目通过 `mkdir(2)` 创建，通过 `rmdir(2)` 移除，但 `configfs` 处理这些操作。`group` 有一组操作来执行这些任务。

```plaintext
Items are created and destroyed inside a config_group. A group is a
collection of items that share the same attributes and operations.
Items are created by mkdir(2) and removed by rmdir(2), but configfs
handles that. The group has a set of operations to perform these tasks.
```

子系统是客户端模块的顶层。在初始化期间，客户端模块将子系统注册到 `configfs`，子系统将作为 `configfs` 文件系统顶部的一个目录出现。子系统也是一个 `config_group`，并且可以执行 `config_group` 能做的所有操作。

```plaintext
A subsystem is the top level of a client module. During initialization,
the client module registers the subsystem with configfs, the subsystem
appears as a directory at the top of the configfs filesystem. A
subsystem is also a config_group, and can do everything a config_group
can.
```

> `USB CONFIGFS` 就是一个 `config_group/subsystem` 。

---

### `struct config_item`

```c
	struct config_item {
		char                    *ci_name;
		char                    ci_namebuf[UOBJ_NAME_LEN];
		struct kref             ci_kref;
		struct list_head        ci_entry;
		struct config_item      *ci_parent;
		struct config_group     *ci_group;
		struct config_item_type *ci_type;
		struct dentry           *ci_dentry;
	};
```

```c
	void config_item_init(struct config_item *);
	void config_item_init_type_name(struct config_item *,
					const char *name,
					struct config_item_type *type);
	struct config_item *config_item_get(struct config_item *);
	void config_item_put(struct config_item *);
```

一般来说，`struct config_item` 被嵌入到一个容器结构中，后者实际上表示子系统正在执行的操作。该结构中的 `config_item` 部分是对象与 `configfs` 交互的方式。

```plaintext
Generally, struct config_item is embedded in a container structure, a
structure that actually represents what the subsystem is doing. The
config_item portion of that structure is how the object interacts with
configfs.
```

无论是在源文件中静态定义还是由父 `config_group` 创建，都必须调用 `_init()` 函数之一对 `config_item` 进行初始化。这会初始化引用计数并设置相应的字段。

```plaintext
Whether statically defined in a source file or created by a parent
config_group, a config_item must have one of the _init() functions
called on it. This initializes the reference count and sets up the
appropriate fields.
```

---

所有 `config_item` 的用户都应该通过 `config_item_get()` 获取其引用，并在使用完成后通过 `config_item_put()` 释放引用。

```plaintext
All users of a config_item should have a reference on it via
config_item_get(), and drop the reference when they are done via
config_item_put().
```

仅仅有一个 `config_item`，它除了出现在 `configfs` 中以外，几乎不能做任何事情。通常子系统希望该项目显示和/或存储属性，以及执行其他操作。为此，它需要一个类型。

```plaintext
By itself, a config_item cannot do much more than appear in configfs.
Usually a subsystem wants the item to display and/or store attributes,
among other things. For that, it needs a type.
```

---

### `struct config_item_type`

```c
	struct configfs_item_operations {
		void (*release)(struct config_item *);
		int (*allow_link)(struct config_item *src,
				  struct config_item *target);
		void (*drop_link)(struct config_item *src,
				 struct config_item *target);
	};
```

```c
	struct config_item_type {
		struct module                           *ct_owner;
		struct configfs_item_operations         *ct_item_ops;
		struct configfs_group_operations        *ct_group_ops;
		struct configfs_attribute               **ct_attrs;
		struct configfs_bin_attribute		**ct_bin_attrs;
	};
```

`config_item_type` 的最基本功能是定义可以对 `config_item` 执行的操作。所有动态分配的项目都需要提供 `ct_item_ops->release()` 方法。当 `config_item` 的引用计数降为零时，会调用此方法。

```plaintext
The most basic function of a config_item_type is to define what
operations can be performed on a config_item. All items that have been
allocated dynamically will need to provide the ct_item_ops->release()
method. This method is called when the config_item's reference count
reaches zero.
```

---

### `struct configfs_attribute`

```c
	struct configfs_attribute {
		char                    *ca_name;
		struct module           *ca_owner;
		umode_t                  ca_mode;
		ssize_t (*show)(struct config_item *, char *);
		ssize_t (*store)(struct config_item *, const char *, size_t);
	};
```

当 `config_item` 希望某个属性作为文件出现在其 `configfs` 目录中时，它必须定义一个描述该属性的 `configfs_attribute`。然后它将该属性添加到 `config_item_type->ct_attrs` 的以 NULL 结尾的数组中。当项目出现在 `configfs` 中时，属性文件将以 `configfs_attribute->ca_name` 文件名出现。`configfs_attribute->ca_mode` 指定文件权限。

```plaintext
When a config_item wants an attribute to appear as a file in the item's
configfs directory, it must define a configfs_attribute describing it.
It then adds the attribute to the NULL-terminated array
config_item_type->ct_attrs. When the item appears in configfs, the
attribute file will appear with the configfs_attribute->ca_name
filename. configfs_attribute->ca_mode specifies the file permissions.
```

如果属性是可读的并且提供了 `->show` 方法，则每当用户空间请求读取该属性时，该方法将被调用。如果属性是可写的并且提供了 `->store` 方法，则每当用户空间请求写入该属性时，该方法将被调用。

```plaintext
If an attribute is readable and provides a ->show method, that method will
be called whenever userspace asks for a read(2) on the attribute. If an
attribute is writable and provides a ->store method, that method will be
be called whenever userspace asks for a write(2) on the attribute.
```

---

### `struct configfs_bin_attribute`

```c
	struct configfs_bin_attribute {
		struct configfs_attribute	cb_attr;
		void				*cb_private;
		size_t				cb_max_size;
	};
```

当需要一个二进制 blob 作为文件内容出现在项目的 `configfs` 目录中时，可以使用 `configfs_bin_attribute`。要实现这一点，将二进制属性添加到 `config_item_type->ct_bin_attrs` 的以 NULL 结尾的数组中，当项目出现在 `configfs` 中时，属性文件将以 `configfs_bin_attribute->cb_attr.ca_name` 文件名出现。`configfs_bin_attribute->cb_attr.ca_mode` 指定文件权限。

```plaintext
The binary attribute is used when the one needs to use a binary blob to
appear as the contents of a file in the item's configfs directory.
To do so, add the binary attribute to the NULL-terminated array
config_item_type->ct_bin_attrs, and the item appears in configfs, the
attribute file will appear with the configfs_bin_attribute->cb_attr.ca_name
filename. configfs_bin_attribute->cb_attr.ca_mode specifies the file
permissions.
```

`cb_private` 成员是为驱动程序使用而提供的，而 `cb_max_size` 成员指定用于该二进制属性的最大 `vmalloc` 缓冲区大小。

```plaintext
The cb_private member is provided for use by the driver, while the
cb_max_size member specifies the maximum amount of vmalloc buffer
to be used.
```

如果二进制属性是可读的，并且 `config_item` 提供了 `ct_item_ops->read_bin_attribute()` 方法，那么每当用户空间请求读取该属性时，将调用该方法。对于 `write(2)` 也是如此。读/写操作是缓冲的，因此只会进行一次读/写；属性本身不需要关心这一点。

```plaintext
If binary attribute is readable and the config_item provides a
ct_item_ops->read_bin_attribute() method, that method will be called
whenever userspace asks for a read(2) on the attribute. The converse
will happen for write(2). The reads/writes are buffered, so only a
single read/write will occur; the attributes need not concern itself
with it.
```

---

### `struct config_group`

`config_item` 不能独立存在。创建 `config_item` 的唯一方法是通过在 `config_group` 上使用 `mkdir(2)`，这将触发子项的创建。

```c
	struct config_group {
		struct config_item		cg_item;
		struct list_head		cg_children;
		struct configfs_subsystem 	*cg_subsys;
		struct list_head		default_groups;
		struct list_head		group_entry;
	};
```

```c
	void config_group_init(struct config_group *group);
	void config_group_init_type_name(struct config_group *group,
					 const char *name,
					 struct config_item_type *type);
```

`config_group` 结构包含一个 `config_item`。适当配置该项意味着 `group` 可以作为自己的项目发挥作用。不过，它可以做更多：它可以创建子项目或子组。这是通过在组的 `config_item_type` 上指定的组操作来实现的。

```plaintext
The config_group structure contains a config_item. Properly configuring
that item means that a group can behave as an item in its own right.
However, it can do more: it can create child items or groups. This is
accomplished via the group operations specified on the group's
config_item_type.
```

```c
	struct configfs_group_operations {
		struct config_item *(*make_item)(struct config_group *group,
						 const char *name);
		struct config_group *(*make_group)(struct config_group *group,
						   const char *name);
		int (*commit_item)(struct config_item *item);
		void (*disconnect_notify)(struct config_group *group,
					  struct config_item *item);
		void (*drop_item)(struct config_group *group,
				  struct config_item *item);
	};
```

---

如果组提供了 `ct_group_ops->make_item()` 方法，则通过 `mkdir(2)` 在组的目录中调用该方法创建子项。子系统分配一个新的 `config_item`（或更可能是其容器结构），对其进行初始化并返回给 `configfs`。然后 `configfs` 将填充文件系统树以反映新项。

```plaintext
A group creates child items by providing the
ct_group_ops->make_item() method. If provided, this method is called from mkdir(2) in the group's directory. The subsystem allocates a new
config_item (or more likely, its container structure), initializes it,
and returns it to configfs. Configfs will then populate the filesystem
tree to reflect the new item.
```

如果子系统希望子项本身也是一个组，则子系统会提供 `ct_group_ops->make_group()` 方法。其他所有行为相同，使用组的 `_init()` 函数对该组进行初始化。

```plaintext
If the subsystem wants the child to be a group itself, the subsystem
provides ct_group_ops->make_group(). Everything else behaves the same,
using the group _init() functions on the group.
```

最后，当用户空间在该项或组上调用 `rmdir(2)` 时，`ct_group_ops->drop_item()` 方法将被调用。由于 `config_group` 也是一个 `config_item`，因此不需要单独的 `drop_group()` 方法。子系统必须在项目分配时初始化的引用上调用 `config_item_put()`。如果子系统没有任何工作要做，则可以省略 `ct_group_ops->drop_item()` 方法，`configfs` 将代表子系统在该项上调用 `config_item_put()`。

```plaintext
Finally, when userspace calls rmdir(2) on the item or group,
ct_group_ops->drop_item() is called. As a config_group is also a
config_item, it is not necessary for a separate drop_group() method.
The subsystem must config_item_put() the reference that was initialized
upon item allocation. If a subsystem has no work to do, it may omit
the ct_group_ops->drop_item() method, and configfs will call
config_item_put() on the item on behalf of the subsystem.
```

重要提示：`drop_item()` 是 `void` 函数，因此不能返回失败。当调用 `rmdir(2)` 时，`configfs` 将从文件系统树中删除该项（假设没有子项保持它忙碌）。子系统负责对此做出响应。如果子系统在其他线程中引用了该项，则内存是安全的。该项实际上可能需要一些时间才能从子系统的使用中消失。但它已经从 `configfs` 中删除。

```plaintext
IMPORTANT: drop_item() is void, and as such cannot fail. When rmdir(2)
is called, configfs WILL remove the item from the filesystem tree
(assuming that it has no children to keep it busy). The subsystem is
responsible for responding to this. If the subsystem has references to
the item in other threads, the memory is safe. It may take some time
for the item to actually disappear from the subsystem's usage. But it
is gone from configfs.
```

当调用 `drop_item()` 时，项的链接已经被拆除。它不再拥有其父项的引用，并且在项目层次结构中没有位置。如果客户端需要在这种拆除发生之前进行一些清理操作，子系统可以实现 `ct_group_ops->disconnect_notify()` 方法。该方法在 `configfs` 从文件系统视图中删除该项后，但在从其父组中删除之前调用。与 `drop_item()` 一样，`disconnect_notify()` 是 `void` 函数，不能返回失败。客户端子系统不应在这里删除任何引用，因为它们仍然必须在 `drop_item()` 中执行。

```plaintext
When drop_item() is called, the item's linkage has already been torn
down. It no longer has a reference on its parent and has no place in
the item hierarchy. If a client needs to do some cleanup before this
teardown happens, the subsystem can implement the
ct_group_ops->disconnect_notify() method. The method is called after
configfs has removed the item from the filesystem view but before the
item is removed from its parent group. Like drop_item(),
disconnect_notify() is void and cannot fail. Client subsystems should
not drop any references here, as they still must do it in drop_item().
```

如果 `config_group` 仍有子项，则无法将其删除。这在 `configfs` 的 `rmdir(2)` 代码中实现。不会调用 `->drop_item()`，因为该项尚未被删除。`rmdir(2)` 将失败，因为目录不为空。

```plaintext
A config_group cannot be removed while it still has child items. This
is implemented in the configfs rmdir(2) code. ->drop_item() will not be
called, as the item has not been dropped. rmdir(2) will fail, as the
directory is not empty.
```

---

### `struct configfs_subsystem`

🔥🔥 **子系统必须在模块初始化期间注册自身。这会告诉 `configfs` 将子系统显示在文件树中**。

```c
	struct configfs_subsystem {
		struct config_group	su_group;
		struct mutex		su_mutex;
	};
```

```c
	int configfs_register_subsystem(struct configfs_subsystem *subsys);
	void configfs_unregister_subsystem(struct configfs_subsystem *subsys);
```

一个子系统由一个顶层 `config_group` 和一个互斥锁组成。该组是创建子 `config_items` 的位置。对于子系统来说，这个组通常是静态定义的。在调用 `configfs_register_subsystem()` 之前，子系统必须通过常规的组初始化函数初始化该组，并且还必须初始化互斥锁。

```plaintext
A subsystem consists of a toplevel config_group and a mutex.
The group is where child config_items are created. For a subsystem,
this group is usually defined statically. Before calling
configfs_register_subsystem(), the subsystem must have initialized the
group via the usual group _init() functions, and it must also have
initialized the mutex.
```

当注册调用返回时，子系统变为活动状态，并通过 `configfs` 可见。此时，可以调用 `mkdir(2)`，子系统必须准备好处理它。

```plaintext
When the register call returns, the subsystem is live, and it
will be visible via configfs. At that point, mkdir(2) can be called and
the subsystem must be ready for it.
```

---

## 示例

这些基本概念的最佳示例是 `samples/configfs/configfs_sample.c` 中的 `simple_children` 子系统/组和 `simple_child` 项。它展示了一个显示和存储属性的简单对象，以及一个创建和销毁这些子项的简单组。

```plaintext
[An Example]

The best example of these basic concepts is the simple_children
subsystem/group and the simple_child item in
samples/configfs/configfs_sample.c. It shows a trivial object displaying
and storing an attribute, and a simple group creating and destroying
these children.
```

> [https://elixir.bootlin.com/linux/v5.10.186/source/samples/configfs/configfs_sample.c](https://elixir.bootlin.com/linux/v5.10.186/source/samples/configfs/configfs_sample.c)
> 可以试试编译，加一些日志看看。
>
> 这里起来时，将 configfs 挂载到了 /tmp/usb 目录。不好，有歧义🌚。
>
> ```plaintext
> [root@milkv-duo]~# ls
> configfs_sample.ko
> [root@milkv-duo]~# ls /tmp/usb
> usb_gadget
> [root@milkv-duo]~# insmod configfs_sample.ko
> [root@milkv-duo]~# ls /tmp/usb
> 01-childless  02-simple-children  03-group-children  usb_gadget
> [root@milkv-duo]~# cd /tmp/usb/
> [root@milkv-duo]/tmp/usb# tree 0*
> 01-childless
> ├── description
> ├── showme
> └── storeme
> 02-simple-children
> └── description
> 03-group-children
> └── description
>
> 0 directories, 5 files
> ```

### 00-init

我们现在已经完成了子系统的定义。为了方便起见，这里列出了所有的子系统。这使得初始化函数可以轻松注册它们。大多数模块只包含一个子系统，并且只会直接调用 `register_subsystem` 进行注册。

> 如前面 [`struct configfs_subsystem`](#struct-configfs_subsystem) 的描述，通过调用
> - `config_group_init(&subsys->su_group);`
> - `mutex_init(&subsys->su_mutex);`
> - `configfs_register_subsystem(subsys);`
>
> 来完成configfs 子系统的注册。

```c
/*
 * We're now done with our subsystem definitions.
 * For convenience in this module, here's a list of them all.  It
 * allows the init function to easily register them.  Most modules
 * will only have one subsystem, and will only call register_subsystem
 * on it directly.
 */
static struct configfs_subsystem *example_subsys[] = {
	&childless_subsys.subsys,
	&simple_children_subsys,
	&group_children_subsys,
	NULL,
};

static int __init configfs_example_init(void)
{
	struct configfs_subsystem *subsys;
	int ret, i;

	for (i = 0; example_subsys[i]; i++) {
		subsys = example_subsys[i];

		config_group_init(&subsys->su_group);
		mutex_init(&subsys->su_mutex);
		ret = configfs_register_subsystem(subsys);
		if (ret) {
			pr_err("Error %d while registering subsystem %s\n",
			       ret, subsys->su_group.cg_item.ci_namebuf);
			goto out_unregister;
		}
	}

	return 0;

out_unregister:
	for (i--; i >= 0; i--)
		configfs_unregister_subsystem(example_subsys[i]);

	return ret;
}

static void __exit configfs_example_exit(void)
{
	int i;

	for (i = 0; example_subsys[i]; i++)
		configfs_unregister_subsystem(example_subsys[i]);
}

module_init(configfs_example_init);
module_exit(configfs_example_exit);
MODULE_LICENSE("GPL");
```

---

### 01-childless

这个第一个例子是一个没有子项的子系统。它不能创建任何 `config_items`，只包含属性。

请注意，我们将 `configfs_subsystem` 封装在一个容器中。如果子系统本身没有直接的属性，这并不是必须的。可以参考下一个例子，02-simple-children，来了解这种子系统。

```c
/*
 * 01-childless
 *
 * This first example is a childless subsystem.  It cannot create
 * any config_items.  It just has attributes.
 *
 * Note that we are enclosing the configfs_subsystem inside a container.
 * This is not necessary if a subsystem has no attributes directly
 * on the subsystem.  See the next example, 02-simple-children, for
 * such a subsystem.
 */

struct childless {
	struct configfs_subsystem subsys;
	int showme;
	int storeme;
};

static inline struct childless *to_childless(struct config_item *item)
{
	return container_of(to_configfs_subsystem(to_config_group(item)),
			    struct childless, subsys);
}

static ssize_t childless_showme_show(struct config_item *item, char *page)
{
	struct childless *childless = to_childless(item);
	ssize_t pos;

	pos = sprintf(page, "%d\n", childless->showme);
	childless->showme++;

	return pos;
}

static ssize_t childless_storeme_show(struct config_item *item, char *page)
{
	return sprintf(page, "%d\n", to_childless(item)->storeme);
}

static ssize_t childless_storeme_store(struct config_item *item,
		const char *page, size_t count)
{
	struct childless *childless = to_childless(item);
	int ret;

	ret = kstrtoint(page, 10, &childless->storeme);
	if (ret)
		return ret;

	return count;
}

static ssize_t childless_description_show(struct config_item *item, char *page)
{
	return sprintf(page,
"[01-childless]\n"
"\n"
"The childless subsystem is the simplest possible subsystem in\n"
"configfs.  It does not support the creation of child config_items.\n"
"It only has a few attributes.  In fact, it isn't much different\n"
"than a directory in /proc.\n");
}

/* https://elixir.bootlin.com/linux/v5.10.186/source/include/linux/configfs.h#L134
#define CONFIGFS_ATTR(_pfx, _name)			\
static struct configfs_attribute _pfx##attr_##_name = {	\
	.ca_name	= __stringify(_name),		\
	.ca_mode	= S_IRUGO | S_IWUSR,		\
	.ca_owner	= THIS_MODULE,			\
	.show		= _pfx##_name##_show,		\
	.store		= _pfx##_name##_store,		\
}

#define CONFIGFS_ATTR_RO(_pfx, _name)			\
static struct configfs_attribute _pfx##attr_##_name = {	\
	.ca_name	= __stringify(_name),		\
	.ca_mode	= S_IRUGO,			\
	.ca_owner	= THIS_MODULE,			\
	.show		= _pfx##_name##_show,		\
}

#define CONFIGFS_ATTR_WO(_pfx, _name)			\
static struct configfs_attribute _pfx##attr_##_name = {	\
	.ca_name	= __stringify(_name),		\
	.ca_mode	= S_IWUSR,			\
	.ca_owner	= THIS_MODULE,			\
	.store		= _pfx##_name##_store,		\
}
*/

// 当项目出现在 configfs 中时，属性文件将以 configfs_attribute->ca_name 文件名出现。
// configfs_attribute->ca_mode 指定文件权限。
// ✨✨✨ 注意这里宏中，指定了 .show/.store 对应的处理函数。
CONFIGFS_ATTR_RO(childless_, showme);
CONFIGFS_ATTR(childless_, storeme);
CONFIGFS_ATTR_RO(childless_, description);
/*
[root@milkv-duo]/tmp/usb/01-childless# ls -lh
total 0
-r--r--r-- 1 root root 4.0K Jan  1 05:15 description
-r--r--r-- 1 root root 4.0K Jan  1 05:15 showme
-rw-r--r-- 1 root root 4.0K Jan  1 05:15 storeme
*/

// 当 config_item 希望某个属性作为文件出现在其 configfs 目录中时，
// 它必须定义一个描述该属性的 configfs_attribute。然后它将该属性添加到
// config_item_type->ct_attrs 的以 NULL 结尾的数组中。
static struct configfs_attribute *childless_attrs[] = {
	&childless_attr_showme,
	&childless_attr_storeme,
	&childless_attr_description,
	NULL,
};

static const struct config_item_type childless_type = {
	.ct_attrs	= childless_attrs,
	.ct_owner	= THIS_MODULE,
};

static struct childless childless_subsys = {
	.subsys = {
		.su_group = {
			.cg_item = {
				.ci_namebuf = "01-childless",
				// config_item_type 的最基本功能是定义可以对 config_item 执行的操作
				.ci_type = &childless_type,
			},
		},
	},
};
```

> 倒着看代码，注意配合看前面的介绍。属性基本只有 `show/store` 两种操作，对应读写，可通过 `echo/cat` 测试。
>
> ```bash
> [root@milkv-duo]/tmp/usb# ls
> 01-childless  02-simple-children  03-group-children  usb_gadget
> [root@milkv-duo]/tmp/usb# cd 01-childless/
> [root@milkv-duo]/tmp/usb/01-childless# ls -lh
> total 0
> -r--r--r-- 1 root root 4.0K Jan  1 05:15 description
> -r--r--r-- 1 root root 4.0K Jan  1 05:15 showme
> -rw-r--r-- 1 root root 4.0K Jan  1 05:15 storeme
> [root@milkv-duo]/tmp/usb/01-childless# cat showme
> 0
> [root@milkv-duo]/tmp/usb/01-childless# cat showme
> 1
> [root@milkv-duo]/tmp/usb/01-childless# cat showme
> 2
> [root@milkv-duo]/tmp/usb/01-childless# cat showme
> 3
> [root@milkv-duo]/tmp/usb/01-childless# cat showme
> 4
> [root@milkv-duo]/tmp/usb/01-childless# cat storeme
> 0
> [root@milkv-duo]/tmp/usb/01-childless# cat storeme
> 0
> [root@milkv-duo]/tmp/usb/01-childless# cat description
> [01-childless]
>
> The childless subsystem is the simplest possible subsystem in
> configfs.  It does not support the creation of child config_items.
> It only has a few attributes.  In fact, it isn't much different
> than a directory in /proc.
> [root@milkv-duo]/tmp/usb/01-childless# echo 0 > showme
> -sh: can't create showme: Permission denied
> [root@milkv-duo]/tmp/usb/01-childless# cat showme
> 5
> [root@milkv-duo]/tmp/usb/01-childless# echo 3 > storeme
> [root@milkv-duo]/tmp/usb/01-childless# cat storeme
> 3
> ```

---

### 02-simple-children

这个例子仅包含一个简单的、带有单一属性的子项。注意，这里没有额外的属性结构，因为子项的属性一开始就是已知的。此外，子系统没有容器，因为它自身没有任何属性。

```c
/*
 * 02-simple-children
 *
 * This example merely has a simple one-attribute child.  Note that
 * there is no extra attribute structure, as the child's attribute is
 * known from the get-go.  Also, there is no container for the
 * subsystem, as it has no attributes of its own.
 */

struct simple_child {
	struct config_item item;
	int storeme;
};

static inline struct simple_child *to_simple_child(struct config_item *item)
{
	return container_of(item, struct simple_child, item);
}

static ssize_t simple_child_storeme_show(struct config_item *item, char *page)
{
	return sprintf(page, "%d\n", to_simple_child(item)->storeme);
}

static ssize_t simple_child_storeme_store(struct config_item *item,
		const char *page, size_t count)
{
	struct simple_child *simple_child = to_simple_child(item);
	int ret;

	ret = kstrtoint(page, 10, &simple_child->storeme);
	if (ret)
		return ret;

	return count;
}

CONFIGFS_ATTR(simple_child_, storeme);

static struct configfs_attribute *simple_child_attrs[] = {
	&simple_child_attr_storeme,
	NULL,
};

static void simple_child_release(struct config_item *item)
{
	kfree(to_simple_child(item));
}

static struct configfs_item_operations simple_child_item_ops = {
	.release	= simple_child_release,
};

static const struct config_item_type simple_child_type = {
	.ct_item_ops	= &simple_child_item_ops,
	.ct_attrs	= simple_child_attrs,
	.ct_owner	= THIS_MODULE,
};

struct simple_children {
	struct config_group group;
};

static inline struct simple_children *to_simple_children(struct config_item *item)
{
	return container_of(to_config_group(item),
			    struct simple_children, group);
}

static struct config_item *simple_children_make_item(struct config_group *group,
		const char *name)
{
	struct simple_child *simple_child;

	simple_child = kzalloc(sizeof(struct simple_child), GFP_KERNEL);
	if (!simple_child)
		return ERR_PTR(-ENOMEM);

	config_item_init_type_name(&simple_child->item, name,
				   &simple_child_type);

	return &simple_child->item;
}

static ssize_t simple_children_description_show(struct config_item *item,
		char *page)
{
	return sprintf(page,
"[02-simple-children]\n"
"\n"
"This subsystem allows the creation of child config_items.  These\n"
"items have only one attribute that is readable and writeable.\n");
}

CONFIGFS_ATTR_RO(simple_children_, description);

static struct configfs_attribute *simple_children_attrs[] = {
	&simple_children_attr_description,
	NULL,
};

static void simple_children_release(struct config_item *item)
{
	kfree(to_simple_children(item));
}

static struct configfs_item_operations simple_children_item_ops = {
	.release	= simple_children_release,
};

/*
 * Note that, since no extra work is required on ->drop_item(),
 * no ->drop_item() is provided.
 */
// 如果组提供了 ct_group_ops->make_item() 方法，则通过 mkdir(2) 在组的目录中调用该方法创建子项。
static struct configfs_group_operations simple_children_group_ops = {
	.make_item	= simple_children_make_item,
};

static const struct config_item_type simple_children_type = {
	.ct_item_ops	= &simple_children_item_ops,
	.ct_group_ops	= &simple_children_group_ops,
	.ct_attrs	= simple_children_attrs,
	.ct_owner	= THIS_MODULE,
};

static struct configfs_subsystem simple_children_subsys = {
	.su_group = {
		.cg_item = {
			.ci_namebuf = "02-simple-children",
			.ci_type = &simple_children_type,
		},
	},
};
```

> [config_group 结构包含一个 config_item。适当配置该项意味着 group 可以作为自己的项目发挥作用。不过，它可以做更多：它可以创建子项目或子组。这是通过在组的 config_item_type 上指定的组操作来实现的。](#struct-config_group)
>
> 这个例子，可以通过 mkdir 创建子项，子项有一个可读写的 storeme 属性。
>
> ```bash
> root@milkv-duo]/tmp/usb/02-simple-children# cat description
> [02-simple-children]
>
> This subsystem allows the creation of child config_items.  These
> items have only one attribute that is readable and writeable.
> [root@milkv-duo]/tmp/usb/02-simple-children# mkdir test01
> [root@milkv-duo]/tmp/usb/02-simple-children# tree .
> .
> ├── description
> └── test01
>     └── storeme
>
> 1 directory, 2 files
> [root@milkv-duo]/tmp/usb/02-simple-children# echo xxx > test01/storeme
> sh: write error: Invalid argument
> [root@milkv-duo]/tmp/usb/02-simple-children# echo 30 > test01/storeme
> [root@milkv-duo]/tmp/usb/02-simple-children# cat test01/storeme
> 30
> [root@milkv-duo]/tmp/usb/02-simple-children# mkdir test02
> [root@milkv-duo]/tmp/usb/02-simple-children# mkdir test03
> [root@milkv-duo]/tmp/usb/02-simple-children# tree .
> .
> ├── description
> ├── test01
> │   └── storeme
> ├── test02
> │   └── storeme
> └── test03
>     └── storeme
>
> 3 directories, 4 files
> [root@milkv-duo]/tmp/usb/02-simple-children# cat test02/storeme
> 0
> [root@milkv-duo]/tmp/usb/02-simple-children# rmdir test03
> [root@milkv-duo]/tmp/usb/02-simple-children# ls
> description  test01  test02
> ```

### 03-group-children

```c
/*
 * 03-group-children
 *
 * This example reuses the simple_children group from above.  However,
 * the simple_children group is not the subsystem itself, it is a
 * child of the subsystem.  Creation of a group in the subsystem creates
 * a new simple_children group.  That group can then have simple_child
 * children of its own.
 */

static struct config_group *group_children_make_group(
		struct config_group *group, const char *name)
{
	struct simple_children *simple_children;

	simple_children = kzalloc(sizeof(struct simple_children),
				  GFP_KERNEL);
	if (!simple_children)
		return ERR_PTR(-ENOMEM);

	config_group_init_type_name(&simple_children->group, name,
				    &simple_children_type);

	return &simple_children->group;
}

static ssize_t group_children_description_show(struct config_item *item,
		char *page)
{
	return sprintf(page,
"[03-group-children]\n"
"\n"
"This subsystem allows the creation of child config_groups.  These\n"
"groups are like the subsystem simple-children.\n");
}

CONFIGFS_ATTR_RO(group_children_, description);

static struct configfs_attribute *group_children_attrs[] = {
	&group_children_attr_description,
	NULL,
};

/*
 * Note that, since no extra work is required on ->drop_item(),
 * no ->drop_item() is provided.
 */
static struct configfs_group_operations group_children_group_ops = {
	.make_group	= group_children_make_group,
};

static const struct config_item_type group_children_type = {
	.ct_group_ops	= &group_children_group_ops,
	.ct_attrs	= group_children_attrs,
	.ct_owner	= THIS_MODULE,
};

static struct configfs_subsystem group_children_subsys = {
	.su_group = {
		.cg_item = {
			.ci_namebuf = "03-group-children",
			.ci_type = &group_children_type,
		},
	},
};
```

> [如果子系统希望子项本身也是一个组，则子系统会提供 `ct_group_ops->make_group()` 方法。](#sturct-config_group)
>
> 这个例子中，可以通过 mkdir 去创建 group。
>
> > ❓❓一个问题：如果 `ct_group_ops` 中同时指定了 `make_group` 和 `make_item` ，那么调用 `mkdir` 时是执行哪一个呢？可以从源码中找答案：[https://elixir.bootlin.com/linux/v5.10.186/source/fs/configfs](https://elixir.bootlin.com/linux/v5.10.186/source/fs/configfs)  `configfs` 的实现看起来有点简单，代码量并不大。
> >
> > [https://elixir.bootlin.com/linux/v5.10.186/source/fs/configfs/dir.c#L1353](https://elixir.bootlin.com/linux/v5.10.186/source/fs/configfs/dir.c#L1353) 在 configfs_mkdir() 中，`make_group` 优先调用。
>
> ```bash
> [root@milkv-duo]/tmp/usb/03-group-children# cat description
> [03-group-children]
> 
> This subsystem allows the creation of child config_groups.  These
> groups are like the subsystem simple-children.
> [root@milkv-duo]/tmp/usb/03-group-children# mkdir grp1
> [root@milkv-duo]/tmp/usb/03-group-children# tree .
> .
> ├── description
> └── grp1
>     └── description
> 
> 1 directory, 2 files
> [root@milkv-duo]/tmp/usb/03-group-children# cat grp1/description
> [02-simple-children]
> 
> This subsystem allows the creation of child config_items.  These
> items have only one attribute that is readable and writeable.
> [root@milkv-duo]/tmp/usb/03-group-children# mkdir grp1/item1
> [root@milkv-duo]/tmp/usb/03-group-children# mkdir grp1/item2
> [root@milkv-duo]/tmp/usb/03-group-children# tree .
> .
> ├── description
> └── grp1
>     ├── description
>     ├── item1
>     │   └── storeme
>     └── item2
>         └── storeme
> 
> 3 directories, 4 files
> [root@milkv-duo]/tmp/usb/03-group-children# rmdir grp1
> rmdir: failed to remove 'grp1': Directory not empty
> [root@milkv-duo]/tmp/usb/03-group-children# cd grp1/
> [root@milkv-duo]/tmp/usb/03-group-children/grp1# rmdir item1
> [root@milkv-duo]/tmp/usb/03-group-children/grp1# rmdir item2
> [root@milkv-duo]/tmp/usb/03-group-children/grp1# cd ..
> [root@milkv-duo]/tmp/usb/03-group-children# rmdir grp1
> [root@milkv-duo]/tmp/usb/03-group-children# ls
> description
> [root@milkv-duo]/tmp/usb/03-group-children#
> ```

## 层次结构导航和子系统互斥锁

`configfs` 提供了一个额外的好处。由于 `config_groups` 和 `config_items` 出现在文件系统中，因此它们被排列成层次结构。子系统**永远**不应触及文件系统部分，但子系统可能对该层次结构感兴趣。为此，层次结构通过 `config_group->cg_children` 和 `config_item->ci_parent` 结构成员进行镜像。

```plaintext
[Hierarchy Navigation and the Subsystem Mutex]

There is an extra bonus that configfs provides. The config_groups and
config_items are arranged in a hierarchy due to the fact that they
appear in a filesystem. A subsystem is NEVER to touch the filesystem
parts, but the subsystem might be interested in this hierarchy. For
this reason, the hierarchy is mirrored via the config_group->cg_children
and config_item->ci_parent structure members.
```

子系统可以在保护子系统互斥锁的情况下导航 `cg_children` 列表和 `ci_parent` 指针，以查看由子系统创建的树。这可能会与 `configfs` 对层次结构的管理产生竞争，因此 `configfs` 使用子系统互斥锁来保护修改。每当子系统想要导航层次结构时，必须在子系统互斥锁的保护下进行。

```plaintext
A subsystem can navigate the cg_children list and the ci_parent pointer
to see the tree created by the subsystem. This can race with configfs'
management of the hierarchy, so configfs uses the subsystem mutex to
protect modifications. Whenever a subsystem wants to navigate the
hierarchy, it must do so under the protection of the subsystem
mutex.
```

只要新分配的项尚未链接到层次结构中，子系统将无法获取互斥锁。同样，只要正在删除的项尚未从层次结构中解除链接，它将无法获取互斥锁。这意味着只要该项在 `configfs` 中，其 `ci_parent` 指针就永远不会为 NULL，并且该项在其父项的 `cg_children` 列表中存在的时间也是

相同的。这使子系统在持有互斥锁时可以信任 `ci_parent` 和 `cg_children`。

```plaintext
A subsystem will be prevented from acquiring the mutex while a newly
allocated item has not been linked into this hierarchy. Similarly, it
will not be able to acquire the mutex while a dropping item has not
yet been unlinked. This means that an item's ci_parent pointer will
never be NULL while the item is in configfs, and that an item will only
be in its parent's cg_children list for the same duration. This allows
a subsystem to trust ci_parent and cg_children while they hold the
mutex.
```

---

## 通过 `symlink(2)` 实现项的聚合

`configfs` 通过 `group->item` 的父/子关系提供了一个简单的分组。然而，在更大的环境中，通常需要超出父/子连接的聚合。这可以通过 `symlink(2)` 实现。

```plaintext
[Item Aggregation Via symlink(2)]

configfs provides a simple group via the group->item parent/child
relationship. Often, however, a larger environment requires aggregation
outside of the parent/child connection. This is implemented via
symlink(2).
```

`config_item` 可以提供 `ct_item_ops->allow_link()` 和 `ct_item_ops->drop_link()` 方法。如果存在 `->allow_link()` 方法，可以将 `symlink(2)` 调用配置项作为链接的源。链接只允许在 `configfs` 的 `config_items` 之间进行。任何在 `configfs` 文件系统之外的 `symlink(2)` 尝试都将被拒绝。

```plaintext
A config_item may provide the ct_item_ops->allow_link() and
ct_item_ops->drop_link() methods. If the ->allow_link() method exists,
symlink(2) may be called with the config_item as the source of the link.
These links are only allowed between configfs config_items. Any
symlink(2) attempt outside the configfs filesystem will be denied.
```

当调用 `symlink(2)` 时，源 `config_item` 的 `->allow_link()` 方法将与自身和目标项一起调用。如果源项允许链接到目标项，它将返回 0。源项可能希望拒绝链接，如果它只希望链接到某一类对象（例如，只能链接到自己的子系统中的对象）。

```plaintext
When symlink(2) is called, the source config_item's ->allow_link()
method is called with itself and a target item. If the source item
allows linking to target item, it returns 0. A source item may wish to
reject a link if it only wants links to a certain type of object (say,
in its own subsystem).
```

当在符号链接上调用 `unlink(2)` 时，源项将通过 `->drop_link()` 方法收到通知。与 `->drop_item()` 方法一样，这是一个 `void` 函数，不能返回失败。子系统负责对更改作出响应。

```plaintext
When unlink(2) is called on the symbolic link, the source item is
notified via the ->drop_link() method. Like the ->drop_item() method,
this is a void function and cannot return failure. The subsystem is
responsible for responding to the change.
```

只要链接到其他项，或其他项链接到它，就无法移除 `config_item`。`configfs` 不允许出现悬空的符号链接。

```plaintext
A config_item cannot be removed while it links to any other item, nor
can it be removed while an item links to it. Dangling symlinks are not
allowed in configfs.
```

---

## 自动创建的子组

一个新的 `config_group` 可能希望拥有两种类型的子 `config_items`。虽然这可以通过 `->make_item()` 方法中的“魔法名称”实现，但通过一种用户空间可以看到这种分歧的方法更加明确。

```plaintext
[Automatically Created Subgroups]

A new config_group may want to have two types of child config_items.
While this could be codified by magic names in ->make_item(), it is much
more explicit to have a method whereby userspace sees this divergence.
```

与其拥有一个行为不同的项，不如让 `configfs` 提供一种方法，即在创建父组时，自动在父组中创建一个或多个子组。因此，`mkdir("parent")` 结果是 `"parent"`，`"parent/subgroup1"`，依次到 `"parent/subgroupN"`。类型 1 的项现在可以在 `"parent/subgroup1"` 中创建，类型 N 的项可以在 `"parent/subgroupN"` 中创建。

```plaintext
Rather than have a group where some items behave differently than
others, configfs provides a method whereby one or many subgroups are
automatically created inside the parent at its creation. Thus,
mkdir("parent") results in "parent", "parent/subgroup1", up through
"parent/subgroupN". Items of type 1 can now be created in
"parent/subgroup1", and items of type N can be created in
"parent/subgroupN".
```

这些自动子组（或默认组）不会排除父组的其他子项。如果存在 `ct_group_ops->make_group()` 方法，可以直接在父组中创建其他子组。

```plaintext
These automatic subgroups, or default groups, do not preclude other
children of the parent group. If ct_group_ops->make_group() exists,
other child groups can be created on the parent group directly.
```

`configfs` 子系统通过使用 `configfs_add_default_group()` 函数将它们添加到父 `config_group` 结构中来指定默认组。每个添加的组将在父组创建时同时填充到 `configfs` 树中。类似地，它们将在父组移除时一起被删除。没有额外的通知。当 `->drop_item()` 方法通知子系统父组正在消失时，这也意味着与该父组关联的每个默认组子项。

```plaintext
A configfs subsystem specifies default groups by adding them using the
configfs_add_default_group() function to the parent config_group
structure. Each added group is populated in the configfs tree at the same
time as the parent group. Similarly, they are removed at the same time
as the parent. No extra notification is provided. When a ->drop_item()
method call notifies the subsystem the parent group is going away, it
also means every default group child associated with that parent group.
```

因此，默认组不能直接通过 `rmdir(2)` 删除。在父组上执行 `rmdir(2)` 时，它们也不会被视为子项。

```plaintext
As a consequence of this, default groups cannot be removed directly via
rmdir(2). They also are not considered when rmdir(2) on the parent
group is checking for children.
```

---

## 依赖子系统

有时其他驱动程序依赖于特定的 `configfs` 项。例如，`ocfs2` 挂载依赖于心跳区域项。如果该区域项通过 `rmdir(2)` 被移除，`ocfs2` 挂载必须 `BUG` 或切换到只读模式。这并不理想。

```plaintext
[Dependent Subsystems]

Sometimes other drivers depend on particular configfs items. For
example, ocfs2 mounts depend on a heartbeat region item. If that
region item is removed with rmdir(2), the ocfs2 mount must BUG or go
readonly. Not happy.
```

`configfs` 提供了两个额外的 API 调用：`configfs_depend_item()` 和 `configfs_undepend_item()`。客户端驱动程序可以调用 `configfs_depend_item()` 来告知 `configfs` 它依赖某个现有项。之后，`configfs` 将在 `rmdir(2)` 调用时返回 `-EBUSY` 错误。当该项不再被依赖时，客户端驱动程序调用 `configfs_undepend_item()`。

```plaintext
configfs provides two additional API calls: configfs_depend_item() and
configfs_undepend_item(). A client driver can call
configfs_depend_item() on an existing item to tell configfs that it is
depended on. configfs will then return -EBUSY from rmdir(2) for that
item. When the item is no longer depended on, the client driver calls
configfs_undepend_item() on it.
```

这些 API 不能在任何 `configfs` 回调中调用，因为它们会产生冲突。它们可能会阻塞和分配内存。客户端驱动程序可能不应自行调用它们，而应该提供供外部子系统调用的 API。

```plaintext
These API cannot be called underneath any configfs callbacks, as
they will conflict. They can block and allocate. A client driver
probably shouldn't calling them of its own gumption. Rather it should
be providing an API that external subsystems call.
```

这个过程如何工作？想象一下 `ocfs2` 的挂载过程。当它挂载时，它请求一个心跳区域项。这是通过调用心跳代码来实现的。在心跳代码中，查找区域项。在这里，心跳代码调用 `configfs_depend_item()`。如果调用成功，心跳知道该区域是安全的，可以分配给 `ocfs2`。如果失败，则表示该区域正在被拆除，心跳代码可以优雅地返回一个错误。

```plaintext
How does this work? Imagine the ocfs2 mount process. When it mounts,
it asks for a heartbeat region item. This is done via a call into the
heartbeat code. Inside the heartbeat code, the region item is looked
up. Here, the heartbeat code calls configfs_depend_item(). If it
succeeds, then heartbeat knows the region is safe to give to ocfs2.
If it fails, it was being torn down anyway, and

 heartbeat can gracefully
pass up an error.
```

---

## 可提交项

注意：可提交项当前未实现。

```plaintext
[Committable Items]

NOTE: Committable items are currently unimplemented.
```

某些 `config_items` 无法拥有有效的初始状态。也就是说，无法为项目的属性指定默认值，以使项目能够执行其工作。用户空间必须配置一个或多个属性，之后子系统才能启动该项目所代表的实体。

```plaintext
Some config_items cannot have a valid initial state. That is, no
default values can be specified for the item's attributes such that the
item can do its work. Userspace must configure one or more attributes,
after which the subsystem can start whatever entity this item
represents.
```

考虑上面的 FakeNBD 设备。没有目标地址和目标设备，子系统无法知道要导入哪个块设备。简单示例假设子系统仅在所有属性都配置完成后连接。这确实可以工作，但现在每个属性存储都必须检查属性是否已初始化。一旦满足条件，每个属性存储都必须触发连接。

```plaintext
Consider the FakeNBD device from above. Without a target address *and*
a target device, the subsystem has no idea what block device to import.
The simple example assumes that the subsystem merely waits until all the
appropriate attributes are configured, and then connects. This will,
indeed, work, but now every attribute store must check if the attributes
are initialized. Every attribute store must fire off the connection if
that condition is met.
```

更好的方式是使用一个明确的操作通知子系统 `config_item` 已准备好。更重要的是，明确的操作可以让子系统提供反馈，说明属性是否以合理的方式进行了初始化。`configfs` 通过可提交项提供了此功能。

```plaintext
Far better would be an explicit action notifying the subsystem that the
config_item is ready to go. More importantly, an explicit action allows
the subsystem to provide feedback as to whether the attributes are
initialized in a way that makes sense. configfs provides this as
committable items.
```

`configfs` 仍然仅使用正常的文件系统操作。通过 `rename(2)` 提交一个项。该项从可以修改的目录移动到不能修改的目录。

```plaintext
configfs still uses only normal filesystem operations. An item is
committed via rename(2). The item is moved from a directory where it
can be modified to a directory where it cannot.
```

任何提供 `ct_group_ops->commit_item()` 方法的组都拥有可提交的项。当该组出现在 `configfs` 中时，不能直接在组中使用 `mkdir(2)`。相反，该组将有两个子目录：“live”和“pending”。`live` 目录不支持 `mkdir(2)` 或 `rmdir(2)`。它只允许 `rename(2)`。`pending` 目录允许 `mkdir(2)` 和 `rmdir(2)`。一个项在 `pending` 目录中创建。其属性可以随意修改。用户空间通过将项重命名到 `live` 目录来提交该项。这时，子系统收到 `->commit_item()` 回调。如果所有必需属性都已填写，方法返回 0，项被移动到 `live` 目录。

```plaintext
Any group that provides the ct_group_ops->commit_item() method has
committable items. When this group appears in configfs, mkdir(2) will
not work directly in the group. Instead, the group will have two
subdirectories: "live" and "pending". The "live" directory does not
support mkdir(2) or rmdir(2) either. It only allows rename(2). The
"pending" directory does allow mkdir(2) and rmdir(2). An item is
created in the "pending" directory. Its attributes can be modified at
will. Userspace commits the item by renaming it into the "live"
directory. At this point, the subsystem receives the ->commit_item()
callback. If all required attributes are filled to satisfaction, the
method returns zero and the item is moved to the "live" directory.
```

由于在 `live` 目录中无法使用 `rmdir(2)`，必须关闭或“取消提交”该项。同样，这可以通过 `rename(2)` 完成，这次是从 `live` 目录重命名回 `pending` 目录。子系统通过 `ct_group_ops->uncommit_object()` 方法收到通知。

```plaintext
As rmdir(2) does not work in the "live" directory, an item must be
shutdown, or "uncommitted". Again, this is done via rename(2), this
time from the "live" directory back to the "pending" one. The subsystem
is notified by the ct_group_ops->uncommit_object() method.
```

---
