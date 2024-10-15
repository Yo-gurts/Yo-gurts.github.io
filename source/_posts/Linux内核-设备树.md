---
title: 设备树语法介绍
top_img: transparent
date: 2023-08-08 13:45:42
updated: 2024-10-15 22:24:42
tags:
  - Linux
  - Kernel
  - DTS
  - Uboot
  - 设备树
categories: Linux
keywords:
description:
---

## 设备树语法

1. 什么是设备？
    直观点：键盘、鼠标、内存、CPU、显示器、`wifi`、网卡、显卡是设备。
    底层点：`SPI` 、`I2C`、`I2S` 、`mipi`、`gpio`、`USB` 等控制器。

2. Linux / Uboot 是如何管理这些设备的呢？
    **父子关系**
    `I2C` 等控制器与通过该控制器接入的设备，明显就是父子关系。
    **根设备**

    `SPI`、 `I2C` 控制器这样看起来没有关联的设备，他们就直接挂靠在根设备下。

    ![image-20231230124213784](../images/Linux%E5%86%85%E6%A0%B8-%E8%AE%BE%E5%A4%87%E6%A0%91/image-20231230124213784.png)

3. 为什么要有设备树？

    这个和 Linux 的设备-驱动模型对应，驱动 - 设备分离，通过 `Kconfig` 定义的配置项决定是否启用驱动。设备是否启用则是由设备树来决定。
    Linux 支持的设备类型很多很多，不可能将所有设备都启用（会浪费大量的内存、CPU资源）。设备树就是用来告诉 Kernel 需要启用哪些设备。

## 语法说明

**设备树 `dtsi` 文件中使用 c 语言的方式注释 `/* 注释 */` 以及 `// 注释`，特别注意 `#` 是有特殊意义的，不是注释！！**

设备之间的父子关系，用文本表示出来是用 `{}` 管理的包含关系。

```c
/ { // ‘/’ 表示根设备
	spi0: spi0@04180000 { // spi0 是 SPI 控制器
        compatible = "snps,dw-apb-ssi";
        reg = <0x0 0x04180000 0x0 0x10000>;
        clocks = <&clk CV181X_CLK_SPI>;

		panel@0 { // panel 是 SPI0 的一个子设备
            compatible = "xxx,st7789v";
            spi-frequency = <48000000>;
        }
    };
};
```

好了，接下来就看 [devicetree.org  的标准文档](https://www.devicetree.org/)，英文的，篇幅不长。


## **节点格式**

一个节点就代表一个设备。

```c
label: node-name@unit-address
```

其中：

> - label：标签（可以省略），它的作用是为了方便地引用 `node`。
> - node-name：节点名字，设备
> - unit-address：单元地址（可以没有）

### 引用节点

`label` 的作用是为了方便地引用 `node`。引用的目的一般是为了 **追加/修改节点内容**，比如 ：

```c
/dts-v1/;
/{
    uart0: uart@fe001000 {
        compatible="ns16550";
        reg=<0xfe001000 0x100>;
    };
};
```

可以使用下面 2 种方法来修改 `uart@fe001000` 这个 `node`：

```c
// 在根节点之外使用 label 引用 node
&uart0 {
    status="disabled";
};
// 或在根节点之外使用全路径，（不使用标签）
&{/uart@fe001000} {
    status="disabled";
};
```

## **属性格式**

简单地说， `properties` 就是 `name=value`， `value` 有多种取值方式。示例：

```c
// 一个 32 位的数据，用尖括号包围起来，可以用十六进制，也可以用十进制，如
interrupts = <0xc>;
interrupts = <17>;

// 一个 64 位数据（只能使用 2 个 32 位数据表示），用尖括号包围起来，如：
clock-frequency = <0x00000001 0x00000000>;

// 也可以传递多个值，具体看驱动如何解析
test-property = <11 22 33>;

// 有结束符的字符串，用双引号包围起来，如：
compatible = "simple-bus";

// 字节序列，用中括号包围起来，如：
local-mac-address = [00 00 12 34 56 78]; // 每个 byte 使用 2 个 16 进制数来表示
local-mac-address = [000012345678];      // 每个 byte 使用 2 个 16 进制数来表示

// 可以是各种值的组合，用逗号隔开，如：
compatible = "ns16550", "ns8250";
example = <0xf00f0000 19>, "a strange property format";
```

## **一些标准属性**

### **compatible 属性**

“compatible” 表示 “兼容”，对于某个 LED，内核中可能有 A、B、C 三个驱动都支持它，那可以这样写：

```c
led {
    compatible = “A”, “B”, “C”;
};
```

内核启动时，**就会为这个 LED 按这样的优先顺序为它找到驱动程序**：A、B、C。

#### **model 属性**

用来表示设备型号而已。没啥用。

model 属性与 `compatible` 属性有些类似，但是有差别。`compatible` 属性是一个字符串列表，表示可以你的硬件兼容 A、B、C 等驱动；model 用来准确地定义这个硬件是什么。比如根节点中可以这样写：

```c
/ {
    compatible = "samsung,smdk2440", "samsung,mini2440";
    model = "jz2440_v3";
};
```

它表示这个单板，可以兼容内核中的 “smdk2440”，也兼容 “mini2440”。

从 `compatible` 属性中可以知道它兼容哪些板，但是它到底是什么板？用 model 属性来明确。

### **status 属性**

status 属性看名字就知道是和设备状态有关的， status 属性值也是字符串，字符串是设备的状态信息，可选的状态如下所示：

![property](../images/Linux%E5%86%85%E6%A0%B8-%E8%AE%BE%E5%A4%87%E6%A0%91/property.png)

### **#address-cells 和 #size-cells 属性**

格式：

```c
# address-cells：address    要用多少个 32 位数来表示；
# size-cells：size          要用多少个 32 位数来表示。
```

比如一段内存，怎么描述它的起始地址和大小？

下例中，`address-cells` 为 1，所以 reg 中用 1 个数来表示地址，即用 0x80000000 来表示地址；size-cells 为 1，所以 reg 中用 1 个数来表示大小，即用 0x20000000 表示大小：

```c
/ {
    # address-cells = <1>;
    # size-cells = <1>;
    memory {
        reg = <0x80000000 0x20000000>;
    };
};
```

### **reg 属性**

`reg` 属性的值，是一系列的 `<address size>`，用多少个 32 位的数来表示 `address` 和 `size`，由其**父节点**的 `#address-cells`、`#size-cells` 决定。示例：

```c
/dts-v1/;
/ {
    # address-cells = <1>;
    # size-cells = <1>;
    // 这里 memory 这个节点的父节点的 address-cells = 1, 表示地址长度为 32 位，
    // size-cells = 1 表示长度是用 32 位的数来表示，size-cells=2 表示长度是用 64 位的数来表示。
    memory {
         reg = <0x8000 0x10 0xA000 0x100>; // 有两个寄存器块，地址位于 0x800，长度为 16 byte，和位于 0xA000，256 字节
    };
};
```

~~不过，从实际文件中来看，reg 的用法更像是 `reg=<0x00 addr 0x00 size>` 如下所示~~🤦‍♂️，比如 I2C 设备的 reg 用来表示设备地址。

```c
	memory@80000000 {
		device_type = "memory";
		reg = <0x00 CVIMMAP_KERNEL_MEMORY_ADDR 0x00 CVIMMAP_KERNEL_MEMORY_SIZE>;
	};


	fast_image {
		compatible = "cvitek,rtos_image";
		reg-names = "rtos_region";
		reg = <0x0 CVIMMAP_FREERTOS_ADDR 0x0 CVIMMAP_FREERTOS_SIZE>;
		ion-size = <CVIMMAP_FREERTOS_RESERVED_ION_SIZE>;	//reserved ion size for freertos
	};
```

其实这里就是上面 `#address-cells` 和 `#size-cells` 属性的作用，因为在 `cv181x_base.dtsi` 中，根节点有设置这两个属性为2，那么就需要用2个32bit来表示。

```c
/ {
	compatible = "cvitek,cv181x";

	#size-cells = <0x2>;
	#address-cells = <0x2>;

	top_misc:top_misc_ctrl@3000000 {
		compatible = "syscon";
		reg = <0x0 0x03000000 0x0 0x8000>;
	};
...
```

### **name 属性**

过时了，建议不用。它的值是字符串，用来表示节点的名字。在跟 `platform_driver` 匹配时，优先级最低。`compatible` 属性在匹配过程中，优先级最高。

### **device_type 属性**

过时了，建议不用。它的值是字符串，用来表示节点的类型。在跟 `platform_driver` 匹配时，优先级为中。`compatible` 属性在匹配过程中，优先级最高。

### **phandle 属性**

全局唯一的ID，用来引用设备。但一般也不用，直接用 `&label` 来引用设备。

`phandle` 属性为 `devicetree` 中唯一的节点指定一个数字标识符，节点中的 `phandle` 性，它的取值必须是唯一的(不要跟其他的 `phandle` 值一样)，例如:

### gpio 属性

```
xxx-gpios =
sda-gpios = <&portb 24 GPIO_ACTIVE_HIGH>;
```

解析后，内核中提取时只要 `-` 前面的值。这里代表 `XGPIOB[24]` 这组脚。后面的 `GPIO_ACTIVE_HIGH` 表示高电平使能。某些`spi` 设备通信时，片选信号线 `cs` 是低电平使能，这里就要改为 `GPIO_ACTIVE_LOW`，这样就不需要去修改代码中的逻辑。

可以理解为内核驱动中将GPIO设置为1，只是将其使能，具体是高电平还是低电平，还要看这个属性。

```c
sda_gpiod = devm_gpiod_get(&pdev->dev, "sda", GPIOD_IN);
```

## 常用的节点

### 根节点

用 `/` 标识根节点，如：

```c
/dts-v1/;
/ {
    model = "SMDK24440";
    compatible = "samsung,smdk2440";

    # address-cells = <1>;
    # size-cells = <1>;
};
```

### CPU 节点

一般不需要我们设置，在 `dtsi` 文件中都定义好了，如：

```c
cpus {
    # address-cells = <1>;
    # size-cells = <0>;

    cpu0: cpu@0 {
     .......
    }
};
```

### memory 节点

芯片厂家不可能事先确定你的板子使用多大的内存，所以 `memory` 节点需要板厂设置，比如：

```c
memory {
    reg = <0x80000000 0x20000000>;
};
```

### chosen 节点

chosen子节点不代表实际硬件，它主要用于给内核传递参数。 这里只设置了“stdout-path =”serial0:115200n8”;”一条属性，表示系统标准输出stdout使用串口serial0。 此外这个节点还用作uboot向linux内核传递配置参数的“通道”， 我们在Uboot中设置的参数就是通过这个节点传递到内核的， 这部分内容是uboot和内核自动完成的，作为初学者我们不必深究。

我们可以通过设备树文件给内核传入一些参数，这要在 `chosen` 节点中设置 `bootargs` 属性：

```c
chosen {
    bootargs = "noinitrd root=/dev/mtdblock4 rw init=/linuxrc console=ttySAC0,115200";
};
// 或者
chosen {
    stdout-path = "serial0";
};
```

### aliases 节点

就是给某些节点设置别名，比如：

```c
aliases {
    i2c0 = &i2c0;
    i2c1 = &i2c1;
    i2c2 = &i2c2;
    i2c3 = &i2c3;
    i2c4 = &i2c4;
    serial0 = &uart0;
    serial1 = &uart1;
    serial2 = &uart2;
    serial3 = &uart3;
    serial4 = &uart4;
    ethernet0 = &ethernet0;
};
```

## 如何获取设备树节点信息

> 参考：[8. Linux设备树](https://doc.embedfire.com/linux/stm32mp1/driver/zh/latest/linux_driver/base_driver_tree.html#id9)

内核提供了一组函数用于从设备节点获取资源（设备节点中定义的属性）的函数，这些函数以of_开头，称为OF操作函数。 常用的OF函数介绍看 [8. Linux设备树](https://doc.embedfire.com/linux/stm32mp1/driver/zh/latest/linux_driver/base_driver_tree.html#id9)

### 提取属性值的of函数

https://doc.embedfire.com/linux/stm32mp1/driver/zh/latest/linux_driver/base_driver_tree.html#of

### 内存映射相关of函数

https://doc.embedfire.com/linux/stm32mp1/driver/zh/latest/linux_driver/base_driver_tree.html#id20

## 参考资料

- [devicetree.org  的标准文档](https://www.devicetree.org/)，英文的，篇幅不长。
- [8. Linux设备树](https://doc.embedfire.com/linux/stm32mp1/driver/zh/latest/linux_driver/base_driver_tree.html#id9) 野火相关教程。
- [微信-整理了一份 Linux 设备树基础知识，建议收藏！](https://mp.weixin.qq.com/s?__biz=MzUxMjEyNDgyNw==&mid=2247507125&idx=1&sn=6100723c3d3d012e40f4706ed16fd2d1&chksm=f96bae41ce1c2757c842713f7828b01b7fa8914b23b5e3d2dfba7aa064d0e822eca24645d1c0&from=industrynews&version=4.1.7.6018&platform=win&poc_token=HI3X0WSjwJWFv7kC545w1yAqOg0r4p8YYUUMe1UV)
- [Device Tree 初探](https://tinylab.org/linux-dts-1/)
- [elinux.org/Device_Tree_Usage](https://elinux.org/Device_Tree_Usage)
- [Linux 设备树解析](http://hulc.xyz/2021/11/11/linux%E8%AE%BE%E5%A4%87%E6%A0%91%E8%A7%A3%E6%9E%90/)
- [Linux 设备树 device_node 转换成 platform_device](http://hulc.xyz/2021/11/16/linux%E8%AE%BE%E5%A4%87%E6%A0%91device-node%E8%BD%AC%E6%8D%A2%E6%88%90platform-device/)
- https://bbs.huaweicloud.com/blogs/411120

---

## 💡编译、反编译

要将设备树二进制文件（**DTB**，Device Tree Blob）反编译为设备树源文件（**DTS**，Device Tree Source），你可以使用 Linux 中的 `dtc`（Device Tree Compiler）工具。以下是具体的步骤：

1. **安装 `dtc` 工具**（如果尚未安装）：
   - 在基于 Debian 的系统（如 Ubuntu）中，可以通过以下命令安装：
     ```bash
     sudo apt-get install device-tree-compiler
     ```

2. **反编译 DTB 文件**：
   使用 `dtc` 命令将 DTB 文件反编译为 DTS 文件。假设你的 DTB 文件名为 `example.dtb`，你可以运行以下命令：
   ```bash
   dtc -I dtb -O dts -o example.dts example.dtb
   ```

   - `-I dtb`：指定输入格式为 DTB。
   - `-O dts`：指定输出格式为 DTS。
   - `-o example.dts`：输出的 DTS 文件名。
   - `example.dtb`：要反编译的 DTB 文件。

3. **查看 DTS 文件**：
   反编译完成后，你可以使用文本编辑器查看生成的 `.dts` 文件。
