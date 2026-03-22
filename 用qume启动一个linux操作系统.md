# 方式 A 的完整原理

你理解这一段，后面操作就不会死记命令。

一个 Linux 系统启动，最核心至少需要两样东西：

## 1. 内核镜像

也就是你编译出来的 `bzImage`

它负责：

- 初始化 CPU
- 初始化内存
- 初始化中断
- 初始化设备
- 挂载根文件系统
- 启动第一个用户态进程（通常是 `/init` 或 `/sbin/init`）

------

## 2. 根文件系统

内核起来以后，必须有个“用户空间”让它进入，不然启动到一半就没东西可跑了。

为了让实验简单，我们不直接上一个完整 Ubuntu 根盘，而是做一个**最小 initramfs**。
 里面只放：

- 一个静态 busybox
- 一个 `init` 脚本

这样内核起来后就能进入 shell。

------

## 3. QEMU 的作用

QEMU 就像一台虚拟计算机。
 它可以：

- 模拟 CPU
- 模拟内存
- 模拟串口控制台
- 直接加载你编译出来的内核

所以你不需要把内核安装进 GRUB，也不需要重启当前 Ubuntu。

------

# 六、正式开始：最推荐的完整操作步骤

下面我按“你现在就能照着做”的方式写。

------

## 第 0 步：准备工作目录

先创建一个实验目录，别把东西扔得到处都是。

```
mkdir -p ~/kernel-lab
cd ~/kernel-lab
```

你以后所有和这个实验有关的文件都放这里。

------

## 第 1 步：安装编译 Linux 内核所需依赖

执行：

```
sudo apt update
sudo apt install -y \
    build-essential \
    bc \
    bison \
    flex \
    libssl-dev \
    libelf-dev \
    libncurses-dev \
    dwarves \
    cpio \
    rsync \
    wget \
    xz-utils \
    busybox-static
```

------

## 第 1 步每个包是干嘛的

### `build-essential`

包含：

- gcc
- g++
- make
- libc 开发头文件

没有它，你根本没法正常编译 C 代码。

### `bc`

内核构建脚本里会用到。

### `bison` 和 `flex`

构建一些解析器/生成器时会用到。你前面编 QEMU 已经见过它们了。

### `libssl-dev`

某些内核构建/签名相关功能会用到。

### `libelf-dev`

内核和模块构建时经常需要 ELF 相关支持。

### `libncurses-dev`

`make menuconfig` 的文本界面依赖它。

### `dwarves`

有些新内核会生成 BTF 调试信息，需要它。

### `cpio`

后面打包 initramfs 要用。

### `rsync`

有时复制根文件系统、安装模块会用。

### `wget`

下载源码。

### `xz-utils`

因为 Linux 内核源码包常见是 `.tar.xz`。

### `busybox-static`

做最小 rootfs 最方便。`static` 版特别适合做 initramfs。

------

# 七、第 2 步：下载 Linux 内核源码

进入实验目录：

```
cd ~/kernel-lab
```

下载内核源码。例如：

```
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.6.87.tar.xz
```

解压：

```
tar -xf linux-6.6.87.tar.xz
cd linux-6.6.87
```

------

## 为什么选这种方式下载

因为这是**官方内核源码包**，最干净、最标准。

相比直接 git clone：

- 压缩包更简单
- 不用处理 git 历史
- 第一次练流程更省心

------

## 另一种下载方式：git clone

你也可以这样：

```
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux
```

优点：

- 可以切分支、看提交历史
- 更适合后面学源码

缺点：

- 体积更大
- 首次上手更复杂

所以你第 1 阶段先用 tarball 更合适。

------

# 八、第 3 步：生成默认配置

在源码目录里执行：

```
make defconfig
```

------

## 这一步为什么要做

内核源码不是开箱即用的。
 Linux 支持海量平台和功能，不可能一次全开。

`make defconfig` 的作用就是：

> 给当前架构生成一份“默认、可用、相对稳妥”的配置文件 `.config`

------

## `.config` 是什么

它是内核的配置清单，决定：

- 编译哪些驱动
- 开哪些文件系统
- 支持哪些 CPU 功能
- 是否带调试功能

内核编译前必须有这份配置。

------

## 为什么不先自己手写 `.config`

因为你现在的目标不是研究几千个配置项，而是先把流程跑通。

第 1 阶段最怕的事就是：

- 一开始乱配
- 把必须功能关掉
- 编出来的内核起不来

所以先用 `defconfig` 是最稳的。

------

# 九、第 4 步：可选地看一下配置界面

```
make menuconfig
```

------

## 这一步的意义

它会打开一个字符界面，让你看到 Linux 内核配置系统长什么样。

这是为了让你建立认知：

- `General setup`
- `Processor type and features`
- `Device Drivers`
- `File systems`
- `Kernel hacking`

你现在只需要“看一眼”和“会保存退出”，不需要大改。

------

## 什么时候这一步可以跳过

如果你现在只想最快跑通流程，可以暂时跳过，直接用 `defconfig` 编译。

------

# 十、第 5 步：编译内核

执行：

```
make -j$(nproc)
```

------

## 这一步会发生什么

内核构建系统会：

1. 读取 `.config`
2. 编译大量 C/汇编代码
3. 链接成内核映像
4. 生成最终可启动文件

------

## `-j$(nproc)` 是什么意思

- `nproc`：查看当前机器有多少 CPU 核心
- `-j`：并行编译

所以这条命令的意思是：

> 用当前机器全部 CPU 核心并行编译，提高速度

------

## 编译成功后最关键的输出文件

你最关心的是：

```
arch/x86/boot/bzImage
```

检查一下：

```
ls -lh arch/x86/boot/bzImage
```

如果这个文件存在，说明你已经拿到了一个可启动的内核镜像。

------

# 十一、第 6 步：准备最小 initramfs

这一步非常关键，因为：

> **内核不是用户空间。**
>
> 光有内核还不够，它启动后还得有个根文件系统和第一个用户进程。

为了简化实验，我们自己做一个极简 initramfs。

------

## 第 6.1 步：创建目录结构

```
mkdir -p ~/kernel-lab/initramfs/{bin,sbin,etc,proc,sys,usr/bin,usr/sbin,dev}
cd ~/kernel-lab/initramfs
```

------

## 为什么要这些目录

最小 Linux 根文件系统也得像个文件系统的样子。
 尤其是：

- `/bin`：放 busybox
- `/proc`：挂载 proc
- `/sys`：挂载 sysfs
- `/dev`：设备节点目录

------

## 第 6.2 步：拷贝 BusyBox

```
cp /usr/bin/busybox ./bin/
```

------

## 为什么用 BusyBox

BusyBox 是“工具箱式”程序，一个二进制可以同时充当很多命令：

- `sh`
- `ls`
- `mount`
- `cat`
- `echo`
- `uname`

特别适合做最小系统。

------

## 第 6.3 步：建立常用命令链接

```
cd bin
for cmd in sh mount echo ls cat uname dmesg pwd ps; do
    ln -sf busybox $cmd
done
cd ..
```

------

## 这一步为什么要做

BusyBox 自己会根据“你用什么名字调用它”来决定扮演哪个命令。

例如：

- 你执行 `/bin/sh`，其实链接到了 `busybox`
- BusyBox 看到自己是以 `sh` 的名字被调用，就进入 shell 模式

这就是它的工作原理。

------

## 第 6.4 步：创建 `init`

在 `~/kernel-lab/initramfs` 目录里执行：

```
cat > init <<'EOF'
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
echo "Boot OK: minimal Linux in QEMU"
uname -a
exec /bin/sh
EOF
chmod +x init
```

------

## 为什么这个 `init` 这么重要

Linux 内核启动后，会尝试找第一个用户空间程序。

它通常会尝试：

- `/init`
- `/sbin/init`
- `/bin/init`
- `/bin/sh`

你这里放一个 `/init`，就是明确告诉内核：

> 启动后先执行这个脚本

这个脚本做了三件事：

### 1. 挂载 `/proc`

让你能看到进程、内核信息。

### 2. 挂载 `/sys`

让你能访问 sysfs。

### 3. 进入 shell

最后 `exec /bin/sh`，让你直接拿到一个最小 shell。

------

## 第 6.5 步：打包 initramfs

回到 `~/kernel-lab/initramfs` 目录，执行：

```
find . | cpio -H newc -ov --owner root:root > ../initramfs.cpio
cd ..
```

------

## 这一步在做什么

它把你刚才做的最小文件系统目录，打包成一个 cpio 格式归档。

这个文件：

```
~/kernel-lab/initramfs.cpio
```

就是内核启动时会加载的初始根文件系统。

------

# 十二、第 7 步：用 QEMU 启动你编译的新内核

执行：

```
qemu-system-x86_64 \
  -kernel ~/kernel-lab/linux-6.6.87/arch/x86/boot/bzImage \
  -initrd ~/kernel-lab/initramfs.cpio \
  -nographic \
  -append "console=ttyS0"
```

------

## 这条命令每部分什么意思

### `qemu-system-x86_64`

启动一个 x86_64 虚拟机。

### `-kernel ...`

直接指定要启动的内核镜像。

### `-initrd ...`

指定初始根文件系统。

### `-nographic`

不用图形窗口，直接把串口输出显示在当前终端。
 这对内核实验特别方便。

### `-append "console=ttyS0"`

把内核控制台重定向到串口 `ttyS0`。
 因为你用了 `-nographic`，所以要靠串口看启动日志。

------

## 启动成功后会看到什么

你应该会看到很多内核启动信息，最后出现你写的那句：

```
Boot OK: minimal Linux in QEMU
```

然后进入 shell。

这时你可以执行：

```
uname -a
```

如果显示的是你刚编译的新内核版本，那就说明：

> 你已经成功完成了“换内核”。

------

# 十三、第 8 步：如何验证你真的换到了新内核

在 QEMU 里的 shell 执行：

```
uname -a
```

或者：

```
cat /proc/version
```

你会看到当前运行的内核版本信息。

这一步非常重要，因为：

> 编译成功 ≠ 启动成功
>  启动成功 ≠ 确实在跑你自己的内核

你必须验证。

------

# 十四、第 9 步：如何切回旧内核

这条路线最爽的地方就在这里：

> 你根本没改 Ubuntu 当前系统的内核。

所以“切回旧内核”非常简单：

- 在 QEMU 里输入 `poweroff`，或者直接关闭 QEMU
- 回到原来的 Ubuntu 终端

你当前系统仍然跑的是原来的 Ubuntu 内核。

------

## 为什么这也算“会切回旧内核”

因为第 1 阶段训练的是“内核切换流程”和“回退意识”。

你这里已经完成了最核心的回退原则：

> **新内核实验与旧系统隔离。**

这是最安全、最工程化的做法。

------

# 十五、如果编译失败怎么办

这是正常情况。Linux 内核不是小项目，失败并不稀奇。

常见做法：

## 1. 看最后几十行错误

```
make -j$(nproc) 2>&1 | tee build.log
```

失败后查看：

```
tail -n 50 build.log
```

------

## 2. 依赖缺失就补

例如缺：

- `libelf`
- `openssl`
- `ncurses`

就装对应开发包。

------

## 3. 配置改坏了就回到默认配置

```
make mrproper
make defconfig
make -j$(nproc)
```

`mrproper` 会清理得很干净，但也会把 `.config` 删掉。