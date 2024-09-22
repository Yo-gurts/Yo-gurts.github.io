---
title: Linux USB 驱动框架
top_img: transparent
date: 2024-09-01 10:54:14
updated: 2024-09-01 10:54:14
tags:
  - Linux
  - USB
categories: Linux
keywords:
description:
---

depends on:

- [ ] bus 通用知识
- [ ] uevent 机制
- [ ] 字符设备

## USB 驱动框架

很典型的Linux 驱动框架，分
- 核心层：完成通用功能。
- 设备驱动层 PDD（实现特定功能，比如 UVC， UAC， HID 这些类型的设备驱动）
- 控制器驱动 HCD（配置寄存器去操作硬件USB IP，完成数据收发）类似与网络中的数据链路层，只负责数据收发。

![image-20240901105545261](../images/Linux-USB%E9%A9%B1%E5%8A%A8%E6%A1%86%E6%9E%B6/image-20240901105545261.png)

> image from paper <USBIP a transparent device sharing technolog over ip network>
>
> 这段 USB Driver Architecture 讲得很好。
>
> USB (Universal Serial Bus) 17) is one of the sophisticated peripheral interfaces based on the recent hardware progress. In the USB 2.0 speciﬁcation announced in April 2000, a host computer controls various USB devices with 3 transfer speeds (1.5 Mbps, 12.0 Mbps, and 480 Mbps) and 4 transfer types (Control, Bulk, Interrupt, and Isochronous). These transfers are serialized and controlled by a dedicated hardware function which is named USB Host Controller. Figure 1 shows the USB device driver model in most operating systems. A **USB Host Controller Driver** (USB HCD) exists in the lowest layer of the device driver stack and abstracts the I/O interface of a host controller into a common API for USB device drivers. **A USB PerDevice Drivers (USB PDD)** is respon-sible for the control of each USB device. The key design of the USB driver stack is that a USB Host Controller and its USB HCD provide USB PDDs with abstracted data input/output for I/O buﬀers. Although a USB device and its USB PDD use the buﬀer to transfer some control data and corresponding reply data, **the USB HCD do not distinguish between control and reply**.
>
> In USB device drivers, a **USB Request Block (URB)** presents USB I/O in a controller-independent form, which includes information about I/O;
> - **I/O buﬀer**
> - I/O direction (Input/Output)
> - I/O speed (1.5 Mbps/12 Mbps/480 Mbps)
> - I/O type (Control/Bulk/Interrupt/Isochronous)
> - I/O destination address
> - **completion handler**
>
> An application or a device driver controls a USB device as follows:
> 1. **A USB PDD converts I/O requests from another driver into URBs and submits these URBs to a USB HCD.**
> 2. The USB HCD transfers data as de-scribed by the URB.
> 3. After I/O of the URB is completed, the completion handler of the URB is called in an interrupt context.
> 4. The USB PDD notiﬁes the upper driver of the requested I/O completion.
>
> A USB device and its USB PDD use 4 diﬀer- ent transfer types depending on characteristics of the device. **Control and Bulk transfer types are asynchronously scheduled into the rest of the bandwidth after the periodical transfers**. Control transfer is the most fundamental one for enumeration and initialization of devices. In 480 Mbps mode, **20% of the bandwidth is reserved for Control transfer**. Bulk transfer is used for the requests without any temporal re-striction, such as storage device I/O, which is the fastest transfer when the bus is available. **Isochronous and Interrupt transfer types are pe-riodically scheduled**. Isochronous transfer can move control data at a constant bit rate, which is useful to read image data from a USB cam-era or to write sound data to a USB speaker. **Interrupt transfer conﬁrms the maximum delay of the requested I/O**. This is used for USB mice and keyboards, which move a small amount of data sporadically.
>
> note: UVC 出流的时候发现最大3个设备总带宽44MB 没问题，但48MB 会提示资源不足。总共 480Mbps -> 60MB -> *80% -> 48MB. 理论上 48M刚好够，不过其他HUB 之类的设备可能也会占一点就导致不够了。

### 代码阅读流程

1. **确定核心代码**（通过 .config + MAKEFILE + KCONFIG）

    ```bash
    cat build/sg2002_wevb_riscv64_sd/.config | grep USB | grep "="
    CONFIG_USB_OHCI_LITTLE_ENDIAN=y # 大小端，一般都是小端
    CONFIG_USB_SUPPORT=y # usb/phy 也是 USB 驱动的总开关
    CONFIG_USB_ARCH_HAS_HCD=y # HOST CONTROLLER，肯定得默认Y
    CONFIG_USB_COMMON=y # usb/common/usb-common.o

    CONFIG_USB_DEFAULT_PERSIST=y # USB电源会话持久性
    CONFIG_USB_AUTOSUSPEND_DELAY=2 # 默认自动休眠时间（2S）
    CONFIG_USB_ROLE_SWITCH=y # 支持角色 host/device 切换 += usb/roles/roles.o

    ########## HOST ##########
    CONFIG_USB=y # += usb/core usb/storage/ usb/misc/  Support for Host-side USB
    # 作为 HOST 时用。要作为device，应该看 USB Gadget。EHCI HCD 代表USB2.0。USB1.1是 UHCI，OHCI
    # Say Y here if your computer has a host-side USB port and you want to use USB devices.  You then need to say Y to at least one of the Host Controller Driver (HCD) options below.  Choose a USB 1.1 controller, such as "UHCI HCD support" or "OHCI HCD support", and "EHCI HCD (USB 2.0) support" except for older systems that do not have USB 2.0 support.  It doesn't normally hurt to select them all if you are not certain.
    # If your system has a device-side USB port, used in the peripheral side of the USB protocol, see the "USB Gadget" framework instead.
    #### controller driver ####
    CONFIG_USB_DWC2=y           # usb/dwc2/dwc2.o USB HCD 驱动
    CONFIG_USB_DWC2_DUAL_ROLE=y # 支持两种角色的工作模式（host/device）

    ########## DEVICE ##########
    CONFIG_USB_GADGET=y         # += 作为USBdevice用。usb/gadget udc/ function/ legacy/ udc/udc-core.o
    CONFIG_USB_GADGET_VBUS_DRAW=2 # Maximum VBUS Power usage (2-500 mA)
    CONFIG_USB_GADGET_STORAGE_NUM_BUFFERS=2 # Number of storage pipeline buffers
    CONFIG_USB_LIBCOMPOSITE=y   # += usb/gadget/libcomposite.o  复合设备驱动
    CONFIG_USB_F_ACM=y          # += usb/gadget/function/usb_f_acm.o
    CONFIG_USB_U_SERIAL=y       # += usb/gadget/function/u_serial.o
    CONFIG_USB_U_ETHER=y        # += usb/gadget/function/u_ether.o
    CONFIG_USB_U_AUDIO=y        # += usb/gadget/function/u_audio.o
    CONFIG_USB_F_SERIAL=y       # += usb/gadget/function/usb_f_serial.o
    CONFIG_USB_F_ECM=y          # += usb/gadget/function/f_ecm.o
    CONFIG_USB_F_EEM=y          # += usb/gadget/function/f_eem.o
    CONFIG_USB_F_RNDIS=y        # += usb/gadget/function/usb_f_rndis.o
    CONFIG_USB_F_MASS_STORAGE=y # += usb/gadget/function/usb_f_mass_storage.o
    CONFIG_USB_F_FS=y           # += usb/gadget/function/usb_f_fs.o
    CONFIG_USB_F_UAC1=y         # += usb/gadget/function/usb_f_uac1.o
    CONFIG_USB_F_UVC=y          # += usb/gadget/function/usb_f_uvc.o
    CONFIG_USB_CONFIGFS=y       # 支持通过configfs对功能进行配置 USB Gadget functions configurable through configfs
    CONFIG_USB_CONFIGFS_SERIAL=y # Generic serial bulk in/out
    CONFIG_USB_CONFIGFS_ACM=y   # Abstract Control Model (CDC ACM)
    CONFIG_USB_CONFIGFS_ECM=y   # Ethernet Control Model (CDC ECM)
    CONFIG_USB_CONFIGFS_RNDIS=y # RNDIS
    CONFIG_USB_CONFIGFS_EEM=y   # Ethernet Emulation Model (EEM)
    CONFIG_USB_CONFIGFS_MASS_STORAGE=y # Mass Storage
    CONFIG_USB_CONFIGFS_F_FS=y  # Function filesystem (FunctionFS)
    # The Function Filesystem (FunctionFS) lets one create USB composite functions in user space in the same way GadgetFS lets one create USB gadgets in user space.
    CONFIG_USB_CONFIGFS_F_UAC1=y # Audio Class 1.0
    CONFIG_USB_CONFIGFS_F_UVC=y # USB Webcam function
    CONFIG_USB_HID=y            # hid/usbhid/usbhid.o 并非所有的PDD驱动实现都放在 usb 目录下
    ```

    再查找配置对应的代码。vscode 中搜索，仅搜索 Makefile, Kconfig 。**注意搜索技巧**。
    ![image-20240901121143789](../images/Linux-USB%E9%A9%B1%E5%8A%A8%E6%A1%86%E6%9E%B6/image-20240901121143789.png)



### core 目录

> https://www.cnblogs.com/wen123456/p/14281890.html

首先，可以通过 Makefile **确定我们需要关注那些 .c 文件**。比如，我们启用了 CONFIG_USB  CONFIG_OF。

```makefile
usbcore-y := usb.o hub.o hcd.o urb.o message.o driver.o
usbcore-y += config.o file.o buffer.o sysfs.o endpoint.o
usbcore-y += devio.o notify.o generic.o quirks.o devices.o
usbcore-y += phy.o port.o
usbcore-$(CONFIG_OF)		+= of.o
obj-$(CONFIG_USB)		+= usbcore.o
```

确定当前目录下会编译多少个模块 `ko`：

1. 通过在当前路径搜索 `module_init/subsys_initcall` 等模块初始化函数。

2. 通过 Kconfig 文件中 `tristate`  的个数。只有为 tristate 才能编译为 ko 形式加载。int 的一般都只是以宏判断的形式嵌入在代码中，不过这个非绝对。

    ```kconfig
    config USB_LEDS_TRIGGER_USBPORT
    	tristate "USB port LED trigger"
    	depends on USB && LEDS_TRIGGERS
    	help
    	  This driver allows LEDs to be controlled by USB events. Enabling this
    	  trigger allows specifying list of USB ports that should turn on LED
    	  when some USB device gets connected.

    config USB_AUTOSUSPEND_DELAY
    	int "Default autosuspend delay"
    	depends on USB
    	default 2
    	help
    	  The default autosuspend delay in seconds.  Can be overridden
    	  with the usbcore.autosuspend command line or module parameter.

    	  The default value Linux has always had is 2 seconds.  Change
    	  this value if you want a different delay and cannot modify
    	  the command line or module parameter.
    ```

总之目前确认了 core 目录下可以编译出两个 ko 文件。`usbcore.ko` 以及 `ledtrig-usbport.ko`.

确定模块之后，就可以从 module 的init 函数去追代码。过程中自然会看到他调用其他 c 文件的方法。

```c
/*
 * Init
 */
static int __init usb_init(void)
{
	int retval;
	if (usb_disabled()) {
		pr_info("%s: USB support disabled\n", usbcore_name);
		return 0;
	}
	usb_init_pool_max(); // USB 驱动中会分配dma内存池，确定内存中内存块的最小的大小，与 ARCH_KMALLOC_MINALIGN 有关。

	usb_debugfs_init(); // debugfs 中增加文件，方便调试

	usb_acpi_register(); // 没配置，暂时不管
	retval = bus_register(&usb_bus_type); // USB 总线注册
	if (retval)
		goto bus_register_failed;
	retval = bus_register_notifier(&usb_bus_type, &usb_bus_nb); // 注册内核通知链，用于设备和接口注册的通知
	if (retval)
		goto bus_notifier_failed;
	retval = usb_major_init(); // 申请 major number。
	if (retval)
		goto major_init_failed;
	retval = usb_register(&usbfs_driver); // 注册接口驱动 usbfs
	if (retval)
		goto driver_register_failed;
	retval = usb_devio_init(); // 初始化字符设备
	if (retval)
		goto usb_devio_init_failed;
	retval = usb_hub_init(); // 注册接口驱动 hub
	if (retval)
		goto hub_init_failed;
	retval = usb_register_device_driver(&usb_generic_driver, THIS_MODULE);
	if (!retval)
		goto out;

	// 下面都是些异常情况的清理过程。
	usb_hub_cleanup();
hub_init_failed:
	usb_devio_cleanup();
usb_devio_init_failed:
	usb_deregister(&usbfs_driver);
driver_register_failed:
	usb_major_cleanup();
major_init_failed:
	bus_unregister_notifier(&usb_bus_type, &usb_bus_nb);
bus_notifier_failed:
	bus_unregister(&usb_bus_type);
bus_register_failed:
	usb_acpi_unregister();
	usb_debugfs_cleanup();
out:
	return retval;
}
```

#### usb_debugfs_init

```c
static void usb_debugfs_init(void)
{
	// 在debugfs 中创建一个 devices 文件。提供了两个 fops：read & llseek
	usb_devices_root = debugfs_create_file("devices", 0444, usb_debug_root,
					       NULL, &usbfs_devices_fops);
}

// core/devices.c
// read 方法会遍历总线，然后dump每一个设备的信息。
static ssize_t usb_device_read(struct file *file, char __user *buf,
			       size_t nbytes, loff_t *ppos)
{
	struct usb_bus *bus;
	ssize_t ret, total_written = 0;
	loff_t skip_bytes = *ppos;
	int id;

	if (*ppos < 0)
		return -EINVAL;
	if (nbytes <= 0)
		return 0;

	mutex_lock(&usb_bus_idr_lock);
	/* print devices for all busses */
	idr_for_each_entry(&usb_bus_idr, bus, id) {
		/* recurse through all children of the root hub */
		if (!bus_to_hcd(bus)->rh_registered)
			continue;
		usb_lock_device(bus->root_hub);
		// 这里 dump 方法将设备信息写入到 buf 中，传递给 user-space
		ret = usb_device_dump(&buf, &nbytes, &skip_bytes, ppos,
				      bus->root_hub, bus, 0, 0, 0);
		usb_unlock_device(bus->root_hub);
		if (ret < 0) {
			mutex_unlock(&usb_bus_idr_lock);
			return ret;
		}
		total_written += ret;
	}
	mutex_unlock(&usb_bus_idr_lock);
	return total_written;
}

const struct file_operations usbfs_devices_fops = {
	.llseek =	no_seek_end_llseek,
	.read =		usb_device_read,
};

// common/common.c
// common 已经是另一个 module 里面的代码了，所以usbcore.ko加载之前必须先加载这个 common.ko
struct dentry *usb_debug_root;
EXPORT_SYMBOL_GPL(usb_debug_root);
static int __init usb_common_init(void)
{
	// debugfs 中创建一个 usb 目录
	usb_debug_root = debugfs_create_dir("usb", NULL);
	ledtrig_usb_init();
	return 0;
}
```

#### bus_register

总线嘛：**一边设备，一边驱动，完成设备和驱动的匹配过程**。

无论是有设备注册、还是驱动加载，总线都调用 `match` 方法去匹配。

`uevent` 方法，会在设备注册时调用，可以给应用层上报信息，通知有设备插入之类的。


```c
	retval = bus_register(&usb_bus_type);

struct bus_type usb_bus_type = {
	.name =		"usb",
	.match =	usb_device_match,
	.uevent =	usb_uevent,
	.need_parent_lock =	true,
};

// 设备与驱动的匹配实现，后面再细看吧。
static int usb_device_match(struct device *dev, struct device_driver *drv)
{
	/* devices and interfaces are handled separately */
	if (is_usb_device(dev)) {
		struct usb_device *udev;
		struct usb_device_driver *udrv;

		/* interface drivers never match devices */
		if (!is_usb_device_driver(drv))
			return 0;

		udev = to_usb_device(dev);
		udrv = to_usb_device_driver(drv);

		/* If the device driver under consideration does not have a
		 * id_table or a match function, then let the driver's probe
		 * function decide.
		 */
		if (!udrv->id_table && !udrv->match)
			return 1;

		return usb_driver_applicable(udev, udrv);

	} else if (is_usb_interface(dev)) {
		struct usb_interface *intf;
		struct usb_driver *usb_drv;
		const struct usb_device_id *id;

		/* device drivers never match interfaces */
		if (is_usb_device_driver(drv))
			return 0;

		intf = to_usb_interface(dev);
		usb_drv = to_usb_driver(drv);

		id = usb_match_id(intf, usb_drv->id_table);
		if (id)
			return 1;

		id = usb_match_dynamic_id(intf, usb_drv);
		if (id)
			return 1;
	}

	return 0;
}

// 通过 uevent 机制上报设备信息。
static int usb_uevent(struct device *dev, struct kobj_uevent_env *env)
{
	struct usb_device *usb_dev;

	if (is_usb_device(dev)) {
		usb_dev = to_usb_device(dev);
	} else if (is_usb_interface(dev)) {
		struct usb_interface *intf = to_usb_interface(dev);

		usb_dev = interface_to_usbdev(intf);
	} else {
		return 0;
	}

	if (usb_dev->devnum < 0) {
		/* driver is often null here; dev_dbg() would oops */
		pr_debug("usb %s: already deleted?\n", dev_name(dev));
		return -ENODEV;
	}
	if (!usb_dev->bus) {
		pr_debug("usb %s: bus removed?\n", dev_name(dev));
		return -ENODEV;
	}

	/* per-device configurations are common */
	if (add_uevent_var(env, "PRODUCT=%x/%x/%x",
			   le16_to_cpu(usb_dev->descriptor.idVendor),
			   le16_to_cpu(usb_dev->descriptor.idProduct),
			   le16_to_cpu(usb_dev->descriptor.bcdDevice)))
		return -ENOMEM;

	/* class-based driver binding models */
	if (add_uevent_var(env, "TYPE=%d/%d/%d",
			   usb_dev->descriptor.bDeviceClass,
			   usb_dev->descriptor.bDeviceSubClass,
			   usb_dev->descriptor.bDeviceProtocol))
		return -ENOMEM;

	return 0;
}
```

https://www.cnblogs.com/downey-blog/p/10507703.html

> - `name`: 该bus的名字，这个名字是这个bus在sysfs文件系统中的体现，对应/sys/bus/$name.
> - `dev_name`: 这个dev_name并不对应bus的名称，而是对应bus所包含的struct device的名字，即对应dev_root。
> - `dev_root`：bus对应的device结构，每个设备都需要对应一个相应的struct device.
> - `match`:bus的device链表和driver链表进行匹配的实际执行回调函数，每当有device或者driver添加到bus中时，调用match函数，为device(driver)寻找匹配的driver(device)。
> - `uevent`: bus时间回调函数，当属于这个bus的设备发生添加、删除、修改等行为时，都将出发uvent事件。
> - `probe`: 当device和driver经由match匹配成功时，将会调用总线的probe函数实现具体driver的初始化。事实上每个driver也会提供相应的probe函数，先调用总线的probe函数，在总线probe函数中调用driver的probe函数。
> - `remove`: 移除挂载在设备上的driver，bus上的driver部分也会提供remove函数，在执行移除时，先调用driver的remove，然后再调用bus的remove以清除资源。

#### bus_register_notifier

通知链，用于设备/接口的添加、删除的通知。

```c
	retval = bus_register_notifier(&usb_bus_type, &usb_bus_nb);

static struct notifier_block usb_bus_nb = {
	.notifier_call = usb_bus_notify,
};

/*
 * Notifications of device and interface registration
 */
static int usb_bus_notify(struct notifier_block *nb, unsigned long action,
		void *data)
{
	struct device *dev = data;

	switch (action) {
	case BUS_NOTIFY_ADD_DEVICE:
		if (dev->type == &usb_device_type)
			(void) usb_create_sysfs_dev_files(to_usb_device(dev));
		else if (dev->type == &usb_if_device_type)
			usb_create_sysfs_intf_files(to_usb_interface(dev));
		break;

	case BUS_NOTIFY_DEL_DEVICE:
		if (dev->type == &usb_device_type)
			usb_remove_sysfs_dev_files(to_usb_device(dev));
		else if (dev->type == &usb_if_device_type)
			usb_remove_sysfs_intf_files(to_usb_interface(dev));
		break;
	}
	return 0;
}
```

#### usb_register

注册 usbfs 驱动，这是一个**接口驱动**。🧠奇怪，这个驱动不需要指定匹配方式吗？哦哦应该是只通过名字来匹配哦。

将 usbfs 驱动提交到设备模型，添加到USB总线的驱动链表里。

设备驱动`struct usb_device_driver`和接口驱动`struct usb_driver`分别用了不同的数据结构表示。此外还有一个 `struct usb_class_driver` 用于表示设备类驱动。

```c
	retval = usb_register(&usbfs_driver);
// devio.c
struct usb_driver usbfs_driver = {
	.name =		"usbfs",
	.probe =	driver_probe,
	.disconnect =	driver_disconnect,
	.suspend =	driver_suspend,
	.resume =	driver_resume,
	.supports_autosuspend = 1,
};

/* use a define to avoid include chaining to get THIS_MODULE & friends */
#define usb_register(driver) \
	usb_register_driver(driver, THIS_MODULE, KBUILD_MODNAME)
```

和下面的 `usb_register_device_driver` 注册设备驱动区分。

#### usb_devio_init

```c
int __init usb_devio_init(void)
{
	int retval;

	// 申请 USB_DEVICE_MAX 个设备号，所有的usb相关的文件节点都是用的这个 MAJOR 设备号
	// #define USB_DEVICE_DEV MKDEV(USB_DEVICE_MAJOR, 0)
	// #define USB_MAXBUS 64
	// #define USB_DEVICE_MAX (USB_MAXBUS * 128)
	retval = register_chrdev_region(USB_DEVICE_DEV, USB_DEVICE_MAX,
					"usb_device");
	if (retval) {
		printk(KERN_ERR "Unable to register minors for usb_device\n");
		goto out;
	}
	// 将字符设备与文件操作关联
	cdev_init(&usb_device_cdev, &usbdev_file_operations);
	// 将字符设备添加到内核设备链表
	retval = cdev_add(&usb_device_cdev, USB_DEVICE_DEV, USB_DEVICE_MAX);
	if (retval) {
		printk(KERN_ERR "Unable to get usb_device major %d\n",
		       USB_DEVICE_MAJOR);
		goto error_cdev;
	}
	usb_register_notify(&usbdev_nb);
out:
	return retval;

error_cdev:
	unregister_chrdev_region(USB_DEVICE_DEV, USB_DEVICE_MAX);
	goto out;
}
```

#### usb_hub_init

注册 hub 接口驱动。

```c
static struct usb_driver hub_driver = {
	.name =		"hub",
	.probe =	hub_probe,
	.disconnect =	hub_disconnect,
	.suspend =	hub_suspend,
	.resume =	hub_resume,
	.reset_resume =	hub_reset_resume,
	.pre_reset =	hub_pre_reset,
	.post_reset =	hub_post_reset,
	.unlocked_ioctl = hub_ioctl,
	.id_table =	hub_id_table,
	.supports_autosuspend =	1,
};

int usb_hub_init(void)
{
	// 注册 hub 接口驱动
	if (usb_register(&hub_driver) < 0) {
		printk(KERN_ERR "%s: can't register hub driver\n",
			usbcore_name);
		return -1;
	}

	/*
	 * The workqueue needs to be freezable to avoid interfering with
	 * USB-PERSIST port handover. Otherwise it might see that a full-speed
	 * device was gone before the EHCI controller had handed its port
	 * over to the companion full-speed controller.
	 */
	// 创建一个 workqueue to process hub events
	hub_wq = alloc_workqueue("usb_hub_wq", WQ_FREEZABLE, 0);
	if (hub_wq)
		return 0;

	/* Fall through if kernel_thread failed */
	usb_deregister(&hub_driver);
	pr_err("%s: can't allocate workqueue for usb hub\n", usbcore_name);

	return -1;
}
```

#### usb_register_device_driver

> https://blog.csdn.net/qq_41483419/article/details/129158142
>
>  **只要是usb设备，都会跟usb_generic_driver匹配上**。
>
> usb_generic_driver中的generic_probe函数，这个函数是一个usb设备的第一个匹配的driver。Generic通用，只要是个usb设备就得先跟他来一段，usb设备驱动界的老大。他的probe干啥了呢？很简单！找个合适的配置，配置一下。从此usb设备就进入配置的时代了。(前期的工作谁做的呢，到这都已经设置完地址了，当然是hub了，hub发现设备后，会进行前期的枚举过程，获得配置，最终调用device_add将该usb设备添加到总线上。这个过程可以专门来一大段，是hub的主要工作，所以需要把hub单独作为一个家族来对待，人家可是走在第一线的默默无闻的工作者，默默的将设备枚举完成后，将这个设备添加到usb总线上，多伟大)
>

```c
usb_register_device_driver(&usb_generic_driver, THIS_MODULE);

// generic.c
struct usb_device_driver usb_generic_driver = {
	.name =	"usb",
	.match = usb_generic_driver_match,
	.probe = usb_generic_driver_probe,
	.disconnect = usb_generic_driver_disconnect,
#ifdef	CONFIG_PM
	.suspend = usb_generic_driver_suspend,
	.resume = usb_generic_driver_resume,
#endif
	.supports_autosuspend = 1,
};
```

usb_device_driver结构体是usb_driver的简化版本,这里注册的是usb设备(非接口)驱动。

usb总线的match方法对usb设备和usb接口做了区分处理,针对usb设备,直接match的,(分析match时候再细化)

然后调用usb设备驱动的probe方法

```c
static int usb_probe_device(struct device *dev)
{
    struct usb_device_driver *udriver = to_usb_device_driver(dev->driver);
    struct usb_device *udev = to_usb_device(dev);
    int error = 0;
    dev_dbg(dev, "%s\n", __func__);
    if (!udriver->supports_autosuspend)  //条件成立
        error = usb_autoresume_device(udev);
    if (!error)
        error = udriver->probe(udev);    //调用usb_device_driver的probe方法
    return error;
}
```

接着调用usb_generic_driver的probe方法

对应到root hub,流程会转入到generic_probe().代码如下:

```c
static int generic_probe(struct usb_device *udev)
{
    /* Choose and set the configuration.  This registers the interfaces
     * with the driver core and lets interface drivers bind to them.
     */
    if (udev->authorized == 0) //至于udev->authorized,在root hub的初始化中,是会将其初始化为1的.后面的逻辑就更简单了.为root hub 选择一个配置然后再设定这个配置.
        dev_err(&udev->dev, "Device is not authorized for usage\n");
    else {
        c = usb_choose_configuration(udev); //Usb2.0 spec上规定,对于hub设备,只能有一个config,一个interface,一个endpoint.实际上,在这里,对hub的选择约束不大,反正就一个配置,不管怎么样,选择和设定都是这个配置.
        if (c >= 0) {
            err = usb_set_configuration(udev, c);
            if (err && err != -ENODEV) {
                dev_err(&udev->dev, "can't set config #%d, error %d\n",
                    c, err);
                /* This need not be fatal.  The user can try to
                 * set other configurations. */
            }
        }
    }
    /* USB device state == configured ... usable */
    usb_notify_add_device(udev);
    return 0;
}
```

#### 总结

USB CORE 模块初始化过程：

- 注册 usb 总线
- 注册 usbfs 接口驱动
- 注册 hub 接口驱动
- 注册 usb generic 设备驱动

> https://blog.csdn.net/vitalma/article/details/136300790

```c
usb_debugfs_init;
---> /sys/kernel/debug/usb/devices; dump USB设备信息
bus_register; 总线节点
---> /sys/bus/usb;
---> /sys/bus/usb/drivers 目录
---> /sys/bus/usb/devices 目录
---> /sys/bus/usb/drivers_probe
---> /sys/bus/usb/drivers_autoprobe
---> /sys/bus/usb/uevent
usb_register;
---> /sys/bus/usb/drivers/usbfs 目录
usb_hub_init;
---> /sys/bus/usb/drivers/hub 目录
usb_register_device_driver:
---> /sys/bus/usb/drivers/usb 目录
```

简单看下其他文件实现：

```c
usbcore-y := usb.o hub.o hcd.o urb.o message.o driver.o
usbcore-y += config.o file.o buffer.o sysfs.o endpoint.o
usbcore-y += devio.o notify.o generic.o quirks.o devices.o
usbcore-y += phy.o port.o
usbcore-$(CONFIG_OF)		+= of.o
obj-$(CONFIG_USB)		+= usbcore.o
buffer.c: DMA mempool，提供一些 HCD 会用到的API
config.c: 低速全速高速超速的最大包大小、解析endpoint/interface/configuration描述符。
devices.c: 就一个作用 dump 所有设备信息。见 [debugfs/usb/devices]
devio.c: usbfs 驱动，实现了usb字符设备的文件操作，read/write/ioctl 等
driver.c: 提供了驱动注册相关的API
endpoint.c: 创建、删除endpoint
file.c: 在 /dev 目录下创建设备，设备名是对于的driver名+miner设备号。
generic.c: generic 设备驱动实现。
hcd.c: HCD ....
hub.c: HUB 接口驱动，这个文件也有很多内容，好像设备枚举阶段都是它完成。
message.c: 这玩意怎么也很多内容
port.c: hub的端口相关
sysfs.c: /sys/bus/usb 相关吧
urb.c: USB Request Block.
```

TODO：

- [ ] 确定这三个驱动的作用。
- [ ] 接口驱动和设备驱动的关系。

- https://www.cnblogs.com/zyly/p/16255617.html
- **[Linux USB驱动分析(一)](https://blog.csdn.net/jun_8018/article/details/116096984)**
- **[Linux USB驱动分析(二)](https://blog.csdn.net/jun_8018/article/details/116137244)**
- **[Linux USB驱动分析(三)](https://blog.csdn.net/jun_8018/article/details/116211644)**
- **[Linux USB总线驱动框架分析](https://zhuanlan.zhihu.com/p/61079354)**
- **[USB主机控制器驱动——OHCI分析](https://blog.csdn.net/qq_30736309/article/details/79583098)**
- **[二、usb子系统初始化](https://blog.csdn.net/jixianghao/article/details/45336873)**
- **[【linux驱动】USB子系统分析](https://blog.csdn.net/fengyuwuzu0519/article/details/104259648)**

#### debugfs/usb/devices

dump USB设备信息

```c
root@h:usb# cat /sys/kernel/debug/usb/devices

T:  Bus=01 Lev=00 Prnt=00 Port=00 Cnt=00 Dev#=  1 Spd=480  MxCh=16
B:  Alloc=  0/800 us ( 0%), #Int=  0, #Iso=  0
D:  Ver= 2.00 Cls=09(hub  ) Sub=00 Prot=01 MxPS=64 #Cfgs=  1
P:  Vendor=1d6b ProdID=0002 Rev= 6.08
S:  Manufacturer=Linux 6.8.0-41-generic xhci-hcd
S:  Product=xHCI Host Controller
S:  SerialNumber=0000:00:14.0
C:* #Ifs= 1 Cfg#= 1 Atr=e0 MxPwr=  0mA
I:* If#= 0 Alt= 0 #EPs= 1 Cls=09(hub  ) Sub=00 Prot=00 Driver=hub
E:  Ad=81(I) Atr=03(Int.) MxPS=   4 Ivl=256ms

T:  Bus=01 Lev=01 Prnt=01 Port=01 Cnt=01 Dev#=  2 Spd=12   MxCh= 0
D:  Ver= 2.00 Cls=00(>ifc ) Sub=00 Prot=00 MxPS=64 #Cfgs=  1
P:  Vendor=0b05 ProdID=19af Rev= 1.00
S:  Manufacturer=AsusTek Computer Inc.
S:  Product=AURA LED Controller
S:  SerialNumber=9876543210
C:* #Ifs= 2 Cfg#= 1 Atr=a0 MxPwr= 16mA
I:* If#= 0 Alt= 0 #EPs= 0 Cls=ff(vend.) Sub=ff Prot=ff Driver=(none)
I:* If#= 2 Alt= 0 #EPs= 1 Cls=03(HID  ) Sub=00 Prot=00 Driver=usbhid
E:  Ad=82(I) Atr=03(Int.) MxPS=  32 Ivl=4ms

T:  Bus=01 Lev=01 Prnt=01 Port=08 Cnt=02 Dev#=  3 Spd=480  MxCh= 4
D:  Ver= 2.00 Cls=09(hub  ) Sub=00 Prot=01 MxPS=64 #Cfgs=  1
P:  Vendor=05e3 ProdID=0608 Rev=60.90
S:  Product=USB2.0 Hub
C:* #Ifs= 1 Cfg#= 1 Atr=e0 MxPwr=100mA
I:* If#= 0 Alt= 0 #EPs= 1 Cls=09(hub  ) Sub=00 Prot=00 Driver=hub
E:  Ad=81(I) Atr=03(Int.) MxPS=   1 Ivl=256ms

T:  Bus=01 Lev=02 Prnt=03 Port=00 Cnt=01 Dev#=  5 Spd=12   MxCh= 0
D:  Ver= 2.00 Cls=e0(wlcon) Sub=01 Prot=01 MxPS=64 #Cfgs=  1
P:  Vendor=0a12 ProdID=0001 Rev=88.91
C:* #Ifs= 2 Cfg#= 1 Atr=c0 MxPwr=  0mA
I:* If#= 0 Alt= 0 #EPs= 3 Cls=e0(wlcon) Sub=01 Prot=01 Driver=btusb
E:  Ad=81(I) Atr=03(Int.) MxPS=  16 Ivl=1ms
E:  Ad=02(O) Atr=02(Bulk) MxPS=  64 Ivl=0ms
E:  Ad=82(I) Atr=02(Bulk) MxPS=  64 Ivl=0ms
I:* If#= 1 Alt= 0 #EPs= 2 Cls=e0(wlcon) Sub=01 Prot=01 Driver=btusb
E:  Ad=03(O) Atr=01(Isoc) MxPS=   0 Ivl=1ms
E:  Ad=83(I) Atr=01(Isoc) MxPS=   0 Ivl=1ms
I:  If#= 1 Alt= 1 #EPs= 2 Cls=e0(wlcon) Sub=01 Prot=01 Driver=btusb
E:  Ad=03(O) Atr=01(Isoc) MxPS=   9 Ivl=1ms
E:  Ad=83(I) Atr=01(Isoc) MxPS=   9 Ivl=1ms
I:  If#= 1 Alt= 2 #EPs= 2 Cls=e0(wlcon) Sub=01 Prot=01 Driver=btusb
E:  Ad=03(O) Atr=01(Isoc) MxPS=  17 Ivl=1ms
E:  Ad=83(I) Atr=01(Isoc) MxPS=  17 Ivl=1ms
I:  If#= 1 Alt= 3 #EPs= 2 Cls=e0(wlcon) Sub=01 Prot=01 Driver=btusb
E:  Ad=03(O) Atr=01(Isoc) MxPS=  25 Ivl=1ms
E:  Ad=83(I) Atr=01(Isoc) MxPS=  25 Ivl=1ms
I:  If#= 1 Alt= 4 #EPs= 2 Cls=e0(wlcon) Sub=01 Prot=01 Driver=btusb
E:  Ad=03(O) Atr=01(Isoc) MxPS=  33 Ivl=1ms
E:  Ad=83(I) Atr=01(Isoc) MxPS=  33 Ivl=1ms
I:  If#= 1 Alt= 5 #EPs= 2 Cls=e0(wlcon) Sub=01 Prot=01 Driver=btusb
E:  Ad=03(O) Atr=01(Isoc) MxPS=  49 Ivl=1ms
E:  Ad=83(I) Atr=01(Isoc) MxPS=  49 Ivl=1ms

T:  Bus=01 Lev=02 Prnt=03 Port=01 Cnt=02 Dev#=  7 Spd=480  MxCh= 0
D:  Ver= 2.00 Cls=00(>ifc ) Sub=00 Prot=00 MxPS=64 #Cfgs=  1
P:  Vendor=14cd ProdID=1212 Rev= 1.00
S:  Manufacturer=Generic
S:  Product=Mass Storage Device
S:  SerialNumber=121220160204
C:* #Ifs= 1 Cfg#= 1 Atr=80 MxPwr=100mA
I:* If#= 0 Alt= 0 #EPs= 2 Cls=08(stor.) Sub=06 Prot=50 Driver=usb-storage
E:  Ad=81(I) Atr=02(Bulk) MxPS= 512 Ivl=0ms
E:  Ad=02(O) Atr=02(Bulk) MxPS= 512 Ivl=0ms

T:  Bus=01 Lev=02 Prnt=03 Port=02 Cnt=03 Dev#= 14 Spd=480  MxCh= 0
D:  Ver= 2.00 Cls=00(>ifc ) Sub=00 Prot=00 MxPS=64 #Cfgs=  1
P:  Vendor=3346 ProdID=1009 Rev= 5.10
S:  Manufacturer=Cvitek
S:  Product=RNDIS
S:  SerialNumber=0123456789
C:* #Ifs= 2 Cfg#= 1 Atr=80 MxPwr=120mA
A:  FirstIf#= 0 IfCount= 2 Cls=02(comm.) Sub=06 Prot=00
I:* If#= 0 Alt= 0 #EPs= 1 Cls=02(comm.) Sub=02 Prot=ff Driver=rndis_host
E:  Ad=82(I) Atr=03(Int.) MxPS=   8 Ivl=32ms
I:* If#= 1 Alt= 0 #EPs= 2 Cls=0a(data ) Sub=00 Prot=00 Driver=rndis_host
E:  Ad=81(I) Atr=02(Bulk) MxPS= 512 Ivl=0ms
E:  Ad=01(O) Atr=02(Bulk) MxPS= 512 Ivl=0ms

T:  Bus=01 Lev=01 Prnt=01 Port=09 Cnt=03 Dev#=  4 Spd=12   MxCh= 0
D:  Ver= 2.00 Cls=00(>ifc ) Sub=00 Prot=00 MxPS= 8 #Cfgs=  1
P:  Vendor=320f ProdID=5088 Rev= 1.04
S:  Manufacturer=Telink
S:  Product=Wireless Gaming Keyboard
C:* #Ifs= 2 Cfg#= 1 Atr=a0 MxPwr=200mA
I:* If#= 0 Alt= 0 #EPs= 1 Cls=03(HID  ) Sub=01 Prot=01 Driver=usbhid
E:  Ad=81(I) Atr=03(Int.) MxPS=   8 Ivl=1ms
I:* If#= 1 Alt= 0 #EPs= 2 Cls=03(HID  ) Sub=01 Prot=01 Driver=usbhid
E:  Ad=82(I) Atr=03(Int.) MxPS=  64 Ivl=1ms
E:  Ad=05(O) Atr=03(Int.) MxPS=  64 Ivl=1ms

T:  Bus=01 Lev=01 Prnt=01 Port=12 Cnt=04 Dev#=  8 Spd=480  MxCh= 4
D:  Ver= 2.00 Cls=09(hub  ) Sub=00 Prot=01 MxPS=64 #Cfgs=  1
P:  Vendor=058f ProdID=6254 Rev= 1.00
C:* #Ifs= 1 Cfg#= 1 Atr=e0 MxPwr=100mA
I:* If#= 0 Alt= 0 #EPs= 1 Cls=09(hub  ) Sub=00 Prot=00 Driver=hub
E:  Ad=81(I) Atr=03(Int.) MxPS=   1 Ivl=256ms

T:  Bus=01 Lev=02 Prnt=08 Port=00 Cnt=01 Dev#=  9 Spd=12   MxCh= 0
D:  Ver= 1.10 Cls=00(>ifc ) Sub=00 Prot=00 MxPS= 8 #Cfgs=  1
P:  Vendor=046d ProdID=c542 Rev= 3.03
S:  Manufacturer=Logitech
S:  Product=Wireless Receiver
C:* #Ifs= 1 Cfg#= 1 Atr=a0 MxPwr= 50mA
I:* If#= 0 Alt= 0 #EPs= 1 Cls=03(HID  ) Sub=01 Prot=02 Driver=usbhid
E:  Ad=82(I) Atr=03(Int.) MxPS=   8 Ivl=4ms

T:  Bus=02 Lev=00 Prnt=00 Port=00 Cnt=00 Dev#=  1 Spd=10000 MxCh= 9
B:  Alloc=  0/800 us ( 0%), #Int=  0, #Iso=  0
D:  Ver= 3.10 Cls=09(hub  ) Sub=00 Prot=03 MxPS= 9 #Cfgs=  1
P:  Vendor=1d6b ProdID=0003 Rev= 6.08
S:  Manufacturer=Linux 6.8.0-41-generic xhci-hcd
S:  Product=xHCI Host Controller
S:  SerialNumber=0000:00:14.0
C:* #Ifs= 1 Cfg#= 1 Atr=e0 MxPwr=  0mA
I:* If#= 0 Alt= 0 #EPs= 1 Cls=09(hub  ) Sub=00 Prot=00 Driver=hub
E:  Ad=81(I) Atr=03(Int.) MxPS=   4 Ivl=256ms
```

前面字段的意思：

```c
// usb/core/devices.c
static const char format_topo[] =
/* T:  Bus=dd Lev=dd Prnt=dd Port=dd Cnt=dd Dev#=ddd Spd=dddd MxCh=dd */
"\nT:  Bus=%2.2d Lev=%2.2d Prnt=%2.2d Port=%2.2d Cnt=%2.2d Dev#=%3d Spd=%-4s MxCh=%2d\n";

static const char format_string_manufacturer[] =
/* S:  Manufacturer=xxxx */
  "S:  Manufacturer=%.100s\n";

static const char format_string_product[] =
/* S:  Product=xxxx */
  "S:  Product=%.100s\n";

#ifdef ALLOW_SERIAL_NUMBER
static const char format_string_serialnumber[] =
/* S:  SerialNumber=xxxx */
  "S:  SerialNumber=%.100s\n";
#endif

static const char format_bandwidth[] =
/* B:  Alloc=ddd/ddd us (xx%), #Int=ddd, #Iso=ddd */
  "B:  Alloc=%3d/%3d us (%2d%%), #Int=%3d, #Iso=%3d\n";

static const char format_device1[] =
/* D:  Ver=xx.xx Cls=xx(sssss) Sub=xx Prot=xx MxPS=dd #Cfgs=dd */
  "D:  Ver=%2x.%02x Cls=%02x(%-5s) Sub=%02x Prot=%02x MxPS=%2d #Cfgs=%3d\n";

static const char format_device2[] =
/* P:  Vendor=xxxx ProdID=xxxx Rev=xx.xx */
  "P:  Vendor=%04x ProdID=%04x Rev=%2x.%02x\n";

static const char format_config[] =
/* C:  #Ifs=dd Cfg#=dd Atr=xx MPwr=dddmA */
  "C:%c #Ifs=%2d Cfg#=%2d Atr=%02x MxPwr=%3dmA\n";

static const char format_iad[] =
/* A:  FirstIf#=dd IfCount=dd Cls=xx(sssss) Sub=xx Prot=xx */
  "A:  FirstIf#=%2d IfCount=%2d Cls=%02x(%-5s) Sub=%02x Prot=%02x\n";

static const char format_iface[] =
/* I:  If#=dd Alt=dd #EPs=dd Cls=xx(sssss) Sub=xx Prot=xx Driver=xxxx*/
  "I:%c If#=%2d Alt=%2d #EPs=%2d Cls=%02x(%-5s) Sub=%02x Prot=%02x Driver=%s\n";

static const char format_endpt[] =
/* E:  Ad=xx(s) Atr=xx(ssss) MxPS=dddd Ivl=D?s */
  "E:  Ad=%02x(%c) Atr=%02x(%-4s) MxPS=%4d Ivl=%d%cs\n";
```

#### file.c

```bash
root@h:usb# dmesg | tail -n 20
[45807.088223] usb 1-9.4: New USB device found, idVendor=0403, idProduct=6001, bcdDevice= 6.00
[45807.088238] usb 1-9.4: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[45807.088244] usb 1-9.4: Product: FT232R USB UART
[45807.088248] usb 1-9.4: Manufacturer: FTDI
[45807.088252] usb 1-9.4: SerialNumber: A50285BI
[45807.096989] ftdi_sio 1-9.4:1.0: FTDI USB Serial Device converter detected
[45807.097079] usb 1-9.4: Detected FT232R
[45807.097821] usb 1-9.4: FTDI USB Serial Device converter now attached to ttyUSB0
[45956.898368] usb 1-12: new full-speed USB device number 16 using xhci_hcd
[45957.039833] usb 1-12: New USB device found, idVendor=320f, idProduct=5055, bcdDevice= 1.04
[45957.039852] usb 1-12: New USB device strings: Mfr=1, Product=2, SerialNumber=0
[45957.039859] usb 1-12: Product: VGN N75 PRO
[45957.039864] usb 1-12: Manufacturer: Telink
[45957.049757] input: Telink VGN N75 PRO  as /devices/pci0000:00/0000:00:14.0/usb1/1-12/1-12:1.0/0003:320F:5055.0007/input/input33
[45957.102483] hid-generic 0003:320F:5055.0007: input,hidraw3: USB HID v1.11 Keyboard [Telink VGN N75 PRO ] on usb-0000:00:14.0-12/input0
[45957.118181] input: Telink VGN N75 PRO  Keyboard as /devices/pci0000:00/0000:00:14.0/usb1/1-12/1-12:1.1/0003:320F:5055.0008/input/input34
[45957.169742] input: Telink VGN N75 PRO  as /devices/pci0000:00/0000:00:14.0/usb1/1-12/1-12:1.1/0003:320F:5055.0008/input/input35
[45957.169984] input: Telink VGN N75 PRO  as /devices/pci0000:00/0000:00:14.0/usb1/1-12/1-12:1.1/0003:320F:5055.0008/input/input36
[45957.170313] input: Telink VGN N75 PRO  Mouse as /devices/pci0000:00/0000:00:14.0/usb1/1-12/1-12:1.1/0003:320F:5055.0008/input/input37
[45957.170779] hid-generic 0003:320F:5055.0008: input,hiddev2,hidraw4: USB HID v1.11 Keyboard [Telink VGN N75 PRO ] on usb-0000:00:14.0-12/input1
root@h:usb# ls
hiddev0  hiddev1  hiddev2
root@h:usb# ls -lh
total 0
crw------- 1 root root 180, 0  9月  1 09:12 hiddev0
crw------- 1 root root 180, 1  9月  1 09:12 hiddev1
crw------- 1 root root 180, 2  9月  1 21:58 hiddev2
root@h:usb#
```

### usbfs 接口驱动

### hub 接口驱动

> https://zhuanlan.zhihu.com/p/61079354

![](https://pica.zhimg.com/80/v2-6257ed39e775173bc62759393ddbb21c_720w.webp)

硬件主机控制器Host Controller之上运行的是HCD，是对主机控制器硬件的一个抽象，实现核心层与控制器之间的对话接口，USB HCD包含多种USB接口规范：

（1）UHCI：Intel提供，通用主机控制接口，USB1.0/1.1；

（2）OHCI：微软提供，开放主机控制接口，USB1.0/1.1；

（3）EHCI：增强主机控制接口，USB2.0；

2.4 USB Device Driver

USB设备驱动框架如下图所示：

![](https://picx.zhimg.com/80/v2-e2d44ba87579ca5f775dc2d79fd48143_720w.webp)

USB设备是由一些配置(configuration)、接口（interface）和端点(endpoint)组成，，即一个USB设备可以含有一个或多个配置，在每个配置中可含有一个或多个接口，在每个接口中可含有若干个端点。一个USB设备驱动可能包含多个子驱动。一个USB设备子驱动程序对应一个USB接口，而非整个USB设备。

USB设备使用各种描述符来说明其设备架构，包括设备描述符、配置描述符、接口描述符、端点描述符、字符串描述符。后面单独讨论USB设备描述符。

USB传输的对象为端点(endpoint)，每一个端点都有传输类型，传输方向，除了端点0外，每一个端点只支持一个方向的数据传输，端点0用于控制传输，既能输出也能输入。输入(IN)、输出(OUT) "都是" 基于USB主机的立场说的。比如鼠标的数据是从鼠标传到PC机, 对应的端点称为"输入端点"。

通过以上分析，USB设备驱动模型可以概括为如下图。

![](https://pic4.zhimg.com/80/v2-d3eab094e80102ae1ad79ed7fc477c2b_720w.webp)

主要包含三个部分：USB控制器驱动，USB核心，USB设备驱动。如上图khubd是USB守护进程，当USB设备插入的时候，守护进程监测到，USB主机控制器就会产生一个hub_irq中断，控制器调用hub的探测函数，来解析设备信息。

### hub.c 源码分析

hub.c 文件内容很多，但hub.h 中对外的接口还好。数据结构主要就一个 `usb_hub, usb_port`。

hub 参数太多了，且一开始不方便理解。先从 usb_port 开始

```c
struct usb_hub {
	struct usb_port		**ports; /* array of usb_port pointers */
};

/**
 * struct usb port - kernel's representation of a usb port
 * @child: usb device attached to the port
 * @dev: generic device interface
 * @port_owner: port's owner
 * @peer: related usb2 and usb3 ports (share the same connector)
 * @req: default pm qos request for hubs without port power control
 * @connect_type: port's connect type
 * @location: opaque representation of platform connector location
 * @status_lock: synchronize port_event() vs usb_port_{suspend|resume}
 * @portnum: port index num based one
 * @is_superspeed cache super-speed status
 * @usb3_lpm_u1_permit: whether USB3 U1 LPM is permitted.
 * @usb3_lpm_u2_permit: whether USB3 U2 LPM is permitted.
 */
struct usb_port {
	struct usb_device *child;
	struct device dev;
	struct usb_dev_state *port_owner;
	struct usb_port *peer;
	struct dev_pm_qos_request *req;
	enum usb_port_connect_type connect_type;
	usb_port_location_t location;
	struct mutex status_lock;
	u32 over_current_count;
	u8 portnum;
	u32 quirks;
	unsigned int is_superspeed:1;
	unsigned int usb3_lpm_u1_permit:1;
	unsigned int usb3_lpm_u2_permit:1;
};
```
