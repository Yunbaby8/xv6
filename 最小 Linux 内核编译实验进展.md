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

**procfs 内核信息导出实验** 在内核中新增 `/proc/hello_kernel` 只读节点，通过 `proc_create()` 和 `seq_file` 实现用户态主动读取内核动态信息，并用 `cat /proc/hello_kernel` 成功验证输出。

**内核模块加载与卸载实验** 编写最小可加载内核模块（`hello_module.ko`），通过 `insmod`/`rmmod` 实现内核功能的动态加载与卸载，并利用 `dmesg` 验证模块初始化与退出函数的执行过程。
procfs 可读写接口实验 在 /proc/hello_module 中新增 write 回调，实现通过 echo 修改内核变量并用 cat 验证结果。
