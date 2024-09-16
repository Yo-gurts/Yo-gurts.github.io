---
title: Cvitek-安全启动
date: 2023-08-03 17:55:02
updated: 2024-05-12 00:13:02
tags:
  - Cvitek
  - 启动流程
  - sha256
  - rsa
  - aes
  - uboot
  - openssl
categories: Cvitek
keywords:
description:
top_img: transparent
---

## Cvitek-安全启动

> 注：仅考虑 `cv181x`

安全启动的意义：

- 防止用户烧录未经授权的固件。（签名）
- 防止通过固件拷贝，来抄袭产品。（签名+加密）

## 原理介绍

### 加密算法介绍

> - [盘点五种最常见加密算法！](https://mp.weixin.qq.com/s/XBYJFZ2jRF2slrlym3svQw)
> - [密码学基本概念](https://mp.weixin.qq.com/s/Hu1VBMSHh82n6bujWxr9nw)

算法整体上可以分为**~~不可逆加密~~**，以及**可逆加密**，可逆加密又可以分为**对称加密**和**非对称加密**。

> 不可逆这部分称为 **消息摘要** 更好，**摘要算法**就是我们常说的散列函数、哈希函数（Hash Function），它能够把任意长度的数据“压缩”成固定长度、而且独一无二的“摘要”字符串，就好像是给这段数据生成了一个数字“指纹”。

![图片](../images/Cvitek-%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8/640.png)

#### 消息摘要

散列算法，就是一种不可逆算法，无法从散列结果中反推出明文。**可以用来检验数据是否被更改**。

散列算法中，明文通过散列算法生成散列值，**散列值是长度固定的数据，和明文长度无关**。

![图片](../images/Cvitek-%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8/640-1694590307639-3.png)

散列算法的具体实现有很多种，常见的包括 `MD5`、`SHA1`、`SHA-224`、`SHA-256` 等等。

散列算法常用于**数字签名**、消息认证、密码存储等场景。

> 本文中，对 `kernel`、`rootfs` 的签名就是基于 `sha256` 实现（不过还用到了下面的 `RSA`）。

| 名称   | 介绍：都是用于将任意长度的数据映射为固定长度的散列值，只是映射长度和实现算法不同                                                                           |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `MD5`  | `MD5`（Message-Digest Algorithm 5），`MD5` 算法的输出长度为 `128` 位，通常用 `32` 个 `16` 进制数表示。                                                     |
| `SHA1` | `SHA-1` 系列存在缺陷，已经不再被推荐使用                                                                                                                   |
| `SHA2` | `SHA-2` 算法包括`SHA-224`、`SHA-256`、`SHA-384`和`SHA-512`四种散列函数，<br />分别将任意长度的数据映射为 `224` 位、`256` 位、`384` 位和 `512` 位的散列值。 |

#### 对称加密 -- AES 算法

对称加密算法，使用同一个密钥进行加密和解密。

![图片](../images/Cvitek-%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8/640-1694591480328-6.png)

加密和解密过程使用的是相同的密钥，因此密钥的安全性至关重要。如果密钥泄露，攻击者可以轻易地破解加密数据。

常见的对称加密算法包括 `DES`、`3DES`、`AES` 等。其中，**`AES` 算法是目前使用最广泛的对称加密算法之一，具有比较高的安全性和加密效率**。**`AES` 算法使用的密钥长度为 `128` 位、`192` 位或 `256` 位**（这里的位是 `bit`）

> 我们使用的 `AES` 秘钥是 128 位。

#### 非对称加密 -- RSA 算法

非对称加密算法需要两个密钥，这两个密钥互不相同，但是相互匹配，一个称为**公钥**，另一个称为**私钥**。

**使用其中的一个加密，则使用另一个进行解密**。例如使用公钥加密，则需要使用私钥解密。

![图片](../images/Cvitek-%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8/640-1694591564320-9.png)

`RSA` 算法的**优点是安全性高**，公钥可以公开，私钥必须保密，保证了数据的安全性；可用于数字签名、密钥协商等多种应用场景。

**缺点是加密、解密速度较慢**，密钥长度越长，加密、解密时间越长；密钥长度过短容易被暴力破解，密钥长度过长则会增加计算量和存储空间的开销。

`RSA` 的秘钥（包括公钥和私钥）由两部分组成：模数和指数

- **公钥**：包括模数 `N` 和**公钥指数** `E`。公钥用于**验证数字签名**。

- **私钥**：包括模数 `N` 和**私钥指数** `D`。私钥用于**生成数字签名**。

  > 具体来说，数字签名过程通常涉及以下步骤：
  >
  > 1. **使用哈希函数对要签名的数据进行哈希处理，生成消息摘要。**
  > 2. **使用私钥对消息摘要进行加密，生成数字签名。**
  > 3. 将原始数据、数字签名和公钥发送给接收方。
  >
  > 验证数字签名的过程如下：
  >
  > 1. **使用公钥对数字签名进行解密，得到消息摘要。**
  > 2. **使用相同的哈希函数对原始数据进行哈希处理，生成另一个消息摘要。**
  > 3. 比较这两个消息摘要，如果相同，则表示签名有效，否则表示签名无效。
  >
  > 因此，公钥通常用于验证签名的有效性，而不是用于生成数字签名。

在一个标准的 `RSA` 密钥对中，公钥的 `E`和 `N` 通常也包含在私钥中，**模数 `N` 是 `RSA` 密钥对的关键部分，它在公钥和私钥中都是相同的**。`E` 和 `D` 是指数，它们通常不同，因为它们在不同的数学运算中使用。

- `E` 通常被设置为常数值 65537（0x10001），因为这个值在二进制中有很多 1，使得加密操作更加高效。
- `N` 是一个大整数，它是两个大质数的乘积，**它决定了 `RSA` 密钥的长度和强度**。

在 RSA 算法中，公钥和私钥都包含模数 `N`，而 `N` 是两个大质数 `p` 和 `q` 的乘积。公钥还包含一个指数 `e`，而私钥包含另一个指数 `d`。这两个指数 `e` 和 `d` 的选择要满足一个特定的条件，也就是满足 `e*d ≡ 1 (mod φ(N))` 的关系。在这里，`φ(N)` 是 Euler's totient 函数，它对于 `N=p*q`，值为`φ(N)=(p-1)(q-1)`。

也就是说，虽然两对密钥（公钥和私钥）有不同的指数 `e` 和 d，但 `e` 和 `d` 是满足相应数学关系的。使得用公钥进行加密的密文用私钥能解密，用私钥进行加密的密文用公钥能解密。在这里，"模 N" 是指在所有的加密和解密操作都在模 N 的意义下进行。实则，"互逆"是在指 `e` 和 `d` 相对于 `φ(N)` 模逆的关系。

> `e*d ≡ 1(mod φ(N))` 是一个同余方程，描述的是 `e` 和 `d` 在模 `φ(N)` 下的乘积同余于 1。
>
> 在数学中，"同余"是一种等价关系，a 和 b "模 m 同余"，如果它们除以 m 得到的余数相同。在此处的上下文中，这意味着当你将 e*d 除以 φ(N)，得到的余数是 1。
> 这个关系确保了公钥（N，e）和私钥（N，d）彼此互逆。即，如果你使用公钥加密消息 m，得到 `c=m^e (mod N)`，那么你可以用私钥解密消息 c，得到 `m=c^d (mod N)`，反之亦然。
> 这样设计的原因在于，如果第三方只知道公钥（N，e），他们无法计算出相应的私钥 d，除非他们能因数分解 N，这在实际中是非常困难的，使得这一加密系统具有很高的安全性。
>
> N 只是公钥和私钥共有的模数而已，它是两个大质数的乘积。要推导出私钥，除了需要知道 N 值外，还需要知道这两个质数，然而，当 N 是一个足够大的数时，要分解 N，找到这两个质数，基本上是不可能的（对于一个 N 值，有且只有一对质数的乘积等于 N）。
>
> 2048 位的 RSA 密钥中的 N 值，是由两个 1024 位的质数相乘得到的，所以它的长度大约在 616 到 617 位之间。在十进制下，它的值大约在 2 的 2048 次方到 2 的 2049 次方之间，即大约在 10 的 617 次方到 10 的 618 次方之间，这是一个非常非常大的数。

#### RSA 示例

使用 OpenSSL 命令行工具，你可以执行 RSA 数字签名和验证操作。下面是一个示例，演示如何生成 RSA 密钥对，使用私钥对文件进行数字签名，然后使用公钥验证签名。

```bash
# 生成私钥
openssl genrsa -out private.pem 2048
# 从私钥中提取公钥
openssl rsa -in private.pem -pubout -out public.pem

echo 123456789 > test.txt
# 使用私钥对文件进行签名
openssl dgst -sha256 -sign private.pem -out signature.bin test.txt
# 使用公钥验证签名
openssl dgst -sha256 -verify public.pem -signature signature.bin test.txt
## 输出 Verified OK
```

在这个示例中：

- `private.pem`是生成的私钥文件
- `public.pem`是从私钥中提取的公钥文件
- `test.txt`是要签名和验证的文件
- `signature.bin`是使用私钥对文件生成的签名文件

下面是一个 python 示例，签名的流程和 `fipsign.py` 中类似。

```python
from Crypto.PublicKey import RSA
from Crypto.Signature import pkcs1_15
from Crypto.Hash import SHA256
from Crypto import Random

# 生成 RSA 密钥对
random_generator = Random.new().read
key = RSA.generate(2048, random_generator)
public_key = key.publickey()

# 原始数据
message = b"This is a test message."

# 计算哈希值
h = SHA256.new(message)

# 使用私钥对原始数据的哈希值进行签名
signature = pkcs1_15.new(key).sign(h)

# 使用公钥验证签名
verifier = pkcs1_15.new(public_key)
try:
    verifier.verify(h, signature)
    print("The signature is valid.")
except (ValueError, TypeError):
    print("The signature is not valid.")

# 注意：在真实应用中，你不应该直接在代码中硬编码密钥，而应该使用安全的方式来存储和访问它们。
```

**问题：为什么要先hash，再对hash 进行签名？**

在数字签名中，通常先对消息进行哈希（hashing），然后再对哈希值进行签名，而不是直接对原始消息进行签名。这样做有几个重要的原因：

1. **性能**：哈希函数比签名算法快得多。哈希函数可以将任意长度的数据压缩成一个固定长度的哈希值，而签名算法则需要对整个消息（或哈希值）进行复杂的数学运算。由于哈希函数的快速性，即使对于非常大的消息，也可以快速生成哈希值，然后再对哈希值进行签名，从而大大**提高了签名的效率**。

2. **安全性**：哈希函数被设计为具有“雪崩效应”（avalanche effect），这意味着即使原始消息中只有一个微小的变化，也会导致哈希值发生显著的变化。这种性质使得哈希值对篡改非常敏感。如果直接对原始消息进行签名，那么任何对消息的微小篡改都可能需要重新进行整个签名过程，这既耗时又容易暴露给攻击者更多的信息。通过对哈希值进行签名，可以确保即使原始消息被篡改，签名也会立即失效，从而**提高了安全性**。

3. **可验证性**：哈希函数是单向的，这意味着从哈希值不能恢复出原始消息（或者说恢复原始消息是非常困难的）。但是，给定相同的消息，任何人都可以计算出相同的哈希值。因此，当接收者收到一个签名和相应的消息时，他们可以自己计算消息的哈希值，并使用签名者的公钥来验证签名是否有效。这种方式允许任何人验证签名的真实性，而不需要与签名者进行通信或交换任何额外的信息。

4. **签名长度**：**直接对长消息进行签名可能会导致生成的签名非常长**，从而增加了存储和传输的负担。通过对哈希值进行签名，可以确保签名长度始终固定且相对较短，这有利于节省空间和带宽。

综上所述，先对消息进行哈希，再对哈希值进行签名是一种常见的做法，它结合了哈希函数和签名算法的优点，提供了高效、安全和可验证的数字签名方案。

### 启动流程简介

![image-20230913160851943](../images/Cvitek-%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8/image-20230913160851943.png)

我们使用的是第三个，**一个完整的启动流程**为：由板端固定的 `ROM` 中的代码（`ZSBL`）初始化环境后，调用 `FSBL`，然后 `FSBL` 调用 `OpenSBI`，进而调用 `U-Boot`，由 `U-Boot` 来启动内核。**烧录流程**（`update`）只是没有 `U-Boot` 来启动内核这一步骤。

> 到 `uboot` 之后，就主要是由 `CONFIG_BOOTCOMMAND` 来控制后续操作，可以看到先尝试显示 `LOGO`，然后检测是否需要升级，`cvi_update` 它会检查是否有 `SD` 卡中有没有 `fip.bin` 或 `USB` 升级。这里使用的是 `||` ，就意味着前面执行成功，就不会执行后续的。升级之后，不会自动进入内核就是这个原因。
>
> ```c
> // u-boot-2021.10/include/configs/cv181x-asic.h:302
> #define CONFIG_BOOTCOMMAND    SHOWLOGOCMD "cvi_update || run norboot || run nandboot || run emmcboot"
> ```

整个启动流程我们用到的硬件存储设备有：`eFuse`、`flash(nor/emmc/nand)`、`SD`、`ddr` 这 4 个。

#### 编译流程

这里只关注会生成哪些文件，以及每个文件对应的模块。

| 模块    | 生成文件                                                                                     |
| ------- | -------------------------------------------------------------------------------------------- |
| fsbl    | bl2.bin，编译 fsbl 之后会创建 fip.bin （该文件打包了 bl2.bin fw_dynamic.bin u-boot-raw.bin） |
| opensbi | fw_dynamic.bin                                                                               |
| uboot   | u-boot-raw.bin                                                                               |
| kernel  | boot.spinor                                                                                  |
| ramdisk | rootfs.spinor                                                                                |

#### 烧录流程

先说升级（烧录流程），它的作用就是将 `opensbi`、`uboot`、`kernel` 等二进制文件烧录到 `flash` 上，每个文件在 `flash` 上的写入地址都是通过文件 `partition.xml` 指定，如：

> 不是指定写入地址，而是指定文件的大小，他们在 `flash` 上紧挨着排列，比如第 `0-1024`存储着 `fip.bin`，`1024-4096` 存储着 `boot.spinor`。即使 `fip.bin` 的实际大小没有 1024，`boot.spinor` 仍然是从第 1024 开始。**要注意 `flash` 的大小限制，不要超出了 `flash` 的大小**。

```xml
<physical_partition type="spinor">
    <partition label="fip" size_in_kb="1024" readonly="false" file="fip.bin"/>
    <partition label="BOOT" size_in_kb="3072" readonly="false" file="boot.spinor"/>
    <partition label="APPCFG" size_in_kb="64" readonly="false" file="app_cfg.bin"/>
    <partition label="APPCFGDEF" size_in_kb="64" readonly="false" file="app_cfg_def.bin"/>
    <partition label="sig" size_in_kb="64" file="sig.bin" />
    <partition label="ENV" size_in_kb="64" file="" />
    <partition label="ENV_BAK" size_in_kb="64" file="" />
    <partition label="ROOTFS" size_in_kb="8192" readonly="false" file="rootfs.spinor" />
    <partition label="MISC" size_in_kb="512" file="logo.jpg" />
    <partition label="DATA"  readonly="false" file="" mountpoint="/mnt/data" type="jffs2" />
</physical_partition>
```

> 写 `partition` 时还要注意对齐问题，这个与 `flash` 有关，比如在 `1812h_nand` 使用的 `flash` 是 `W25N02KVxxIR/U`，查看其数据手册可以看到，最小的块应该是 `128KB`，如果仍然将 `sig.bin` 的大小设置为 `size_in_kb="64"` ，烧录的时候就会报错！
>
> 另外，也要注意 `parititon` 中的大小要和 `cv181x-asic.h` 中设置的 `CONFIG_NANDBOOTCOMMAND` 中一致。
>
> ```bash
> Flexible Architecture with 128KB blocks
> – Uniform 128K-Byte Block Erase
> – Flexible page data load methods
> ```

由于 `fw_dynamic.bin u-boot-raw.bin` 放入了 `fip.bin` 文件中，因此，我们进行 `update` 的时候只需要准备 3 个文件：

```bash
fip.bin
boot.spinor
rootfs.spinor
```

虽然 `upgrade.zip` 解压后，能看到 `partition.xml` ，不过这玩意可以不要，因为编译过程中，已经从该文件生成了分区的 `.h` 头文件嵌入到的代码中，实际烧录时并不会用到。

另外，由于 `partition` 的分区划分，不同区域固定起始地址并且不会重叠，因此升级过程中，我们也可以只烧录一部分，比如与上一次烧录相比，只有 `u-boot` 发生了变化，那么我们的 `SD` 卡中只需要准备 `fip.bin` 就行。如果 `kernel` 发生了变化，就需要 `fip.bin` 和 `boot.spinor`，避免重复烧录 `rootfs.spinor` 可以加快一点点效率。

![sd 卡升级](../images/Cvitek-%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8/sd%E5%8D%A1%E5%8D%87%E7%BA%A7.png)

升级流程为：

1. `rom` 中 `bl1` 代码将 `bld.bin`（`BL2` 代码）搬移到 `SRAM` 中；
2. 执行 `BL2` 代码，初始化 `ddr`，将 `bldp.bin` 和 `u-boot.bin` 搬移到 `ddr` 中；
3. 执行 `bldp.bin`（`BL31` 代码）；
4. 执行 `uboot（BL33）`代码，将 `boot.xxx`、`rootfs.xxx`、`fip.bin` 拷贝到 `spinor(nand/emmc)`（中间需要经过 `ddr`，上图中没有展示该过程）；
5. 继续执行 `uboot` 代码，将 `boot.xxx` 从 `spinor(nand/emmc)` 读入 `ddr`，开始启动 `kernel`；

**4、5 两步由 `uboot` 下 `cvi_update` 和 ` run norboot/nandboot/emmcboot` 指令完成**，具体来说，`cvi_update` 主要完成的事情有：

1. 从升级的源头上拷贝 `boot.xxx、rootfs.xxx、fip.bin` 到 `ddr`；（这里源头可以是 `uart`、`sd` 卡、`usb` 甚至是 `ethernet`）
2. 将以上文件写入存储介质（`spinor、spinand、emmc`）；后续就可以直接从 `flash` 启动。

#### 启动流程

启动流程的前三级 `fsbl, opensbi, uboot` 和烧录流程是一致的，只有到 `uboot` 这里执行的内容不同。

再说启动流程，在 `uboot` 启动内核时，`nor flash` 与 `emmc/nand` 有点区别，因为 `nor` 可以直接运行代码，而 `emmc/nand` 需要先将 `kernel` 从 `flash` 中加载到 `ddr`，然后再运行。不过，下面我们使用安全启动时，由于需要校验 `kernel` 的签名，所以即使是 `nor` 也需要将 `kernel` 加载到 `ddr`。

### 签名加密流程介绍

从上面的烧录流程可以看到，前期到 `uboot` 运行阶段都与 `fip.bin` 有关，因此首先我们要保证 `fip.bin` 的安全性，要保证它不被篡改。不过这里并没有采用直接对 `fip.bin` 这整个文件进行加密，而是对 `fip.bin` 中每个小模块逐个加密、签名。一个 `fip.bin` 文件的构成如下图所示：

![_images/image1.jpg](../images/Cvitek-%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8/image1.jpg)

主要也就是前面说的 `fsbl, opensbi, u-boot`，不过还额外增加了各级的签名（可选的）。生成 `fip.bin` 的脚本是`fsbl/plat/cv181x/ fiptool.py `，对 `fip.bin` 进行签名和加密的脚本也在该目录下。

> 具体的 `fip` 构成可以看 `fsbl/plat/cv181x/fiptool.py:143` `FIP` 这个类的定义，`FIP` 就是按这里定义的顺序，由五个部分组成：`param1, body1, param2, body2, ldr_2nd_hdr`。下面是生成 `fip` 的编译日志：
>
> ```bash
> fsbl/plat/cv181x/fiptool.py -v genfip \
>         '/data/song.yu/cvi_mmf_sdk-intl/fsbl/build/cv1811h_wevb_0007a_spinor/fip.bin' \
>         --MONITOR_RUNADDR="${MONITOR_RUNADDR}" \
>         --BLCP_2ND_RUNADDR="${BLCP_2ND_RUNADDR}" \
>         --CHIP_CONF='/data/song.yu/cvi_mmf_sdk-intl/fsbl/build/cv1811h_wevb_0007a_spinor/chip_conf.bin' \
>         --NOR_INFO='FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF' \
>         --NAND_INFO='00000000'\
>         --BL2='/data/song.yu/cvi_mmf_sdk-intl/fsbl/build/cv1811h_wevb_0007a_spinor/bl2.bin' \
>         --BLCP_IMG_RUNADDR=0x05200200 \
>         --BLCP_PARAM_LOADADDR=0 \
>         --BLCP=test/empty.bin \
>         --DDR_PARAM='test/cv181x/ddr_param.bin' \
>         --BLCP_2ND='/data/song.yu/cvi_mmf_sdk-intl/freertos/cvitek/install/bin/cvirtos.bin' \
>         --MONITOR='../opensbi/build/platform/generic/firmware/fw_dynamic.bin' \
>         --LOADER_2ND='/data/song.yu/cvi_mmf_sdk-intl/u-boot-2021.10/build/cv1811h_wevb_0007a_spinor/u-boot-raw.bin' \
>         --compress='lzma'
> ```

#### 准备秘钥

首先我们需要准备好秘钥，从秘钥的后缀可以看出其类型，`.key` 为 `AES` 加密使用的，`.pem` 为 `RSA` 算法使用的。

| 文件          | 作用                                                         | 使用位置   |
| ------------- | ------------------------------------------------------------ | ---------- |
| rsa_hash0.pem | 用于给 `bl_priv.pem` 签名的 `RSA` 私钥，公钥（的 `hash` 值）被烧到 `HASH0_PUBLIC` | `rom code` |
| bl_priv.pem   | 用于给 `fsbl/opensbi/u-boot` 生成签名的 `RSA` 私钥           | `fsbl`     |
| loader_ek.key | 用于给 `bl_ek.key` 加、解密的 `AES` 秘钥，也会被烧写到 `eFuse` | `rom code` |
| bl_ek.key     | 用于给 `fsbl/opensbi/u-boot` 加、解密的 `AES` 秘钥           | `fsbl`     |

`RSA` 密钥使用 `2048 bits` 和第 4 费马数，**用于签名**：

> **虽然这里说“签名”，~~但和前面所说的“消息摘要”并不是一个概念，需要注意~~！🙄🙄是一个概念，但签名会用到私钥，而不仅仅是直接使用 sha256 算法之类的。**私钥签名，公钥验证。
>
> `PKCS#1 v1.5` 是一种公钥密码学标准，用于数字签名和验证消息的完整性。它通常与 `RSA` 密钥对一起使用。
>
> 1. **数字签名**：签名过程会使用**私钥**生成一个与消息相关联的**数字签名**，以便在以后验证消息的完整性和真实性。
> 2. **数字签名验证**：验证过程会使用与签名相关的**公钥**来验证消息的完整性，以确保消息没有被篡改，并且确实是由私钥持有者签名的。

```bash
host$ openssl genrsa -out rsa_hash0.pem -F4 2048
host$ openssl genrsa -out bl_priv.pem -F4 2048
```

> 为什么这里两个 `rsa` 秘钥都是私钥❓公钥在哪呢❓
>
> 答：`RSA` 算法的私钥中实际上包含了公钥的一部分信息，具体来说，包括了**模数**和**公钥指数**，通过这两个参数就可以推出公钥😮。不过公钥是无法推出私钥的，这是 `rsa` 算法的安全保障。

`AES` 的秘钥**使用长度 16 的随机数**（`AES-128`），**用于加密**：（只签名不加密就用不到这个）

```bash
host$ head -c 16 /dev/random > loader_ek.key
host$ head -c 16 /dev/random > bl_ek.key
```

#### fip 组成结构

这里得看一下 `fip.bin` 的构成，在 `fsbl/plat/cv181x/fiptool.py:143` 中定义，下面有省略：

```python
class FIP:
    # FIP 就是按这里定义的顺序：param1, body1, param2, body2, ldr_2nd_hdr
    param1 = OrderedDict(
        [
            Entry.make("MAGIC1", 8, int, b"CVBL01\n\0"),
            Entry.make("MAGIC2", 4, int),
            Entry.make("BL2_IMG_CKSUM", 4, int),
            Entry.make("BL2_IMG_SIZE", 4, int),
            Entry.make("BLD_IMG_SIZE", 4, int),
            Entry.make("BL_EK", 32, bytes),             # 存储 bl_ek ？
            Entry.make("ROOT_PK", 512, bytes),          # 存储 rsa_hash0.pem ? 512 字节只是最大长度吧
            Entry.make("BL_PK", 512, bytes),            # 存储 bl_priv.pem ?
            Entry.make("BL_PK_SIG", 512, bytes),        # 存储 bl_priv.pem 的签名
            Entry.make("CHIP_CONF_SIG", 512, bytes),
            Entry.make("BL2_IMG_SIG", 512, bytes),      # 存储 bl2(fsbl) 镜像的签名
            Entry.make("BLCP_IMG_SIG", 512, bytes),     # 存储 blcp 镜像的签名
        ]
    )

    body1 = OrderedDict(
        [
            Entry.make("BLCP", None, bytes),            # 填充 blcp
            Entry.make("BL2", None, bytes),             # 填充 bl2
        ]
    )
```

可以看到，在 `param1` 中存储着 `BL2` （也就是 `fsbl` 镜像）的签名。从 `fsbl/plat/cv181x/fipsign.py:sign` 中可以看到具体的赋值过程：

```python
    # 签名的主函数
    def sign(self):
        logging.info("sign fip.bin")

        # 处理 fip 的 param1 部分
        self.param1["FIP_FLAGS"].content = self.FIP_FLAGS_SCS_MASK | self.param1["FIP_FLAGS"].toint()
        self.sign_bl_pk()
        cc = self.param1["CHIP_CONF"].content
        cc_size = unpack("<I", self.param1["CHIP_CONF_SIZE"].content)[0]
        logging.debug("CHIP_CONF_SIZE=%#x", cc_size)
        cc = cc[:cc_size]

        # 主要关注这里 3 个 sign_by_bl_priv
        self.param1["CHIP_CONF_SIG"].content = self.sign_by_bl_priv(cc)
        self.param1["BL2_IMG_SIG"].content = self.sign_by_bl_priv(self.body1["BL2"].content)

        if self.body1["BLCP"].content:
            logging.debug("sign blcp")
            self.param1["BLCP_IMG_SIG"].content = self.sign_by_bl_priv(self.body1["BLCP"].content)

        self.sign_fip2()
```

主要关注这里 3 个 `sign_by_bl_priv`，将 `BL2` 镜像通过 `bl_priv.pem` 私钥进行签名，然后将签名结果放在 `param1["BL2_IMG_SIG"]`。板端通过写在 `efuse` 的公钥，对 `BL2` 又计算一次签名，和 `fip` 中的签名对比，如果一致则说明该 `fip` 确实是私钥持有者提供的。

上面只是对 `bl2` 进行了签名，`FIP` 的后半部分还有 `opensbi`， `uboot`（同样有省略）：

```python
    param2 = OrderedDict(
        [
            Entry.make("MAGIC1", 8, int, b"CVLD02\n\0"),
            Entry.make("MONITOR_CKSUM", 4, int),                # ATF-BL31 or OpenSBI
            Entry.make("MONITOR_LOADADDR", 4, int),
            Entry.make("LOADER_2ND_RESERVED0", 4, int),         # uboot
            Entry.make("LOADER_2ND_LOADADDR", 4, int),          # Reserved
            Entry.make("RESERVED_LAST", 4096 - 16 * 5, bytes),
        ]
    )

    body2 = OrderedDict(
        [
            Entry.make("MONITOR", None, bytes),     # 填充 opensbi 镜像
            Entry.make("LOADER_2ND", None, bytes),  # 填充 uboot 镜像
        ]
    )

    ldr_2nd_hdr = OrderedDict(
        [
            Entry.make("MAGIC", 4, int),
            Entry.make("CKSUM", 4, int),
            Entry.make("SIZE", 4, int),
        ]
    )
```

与 `param1` 不同，这里 `param2` 中并没有成员专门记录 `uboot` 的签名，而是直接放在了镜像的末尾，而且也没有对 `opensbi` 进行签名。

```python
e = self.body2["LOADER_2ND"] # e 就是 uboot 的镜像
# 在末尾预留签名的空间，再填充 \0 按 IMAGE_ALIGN 对齐
e.content = self.pad(e.content + b"\xCE" * sig_size, IMAGE_ALIGN)
# SIZE is after CKSUM, update it before signing
self._update_ldr_2nd_hdr()

image = bytearray(e.content)
# 计算签名，不包含最后预留给存储”签名“的内容。
sig = self.sign_by_bl_priv(image[self.ldr_2nd_hdr["CKSUM"].end : -sig_size])
assert sig_size == len(sig)
image[-sig_size:] = sig

e.content = image
```

#### efuse 介绍

`eFuse` 也就是**可编程电子熔丝**，简单来说，它存储的数据只能写 `1` 不能写 `0`，最初全为 0，一旦写入就无法更改。具有以下主要特点：

1. **不可逆性**：一旦配置，`eFuse` 通常无法被擦除或重置。这意味着一旦信息被写入 `eFuse`，它将永久存储在其中，不能再次修改或清除。这种不可逆性增强了存储在 `eFuse` 中的信息的安全性。（不可擦除）
2. **电气可编程**：`eFuse` 可以通过电气编程方式进行配置，而不需要使用额外的设备或工具。这种可编程性使得它在制造过程中或设备的生命周期中可以根据需要进行配置。（可写入）
3. **高可靠性**：`eFuse` 通常具有高度的可靠性和稳定性，可以在广泛的温度范围和环境条件下正常工作。这使得它适用于各种应用，包括极端环境下的用途。
4. **物理安全**：`eFuse` **通常集成在芯片内部**，难以物理上访问或破坏。这增强了存储在其中的敏感信息的物理安全性，防止了硬件级别的攻击。

`eFuse` 的上述特点，很适合用来存储板端的秘钥。

#### fip 签名流程

> 直接使用 `fipsign.py` 这个脚本，`fip.bin` 为原始镜像，`fip_sign.bin` 为签名后的镜像。注意这里仅使用了 `RSA` 秘钥。
>
> ```bash
> $ ./fipsign.py sign \
>       --root-priv=rsa_hash0.pem \
>       --bl-priv=bl_priv.pem \
>       fip.bin fip_sign.bin
> ```
>

![安全启动.drawio](../images/Cvitek-%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8/%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8.drawio.png)

> 图中 `PK` 表示公钥的 `N` 值，`_SIG/sign` 表示数字签名。

1. 修改 `FIP_FLAGS` 表明这个 `fip.bin` 中有签名，需要验证。

2. 从私钥 `rsa_hash0.pem` 中提取 `N` 值并写到 `fip.bin` 中，该 `N` 值和 `rom` 中固化的 `E=0x10001` 可组成公钥，用于签名验证。

3. `N` 值的 `sha256` 散列值会被写入到 `efuse` 的 `LOCK_HASH0_PUBLIC `，验证时先计算 `fip.bin` 中的 `N` 值的散列值，与 `efuse` 中存储的散列值比较，来确认 `N` 值（或者说公钥）是否与最初的一致。

4. 提取 `bl_priv.pem` 的 `N` 值写入到 `fip.bin` 中。

5. 计算 `bl_priv.pem` 的 `N` 值的数字签名，并写入到 `fip.bin` 中。该签名是通过 `rsa_hash0.pem` 这个私钥计算 ，可用 `rsa_hash0.pem` 的公钥验证。

    > 实际是 `bl_priv.pem` 的 `N` 值的 SHA256 散列值的数字签名，这里简单点说方便理解。

6. 计算 `bl2.bin(fsbl)` 的数字签名，该签名通过 `bl_priv.pem` 这个私钥计算，用 `BL_PK` 验证。

7. 计算 `fw_dynamic.bin(opensbi)` 的数字签名，该签名通过 `bl_priv.pem` 这个私钥计算，用 `BL_PK` 验证。

8. 计算 `u-boot-raw.bin(uboot)` 的数字签名，该签名通过 `bl_priv.pem` 这个私钥计算，用 `BL_PK` 验证。

#### fip 签名+加密流程

> 直接使用 `fipsign.py` 这个脚本，`fip.bin` 为原始镜像，`fip_enc.bin` 为签名+加密后的镜像。这里使用了 `RSA` 和 `AES` 秘钥。
>
> ```bash
> $ ./fipsign.py sign-enc \
>          --root-priv=rsa_hash0.pem \
>          --bl-priv=bl_priv.pem \
>          --ldr-ek=loader_ek.key \
>          --bl-ek=bl_ek.key \
>          fip.bin fip_enc.bin
> ```

`fip` 的签名并加密也就是在上面签名的基础上，再对各个 `bin` 文件进行加密。加密主要依赖两个 `AES` 秘钥，`loader_ek` 和 `bl_ek`。前者会被写入 `efuse`，并用于对 `bl_ek` 的加、解密。`bl_ek` 加密后放入 `fip.bin` 中，`bl_ek` 用于对各个 `bin` 文件进行加密。验证时先通过 `efuse` 中的明文的 `loader_ek` 秘钥对 fip.bin 中加密后的 `bl_ek` 进行解密得到 `bl_ek`。然后用它对各个 `bin` 文件进行解密。

验证的流程相反，先解密后再验证签名。

![安全启动-sign-enc-fip.drawio](../images/Cvitek-%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8/%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8-sign-enc-fip.drawio.png)

1. 将 `loader_ek.key` 明文写入 `efuse`。
2. 用 `loader_ek.key` 加密 `bl_ek.key` 后写入 `fip.bin`。
3. 用 `bl_ek.key` 对 `bl2.bin` 进行加密。
4. 用 `bl_ek.key` 对 `fw_dynamic.bin` 进行加密。
5. 用 `bl_ek.key` 对 `u-boot-raw.bin` 进行加密。

具体签名的实现涉及到 `fsbl/plat/cv181x/` 目录下的 `fipsign.py` 和 `fiptool.py`。前面 [fip 组成结构](#fip 组成结构) 也提到了部分实现。

> `param2/ldr_2nd_hdr` 中记录的信息也会随着签名、加密更新，这里并不关注这些细节。

```asciiarmor
encrypt_fip
  ├─►read_fip
  ├─►sign
  │   ├─►sign_bl_pk
  │   ├─►sign_by_bl_priv(BL2
  │   ├─►sign_by_bl_priv(MONITOR
  │   └─►sign_by_bl_priv(LOADER_2ND_LOADADDR
  └─►encrypt
      ├─►_aes_encrypt(LOADER_EK, BL_EK)  # (秘钥，待加密内容)
      ├─►_aes_encrypt(BL_EK, BL2)
      ├─►_aes_encrypt(BL_EK, MONITOR)
      └─►_aes_encrypt(BL_EK, LOADER_2ND_LOADADDR)
```

------

#### fip 校验流程

`fip.bin`  的校验由两部分组成，一部分代码是位于 `ROM` 中（没有权限查看，只能合理猜测 `fsbl` 中缺少的就是在 `rom` 中完成的）：不用猜测，测试就行，`ROOT_PK`，`BL_PK`，`BL2`的签名校验都是在 `rom code` 中完成。~~从 `efuse` 读取 AES 秘钥 `LOADER_EK`~~。校验不通过则不进行烧录，直接从 `flash` 加载 `fsbl`，日志信息如下：

```
C.SCS/3/3.URPL.SDI/25000000/6000000.BS/SD.PS.SD/0x0/0x1000/0x1000/0.VRK4. E:verify root (-14)
 W:DL cancelled. Load flash. (1).
BS/NOR.PS.VRK4.VBK.DK4.VCC.PE.BS.DB2.VB2.BE.J.
FSBL Jb2829:ga05fb6172-dirty:2023-09-19T10:28:45+08:00

C.SCS/3/3.URPL.SDI/25000000/6000000.BS/SD.PS.SD/0x0/0x1000/0x1000/0.VRK4. E:verify bl_pk (-14)
C.SCS/3/3.URPL.SDI/25000000/6000000.BS/SD.PS.SD/0x0/0x1000/0x1000/0.VRK4. E:verify BL2 (-14)
```

而 `FSBL` 中的解密、验签的流程如下：加载时，才去解密 + 校验。

```asciiarmor
 bl2_main
    │
    └─►load_rest
          │
          ├──►load_blcp_2nd ───┐
          │                    │
          ├──►load_monitor ────┼──►dec_verify_image
          │                    │       │
          └──►load_loader_2nd ─┘       ├──►security_is_tee_encrypted
                                       │         │
                                       │         └───►cryptodma_aes_decrypt
                                       └──►verify_rsa
```

### kernel / rootfs 的签名

除 `fip.bin` 外，还有 `kernel` 的产物 `boot.xxx` 和文件系统的产物 `rootfs.xxx`。

#### 准备秘钥

> 1️⃣这里的秘钥的名称与上面不一样，因为是不同时期针对不同板子写的，不过逻辑是一样的，后面再考虑统一名称。
> 2️⃣这里没有进行加密，所以不需要生成 `AES` 秘钥。

| 文件           | 作用                                                                     |
| -------------- | ------------------------------------------------------------------------ |
| NTKC_PRIV.pem  | 用于给 `REE_OS_PK` 签名的私钥。公钥（的 `hash` 值）被烧到 `HASH0_PUBLIC` |
| REEOS_PRIV.pem | 用于给 `boot, rootfs` 签名的私钥                                         |
| boot.crl       | 黑名单，`REEOS_PK.pem` 不能是 `boot.crl` 中的值                          |

和上面一样，使用 `openssl` 生成 `2048` 位的秘钥：

```bash
host$ openssl genrsa -out NTKC_PRIV.pem -F4 2048
host$ openssl genrsa -out REEOS_PRIV.pem -F4 2048
```

#### 签名流程

与前面直接将签名信息直接写入到 `fip.bin` 不同，对 `kernel` 和 `rootfs` 的签名专门生成了一个 `sig.bin` 文件来存储。其结构如下图所示，分为三部分，公钥、签名、文件大小信息。

> 1️⃣在这里的设计中，文件系统 `rootfs` 有两份，一份就是正常大小的 `upgrade`，另一份为删减了一些非必要文件的 `minerfs`。不过在后续的流程中，我并没有对 `rootfs` 进行裁剪，也就是说 `upgrade == minerfs == rootfs.spinor` ，先走通流程，后续再考虑是否精简。
>2️⃣只进行了签名操作，没有进行加密！
> 3️⃣文件大小信息是板端重新对 `kernel/rootfs` 计算签名的时候用到的。

![安全启动-sign-boot-rootfs.drawio](../images/Cvitek-%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8/%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8-sign-boot-rootfs.drawio.png)

1. 由 `NTKC_PRIV.pem` 私钥得到公钥。 `NTKC_PRIV.pem` 的公钥需要记录在 `efuse` 里面，而 `efuse` 空间有限，签名 `rsa_hash0.pem` 已经用了，所以这两个必须相等 `NTKC_PRIV.pem == rsa_hash0.pem` 。
2. 由 `REEOS_PRIV.pem` 私钥得到公钥。该私钥感觉也可以与前面的 `bl_priv.pem` 相同。
3. 用 `NTKC_PRIV.pem` 计算 `REEOS_PK.pem` 的数字签名
4. 用 `NTKC_PRIV.pem` 计算数字签名。`boot.crl` 是作为黑名单使用的，即 `REEOS_PK.pem` 不能是 `boot.crl` 中的值（这里面的值是已经泄漏的秘钥）。
5. 用 `REEOS_PRIV.pem` 计算数字签名。
6. 用 `REEOS_PRIV.pem` 计算数字签名。这里不区分 `minerfs/upgrade` ，都记录为 `rootfs` 的签名。

对 `kernel` 和 `rootfs` 的签名的实现是在 `make_sig_img.sh` 这个脚本中：

```bash
#!/bin/bash

set -e -o pipefail

SIG_TMP_PATH="${BUILD_PATH}/tools/common/sign_tools/sig_files"
NTKC_PRIV_PATH="${BUILD_PATH}/tools/common/sign_tools/NTKC_PRIV.pem"
REEOS_PRIV_PATH="${BUILD_PATH}/tools/common/sign_tools/REEOS_PRIV.pem"
BOOT_CRL_PATH="${BUILD_PATH}/tools/common/sign_tools/boot.crl"

# sig.bin 中不同部分的分割符
SIG_IMG_SEP=$(printf '\xe8\x6c\xdc\x26\x7b\xcb\xf6\x4b\x8d\xd8\xa2\x2c\x86\xa6\x58\x73\xf9\x42\xbc\x3a\x70\x58\x02\x43\xbe\x79\x56\x66\x23\x2a\x0a\x7e')

function int_to_bytes()
{
    python3 -c 'import sys; sys.stdout.buffer.write(int(sys.argv[1]).to_bytes(4, "little"))' $1
}

# create sig.bin
function make_sig_img()
(
    # 这里顺序要和 u-boot 中 cvi_verify.c 中对应
    echo "${FUNCNAME[0]}()"
    {
        cat $SIG_TMP_PATH/NTKC_PK.pem
        printf '%s' "$SIG_IMG_SEP"
        cat $SIG_TMP_PATH/REEOS_PK.pem
        printf '%s' "$SIG_IMG_SEP"
        cat "${BOOT_CRL_PATH}"
        printf '%s' "$SIG_IMG_SEP"
        cat $SIG_TMP_PATH/REEOS_PK.pem.sig
        printf '%s' "$SIG_IMG_SEP"
        cat $SIG_TMP_PATH/boot.crl.sig
        printf '%s' "$SIG_IMG_SEP"
        cat $SIG_TMP_PATH/boot.$STORAGE_TYPE.sig
        printf '%s' "$SIG_IMG_SEP"
        cat $SIG_TMP_PATH/minerfs.sig
        printf '%s' "$SIG_IMG_SEP"
        cat $SIG_TMP_PATH/upgrade.sig
        printf '%s' "$SIG_IMG_SEP"
        int_to_bytes "$(stat -c '%s' $OUTPUT_DIR/rawimages/boot.$STORAGE_TYPE)"
        int_to_bytes "$(stat -c '%s' $OUTPUT_DIR/rawimages/rootfs.$STORAGE_TYPE)"
        int_to_bytes "$(stat -c '%s' $OUTPUT_DIR/rawimages/rootfs.$STORAGE_TYPE)"
        printf '%s' "$SIG_IMG_SEP"
    } > "$OUTPUT_DIR/rawimages/sig.bin"

    # 在文件 sig.bin 前面加上一段前缀字符串，表示是 cvitek 的镜像
    python3 "$BUILD_PATH/tools/common/image_tool/raw2cimg.py" "$OUTPUT_DIR/rawimages/sig.bin" "$OUTPUT_DIR" "$FLASH_PARTITION_XML"
    echo "Save to $OUTPUT_DIR/sig.bin"
)

# 对镜像用私钥进行加密、签名
function sign_image()
(
    echo "${FUNCNAME[0]}()"
    # 由私钥得到公钥
    openssl rsa -in ${NTKC_PRIV_PATH} -pubout -out "$SIG_TMP_PATH/NTKC_PK.pem"
    openssl rsa -in ${NTKC_PRIV_PATH} -pubout -outform DER | sha256sum > "$SIG_TMP_PATH/NTKC_PK.der.sha256"
    openssl rsa -in "${REEOS_PRIV_PATH}" -pubout -out "$SIG_TMP_PATH/REEOS_PK.pem"

    # 生成数字签名
    openssl dgst -sha256 -sign ${NTKC_PRIV_PATH} -out "$SIG_TMP_PATH/REEOS_PK.pem.sig" "$SIG_TMP_PATH/REEOS_PK.pem"
    openssl dgst -sha256 -sign ${NTKC_PRIV_PATH} -out "$SIG_TMP_PATH/boot.crl.sig" ${BOOT_CRL_PATH}

    openssl dgst -sha256 -sign "${REEOS_PRIV_PATH}" -out "$SIG_TMP_PATH/boot.$STORAGE_TYPE.sig" "$OUTPUT_DIR/rawimages/boot.$STORAGE_TYPE"
    # 这里没有区分 upgrade/minerfs，暂时不确定是否需要区分，所以保留两个
    openssl dgst -sha256 -sign "${REEOS_PRIV_PATH}" -out "$SIG_TMP_PATH/minerfs.sig" "$OUTPUT_DIR/rawimages/rootfs.$STORAGE_TYPE"
    openssl dgst -sha256 -sign "${REEOS_PRIV_PATH}" -out "$SIG_TMP_PATH/upgrade.sig" "$OUTPUT_DIR/rawimages/rootfs.$STORAGE_TYPE"
    echo "sign_image done!"
)

mkdir -p "$SIG_TMP_PATH"
(
    cd "$SIG_TMP_PATH"
    echo "Clear $SIG_TMP_PATH"
    rm -f ./*
)

echo "start sign"
sign_image
make_sig_img
```

#### 校验流程

1. 计算 `sig.bin` 中 `NTKC_PK.pem` 的散列值，与 `efuse` 中存储的散列值对比，如果相同则说明 `NTKC_PK.pem` 可信。
2. 利用 `NTKC_PK.pem` 计算 `REEOS_PK.pem` 的数字签名，和 `REEOS_PK.pem.sig` 对比，相同则说明 `REEOS_PK.pem` 可信。
3. 利用 `REEOS_PK.pem` 为 `boot / rootfs` 计算数字签名，并与对应的 `.sig` 对比，相同则说明对应的文件没有被篡改，可信。
4. 校验完毕，可以正常启动 `kernel` 了。

## 注意事项

### 1️⃣使用注意

1. **配置参数**，`fip.bin` 的加密通过设置配置选项 `CONFIG_FSBL_SECURE_BOOT_SUPPORT=y` 启用，而 `kernel` 和 `rootfs` 则是通过环境变量 `UBOOT_VBOOT=1` 来启用。分开配置是必要的，因为第一次需要未加密的 `fip.bin` 来实现烧写 `efuse`，在 `efuse` 还未烧写的情况下，加密了的 `fip.bin` 无法加载。
2. **修改** `partition.xml`，其实就是增加一项 `sig.bin`，需要注意其大小对齐，具体大小要看对应 `flash` 数据手册，不过一般为 `64k/128k`。
3. **修改** `cv18xx-asic.h`，在 `uboot`启动 `kernel` 之前，需要先对 `kernel` 和 `rootfs` 进行签名校验，而这一过程需要先将对应的内容从 `flash` 读取到内存中，目前只测试了 `spinor/spinand`，`emmc` 还未测试。**另外注意**：读取时各个分区占据的内存不要重叠，分区的大小最好与 `partition.xml` 中一致。

### 2️⃣raw2cimg.py

在我们的编译流程中，编译得到的原始文件比如 `boot.spinand`， `rootfs.spinor` 等是放在 `$OUTPUT_DIR/rawimages` 目录下的，不过编译流程中还会调用 `raw2cimg.py` 对这些二进制文件加上一段前缀`Header`，`upgrade.zip` 中打包的也是这些添加"`header`"后的文件。

```bash
$ hexdump sig.bin
0000000 4943 474d 0001 0000 0040 0000 0001 0000
0000010 0bb5 0000 6973 0067 0000 0000 0000 0000
0000020 0000 0000 0000 0000 0000 0000 0000 0000
*
0000040 0001 0000 0b75 0000 0000 00b2 0000 0002
0000050 dd1b 0ee4 0000 0000 0000 0000 0000 0000
0000060 0000 0000 0000 0000 0000 0000 0000 0000
*
0000080 2d2d 2d2d 422d 4745 4e49 5020 4255 494c # 这一行开始是正式内容
0000090 2043 454b 2d59 2d2d 2d2d 4d0a 4949 4942
```

~~问题在于💢：前面 `sig.bin` 中使用的是 `rawimages` 下的文件生成的签名，而烧写到 `flash` 的是增加了一段前缀后的文件，两个文件有差异，那么在进行 `kernel` 和 `rootfs` 签名校验的时候，应该无法通过才对❗💢💢但它确实能通过校验，并且使用 `rawimages` 下的文件反而无法通过🤦‍♂️也没有找到哪里有对前缀进行裁剪之类的代码🤦‍♂️🤦‍♂️🤦‍♂️~~。

**是在烧录的过程中就进行了裁剪**。在 `cvi_update` 过程中，会检查各个文件是否存在 `Header`，如果不存在或者不一致，会无法烧录到 `flash`。此外，在实际烧录时也会裁剪掉 `header`。

```asciiarmor
 do_cvi_update
     │
     └►_storage_update
          │
          └─►_checkHeader
               │
               ├──►uint32_t pos = HEADER_SIZE;
               └──►snprintf(cmd, 255, "fatload %s %p %s 0x%x 0x%x;", strStorage,
                   			 (void *)UPDATE_ADDR, file, load_size, pos);
```

使用未添加 `Header` 的 `sig.bin`，`boot.spinor`，`rootfs.spinor` 烧录的时候，会提示下面的错误：

```
File:sig.bin Magic number is wrong, skip it
File:rootfs.spinor Magic number is wrong, skip it
```

### 3️⃣烧录 efuse

目前是将烧录 `efuse` 的流程是绑定在对 `kernel` 签名的。只要启用了 `UBOOT_VBOOT=1`，启动过程中就会检查是否设置了安全启动（通过读取 `efuse` 的数据判断），如果没有设置就会烧写 `efuse`，包括 `HASH0_PUBLIC` 和 `loader_ek`。

如果每次编译的 `boot.xxx` 都带有这些信息，容易导致秘钥泄漏。前者没有问题，因为烧写到 `efuse` 的是公钥，即使知道公钥也推不出私钥。不过后者是 `AES` 明文的秘钥，存在隐患。~~所以，在非必要的时候应该注释掉相关内容~~。已增加配置环境变量 `UBOOT_WRITE_EFUSE=1` 来控制。

### ⭐更新

由于客户要求，只签名，不对 `fip.bin` 进行加密，希望能以宏进行控制，因此需要对部分代码进行更新。

1. 使用环境变量 `ONLY_SIGN_FIP=1` 来表示仅对 `fip.bin` 进行签名，未设置或设置为0表示签名+加密。
2. 代码主要更新：
    1. 编译流程中 `fip_v2.mk` 调用 `fipsign.py` 之前检查环境变量。
    2. 编译流程中 `build/Makefile` 将 `loader_ek` 复制到 `uboot` 目录下的过程，需要先检查环境变量。
    3. `efuse.c` 中烧写 `efuse` 时，检查环境变量，不烧写 `loader_ek`。
    4. 在 `uboot/*/cvitek.mk` 中还要增加 `-DONLY_SIGN_FIP`，这样才能影响到 `efuse.c` 中的代码。

### 4️⃣使用方法

1. 准备以下几个秘钥文件，如何生成秘钥可以看前面的介绍。

    | fip.bin       | kernel/rootfs  |                                                                                   |
    | ------------- | -------------- | --------------------------------------------------------------------------------- |
    | rsa_hash0.pem | NTKC_PRIV.pem  | 用于给 `bl_priv.pem` 签名的 `RSA` 私钥，公钥（的 `hash` 值）被烧到 `HASH0_PUBLIC` |
    | bl_priv.pem   |                | 用于给 `fsbl/opensbi/u-boot` 生成签名的 `RSA` 私钥                                |
    | loader_ek.key |                | 用于给 `bl_ek.key` 加、解密的 `AES` 秘钥，也会被烧写到 `eFuse`                    |
    | bl_ek.key     |                | 用于给 `fsbl/opensbi/u-boot` 加、解密的 `AES` 秘钥                                |
    |               | REEOS_PRIV.pem | 用于给 `boot, rootfs` 签名的私钥                                                  |
    |               | boot.crl       | 黑名单，`REEOS_PK.pem` 不能是 `boot.crl` 中的值，**不能为空文件**                 |

2. 在一个空白的板子上，此时板子的 `efuse` 还没有写过，首先需要烧写 `efuse`，此时不能加密 `fip.bin`。

    `export UBOOT_VBOOT=1 UBOOT_WRITE_EFUSE=1; source build/xxx`

3. **完成烧录后，拔卡，重新上电**，才会进行 `efuse` 的烧写。

4. 烧写完毕，后续编译的镜像就必须对 `fip.bin` 进行加密了，否则无法烧录。
    完成烧录后，也不要再使用 `UBOOT_WRITE_EFUSE=1`，否则 `boot.xxx` 中会带有 `loader_ek.key` 的明文，容易导致秘钥泄漏。

    `export UBOOT_VBOOT=1; source build/xxx`，再通过 `menuconfig` 启用 `CONFIG_FSBL_SECURE_BOOT_SUPPORT`(这个可以写在对应板卡目录下的 xxx_defconfig 文件中 )

## 常见问题

1. 秘钥都是写在 efuse 中的，那别人岂不是可以直接读 efuse 获取到秘钥？这样不很容易破解吗？

   > efuse 中存了两个秘钥：
   >
   > 1. HASH0_PUBLIC 区存放的 RSA 算法的公钥的 sha256 哈希值。
   > 2. LOADER_EK 区存放的 AES 的秘钥。
   >
   > 正常情况下是可读写的，不过烧写下面两个字段，就可以禁用对应区域的读写。[see doc](https://doc.sophgo.com/cvitek-develop-docs/master/docs_latest_release/CV180x_CV181x/zh/01.software/BSP/eFuse_User_Guide/build/html/2_eFuse_User_Guide/eFuse_Overview.html)
   >
   > | LOCK_LOADER_EK | 锁定安全启动AES加密密钥区域，让此区域无法读写 |
   > | -------------- | --------------------------------------------- |
   > | LOCK_DEVICE_EK | 锁定DEVICE_EK，让此区域无法读写               |
   >
   > **那如果不能读写？启动过程又是怎么验签和解密的呢？**
   >
   > 这两个秘钥的内容是在 rom code 中读取的，rom code 有特权吧。efuse 区分安全介面和非安全介面，锁读写只是非安全界面的读写，安全界面至少能读。

2. ~~fip 的签名校验都是在 fsbl 中完成~~，如果能获得 fsbl 的源码，修改后能跳过签名校验过程吗？

   > 看 [fip 校验流程](#fip 校验流程) ，在 rom code 中会首先对 fip 进行校验，也就是说，即使你能修改 fsbl 的流程，但没有 RSA 秘钥，无法对 fip 进行签名，那么在 rom code 阶段就被卡住了。
   >
   > 如果秘钥都泄漏了，那改不改 fsbl 的流程都一样了。

3. 公钥不小心泄漏了，能被人利用吗？前面介绍的公钥是直接存放在 fip.bin 中的，fip.bin 本身又不大，知道公钥的位数，总能穷举出来吧。

   > 除了穷举，还可以随便创建一个公钥私钥，用私钥签名 `fip.bin` 然后查看公钥对应 `fip.bin` 中的偏移量。不就可以知道任意一个 `fip.bin` 中的公钥了吗？
   >
   > 假设已经知道了公钥，rom code 里面如果仅仅对公钥本身进行了校验，那也就是说我可以使用自己编译的一个 fip.bin，替换里面的公钥，就可以进入 fsbl 阶段了。那我岂不是可以修改 fsbl，跳过签名、解密的过程？这不就破解了？？？uboot就可以烧录任何固件了.... ？？？
   >
   > 👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇👇
   >
   > 公钥确实容易获取，查看 `fiptool.py` 可以看到 `ROOT_PK` 在 `fip.bin` 中的偏移量为 `0X400`。不过这并没有安全风险。因为目前是一级一级校验的。
   >
   > - `rom code` 中会校验 `BL2(FSBL)` 的签名，保证了 `FSBL` 无法被篡改。或者说篡改后的 `FSBL` 无法通过 `ROM` 的校验。*对应下图中的 2,4,5,6 步*。
   > - `FSBL` 中校验 `UBOOT`，也就保证了 `uboot` 无法被篡改。*对应下图中的 7,8 步*。
   > - `uboot` 校验 `kernel/rootfs`，保证了它们无法被篡改。
   >
   > ![安全启动-sign-enc-fip.drawio](../images/Cvitek-%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8/%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8-sign-enc-fip.drawio.png)
   >
   > 回到上面的问题，`rom` 中并不是只校验了 `ROOT_PK`，而是校验了 `bl2` 的签名。即使 `root pk` 泄露了，也不会影响 `bl2` 的签名校验（这需要 root 和 bl 的私钥）。不存在安全风险。
   >
   > 另一方面，目前 `kernel` 和 `rootfs` 的校验做的不太好，理论上 `uboot` 阶段去校验 `kernel` 和 `rootfs` 的签名时，应该使用新的秘钥，而不是 `root pk`。也就是 `NTKC_PRIV.pem` 的公钥也可以记录在 `fip.bin` 中，`fsbl` 中对 `uboot` 验签的时候，用 `bl_pk` 对 `sig.bin` 进行验签，也就是将 `sig.bin` 中的 `NTKC_PK.pem` 换为 `bl_pk`，这样就是一层一层的依赖关系。而不是现在 `uboot` 直接用 `root` 的公钥。。不过也还好，影响不大。
   >
   > ![安全启动-sign-boot-rootfs.drawio](../images/Cvitek-%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8/%E5%AE%89%E5%85%A8%E5%90%AF%E5%8A%A8-sign-boot-rootfs.drawio.png)

4. **安全启动的意义**

   > - 防止用户烧录未经授权的固件。（签名）
   >
   >     > 启用安全启动之后，只能烧录签名后的固件。
   >
   > - 防止通过固件拷贝，来抄袭产品。（签名+加密）
   >
   >     > A 卖芯片，提供 SDK。
   >     >
   >     > B 买芯片，画板子，用 A 提供的 芯片 + SDK 做产品卖。
   >     >
   >     > C 抄 B 的板子，并直接拷 B `flash` 中的固件，再买 A 的同款芯片。理论上用很低的成本就完成了对 B 产品的完全复制。
   >     >
   >     > 如何避免呢？
   >     >
   >     > 1. 安全启动（仅签名），C 的视角来看，固件有了，问题在于芯片上的 `efuse` 烧写。不过仅签名的话，只会用到 `root_pk`，而这是可以从固件 `fip.bin` 中获得的。因此 `C` 还是可以抄袭。
   >     > 2. 安全启动（签名 + 加密），C 的视角来看，固件有了，还差 `LOADER_EK`，而这无法读取。因此，无法抄袭。

5. 从上面可以看到，还有一个关键在于 `efuse` 中的 `AES` 秘钥，绝不能被窃取。如何实现呢？安全介面和非安全介面

   >

## 相关资料

- [安全启动使用指南](https://doc.sophgo.com/cvitek-develop-docs/master/docs_latest_release/CV180x_CV181x/zh/01.software/BSP/Secure_Boot_User_Guide/build/html/3_Secure_Image_Generation.html)
- [eFuse 使用指南](https://doc.sophgo.com/cvitek-develop-docs/master/docs_latest_release/CV180x_CV181x/zh/01.software/BSP/eFuse_User_Guide/build/html/index.html)
- [图解|什么是 RSA 算法](https://cloud.tencent.com/developer/article/1693368)
- [盘点五种最常见加密算法！](https://mp.weixin.qq.com/s/XBYJFZ2jRF2slrlym3svQw)
- [密码学基本概念](https://mp.weixin.qq.com/s/Hu1VBMSHh82n6bujWxr9nw)
