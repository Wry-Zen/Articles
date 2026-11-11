# 一：Linux 与 Android 的差异

>安卓乱炖大杂烩的开始

## Android 不是普通 Linux

- Android 使用 Linux Kernel，但没有沿用 GNU/Linux 发行版的传统用户空间体系。它采用 Bionic 作为 C 库，以 Android init 管理系统启动和服务，并使用 Android 自己的属性系统、IPC 机制和目录布局。
- 我们常说的Linux系统全称更精确来说叫GNU/Linux，而安卓如果禁用这种命名方式，不严谨的称呼可以叫做AOSP/Linux
- AOSP ≠ 完整的商业 Android 产品。AOSP 提供 Android 的开源基础，实际设备通常还会包含 SoC 厂商组件、设备厂商定制，以及可能存在的 Google Mobile Services（GMS）等专有组件。不同厂商和地区的设备在此基础上的差异也比较明显。本系列主要从 AOSP 视角展开。

### 启动链对比

```text
GNU/Linux:
UEFI → GRUB → Linux Kernel + initramfs → PID 1（systemd 等）
     → System Services → Login Shell

Android:
BootROM → Bootloader → Linux Kernel + ramdisk
       → Android init → Native Services
       → Zygote → system_server → Framework → App
```

>注：Android 的 Bootloader 因 SoC 而异——本文平台(RK3588S)是 U-Boot，高通是 XBL/ABL，MTK 是 preloader/LK

---

## 逐层差异表

>本节名词密集，混个脸熟即可，后续章节会按需要展开

| 层级 | GNU/Linux 发行版 | Android |
| ------ | ----------- | --------- |
| 硬件初始化 | BIOS/UEFI | SoC BootROM |
| 引导 | GRUB等 | U-Boot、ABL、LK 等 |
| 内核 | Linux | + Android 相关补丁及厂商驱动 |
| PID 1 | systemd、SysV init 等 | Android init（/init） |
| 服务管理 | systemd 等 | init + servicemanager + system_server |
| C 库 | glibc等 | Bionic |
| IPC | D-Bus socket pipe等 | Binder 为核心，同时使用 socket、共享内存等 |
| 图形 | X11/Wayland | SurfaceFlinger + HWC HAL |
| 应用管理 | apt、pacman、rpm 等 | PackageManager + APK |
| 目录布局 | FHS(/) | Android 专用目录布局 |
| 分区管理 | GPT 分区及发行版自身方案 | 静态分区、动态分区、A/B 等 |

>Android 与 GNU/Linux 发行版的主要区别在 Kernel→PID 1 及以后；之前的差异更多源于 SoC(ARM) 的设备形态，会在后面讲到

---

## 设计哲学差异

| | 标准 Linux | Android |
| --- | ----------- | --------- |
| 目标 | 通用多用户 OS | 以应用为中心，每 app 独立 UID |
| 设计优先级 | 兼容性、灵活性 | 性能、安全、资源效率 |
| 配置方式 | 文本文件(/etc) | Android Property(key-value) |
| 升级方式 | 包管理器逐包更新 | 分区镜像 OTA 整体替换(A/B 无缝升级) |
| 隔离方式 | 用户权限、namespace | 每 app 独立 UID 沙箱 + 强制访问控制(SELinux) + 系统调用过滤(seccomp) + 分区隔离 |

>Android 把 Linux 的多用户机制拿来隔离**应用**而非人——每个 app 一个 uid，这正是应用沙箱的基石，也是AOSP极为重要的权限管理方案(代价是应用间文件互通远不如桌面系统直觉)

---

## 为什么谷歌选择了自己蹚一条路

>Android init 设计于 2005-2007 年，systemd 2010 年公布、2011 年随 Fedora 15 进入主流。Google 不是"拒绝"了 systemd，而是当时根本没得选——桌面 Linux 主流还停留在 sysvinit/Upstart(当年的服务管理器)。所以更准确的问题是：**为什么 Google 没有沿用桌面 Linux 的用户空间**

### 1. init.rc VS 通用服务管理框架

- systemd 统筹一切：服务、定时器、状态监测、资源管理、日志，几乎接管整个系统
- Android 只需要按声明式脚本启动一组固定的、有序的服务，同时尽可能降低性能需求和功耗，用不上 systemd 那样的通用性，也不愿付出它的复杂度

### 2. Binder VS socket/D-Bus

- 手机场景 IPC 极频繁(60fps = 16ms/帧)。socket 每次事务要两次拷贝(用户→内核→用户)，且没有内核强制的发送方身份，需要应用层自行做权限检查
- Binder 提供内核强制的进程身份验证、引用计数、死亡通知；事务只需**一次拷贝**——数据直接拷入目标进程预分配的 mmap 缓冲区。此外还有 ashmem/dma-buf 这类零拷贝的共享方案

### 3. Bionic VS glibc

- glibc 是 LGPL 许可，静态链接对嵌入式不友好，且体积大
- Bionic 采用 BSD 许可、体积小、刻意只实现 POSIX 子集——为省资源不惜牺牲兼容性，这在桌面发行版看来是不可理喻的取舍

>这三点已经不难看出AOSP的理念全部基于***在资源受限的移动/嵌入式场景最大化的降低开销***，而恰巧手机就是那个受限的场景：有限的体积和电池供电

---

## 什么是AOSP？

>AOSP（Android Open Source Project）是 Android 的开源软件项目，包含 Linux Kernel 相关代码、用户空间系统组件、Framework、运行时以及应用开发接口等。Android 并不是在 Linux Kernel 之上简单替换几个系统工具，而是围绕移动设备重新构建了一整套操作系统架构。从 Bionic、Android init 到 Binder，再到 Zygote 和 Framework，这些组件共同构成了 Android 区别于传统 GNU/Linux 发行版的用户空间体系。

## 本系列路线

- **第一阶段(1.1-1.5)**：沿启动链从电源键走到 App——Bootloader(1.2) → Kernel(1.3) → init(1.4) → Zygote 与进程模型(1.5)
- **第二阶段(2.1-2.5)**：切换到源码树视角——AOSP 目录总览(2.1) → kernel 与 u-boot(2.2) → device 与 vendor(2.3) → hardware 与 frameworks(2.4) → system 与 BSP 总结(2.5)

>两条线最终会在同一处汇合，所有的差异最终都会凝聚为AOSP

初版2026-11-4
二改2026-11-11
