# 二：Bootloader

![whereicome](whereicome.png)

Bootloader 是 SoC 上电后，BootROM 验证并加载的第一阶段可更新引导代码。它负责完成早期硬件初始化、加载后续启动组件，并最终将控制权交给 Linux Kernel。

需要注意，Bootloader 并不一定是一个独立的程序。在实际设备中，启动过程通常由多个阶段组成，不同阶段可能由不同的固件或引导程序完成。
以 Rockchip RK3588S 为例，典型启动链路可以简化为：

```text
SoC 上电 -> BootROM -> DDR 初始化固件 -> SPL / Loader
-> BL31（TF-A）-> BL32（可选，如 OP-TEE）-> U-Boot -> Linux Kernel
```

其中，BootROM 是固化在芯片内部的代码，负责从启动介质中读取并启动后续组件。DDR 初始化固件负责初始化外部内存；SPL、Loader 和 U-Boot 等组件则逐步完成后续启动工作。

具体启动链路会受到 SoC 厂商、固件版本和设备配置的影响，因此不同平台的实现可能存在差异。

---

## 1.Bootloader 的主要职责(了解为主)

Bootloader 的主要工作可以概括为以下几个方面

### 初始化硬件

在 Linux Kernel 启动之前，需要完成必要的硬件初始化，为后续程序提供运行环境，包括：

- 初始化 DDR，使系统能够使用外部内存
- 初始化时钟、电源管理和部分基础外设
- 配置必要的处理器运行环境

>这些工作不一定全部由 U-Boot 完成。在多阶段启动架构中，DDR 初始化通常由更早期的固件完成，其他硬件初始化也可能分散在不同启动阶段。

### 建立处理器运行环境

在将控制权交给 Linux Kernel 之前，需要确保处理器处于符合内核启动要求的状态。相关工作可能包括：

- 配置 CPU 异常级别（Exception Level）
- 设置 Cache 和 MMU 状态
- 初始化必要的中断控制器
- 准备内核启动所需的寄存器和参数

>在 ARM64 平台上，BL31（通常由 TF-A 提供）负责处理 EL3 相关工作，并提供 Secure Monitor 等安全服务。BL32 则是可选的安全世界组件，例如 OP-TEE。这些工作由多个启动阶段共同完成，并非全部属于 U-Boot 的职责。

### 建立安全启动环境

Android 设备通常会使用 Verified Boot（AVB）建立启动过程中的镜像验证机制。启动链中的各个阶段会根据验证规则检查后续启动组件，确保加载的固件和系统镜像符合预期。在 ARM TrustZone 架构中，还可以通过安全世界组件实现可信执行环境。例如：

- BL31：提供 EL3 Secure Monitor。
- BL32：可选的安全世界运行环境，例如 OP-TEE。
- TZASC：用于配置对特定内存区域的访问权限。
- SMC：Normal World 调用 EL3 服务的接口。

>这些机制分别承担启动验证、可信执行和安全资源隔离等职责。

### 加载 Linux Kernel

完成必要的硬件初始化和启动验证后，Bootloader 需要准备 Linux Kernel 的启动环境。在 Android 设备上，这通常涉及：

- 读取 boot.img、vendor_boot.img 等启动镜像。
- 根据 AVB 验证链验证相关镜像。
- 加载 Linux Kernel。
- 准备 ramdisk、DTB/DTBO 等启动数据。
- 设置 Kernel Command Line 和 Bootconfig。
- 将控制权交给 Kernel 入口。

>不同 Android 版本的镜像布局有所不同。例如，Android 12 及之后的设备可能使用 vendor_boot.img 和 init_boot.img 存放不同类型的 ramdisk 内容。因此，不能简单认为所有设备的 ramdisk 都位于 boot.img 中。

### 将控制权交给 Linux Kernel

完成上述工作后，Bootloader 会按照平台规定的内核启动协议准备寄存器、启动参数和设备树等信息，然后跳转到 Linux Kernel 的入口地址。
从这一刻开始，系统进入 Linux Kernel 的启动流程。

### 2. Android ramdisk

>这一段尤其需要着重展开，引导架构的变化和多阶段exec是安卓极具特色的探究尝试

在 Android 启动过程中，Bootloader 会根据设备的启动镜像布局准备 ramdisk，并将其交给 Linux Kernel。

Ramdisk 是一个基于内存的临时根文件系统，通常以压缩的 cpio archive 形式存储。Kernel 启动后，可以将其解包并挂载为初始根文件系统。Android 利用 ramdisk 提供早期用户空间环境，其中包含启动系统所需的关键程序和配置文件。

#### 2.1 Ramdisk 的主要内容

传统 Android ramdisk 通常包含：

| 文件 | 作用 |
| --- | --- |
| `/init` | Android 用户空间初始化程序 |
| `/init.rc` | init 的主要启动配置文件 |
| `/ueventd.rc` | 设备节点和权限相关配置 |
| `/fstab.*` | 分区挂载配置 |
| `prop.default` 等属性文件 | 提供系统启动所需的默认属性 |
| SELinux 相关文件 | 提供早期安全策略相关配置 |

其中，/init 是整个 Android 用户空间启动流程的入口。Linux Kernel 启动完成后会执行它并将其作为用户空间的第一个进程，即 PID 1。

#### 2.2 传统 Android：ramdisk

在Android 8.x 及以前的典型布局中，ramdisk 不仅仅是一个启动环境，它本身就是系统运行期间的根文件系统

```text
/  (ramdisk)
├── init
├── init.rc
├── ueventd.rc
├── fstab.*
├── dev/
├── proc
├── system/       ← system.img 挂载点
├── vendor/       ← vendor.img 挂载点
└── data/         ← data 分区挂载点
```

这种布局下，ramdisk 始终保留在根目录中，而系统分区则挂载到对应的目录。由于 ramdisk 是基于内存的临时文件系统，因此对其中内容进行的运行时修改不会写回原始启动镜像。设备重启后，这些修改也不会保留。这也是为什么修改传统 Android ramdisk通常需要重新打包 boot.img 并刷入设备。

#### 2.3 System-as-root：根文件系统的重新组织

随着 Android 版本演进，Android 引入了 System-as-root（SAR）启动布局。SAR 的核心变化在于最终根文件系统由 system 分区提供而不是始终由传统 ramdisk 提供。需要注意，Android 9 和 Android 10 的 SAR 实现并不完全相同

在部分 Android 9 SAR 设备中，系统可以直接使用 system 分区提供根文件系统。例如，根目录下的 /init 可以是一个指向 /system/bin/init 的符号链接。这种布局不一定需要传统意义上的 ramdisk 根切换流程，具体实现取决于设备的分区布局和启动方式。由于存在时间极短这里只简单带过。

Android 10 引入了新的启动布局。在采用 ramdisk 启动布局的设备上，ramdisk 仍然承担早期启动环境的职责。First-stage init 会先挂载 system 等必要分区，然后通过 switch_root 将根文件系统切换到 system 提供的文件系统。其过程可以简化为：

```text
Kernel 启动 -> 挂载初始 ramdisk -> 执行 /init ->
First-stage init -> 挂载 system 等分区 -> switch_root ->
system 成为最终根文件系统 -> 继续执行 init 后续阶段
```

SAR 并不意味着 ramdisk 完全消失，而是将早期启动环境与最终根文件系统的职责分离，后文会详细讲解包括First-stage init在内的三大启动阶段

#### 2.4 init与自己的三次交接棒

从内核拉起init到init开始初始化完整系统

```text
1.Kernel 挂载初始根文件系统，启动 /init（PID 1）

2.First-stage init 执行早期初始化
挂载必要的系统分区，如 system、vendor

3.执行 switch_root，将 system 分区对应的挂载切换为新的根文件系统 /

4.原始 ramdisk 不再需要，释放其相关内容

5.exec /system/bin/init selinux_setup
6.exec /system/bin/init second_stage

7.init开始拉起其他进程
```

相比常见的 GNU/Linux 发行版启动方式，Android init 的一个显著特点是将 PID 1 的早期启动过程明确拆分为 First-stage init、SELinux setup 和 Second-stage init，并通过多次 exec 完成阶段交接。这种设计与 Android 对设备驱动、动态分区以及 SELinux 的启动依赖密切相关。[AOSP源码][INIT]直接写明了三种不同的参数对应不同的分支

```code
if (argc > 1) {
        if (!strcmp(argv[1], "subcontext")) {
            android::base::InitLogging(argv, &android::base::KernelLogger);
            const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();
            return SubcontextMain(argc, argv, &function_map);
        }
        if (!strcmp(argv[1], "selinux_setup")) {
            return SetupSelinux(argv);
        }
        if (!strcmp(argv[1], "second_stage")) {
            return SecondStageMain(argc, argv);
        }
      }
```

#### 2.4.1 为什么需要 First-stage init

First-stage init 主要负责完成系统进入正常用户空间之前必须完成的工作。其中一个重要任务是加载必要的内核模块。Android 设备的硬件平台和驱动实现存在较大差异。部分设备驱动并没有直接编译进 Kernel，而是以可加载内核模块（LKM）的形式提供。~~***其实就是ARM自己稀烂且零碎的生态导致各家都一堆私有驱动***~~
如果挂载 system 或 vendor 分区所依赖的驱动模块恰好位于这些分区中，就会产生无法解决的鸡生蛋问题：**挂载system分区需要驱动，而驱动又位于system分区内需要挂载才能读取**。为了解决这个问题，设备可以将早期启动所需的驱动模块放入 ramdisk 或 vendor ramdisk 中，由 First-stage init 提前加载。
Android 12 引入 GKI后这种机制更加重要。GKI 提供通用内核基础，但设备厂商的部分硬件驱动仍然需要通过厂商模块提供。

#### 2.4.2 为什么 SELinux 初始化被放在 First-stage 和 Second-stage 之间？

SELinux 是 Android 安全机制的重要组成部分，用于限制进程能够访问的资源。
在 First-stage init 执行期间，系统还没有加载完整的 SELinux policy。此时 init 运行在内核提供的初始 SELinux domain 中而不是正常的 init domain。但完整的 SELinux policy 通常位于 system、vendor 等分区中。因此系统必须先完成早期分区挂载才能读取并加载这些策略文件，[源码中的注释][SE]也证明了这点

谷歌选择了将 SELinux setup 放在一二阶段中间

```text
First-stage init -> 挂载 system / vendor 等分区 ->
SELinux setup -> 加载 SELinux policy ->
Second-stage init -> 解析 init.rc，启动系统服务
```

这种安排带来了明确的安全收益：在完整的SELinux policy 可用后尽早启用强制访问控制，缩短 init 在完整策略尚未加载时运行的时间，减少早期启动阶段缺少 SELinux 强制访问控制约束的操作范围。

需要说明的是，在 SELinux policy 加载之前系统并不是完全没有安全限制。Linux 本身的 DAC 权限控制、UID/GID、Capabilities 等机制仍然存在，只是 Android 完整的 SELinux 强制访问控制策略尚未生效。

在源码实现中，SELinux setup 阶段还会设置 `/system/bin/init` 的安全上下文，并通过后续的 `exec` 触发 SELinux domain transition，使 init 进入正常的 init domain，最后进入拉起完整系统的 Second-stage

#### 2.4.3 为什么每个阶段都要使用exec？

虽然 First-stage init、SELinux setup 和 Second-stage init 是三个不同的启动阶段，但它们始终运行在同一个进程中，PID 始终为 1。

> 所以为什么不直接创建新的 init 进程？

因为 PID 1 在 Linux 中具有特殊地位，它是用户空间的第一个进程，承担系统初始化、孤儿进程接管和子进程回收等职责。通过 fork() 创建的子进程会获得新的 PID，无法取代当前的 PID 1。

因此，Android 使用 exec 替换当前进程的程序映像，而不是创建新的进程。这样既能让不同阶段拥有独立的程序执行环境，又能保证整个启动过程中 PID 1 的身份不发生变化。

关于 exec 的具体实现，可以参考：[Linux创建进程的奇怪方式][EXEC]

#### 2.5 总结

从传统 ramdisk 到 System-as-root，最核心的变化并不是简单地改变 system.img 的挂载位置，而是重新定义了根文件系统的来源和生命周期。

传统方案中，ramdisk 同时承担早期启动环境和最终根文件系统的职责；而新的方案将这两个职责分离：ramdisk 负责早期启动，system 分区提供最终根文件系统。

这一变化也为后续 Android 启动架构的演进提供了基础。后文的GKI将会再次见面

---

初版2026-11-11

[INIT]: https://cs.android.com/android/platform/superproject/+/android-latest-release:system/core/init/main.cpp?hl=zh-cn "源码网站"
[EXEC]: https://www.bilibili.com/video/BV1hjkEBkEtb
[SE]: https://cs.android.com/android/platform/superproject/+/android-latest-release:system/core/init/selinux.cpp;l=1?q=selinux.cpp&sq=&ss=android%2Fplatform%2Fsuperproject&hl=zh-cn "selinux.cpp line20~22"
