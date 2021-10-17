---
title: "Linux 启动流程"
description: ""
date: 2021-02-24T22:05:45+08:00
lastmod: 2021-02-24T22:05:45+08:00
draft: false
images: []
menu:
  infrastructure:
    parent: "Linux"
weight: 800
---

启动一台 Linux 机器的过程分为两个部分：Boot 和 Startup。其中，Boot 起始于计算机启动，在内核初始化完成且 Systemd 进程开始加载后结束。紧接着， Startup 接管任务，使计算机达到一个用户可操作的状态。

## Boot

如上图所示，Boot 过程又可以详细地分为三个部分：

- BIOS POST
- Boot Loader
- 内核初始化

### BIOS POST

开机自检（Power On Self Test，POST）是 [基本输入输出系统](https://zh.wikipedia.org/wiki/BIOS)（Basic I/O System，BIOS）的一部分，也是启动 Linux 机器的第一个步骤。其工作对象是计算机硬件，因此对于任何操作系统都是相同的。**POST 检查硬件的基本可操作性**，若失败则 Boot 过程将会被终止。

POST 检查完毕后会发出一个 BIOS 中断调用 [INT 13H](https://en.wikipedia.org/wiki/INT_13H)，它将在任何可连接且可引导的磁盘上搜索含有有效引导记录的引导扇区（boot sector），通常是 [主引导扇区](https://zh.wikipedia.org/wiki/%E4%B8%BB%E5%BC%95%E5%AF%BC%E8%AE%B0%E5%BD%95)。引导扇区中的主引导记录（Master Boot Record，MBR）将被加载到 RAM 中，然后控制权就会转移到其手中。

### Boot Loader

大多数 Linux 发行版使用三种 Boot Loader 程序：GRUB、GRUB2 和 LILO，其中 GRUB2 是最新且使用最为广泛的。GRUB2 代表“GRand Unified Bootloader, version 2”，**它能够让计算机找到操作系统内核并将其加载到内存中**。GRUB2 还允许用户选择从几种不同的内核中引导计算机，如果更新的内核版本出现兼容性问题，我们就可以恢复到先前内核版本。GRUB2 的引导过程可以分为以下三个阶段：

#### stage 1

上文提到，BIOS 中断调用会定位主引导扇区，其结构如下图所示：

![20211015161614](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20211015161614.png)

主引导记录首部的引导代码便是 stage 1 文件 boot.img，大小仅有 446 字节。其作用是检查分区表是否正确，然后**定位和加载 stage 1.5**。当 stage 1.5 加载到 RAM 后，控制权也随之转移。

#### stage 1.5

446 字节的 stage 1 文件放不下能够识别文件系统的代码，只能通过计算扇区的偏移量来定位和加载 stage 1.5，因此 stage 1.5 文件 core.img 必须位于主引导记录本身和驱动器的第一个分区（partition）之间。第一个分区从扇区 63 开始，与位于扇区 0 的主引导记录之间有 62 个扇区（每个 512 字节），因此有足够的空间存储大小为 25389 字节的 core.img 文件。

core.img 文件中包含一些常见的文件系统驱动程序，如 EXT、FAT和NTFS等，它和 boot.img 文件均位于 /boot/grub2 目录下：

```shell
[root@bastion ~]# ls /boot/grub2/i386-pc/ | grep img
boot.img
core.img
```

core.img 文件可以识别文件系统，因此它的作用是根据系统路径**定位和加载 stage 2**。同样，当 stage 2 加载到 RAM 后，控制权也随之转移。

#### stage 2

stage 2 文件并非是一个 .img 的镜像，而是一些运行时内核模块：

```shell
[root@bastion ~]# ls /boot/grub2/i386-pc/ | grep .mod | head
acpi.mod
adler32.mod
affs.mod
afs.mod
ahci.mod
all_video.mod
aout.mod
appendedsig.mod
appended_signature_test.mod
archelp.mod
```

它们的任务是根据 grub.cfg 文件的配置**定位和加载内核文件**，然后将控制权转交给 Linux 内核。

### 内核初始化

不同内核及其相关文件位于 /boot 目录中，均以 vmlinuz 开头：

```shell
[root@bastion ~]# ls /boot/ | grep vmlinuz
vmlinuz-0-rescue-20210623110808105647395700239158
vmlinuz-4.18.0-305.12.1.el8_4.x86_64
vmlinuz-4.18.0-305.3.1.el8.x86_64
```

## Startup

## 参考文献

[BIOS - Wikipedia](https://zh.wikipedia.org/wiki/BIOS)

[INT 13H - Wikipedia](https://en.wikipedia.org/wiki/INT_13H)

[主引导记录](https://zh.wikipedia.org/wiki/%E4%B8%BB%E5%BC%95%E5%AF%BC%E8%AE%B0%E5%BD%95)

[An introduction to the Linux boot and startup processes](https://opensource.com/article/17/2/linux-boot-and-startup)
