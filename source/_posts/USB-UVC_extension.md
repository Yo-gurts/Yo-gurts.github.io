---
title: UVC 扩展单元
top_img: transparent
date: 2024-05-15 22:26:44
updated: 2024-05-17 14:40:44
tags:
  - USB
  - UVC
categories: USB
keywords:
description:
---

## 常见问题

- [x] UVC 扩展单元的作用是啥？

    >《UVC 1.5 Class specification》 的 `2.7, 3.7.2` 中介绍了 UVC 的控制接口包含：输入终端、输出终端、相机终端、选择器单元、处理单元、编码单元、扩展单元。前几个都是类标准中有指定应该实现什么功能的，扩展单元就是专门给厂商实现自定义的一些功能。
    >
    >比如我们这里客户需要能给设备发送重启指令。

- [x] 和 UVC 的关系是啥？

    > UVC 扩展单元是 UVC 的一个可选的小功能。

- [x] 如何实现与测试？

    > 测试可使用：
    >
    > - `Bus Hound`，可给设备发送任意命令。分免费版和付费版，我们这里只用来发指令，免费版就够用了。官方网址：https://www.perisoft.net/bushound/index.htm
    > - [UVCXU摄像头扩展单元调试工具UVCXU-USB中文网官方版](https://www.usbzh.com/article/detail-761.html)，需要加QQ群下载。
    > - `wireshark` 抓包分析。
    >
    > > `BusHound` 是直接转到驱动去发的，没有通过 `windows` 视频类驱动，而UVCXU调试工具时调用了  `windows` 视频类驱动，就会受到 windows 的一些限制。见：https://www.usbzh.com/article/detail-538.html

## UVC 扩展单元描述符

扩展单元描述符允许硬件设计者定义任意一组控制接口，使类驱动程序可以在设备与供应商提供的主机软件之间进行通讯控制。

```C
UINT8 bLength;
UINT8 bDescriptorType;
UINT8 bDescriptorSubtype;
UINT8 bUnitID;
UINT8 guidExtensionCode[16];
UINT8 bNumControls;
UINT8 bNrInPins;
UINT8 baSourceID[bNrInPins];
UINT8 bControlSize;
UINT8 bmControls[bControlSize];
UINT8 iExtension;
```

---

- `bLength`：描述符的长度，`24+p(bNrInPins)+n(bControlSize)`，就是标记描述符这个结构体的大小。

---

- `bDescriptorType`：描述符类型，`CS_INTERFACE`，值为 `0x24`。

- `bDescriptorSubtype`：描述符子类型。 `VC_EXTENSION_UNIT`，值为 `0x06`。

    > 这两个值都是按照标准来的，指定描述符的类型，必须按照标准填固定值。

---

- `bUnitID`：唯一标识描述符。同一视频功能内的任何其他单元或终端不得具有相同的功能 `ID`。非零值。

    > 上面提到的输入终端、输出终端、相机终端、选择器单元、处理单元、编码单元、扩展单元，每个终端 `terminal` 或者单元 `unit` 都会有一个唯一的标识符，`bTerminalID` 或者 `bUnitID`，收到数据时会依据这个值，交给对应的模块处理。

---

- `guidExtensionCode`：供应商扩展单元编码

    > 这个值似乎没啥用处，应该只是填表示供应商的信息。

---

- `bNumControls`：该扩展单元的控制数量。这个控制是指控制请求，对应扩展单元的[特定类请求](https://www.usbzh.com/article/detail-172.html)的选择子的数量。

- `bControlSize`：`bmControls` 的大小。

- `bmControls[bControlSize]`：扩商指定自定义支持的 `bControlSize * 8` 个控制。

    > 首先，`bmControls` 是个位图，每一位代表一个功能。可以看[看 `linux` 内核中的定义](https://elixir.bootlin.com/linux/v5.10/source/include/uapi/linux/usb/video.h#L88)。
    >
    > ```c
    > /* A.9.4. Camera Terminal Control Selectors */
    > #define UVC_CT_CONTROL_UNDEFINED			0x00
    > #define UVC_CT_SCANNING_MODE_CONTROL			0x01
    > #define UVC_CT_AE_MODE_CONTROL				0x02
    > #define UVC_CT_AE_PRIORITY_CONTROL			0x03
    > #define UVC_CT_EXPOSURE_TIME_ABSOLUTE_CONTROL		0x04
    > #define UVC_CT_EXPOSURE_TIME_RELATIVE_CONTROL		0x05
    > #define UVC_CT_FOCUS_ABSOLUTE_CONTROL			0x06
    > #define UVC_CT_FOCUS_RELATIVE_CONTROL			0x07
    > ```
    >
    > 这些宏的值就是选择子。选择子与 `bmControls` 这个数组的对应关系如下。
    >
    > | BIT   | 选择子 |
    > | ----- | ------ |
    > | BIT0  | 1      |
    > | BIT1  | 2      |
    > | BIT2  | 3      |
    > | …     | …      |
    > | BIT31 | 32     |
    >
    > `bmControls[bControlSize]` 的值中，对应 `bit` 为1，就表示支持该功能。比如我只支持`UVC_CT_SCANNING_MODE_CONTROL` 这一个功能，那么
    >
    > ```c
    > bNumControls = 1;
    > bControlSize = 1;
    > bmControls[0] = cpu_to_le16(0x1);
    > ```
    >
    > `bNumControls` 就是 `bmControls` 中1的个数。不过 `host/PC` 端驱动不一定会用这个值，windows 好像就直接自己判断的 `bmControls` 中1的个数。
    >
    > 在上面的 Camera Terminal Control Selectors 的定义中，每一位对应的功能都是 UVC 标准中制定的。而扩展描述符就是留给厂商自由发挥的。
    >
    > **选择子**：**就是对应功能的 ID**。给设备发消息时需要指定选择子。

---

- `bNrInPins`：输入管脚数 p

- `baSourceID`：各个输入管脚连接的实体或端点ID(从第一个到最后一个)

    > 不知道这两个有啥用。

---

- `iExtension`：扩展单元的[字符串描述符](https://www.usbzh.com/article/detail-53.html)索引。用来提供一些额外的信息，用不着写0就行。

    > 为了提供比较友好的设备标识，USB规范中定义了字符串描述符，即使用人类的自然语言来描述设备的功能，生产厂家，生产序列号等。这里的自然语言可以为英文，也可以为中文，也可以为其它的自然语言。
    >
    > 为了解决字符串的跨国性，USB字符串使用UNICODE编码。

描述符配置对了，基本就成功了一大半。

## Unit and Terminal Control Requests

大概猜测了下，这些请求是把每个 `control_selector` 当成一个数值属性来看，比如以灯光亮度来理解。

- `GET_LEN`: 表示 `SET_CUR` 时传输的数据长度（字节）。比如亮度的范围在 0~255 以类的话，一个字节就可以表示。而范围如果超过 255，那一个字节显然不够用。**注意：标准中要求 `GET_LEN` 的返回值为2字节的**。
- `GET_INFO`: 用于获取设备支持的特定请求。一般就使能 `D0, D1` 就行，返回的值各 `bit` 代表的含义见下表：

  | 位     | 描述                                                   | 位状态     |
    | ------ | ------------------------------------------------------ | ---------- |
    | D0     | 1=Supports GET value requests                          | Capability |
    | D1     | 1=Supports SET value requests                          | Capability |
    | D2     | 1=Disabled due to automatic mode(under device control) | State      |
    | D3     | 1=Autoupdate Control                                   | Capability |
    | D4     | 1=Asynchronous Control                                 | Capability |
    | D5     | 1=Disabled due to incompatibility with Commit state.   | State      |
    | D7..D6 | 保留，置为0                                            | —          |

  如果[编码单元](https://www.usbzh.com/article/detail-82.html)控制的实现使得设备可以启动该控件的最小和/或最大设置属性的更改，那么该设备应该能够发送控制更改中断来通知主机新的 `GET_MIN` 和/或 `GET_MAX` 设置，因此必须设置 `D3`（自动更新控制）。

---

- `GET_MIN`: 支持的亮度最小值。比如 0

- `GET_MAX`: 支持的亮度最大值。比如 100

- `GET_RES(resolution)`: 获取调整的精度。比如 10，这样的话，调用 `SET_CUR` 时，就只能设置 10 的整数倍。

    这三个值存在关联的，标准中似乎有要求 `min/res`，`max/res`，`cur/res` 都是整数，不能是小数。**如果属性是字符串的话，那这里 `RES` 就得设置为**`1`。

---

- `GET_DEF(default)`: 默认值，设备枚举完成之后，会立马调用该请求获取默认值。
- `SET_CUR`: 设置亮度为特定值。
- `GET_CUR`: 获取当前的亮度值。

## 参考资料
