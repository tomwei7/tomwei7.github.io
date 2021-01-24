---
layout: post
title: Jetson Nano USB Boot 配置
tags: Jetson Nano,USB,Boot,SSD
---
Jetson Nano 是 NVIDIA 推出的 Aot 嵌入式平台，拥有 4 核心主频 1.43GHZ 的 Cortex-A57 CPU 和128组472GFLOPS （FP16）的Maxwell架构 GPU 以及 4GB LPDDR4 内存。 配置十分强大，可是和树莓派一样默认使用 SD 卡作为存储，SD 卡到性能直接影响了日常到使用体验。而大容量高速的 SD 卡价格实在感人。考虑到 Jetson Nano 拥有四个 USB3.0 5G 速率的接口，使用 USB 外接 SSD 也许是更好的解决方法。

这里首先介绍下 Jetson Nano 的启动过程，Jetson Nano 使用两阶段启动的方式，首先 Bootloader 会从 sdcard 加载一个比较小的 initrd (initial ramdisk)，可以理解为一个很小的系统，通过这个小系统再去加载 Linux Kernal。

```bash
+------------------+        +--------+       +--------------+
|                  |        |        |       |              |
|bootloader(u-boot)+------->+ initrd +------>+ Linux kernel |
|                  |        |        |       |              |
+------------------+        +--------+       +--------------+
```

Jetson Nano 官方文档并没有提到可以通过 USB 启动，默认的 initrd 系统也不包含 USB 驱动，所以initrd 没办法加载 USB 进行启动，Github 上有个项目叫做 [https://github.com/JetsonHacksNano/rootOnUSB](https://github.com/JetsonHacksNano/rootOnUSB) ，通过重新编译 initrd 添加 USB 驱动的方式，让 initrd 可以从 USB 设备启动 Linux。

但是经过尝试发现这种方式已经没有办法从 USB 启动 Jetson Nano 会让系统陷入无限重启的循环。考虑到这个项目最后更新还是17月以前，而现在 Jetson Nano 系统版本已经更新多次，可能已经无法兼容了。

无奈只能接上了串口可以调试启动过程，无意中发现现在版本的 U-Boot（Jetson Nano 的 Bootloader）在尝试检测 USB 设备，尝试翻阅 NVIDIA 相关文档也说明了这一点 [https://docs.nvidia.com/jetson/l4t/index.html#page/Tegra Linux Driver Package Development Guide/uboot_guide.html#wwpID0E06E0HA](https://docs.nvidia.com/jetson/l4t/index.html#page/Tegra%20Linux%20Driver%20Package%20Development%20Guide/uboot_guide.html#wwpID0E06E0HA)

> Boot Sequence and Sysboot Configuration Files
> U-Boot functionality includes a default booting scan sequence. It scans bootable devices in the following order:
> - External SD card
> - Internal eMMC
> - USB device or NVMe device (Jetson TX2 series only)
> - NFS network via DHCP/PXE

所以其实官方的 Bootloader 已经早就支持了 USB 启动了，但是由于默认的镜像是为 SD 卡准备的，并不能直接用于 USB 设备。这里需要进行一点点修改。

首先直接将下载的 Jetson Nano 的 SD 镜像写入 SSD 中这里的过程和写入 SD 卡一样，由于 SD 镜像的文件系统是 ext4 格式，这里需要一台 Linux 系统的机器来挂载 ext4 的文件系统并修改启动参数。

将写好的 SSD 盘使用 `parted` 打开，可以看到有这些分区

```bash
>> sudo parted /dev/sda
GNU Parted 3.2
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print
Model: ATA INTEL SSDSC2BB16 (scsi)
Disk /dev/sda: 160GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name     Flags
 2      1049kB  1180kB  131kB                TBC
 3      2097kB  2556kB  459kB                RP1
 4      3146kB  3736kB  590kB                EBT
 5      4194kB  4260kB  65.5kB               WB0
 6      5243kB  5439kB  197kB                BPF
 7      6291kB  6685kB  393kB                BPF-DTB
 8      7340kB  7406kB  65.5kB               FX
 9      8389kB  8847kB  459kB                TOS
10      9437kB  9896kB  459kB                DTB
11      10.5MB  11.3MB  786kB                LNX
12      11.5MB  11.6MB  65.5kB               EKS
13      12.6MB  12.8MB  197kB                BMP
14      13.6MB  13.8MB  131kB                RP4
 1      14.7MB  160GB   160GB   ext4         APP
```

需要挂载的分区是 APP 这里将 APP 分区即 `/dev/sda1` 挂载到 `/mnt`

```bash
sudo mount /dev/sda1 /mnt
```

打开 `/mnt/boot/extlinux/extlinux.conf` 文件，可以看到镜像默认将 root 指向了 `/dev/mmcblk0p1` 即 sd 卡的第一个分区

```bash
LABEL primary
      MENU LABEL primary kernel
      LINUX /boot/Image
      INITRD /boot/initrd
      APPEND ${cbootargs} quiet root=/dev/mmcblk0p1 rw rootwait rootfstype=ext4 console=ttyS0,115200n8 console=tty0 fbcon=map:0 net.ifnames=0
```

直接将 `/dev/mmcblk0p1` 改成 `/dev/sda1` 即可

```bash
LABEL primary
      MENU LABEL primary kernel
      LINUX /boot/Image
      INITRD /boot/initrd
      APPEND ${cbootargs} quiet root=/dev/sda1 rw rootwait rootfstype=ext4 console=ttyS0,115200n8 console=tty0 fbcon=map:0 net.ifnames=0
```

这里更加保险的做法应该是使用分区的 UUID 或者 LABEL 来指定磁盘，但是经过测试不管是 UUID 还是 LABEL 均会导致启动失败。所以要注意使用 USB 启动时，在 Jetson Nano 开机的时候只能插一个磁盘，不然可能会导致启动失败

现在从 Jetson Nano 中移除现有的 SD 卡，插入 USB SSD 重新启动就能从 USB 启动。

如果使用 USB 外接 SSD 需要注意，最好使用 5V4A 的电源，而不要使用 microUSB 供电。非常容易导致供电不足。

最后附上 USB SSD 与 SD 卡的性能测试，虽然 Jetson Nano 的 USB 3.0 接口比较拉胯，但是依然可以秒杀 SD 卡，平时使用在编译时也可以感觉到明显的性能提升。

```bash
# USB SSD (Intel DC S3500 160G)
READ: bw=263MiB/s (276MB/s), 263MiB/s-263MiB/s (276MB/s-276MB/s), io=8192MiB (8590MB), run=31169-31169msec
WRITE: bw=101MiB/s (106MB/s), 101MiB/s-101MiB/s (106MB/s-106MB/s), io=4096MiB (4295MB), run=40686-40686msec

# SDCard (KIOXIA EXCERIA 32G U1)
READ: bw=85.6MiB/s (89.8MB/s), 85.6MiB/s-85.6MiB/s (89.8MB/s-89.8MB/s), io=8192MiB (8590MB), run=95709-95709msec
WRITE: bw=12.2MiB/s (12.8MB/s), 12.2MiB/s-12.2MiB/s (12.8MB/s-12.8MB/s), io=512MiB (537MB), run=42007-42007msec
```
