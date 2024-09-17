---
title: Alios USB camera 常见需求处理
top_img: transparent
date: 2024-06-04 11:19:42
updated: 2024-06-04 11:19:42
tags:
  - Alios
  - USB
categories: Alios
keywords:
description:
---

[TOC]

## 配置说明

### 组件配置

每个组件都有一个 `.yaml` 配置文件，主要关注：

```yaml
def_config: # 组件的可配置项
  CONFIG_XXX: 1
```

`def_config` 下的配置都会以宏定义的形式来影响代码。就相当于在编译时多了 `-DCONFIG_XXX=1` 这样。

### Solution 配置

Solution 也有一个 `yaml` 的配置文件，也包含 `depends`、`def_config` 等配置项，这里单独来讲主要是想强调 `def_config` 这个配置，Solution 下的配置文件中，可以修改任意组件的 `def_config` 。也就是说，每个组件下的配置文件中，`def_config` 下配置相当于是*默认配置*。如果 Solution 下的配置项与组件的下配置不一致，以 Solution 的为准。

**❗❗强调：Solution 下的配置会对所有组件生效，所以在定义配置的时候，注意配置名是否与其他组件重复了（可以重复，但要确认这是你期望的）。**

## 用户常用改动

- [x] 板卡内存大小
    ```yaml
    CONFIG_DRAM_CFG: "128m"
    ```

- [x] 配置 UVC 数量。有的客户是单目，有的客户是双目。

    ```yaml
    CONFIG_USBD_UVC_NUM: 2
    ```

- [x] 修改 PID, VID。（设备描述符里面）

- [x] 修改设备名称（视频控制接口描述符里面。双目场景一般一个rgb摄像头 + 一个红外摄像头，需要分别设置名字以区分）。

- [x] 修改序列号。（设备描述符里面）

- [x] 修改 PRODUCT ID（设备描述符里面）

- [x] 视频码率控制。（MJPEG/H264 H265）
    ```yaml
      CONFIG_UVC_MJPEG_BITRATE: 20480          # UVC 推流时，设置视频编码器 MJPEG 的码率，单位 kbps
      CONFIG_UVC_H264_H265_BITRATE: 4096       # UVC 推流时，设置视频编码器 H264/H265 的码率，单位 kbps
    ```

- [ ] 灯光亮度控制。（通过 PWM 调控）

- [x] 分辨率。（不同客户需要支持的分辨率不一样）

- [x] 支持高分辨率（vpss 仅一个通道支持2880x1620)

- [x] [USB camera 通用功能（亮度、色调、饱和度等调整）](#USBcamera通用功能)

    ```yaml
      CONFIG_UVC_COMM_FUNC: 0                  # USB camera 使能亮度、色调、对比度等调整功能
    ```

- [x] [设置裁剪以保证任意格式输出不会变形](#保持输出长宽比)

    ```yaml
    CONFIG_UVC_CROP_BEFORE_SCALE: 1
    ```



### 板卡内存大小

1811c，1810c 等内存大小不一样，需要更具客户使用的具体芯片来调整。

**配置选项**：

```yaml
CONFIG_DRAM_CFG: "128m"
```

**默认配置**：`64m`

**用户（特定项目）配置位置**：`solutions/usb_cam/customization/{项目xxx}/package.xxx.yaml`

### UVC 数量

**配置选项**：

```yaml
CONFIG_USBD_UVC_NUM: 1
```

**默认配置**：`1`

**组件配置位置（默认配置）**：`components/cvi_platform/package.yaml`

**用户（特定项目）配置位置**：`solutions/usb_cam/customization/{项目xxx}/package.xxx.yaml`

### PID/VID 修改

暂未定义宏。直接修改对应文件。

**配置位置**：`components/cvi_platform/protocol/usb_devices/usbd_composite/src/usbd_comp.c`

```c
#define CVITEK_VENDOR_ID        0x3346    /* cvitek vendor id */
#define CVITEK_PRODUCT_ID       0x0001    /* Webcam A/V gadget */
#define CVITEK_DEVICE_BCD       0x0001    /* 0.01 */

# 修改上述宏，在设备描述符中使用
static struct usb_device_descriptor comp_device_descriptor = {
    .bLength            = USB_DT_DEVICE_SIZE,
    .bDescriptorType    = USB_DT_DEVICE,
    .bcdUSB             = USB_2_0,
    .bDeviceClass       = USB_CLASS_MISC,
    .bDeviceSubClass    = 0x02,
    .bDeviceProtocol    = 0x01,
    .bMaxPacketSize0    = 0x40,
    .idVendor           = cpu_to_le16(CVITEK_VENDOR_ID),   # VID
    .idProduct          = cpu_to_le16(CVITEK_PRODUCT_ID),  # PID
    .bcdDevice          = cpu_to_le16(CVITEK_DEVICE_BCD),
    .iManufacturer      = USB_STRING_MFC_INDEX,
    .iProduct           = USB_STRING_PRODUCT_INDEX,
    .iSerialNumber      = USB_STRING_SERIAL_INDEX,
    .bNumConfigurations = 1,
};
```

### 修改序列号

序列号`iSerialNumber`是在设备描述中指定的（指定字符串描述符的ID）。要修改的话只需修改对应的字符串描述符。

**配置位置**：`components/cvi_platform/protocol/usb_devices/usbd_composite/src/usbd_comp.c`

```
static const struct usb_descriptor_header * const comp_string_descriptors[] = {
    (struct usb_descriptor_header *) &comp_string_descriptor_zero,
    (struct usb_descriptor_header *) &comp_string_descriptor_manufacturer,
    (struct usb_descriptor_header *) &dev_string_descriptor_product,
    (struct usb_descriptor_header *) &comp_string_descriptor_serial, // 序列号字符串描述符
    (struct usb_descriptor_header *) &comp_string_descriptor_zero,
    (struct usb_descriptor_header *) &uac_string_descriptor_audio,
    (struct usb_descriptor_header *) &uvc_string_descriptor_camera1_name,
    (struct usb_descriptor_header *) &uvc_string_descriptor_camera2_name,
    (struct usb_descriptor_header *) &uvc_string_descriptor_camera3_name,
    NULL,
};

// 修改 `comp_string_descriptor_serial` 即可。
// 由于 USB 协议规定序列号是 unicode 编码，所以要用 `cpu_to_le16` 转换。
// 并注意字符串的长度和声明的一致。
DECLARE_UVC_STRING_DESCRIPTOR(10);
static const struct UVC_STRING_DESCRIPTOR(10) comp_string_descriptor_serial = {
    .bLength            = UVC_STRING_DESCRIPTOR_SIZE(10),
    .bDescriptorType    = USB_DESCRIPTOR_TYPE_STRING,
    .wData              = {
        cpu_to_le16('2'),
        cpu_to_le16('0'),
        cpu_to_le16('2'),
        cpu_to_le16('4'),
        cpu_to_le16('0'),
        cpu_to_le16('1'),
        cpu_to_le16('1'),
        cpu_to_le16('9'),
        cpu_to_le16('0'),
        cpu_to_le16('0'),
    },
};
```

### 动态修改序列号✨

基本都会用到。

### 修改 PRODUCT ID

`product ID` 也是在设备描述中指定的`iProduct`（指定字符串描述符的ID）。要修改的话只需修改对应的字符串描述符。

**配置位置**：`components/cvi_platform/protocol/usb_devices/usbd_composite/src/usbd_comp.c`

```c
static const struct UVC_STRING_DESCRIPTOR(10) dev_string_descriptor_product = {
    .bLength            = UVC_STRING_DESCRIPTOR_SIZE(10),
    .bDescriptorType    = USB_DESCRIPTOR_TYPE_STRING,
    .wData              = {
        cpu_to_le16('C'),
        cpu_to_le16('V'),
        cpu_to_le16('I'),
        cpu_to_le16('T'),
        cpu_to_le16('E'),
        cpu_to_le16('K'),
        cpu_to_le16('-'),
        cpu_to_le16('D'),
        cpu_to_le16('E'),
        cpu_to_le16('V'),
    },
};
```

### 设备名称修改

设备名称也是通过字符串描述符传递。视频控制接口描述符中的 `iInterface` 指定了名称对应的字符串描述符的下标。

**配置位置**：`components/cvi_platform/protocol/usb_devices/usbd_composite/src/usbd_comp.c`

```c
static const struct usb_descriptor_header * const comp_string_descriptors[] = {
    (struct usb_descriptor_header *) &comp_string_descriptor_zero,
    (struct usb_descriptor_header *) &comp_string_descriptor_manufacturer,
    (struct usb_descriptor_header *) &dev_string_descriptor_product,
    (struct usb_descriptor_header *) &comp_string_descriptor_serial,
    (struct usb_descriptor_header *) &comp_string_descriptor_zero,
    (struct usb_descriptor_header *) &uac_string_descriptor_audio,
    (struct usb_descriptor_header *) &uvc_string_descriptor_camera1_name, // 3个设备名称描述符
    (struct usb_descriptor_header *) &uvc_string_descriptor_camera2_name,
    (struct usb_descriptor_header *) &uvc_string_descriptor_camera3_name,
    NULL,
};

/* UVC Camera max name length is 32  */
DECLARE_UVC_STRING_DESCRIPTOR(32);

static const struct UVC_STRING_DESCRIPTOR(32) uvc_string_descriptor_camera1_name = {
    .bLength            = UVC_STRING_DESCRIPTOR_SIZE(14), // name length is 14
    .bDescriptorType    = USB_DESCRIPTOR_TYPE_STRING,
    .wData              = {
        cpu_to_le16('C'),
        cpu_to_le16('V'),
        cpu_to_le16('I'),
        cpu_to_le16('T'),
        cpu_to_le16('E'),
        cpu_to_le16('K'),
        cpu_to_le16(' '),
        cpu_to_le16('C'),
        cpu_to_le16('A'),
        cpu_to_le16('M'),
        cpu_to_le16('E'),
        cpu_to_le16('R'),
        cpu_to_le16('A'),
        cpu_to_le16('1'),
    },
};
```

目前最大支持三个摄像头，所以这里预留了三个设备描述符。如需修改名称，修改对应的描述符即可。

❗❗**注意**：修改之后，Windows 需要在设备管理器中卸载设备，再插上设备才能看的修改后的名字。(Windows自身有缓存)

### 视频码率控制

目前 `H264/H265` 都是吃配置 `CONFIG_UVC_H264_H265_BITRATE`。

`MJPEG` 格式的码率则是通过 `CONFIG_UVC_MJPEG_BITRATE` 配置。

**配置选项**：

```yaml
  CONFIG_UVC_MJPEG_BITRATE: 20480          # UVC 推流时，设置视频编码器 MJPEG 的码率，单位 kbps
  CONFIG_UVC_H264_H265_BITRATE: 4096       # UVC 推流时，设置视频编码器 H264/H265 的码率，单位 kbps
```

**组件配置位置（默认配置）**：`components/cvi_platform/package.yaml`

**用户（特定项目）配置位置**：`solutions/usb_cam/customization/{项目xxx}/package.xxx.yaml`

### 视频格式设置

目前支持 `YUY2`，`H264`，`H265`，`MJPEG` 4 种格式出流。其中 H264, H265 有一个 DISABLE 配置选项，因为这两种编解码方式需要一个固件，H264 的固件大小约为253k，h265 约为 147k，某些场景下flash 大小不够需要严格控制固件大小，就可以考虑disable h264/h265。

不过目前release 版本中，codec 是以.a 静态库的形式提供的，默认是使能了的。所以该配置意义不大。

**配置选项**：

```yaml
  CONFIG_DISABLE_VENC_H264: 0              # 是否禁用 H264 编码
  CONFIG_DISABLE_VENC_H265: 0              # 是否禁用 H265 编码
```

**用户（特定项目）配置位置**：`solutions/usb_cam/customization/{项目xxx}/package.xxx.yaml`

### 分辨率设置

可为不同视频格式配置多种不同的分辨率，直接修改对于的描述数组。

**配置位置**：`components/cvi_platform/protocol/usb_devices/usbd_class/usbd_uvc/src/usbd_uvc.c`

```
static struct uvc_frame_info_st yuy2_frame_info[] = {
    {1, 800, 600, 15, 0},
    {2, 640, 360, 15, 0},
    {3, 400, 300, 15, 0},
    {5, 480, 320, 15, 0},
    {6, 480, 360, 15, 0},
    {7, 1280, 720, 15, 0},
};

static struct uvc_frame_info_st mjpeg_frame_info[] = {
	{1, 240, 320, 30, 0},
	{2, 320, 240, 30, 0},
	{3, 480, 320, 30, 0},
	{4, 640, 480, 30, 0},
	{5, 656, 468, 30, 0},
	{6, 720, 480, 30, 0},
	{7, 800, 480, 30, 0},
	{8, 864, 480, 30, 0},
	{9, 800, 600, 30, 0},
	{10, 1280, 720, 30, 0},
	{11, 1920, 1080, 30, 0},
	{12, 1600, 1200, 30, 0},
};

static struct uvc_frame_info_st h264_frame_info[] = {
    {1, 800, 600, 30, 0},
    {2, 1280, 720, 30, 0},
    {3, 640, 480, 30, 0},
    {4, 400, 300, 30, 0},
    {5, 1920, 1080, 30, 0},
};
```

格式为：`{序号，宽，高，帧率，是否旋转}`。

### 支持高分辨率

视频流处理过程中，会用到一些中间变量来存放图像数据，目前该变量的大小是预先确定的，即 `DEFAULT_FRAME_SIZE`。默认分配的大小是1920x1080x1.5。

在大多数情况下，MJEPG/H264/H265 等需要存放的都是编码后的数据，即使是 2560x1440 经过编码后也完全能存放下，但我们还支持未压缩的 YUY2 格式，若需要输出 1920x1080 的 YUY2 格式，则上述空间就不够用了。需要设置为 1920x1080x2，另外，处理过程中还需要再原始图像数据的基础上，每1012字节加上12字节的header，所以还需额外分配 些空间。目前建议设置  1920x1120x2

**配置位置**：`components/cvi_platform/protocol/usb_devices/usbd_class/usbd_uvc/src/usbd_uvc.c`

```c
#define WIDTH  (unsigned int)(1920)
#define HEIGHT (unsigned int)(1080)

#define CAM_FPS        (30)
#define INTERVAL       (unsigned long)(10000000 / CAM_FPS)
#define MAX_FRAME_SIZE (unsigned long)(WIDTH * HEIGHT * 2)
#define DEFAULT_FRAME_SIZE (unsigned long)(WIDTH * HEIGHT * 3 / 2)
```

修改为

```c
#define WIDTH  (unsigned int)(1920)
#define HEIGHT (unsigned int)(1120)

#define CAM_FPS        (30)
#define INTERVAL       (unsigned long)(10000000 / CAM_FPS)
#define MAX_FRAME_SIZE (unsigned long)(WIDTH * HEIGHT * 2)
#define DEFAULT_FRAME_SIZE (unsigned long)(WIDTH * HEIGHT * 2)
```

### 支持 2880x1440 等高分辨率

参考 《Cvitek Debug SOP》

这个问题是由vpss各scaler硬件性能差异导致的，这种情况下需要使用最强的scaler-1进行所需图像处理。具体的，对于single mode，需要设置vpss chn的chn号为1，**对于dual mode，需要设置 grp 属性中的u8VpssDev为1且设置vpss chn的chn号为0**。

**推荐解决办法：即使单个设备也使用dual mode**。

备选方案：使用 single mode 并设置通道为1的话，目前还会启用通道0，会占用额外的VB，也可以合入这笔代码 https://gerrit-ai.sophgo.vip:8443/#/c/114453/，不启用chn0。这样还需修改uvc 使用的通道号为1.  VPSS 和venc的绑定关系在：`components/cvi_platform/protocol/usb_devices/usbd_class/usbd_uvc/src/usbd_uvc.c` 中`uvc`这个全局变量下配置。

```
static struct uvc_device_info uvc[USBD_UVC_MAX_NUM] = {
  {
    // .ep = 0x81,
    .format_info = uvc_format_info,
    .formats = ARRAY_SIZE(uvc_format_info),
    .video = {0, 0, 1}, // {venc chn, vpss grp, vpss chn}
  },
```

### USBcamera通用功能

> 其他项目也会用到 usbcamera，但并不需要调整色调等功能，所以目前 SDK 默认并未启用这些功能。
>
> 如需启用，需要通过配置 `CONFIG_UVC_COMM_FUNC:1`，则将目前支持的功能都启用。

**配置选项**：

```yaml
  CONFIG_UVC_COMM_FUNC: 0                  # USB camera 使能亮度、色调、对比度等调整功能
```

**默认配置**：`0` 未启用。

**组件配置位置（默认配置）**：`components/cvi_platform/package.yaml`

**用户（特定项目）配置位置**：`solutions/usb_cam/customization/{项目xxx}/package.xxx.yaml`

**注意**：由于有多目 sensor 的场景，在特定摄像头下进行配置的时候，会区分设备的。设备需要与对应的 VPSS GRP, VI_PIPE 绑定，必须满足以下关系：

SENSOR0 -> VI_PIPE0 -> GRP0

SENSOR1 -> VI_PIPE1 -> GRP1

否则设置无法生效。

**VI 绑定 VPSS** 是在 `solutions/usb_cam/customization/{xxx}/param/custom_vpssparam.c` 中，Group 配置中有如下内容：

```c
        .bBindMode = CVI_TRUE,
        .astChn[0] = {
            .enModId = CVI_ID_VI,
            .s32DevId = 0,
            .s32ChnId = 1,  // VI CHN ID
        },
        .astChn[1] = {
            .enModId = CVI_ID_VPSS,
            .s32DevId = 1,  // group ID
            .s32ChnId = 0,
        },
```

目前支持的功能有：

- [ ] PROCESSING UNIT
  - [x] 亮度
  - [x] 对比度
  - [x] 色调
  - [x] 饱和度
  - [x] 锐度
- [ ] INPUT TERMINAL(CAMERA TERMINAL)
  - [x] PANTILE上下左右移动(绝对)
- [ ] EXTENSION UNIT
  - [x] REBOOT（发送 reboot 会重启，无插拔烧录时用）

### 保持输出长宽比

当sensor的输入图像为 16:9 时，若输出的分辨率为 4:3 ，会看到输出的图像明显发生畸变 / 或者说有黑边（**取决于 vpss 通道的 ASPECT_RATIO_S 参数设置**）。可以使能该配置 `CONFIG_UVC_CROP_BEFORE_SCALE:1`，在进行缩放前先进行裁剪，使裁剪后的大小与输出的长宽比一致，并且尽可能提供更大的视野。这样经过缩放就不会发生畸变。

**配置选项**：

```yaml
CONFIG_UVC_CROP_BEFORE_SCALE: 1
```

**默认配置**：`1`

**组件配置位置（默认配置）**：`components/cvi_platform/package.yaml`

**用户（特定项目）配置位置**：`solutions/usb_cam/customization/{项目xxx}/package.xxx.yaml`

### 保持输出长宽比2--是否插黑边

VPSS 在缩放处理时，可以指定参数是否保存长宽比。

如 1280x720的屏幕，如果输出的画面是 800x400，

- AUTO 保持比例的话就会拉伸为960x720，左右两边就会有黑边。
- NONE 不进行处理，原始大小输出。（画面可能会变形，比如16:9的输入，输出为4:3时）
- MANUAL 手动指定显示的位置，以及显示画面的宽、高。

```c
typedef struct _ASPECT_RATIO_S {
    ASPECT_RATIO_E enMode;      // 是否固定长宽比：NONE, AUTO, MANUAL
    CVI_BOOL bEnableBgColor;    // 是否要让画面以外以背景色覆盖。
    CVI_U32 u32BgColor;         // 填充背景颜色，RGB 888，可通过宏 RGB_8BIT(0, 0, 0xFF) 来指定。bit[7:0]為 B，bit[15:8]為 G，bit[23:16]為 R
    RECT_S stVideoRect;         // 在手动模式时才生效，这个 RECT
} ASPECT_RATIO_S;

typedef enum _ASPECT_RATIO_E {
    ASPECT_RATIO_NONE = 0,      // 无动作，满屏
    ASPECT_RATIO_AUTO,          // 视频保持比例，自动计算视频区域
    ASPECT_RATIO_MANUAL,        // 手动决定视频区域
    ASPECT_RATIO_MAX
} ASPECT_RATIO_E;

typedef struct _RECT_S {
    CVI_S32 s32X;
    CVI_S32 s32Y;
    CVI_U32 u32Width;
    CVI_U32 u32Height;
} RECT_S;
```

**用户（特定项目）配置位置**：`solutions/usb_cam/customization/{项目xxx}/param/custom_vpssparam.c` 为每个 vpss 通道单独配置。

```c
 PARAM_CLASSDEFINE(PARAM_VPSS_CHN_CFG_S, CHNCFG, GRP0, CHN) [] =
 {
    {.u8Rotation = ROTATION_0,
     .stVpssChnAttr = {
             .u32Width = 2880,
             .u32Height = 1620,
             .enVideoFormat = VIDEO_FORMAT_LINEAR,
             .enPixelFormat = PIXEL_FORMAT_NV21,
             .stFrameRate.s32SrcFrameRate = -1,
             .stFrameRate.s32DstFrameRate = -1,
             .bFlip = CVI_FALSE,
             .bMirror = CVI_FALSE,
             .u32Depth = 0,
             .stAspectRatio.enMode = ASPECT_RATIO_AUTO, // 保持比例，会有黑边，未配置的话就是拉伸
             .stAspectRatio.bEnableBgColor = CVI_TRUE,
             .stAspectRatio.u32BgColor    = COLOR_RGB_BLACK,
             .stNormalize.bEnable = CVI_FALSE,
         }
    },
```

### 启用 pqtool 支持

❗❗**注意编译时要指定正确的 `CHIP_ARCH` ，否则会导致 pqtool 工具使用时报错**。通过 `export CHIP_ARCH=cv181x; make usb_cam PROJECTS="XXXX"`  编译。

```yaml
  CONFIG_PQTOOL_SUPPORT: 1
  CONFIG_USBD_CDC_RNDIS: 1
  CONFIG_USBD_UVC: 1
  CONFIG_USB_DWC2_DMA_ENABLE: 0
  CONFIG_APP_RTSP_SUPPORT: 0
```

启用 pqtool_support 以及 rndis 网络才能让pc端工具连上。使用 UVC 出流效果比 rtsp 更好。
DMA 与 rndis 不兼容，所以要关闭。不过目前不开 DMA 会有概率屏闪，所以仅推荐在调试时关闭DMA。
RTSP 与 UVC 不兼容，所以要关闭。

**目前 RNDIS  有概率出错，发现连不上的话在设备管理器中，禁用 rndis 对应的设备再启用。**

板端默认的 IP 地址是 `192.168.11.10`，PC端对应的 rndis 网卡的 IP 也需要手动设置到相同网段，才能使用 Pqtool 工具进行连接。

### 快启说明

为了能使用USB烧录，在rom阶段会等待1s检测usb连接，还有uart烧录的引脚被占用，也会等待1s。

- 快启1：通过写 efuse 跳过 rom 阶段的烧录检测。
  - 配置：`CONFIG_ENABLE_FASTBOOT=1`，启用后，app_main 中会调用 `CVI_EFUSE_EnableFastBoot` 去烧写 efuse 使能快启。
  - 收益：1s整或2s整。

- 快启2：boot0 阶段通过拉高 SPI 的速率加速从flash load固件的过程。来提升启动速度。
  - 配置：`CONFIG_QUICK_STARTUP_SUPPORT`。启用后 boot0 中会拉高 spi 的时钟。
  - 收益：看yoc.bin，或者是否需要从flash加载模型吧，收益不大，可能就几十ms。

### AE 快速收敛

若 sensor 的自动曝光调整较慢，在刚打开sensor时，可能会看到明显的画面亮度调整的过程。可以调整pq参数，调大自动曝光的调整速率。

### 摄像头开启关闭时进行额外处理

目前是为每个摄像头建立了一个处理线程，完成 UVC 推流。该线程中每一次调用`vedio_streaming_send`发送一帧，通过检查stream_on的变化情况，确定当前的状态。stream_on 由 0->1 时表示打开了设备。反之关闭摄像头。

```c
static void *send_to_uvc(void *arg)
{
	uint32_t i = 0;
	int64_t dev_index = (int64_t)arg;
	bool last_stream_on = false;
	bool cur_stream_on = false;

	while (av_session_init_flag) {
		cur_stream_on = uvc[dev_index].streaming_on;
		if (cur_stream_on != last_stream_on) {
			last_stream_on = cur_stream_on;
			if (last_stream_on) { // 开始推流
				// led gpio up, restart sensor, etc.
			} else { // 停止推流
				// led gpio down, standby sensor, etc.
			}
		}

		if(cur_stream_on) {
			vedio_streaming_send(&uvc[dev_index], dev_index);
		} else {
			aos_msleep(1);
		}
	}

	return 0;
}
```

#### 灯光控制

推流时开灯，停止推流时关灯。简单的开关可直接在上面的流程中实现，有些场景可考虑使用UVC 处理单元中的 backlight， 通过pwm去控制灯光。

#### standby sensor 降低功耗

在不出流的时候（待机状态），将 sensor standby，以降低整机功耗。standby 一般有两中方式：

- 软件standby：通过I2C发送指令，让sensor进入standby模式（仍会有较低的功耗，以确保能接受I2C的唤醒指令）
- 硬件standby：拉低sensor的powerdown引脚，进行standby。功耗比软件更低。

### 超频

clkstat 命令可以看时钟。



### 带宽控制

### 测试

```powershell
# 初始化计数器
$counter = 0

while ($true) {
    # 增加计数器
    $counter++

    # 输出当前次数
    Write-Output "当前次数: $counter"

    # 启动名为 'xxx.exe' 的程序
    $process = Start-Process "C:\Program Files (x86)\Noël Danjou\AMCap\amcap.exe" -PassThru

    # 等待3秒
    Start-Sleep -Seconds 3

    # 杀死进程
    Stop-Process -Id $process.Id -Force

    # 等待1秒
    Start-Sleep -Seconds 1
}
```
