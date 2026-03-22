# Linux 启动流程六列表

1️⃣ QEMU 启动虚拟硬件
        ↓
2️⃣ 加载内核 bzImage
        ↓
3️⃣ 加载 initramfs 到内存
        ↓
4️⃣ 内核启动（初始化硬件）
        ↓
5️⃣ initramfs 被挂载为 rootfs（/）
        ↓
6️⃣ 内核执行 /init
        ↓
7️⃣ init 挂载 proc 和 sys
        ↓
8️⃣ init 启动 shell（busybox）
        ↓
9️⃣ 你看到命令行 #

| 阶段                               | 日志关键字                                                   | 正在干什么                                                   | 为什么必须现在做                                             | 属于哪个子系统                       | 后续影响                                                     |
| ---------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------ | ------------------------------------------------------------ |
| 1. 内核自报家门                    | `Linux version 6.6.87`                                       | 内核打印自己的版本、编译信息，说明现在已经跳进内核执行了     | 必须先确认“当前跑的是谁”，否则后面所有日志都没有上下文       | 内核启动主线 / `init` / 架构早期启动 | 你能确认这是你自己编译的内核，后续改源码都能在这里验证       |
| 2. 读取启动参数                    | `Command line: console=ttyS0`                                | 内核读取 bootloader 传下来的启动参数                         | 启动参数会影响 console、root、调试方式等，必须最早知道       | 启动参数解析 / early boot            | 决定后面日志输出到串口 `ttyS0`，也影响启动行为               |
| 3. 识别机器平台                    | `DMI: QEMU Standard PC (i440FX + PIIX, 1996)`                | 识别当前硬件/虚拟平台类型                                    | 内核必须知道自己跑在什么平台上，后面很多设备初始化路径取决于平台 | x86 平台识别 / DMI / 固件信息        | 决定你后面会看到 PIIX、E1000、QEMU DVD-ROM、PS/2 等典型 QEMU 设备 |
| 4. 接收物理内存地图                | `BIOS-provided physical RAM map` / `BIOS-e820`               | BIOS/固件把物理地址空间哪些可用、哪些保留告诉内核            | 内核不能一上来就乱用内存，必须先知道哪些物理地址安全可用     | x86 内存发现 / e820                  | 后面 node、zone、页分配器都以这个为基础                      |
| 5. 修正固件内存图                  | `e820: update ... usable ==> reserved` / `e820: remove ...`  | 内核修正一些不该分配的特殊地址范围，比如低端保留页、传统 PC 洞 | 固件给的图不一定能原样照搬，内核必须先过滤危险区域           | x86 内存管理早期阶段                 | 防止把 BIOS/VGA/保留空间误当普通内存使用                     |
| 6. 确定物理页范围                  | `last_pfn = ...`                                             | 把物理内存换算成 Linux 内部的页框编号范围                    | Linux 内存管理按“页”工作，不先算 PFN，后面无法建页分配器     | 物理内存管理 / 页框模型              | 决定总页数、zone 边界、伙伴系统规模                          |
| 7. 开启基本执行保护                | `NX (Execute Disable) protection: active`                    | 启用不可执行页等硬件执行保护能力                             | 安全能力应尽早建立，越早越能防止错误执行或利用               | x86 CPU 特性 / 内存保护              | 后面内核和用户空间页表可使用 NX 属性增强安全性               |
| 8. 识别 CPU 计时能力               | `tsc: Fast TSC calibration using PIT` / `tsc: Detected ... MHz` | 校准 TSC，估算 CPU 计时频率                                  | 后面延时、调度、定时器都离不开时间基准，必须很早建立         | 时钟系统 / CPU 计时                  | 为 delay loop、clocksource 选择、调度时间统计提供基础        |
| 9. 发现 initramfs 所在内存         | `RAMDISK: [mem ...]`                                         | 确认 initramfs/initrd 镜像在内存中的位置                     | 后面要解压早期根文件系统，必须先知道压缩包在哪               | 启动镜像 / initramfs                 | 为后面 `Unpacking initramfs...` 做准备                       |
| 10. 读取 ACPI 表入口               | `ACPI: RSDP / RSDT / FACP / DSDT / APIC / HPET`              | 读取平台 ACPI 描述表，获取电源、中断、设备拓扑信息           | 在 x86 上，很多平台信息不是猜的，而是必须从 ACPI 表读        | ACPI / 平台描述                      | 后面电源管理、APIC、PCI 根桥、中断路由都依赖 ACPI            |
| 11. 保留 ACPI 表内存               | `ACPI: Reserving ... table memory`                           | 把 ACPI 表所在物理页标记为保留，避免被页分配器覆盖           | 这些表后面运行过程中还要读取，所以现在必须保护               | ACPI / 内存保留                      | 防止平台描述数据被破坏，确保中断和设备信息可继续使用         |
| 12. 建立 NUMA/node 框架            | `No NUMA configuration found` / `Faking a node` / `NODE_DATA(0)` | 没有真实 NUMA，就伪造一个 node 0，建立统一内存节点模型       | Linux 内部很多内存路径都基于 node/zone 框架，即使单节点也要建 | 内存管理 / NUMA 抽象                 | 后面 zonelist、页分配器、per-node 数据都可统一处理           |
| 13. 划分内存 zone                  | `Zone ranges:` / `DMA` / `DMA32` / `Normal empty`            | 把物理内存划分为 DMA、DMA32 等区域                           | 不是所有设备都能访问所有内存，必须先分区，后面 DMA 分配才安全 | 页分配器 / zone 管理                 | 驱动申请 DMA 内存时会受 zone 限制                            |
| 14. 建立早期可用内存区间           | `Early memory node ranges` / `Initmem setup node 0`          | 把 usable 区域正式转成 Linux 内部可管理的早期内存范围        | 只有完成这个步骤，内核才能逐步接管“可分配内存”               | 早期内存初始化 / `mm`                | 为伙伴系统、slab、页描述符建立准备条件                       |
| 15. 统计 zone 中不可用页           | `zone DMA: ... unavailable ranges`                           | 在 zone 内标记那些虽然落在范围内，但不能分配的洞             | zone 不是一个全连续大块，必须先剔除洞，避免错误分配          | 页分配器 / zone 细化                 | 提高后续页分配准确性和稳定性                                 |
| 16. 初始化 APIC/中断拓扑           | `IOAPIC[0] ...` / `ACPI: INT_SRC_OVR` / `LAPIC_NMI`          | 建立本地 APIC、IOAPIC、中断映射关系                          | CPU 能处理中断，设备才能工作；所以中断体系必须早起           | 中断子系统 / x86 APIC                | 后面 PCI 设备、时钟中断、串口中断都依赖它                    |
| 17. 读取 HPET 和 PM timer          | `ACPI: HPET ...` / `ACPI: PM-Timer IO Port`                  | 发现高精度定时器和 ACPI PM 计时器                            | 时间源需要多个候选项，内核早期先枚举出来，后面才能择优切换   | 时钟系统 / ACPI                      | 为 `clocksource` 最终选择提供备选                            |
| 18. 确定 CPU 数量                  | `smpboot: Allowing 1 CPUs`                                   | SMP 框架决定本次启动最多启用几个 CPU                         | 调度、percpu、RCU 都依赖 CPU 拓扑，必须尽早确定              | SMP / CPU 拓扑                       | 后面出现 `1 CPU`，说明是单核虚拟机                           |
| 19. 建立时钟源初始版本             | `clocksource: refined-jiffies`                               | 先启用一个简单、稳定的基础时钟源                             | 刚开机时未必所有高级时钟都准备好了，先有个能工作的很重要     | 时钟系统 / clocksource               | 后面会被更好的 TSC/HPET 替代                                 |
| 20. 建立 percpu 区域               | `setup_percpu` / `percpu: Embedded ... pages/cpu`            | 为每个 CPU 分配专属变量区域                                  | 调度器、RCU、统计、软中断等都要 percpu 数据，必须早建        | percpu / SMP 基础设施                | 后面大量 CPU 本地状态都能无锁或低锁访问                      |
| 21. 建立 dentry/inode 等缓存       | `Dentry cache hash table entries` / `Inode-cache hash table entries` | 初始化 VFS 关键缓存结构                                      | 文件系统以后一定会用到，路径查找和 inode 管理不能等太晚      | VFS / slab / 哈希缓存                | 后面挂载文件系统、查找文件、执行 `/init` 都要靠它            |
| 22. 建立 zonelist 和分配策略       | `Built 1 zonelists` / `Policy zone: DMA32`                   | 确定内存分配时 zone 的回退顺序和策略                         | 没有分配策略，页分配器还不能稳定工作                         | 页分配器 / zonelist                  | 后面各种 `alloc_pages`、slab 分配都按这个顺序走              |
| 23. 报告可用内存总量               | `Memory: 86012K/130552K available`                           | 给出总内存与当前已可用内存统计                               | 到这一步内核已经基本完成对物理内存的初步接管                 | 内存管理总览                         | 让你知道不是所有 RAM 都会直接变成“可用内存”                  |
| 24. 初始化 SLUB                    | `SLUB: ...`                                                  | 启动内核小对象分配器                                         | 后面大量内核对象不是按整页分配，而是按 slab/cache 分配       | slab/SLUB 分配器                     | 影响 dentry、inode、task_struct 等对象的分配效率             |
| 25. 初始化 RCU                     | `rcu: Preemptible hierarchical RCU implementation`           | 建立 RCU 并发读优化机制                                      | 很多内核全局数据结构的并发访问很早就要用，必须先把 RCU 启起来 | 并发控制 / RCU                       | 后面调度、VFS、网络等子系统都可能依赖 RCU                    |
| 26. 初始化中断号空间               | `NR_IRQS: ... nr_irqs: ...`                                  | 建立整个系统的 IRQ 编号和预留空间                            | 没有 IRQ 管理框架，设备中断无法注册                          | 中断管理                             | 后续串口、网卡、键盘等驱动才能申请中断                       |
| 27. 启用 console                   | `Console: colour VGA+ 80x25` / `printk: console [ttyS0] enabled` | 把 printk 正式绑定到可工作的控制台设备上                     | 前面日志可能是早期输出机制，正式 console 必须尽快接管        | printk / console / TTY               | 以后日志会稳定输出到串口 `ttyS0`                             |
| 28. 切换到对称 I/O 中断模式        | `APIC: Switch to symmetric I/O mode setup`                   | 中断处理从传统模式进入 APIC 对称 I/O 模式                    | 现代设备和多 CPU 中断管理都依赖它                            | 中断子系统 / APIC                    | 后面 PCI IRQ 路由更加标准化                                  |
| 29. 切入更好的早期时钟源           | `clocksource: tsc-early` / `Calibrating delay loop`          | 用更高性能的 TSC 计时替换简单时钟                            | 系统运行起来后，需要更快更准的计时基础                       | 时钟系统                             | 提升时间统计、调度、延时精度                                 |
| 30. 启用 CPU 安全与 FPU 特性       | `Spectre V1/V2` / `x86/fpu: x87 FPU will use FXSAVE`         | 配置投机执行缓解和 FPU 保存恢复机制                          | CPU 运行特性应在用户态和复杂内核路径开启前处理好             | x86 CPU 特性 / 安全                  | 影响安全性、上下文切换、浮点状态管理                         |
| 31. 初始化安全框架                 | `LSM: initializing lsm=...` / `SELinux: Initializing`        | 启动 Linux Security Modules 框架和 SELinux                   | 安全策略必须在系统更复杂之前接入核心路径                     | 安全框架 / LSM                       | 后面文件访问、进程权限、对象创建会进入 LSM 钩子              |
| 32. 真正识别 CPU 并上线            | `CPU0: AMD QEMU Virtual CPU...` / `smp: Brought up 1 node, 1 CPU` | 把 CPU0 正式纳入可调度运行状态                               | 没有可运行 CPU，后面的线程、软中断、用户态都无从谈起         | SMP / 调度基础                       | 系统进入“正式可运行”状态                                     |
| 33. 初始化 devtmpfs                | `devtmpfs: initialized`                                      | 准备 `/dev` 设备节点的基础设施                               | 后面设备驱动会不断注册设备，没有 devtmpfs 很难暴露给用户空间 | 设备模型 / devtmpfs                  | 用户空间后续能看到 `/dev/ttyS0`、`/dev/sr0` 等               |
| 34. 读取 RTC 时间                  | `PM: RTC time: ... date: ...`                                | 从硬件 RTC 读取当前时间                                      | 需要尽快给系统一个现实世界时间基准                           | RTC / 时间管理                       | 影响日志时间、文件时间戳、用户空间时间初始化                 |
| 35. 初始化网络核心协议族           | `NET: Registered PF_NETLINK/PF_ROUTE`                        | 先注册网络协议框架本身，而不是某块网卡                       | 没有网络协议栈，网卡驱动就算起来也不能真正提供网络服务       | 网络协议栈                           | 为 IPv4/IPv6、本地 socket、netlink 等铺路                    |
| 36. 允许 PCI 配置访问              | `PCI: Using configuration type 1 for base access`            | 决定用什么方式访问 PCI 配置空间                              | 不先能读 PCI 配置空间，就无法发现 PCI 设备                   | PCI 总线                             | 后面正式开始枚举 host bridge、VGA、IDE、网卡                 |
| 37. 让 ACPI 解释器上线             | `ACPI: Interpreter enabled` / `ACPI: PM: (supports S0 S3 S4 S5)` | 正式启用 ACPI 解释执行环境，读取平台电源和设备方法           | 很多 ACPI 设备方法和资源分配必须依赖解释器                   | ACPI                                 | 支持电源按钮、睡眠态、设备方法调用                           |
| 38. 建立 PCI 根桥                  | `ACPI: PCI Root Bridge [PCI0]` / `PCI host bridge to bus 0000:00` | 确定 PCI 树的根，总线范围和窗口                              | 没有 root bridge，就没有后续 PCI 设备树可枚举                | PCI / ACPI                           | 后面 bus 00 上所有设备才能出现                               |
| 39. 宣告 PCI 资源窗口              | `root bus resource [io ...]` / `[mem ...]`                   | 给 PCI 设备准备 I/O 端口和 MMIO 的地址窗口                   | 驱动访问设备寄存器必须先有资源分配范围                       | PCI 资源管理                         | 后续 BAR 映射和驱动寄存器访问依赖这些窗口                    |
| 40. 枚举 PCI 设备                  | `pci 0000:00:00.0` / `00:01.1` / `00:02.0` / `00:03.0`       | 扫描总线上每个设备，读取 vendor/device/class                 | 必须先“发现设备”，驱动才能谈得上匹配和绑定                   | PCI 枚举                             | 发现了 host bridge、PIIX3/PIIX4、IDE、VGA、E1000 网卡        |
| 41. 读取并分配设备资源             | `reg 0x10` / `reg 0x14` / `legacy IDE quirk`                 | 识别各设备 BAR、I/O 端口、MMIO、特殊兼容模式                 | 驱动后面访问设备寄存器前，必须先知道地址在哪                 | PCI 资源 / 驱动准备                  | IDE、VGA、网卡后面才能真正被对应驱动控制                     |
| 42. 配置 PCI 中断链路              | `ACPI: PCI: Interrupt link LNKA ...`                         | 把 PCI 设备的中断路由到合适 IRQ                              | 设备不只是有寄存器，还要能发中断通知 CPU                     | PCI IRQ 路由 / ACPI / APIC           | 后面网卡、IDE、串口等设备中断才能工作                        |
| 43. 初始化 SCSI/USB/基础设备框架   | `SCSI subsystem initialized` / `usbcore: registered ...`     | 启动总线和中层框架，等待具体设备驱动挂接                     | 没有框架，具体驱动无法统一接入                               | SCSI 子系统 / USB 核心 / 驱动框架    | 光驱、USB 存储、HID 以后都能按统一方式出现                   |
| 44. 切换到更稳定时钟源             | `clocksource: Switched to clocksource tsc-early`             | 经过检测后，把系统正式切到更合适的 clocksource               | 启动更后期已经有足够信息，可以选择性能更好的时间源           | 时钟系统                             | 提高整个系统运行时的计时质量                                 |
| 45. 初始化 PnP ACPI                | `pnp: PnP ACPI init` / `found 6 devices`                     | 识别非 PCI 的传统平台设备                                    | 不是所有设备都挂在 PCI 上，键盘控制器、RTC 等常走 PnP/ACPI 路径 | PnP / ACPI 设备模型                  | 后面 i8042、RTC、电源按钮等能被发现                          |
| 46. 初始化 IPv4/本地 socket 等协议 | `NET: Registered PF_INET` / `PF_UNIX/PF_LOCAL`               | 把常用网络协议族注册到内核                                   | 用户空间和内核都会用到 socket，必须先建协议栈                | 网络子系统                           | 后面 eth0 上线后，IPv4/本地 IPC 立即可用                     |
| 47. 解压 initramfs                 | `Unpacking initramfs...`                                     | 把早期根文件系统从内存压缩包中展开                           | 后面要过渡到用户空间，必须先有一个可工作的初始根环境         | initramfs / 启动镜像                 | 为执行 `/init`、加载更多内容、挂载真实 root 做准备           |
| 48. 初始化密钥与证书框架           | `Initialise system trusted keyrings` / `x509`                | 建立系统可信密钥和证书解析能力                               | 模块签名、firmware、regulatory 数据等可能很早就需要验证      | 密钥环 / 安全证书                    | 后面加载内建证书、校验某些数据                               |
| 49. 释放 initrd 原始内存           | `Freeing initrd memory: 2148K`                               | initramfs 解压完成后，原始压缩镜像内存被释放                 | 启动时临时占用的内存不能一直浪费，能回收就立刻回收           | 内存回收 / initramfs                 | 增加系统可用内存                                             |
| 50. 注册电源按钮                   | `input: Power Button` / `ACPI: button: Power Button [PWRF]`  | 把 ACPI 电源按钮注册成输入设备                               | 平台基础控制设备要尽快就位                                   | ACPI / input 子系统                  | 用户空间以后能响应按电源键关机                               |
| 51. 初始化串口驱动                 | `Serial: 8250/16550 driver` / `ttyS0 ... is a 16550A`        | 正式初始化标准串口控制器并识别 COM1                          | 你命令行指定了串口 console，所以串口必须真正被驱动接管       | TTY / serial / 8250                  | 串口日志、串口 shell、调试输出都更稳定                       |
| 52. 初始化块设备/loop              | `loop: module loaded`                                        | 启用 loop 设备等块设备基础设施                               | 文件系统镜像、某些挂载方式可能会用到                         | 块设备层                             | 为后面更复杂存储操作留好接口                                 |
| 53. 初始化 ATA/IDE 控制器          | `scsi host0: ata_piix` / `ata1` / `ata2`                     | PIIX IDE 控制器被 `ata_piix` 驱动接管，建立 ATA 通道         | 没有控制器驱动，磁盘/光驱设备不会出现                        | ATA/libata / 存储驱动                | 后面 QEMU DVD-ROM 才能被识别为块设备                         |
| 54. 注册网卡驱动代码               | `e100` / `e1000` / `e1000e`                                  | 内核把若干 Intel 网卡驱动注册到系统                          | 必须先有驱动可用，才能去匹配具体 PCI 网卡                    | 网络驱动层                           | 后面 `8086:100e` 设备会被 e1000 绑定                         |
| 55. 识别并注册光驱                 | `ata2.00: ATAPI: QEMU DVD-ROM` / `sr0`                       | ATA 控制器发现 ATAPI 光驱，并通过 SCSI 中层注册为 `sr0`      | 设备发现后必须转换成统一块设备抽象供上层使用                 | libata / SCSI 中层 / 块设备          | 用户空间后面能访问 `/dev/sr0`                                |
| 56. 网卡驱动绑定硬件               | `e1000 0000:00:03.0 eth0`                                    | PCI 网卡 `8086:100e` 被 e1000 驱动绑定，创建设备 `eth0`      | 网络协议栈已经起来了，现在具体硬件终于能接入                 | 网卡驱动 / netdev                    | 这时系统才真正拥有可用网络接口                               |
| 57. 初始化键盘控制器               | `i8042: PNP: PS/2 Controller` / `serio: i8042 KBD`           | 识别传统 PS/2 控制器和键盘/鼠标端口                          | 输入设备要先有控制器层，再有具体键盘设备                     | 输入子系统 / serio / i8042           | 后面键盘被注册为 input 设备                                  |
| 58. 初始化 RTC 设备                | `rtc_cmos ... registered as rtc0`                            | CMOS RTC 驱动绑定设备，注册为 `rtc0`                         | 前面只是读了时间，这里是正式把 RTC 变成设备对象              | RTC 驱动 / 平台设备                  | 用户空间可通过 `/dev/rtc0` 访问 RTC                          |
| 59. 初始化 HID/USB 输入支持        | `usbhid: USB HID core driver`                                | 建立 USB 人机输入设备支持                                    | 后续若有 USB 键盘鼠标需要统一接入                            | USB / HID 子系统                     | 为更广泛的输入设备留接口                                     |
| 60. 初始化 IPv6 与原始报文接口     | `PF_INET6` / `PF_PACKET`                                     | 扩展网络协议支持到 IPv6、packet socket                       | 网络功能逐渐补齐，满足更完整系统能力                         | 网络协议栈                           | 抓包、IPv6、隧道等功能可用                                   |
| 61. 开启 netconsole                | `printk: console [netcon0] enabled` / `netconsole: network logging started` | 把日志也可以发到网络 console                                 | 系统已有网络栈和网卡后，可以扩展日志输出能力                 | printk / netconsole / 网络           | 远程调试和网络日志采集可用                                   |
| 62. 加载内建证书                   | `Loading compiled-in X.509 certificates`                     | 加载编译进内核的可信证书                                     | 安全相关框架已就位，这时可正式导入证书                       | 安全 / 密钥环                        | 支持签名校验、证书验证等                                     |
| 63. 尝试加载监管数据库             | `cfg80211: failed to load regulatory.db`                     | 无线监管数据库加载失败                                       | 这是无线框架的附属资源，不是启动主线的关键错误               | cfg80211 / firmware 加载             | 对你当前这台没有无线网卡的 QEMU 机器基本没影响               |
| 64. 报告无声卡                     | `No soundcards found`                                        | ALSA 扫描后没有发现声卡                                      | 声音不是启动关键路径，能否找到声卡不影响系统核心启动         | ALSA / 声卡子系统                    | 对当前虚拟机通常是正常现象                                   |
| 65. 释放 init 段内存               | `Freeing unused kernel image (initmem) memory`               | 把只在初始化期间使用的代码/数据段释放回收                    | 启动主干已结束，不需要继续长期占用这些内存                   | 内存回收 / `__init` 段               | 减少内核常驻内存占用                                         |
| 66. 只读保护内核数据               | `Write protecting the kernel read-only data`                 | 给只读代码/数据页加写保护                                    | 系统进入稳定运行前，必须把该锁死的内存锁死                   | 内核内存保护 / 安全                  | 提升稳定性和防护能力，防止误写内核只读段                     |

------

# 你现在可以把整个启动压缩成 10 个动作

如果上面表太长，你先记这 10 个主动作：

| 阶段 | 一句话理解                                          |
| ---- | --------------------------------------------------- |
| 1    | 先确认“我是谁、命令行是什么、跑在哪台机器上”        |
| 2    | 接收 BIOS 给的物理内存地图                          |
| 3    | 把物理内存转成 Linux 自己的 node/zone/page 管理模型 |
| 4    | 建立 CPU、percpu、RCU、中断、时钟这些最底层执行环境 |
| 5    | 启用正式 console，让日志稳定输出                    |
| 6    | 读取 ACPI，理解平台拓扑、电源、中断路由             |
| 7    | 扫描 PCI，发现 IDE、VGA、E1000 等设备               |
| 8    | 解压 initramfs，准备早期用户空间                    |
| 9    | 驱动绑定硬件：串口、光驱、网卡、键盘、RTC           |
| 10   | 回收初始化内存，开启只读保护，进入稳定运行          |