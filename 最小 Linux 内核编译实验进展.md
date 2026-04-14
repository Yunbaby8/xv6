最小 Linux 内核编译实验
把 Linux 6.6.87 源码成功编译出了可启动的 bzImage。
QEMU 启动自编译内核实验
用 QEMU 成功启动了自己编译的内核。
最小 initramfs / rootfs 启动实验
用自己构建的最小 rootfs 和 /init 跑进了 shell。
内核版本后缀修改实验
通过修改 CONFIG_LOCALVERSION，让运行中的内核版本显示成了你自己的后缀。
Git 本地仓库初始化实验
在 Linux 源码目录里初始化了 Git 仓库，并完成了第一次提交。
启动路径 printk 插桩实验
在 init/main.c 的 start_kernel() 里加入了 printk，并在启动日志里成功看到了自己的输出。