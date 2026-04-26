# procfs 内核信息导出实验笔记

## 1. 实验目标

本实验是在 Linux 内核中新增一个 procfs 节点：

```
/proc/hello_kernel
```

用户态执行：

```
cat /proc/hello_kernel
```

可以从内核读取动态生成的信息，例如：

```
Hello from kernel
jiffies: 123456
release: 6.6.87-xxx
```

------

# 2. 核心代码解析

## 2.1 `hello_kernel_show()`

```
static int hello_kernel_show(struct seq_file *m, void *v)
{
    seq_printf(m, "Hello from kernel\n");
    seq_printf(m, "jiffies: %lu\n", jiffies);
    seq_printf(m, "release: %s\n", init_uts_ns.name.release);

    return 0;
}
```

作用：
 定义用户读取 `/proc/hello_kernel` 时看到的内容。

参数：

```
struct seq_file *m
```

表示 seq_file 输出对象，可以理解为内核准备好的输出缓冲区。

```
void *v
```

seq_file 框架预留参数，本实验没有使用。

返回值：

```
0
```

表示输出成功。

这里通过 `seq_printf()` 向用户态输出三类信息：

- 固定字符串：`Hello from kernel`
- 当前内核节拍计数：`jiffies`
- 当前内核版本：`init_uts_ns.name.release`

------

## 2.2 `hello_kernel_open()`

```
static int hello_kernel_open(struct inode *inode, struct file *file)
{
    return single_open(file, hello_kernel_show, NULL);
}
```

作用：
 当用户打开 `/proc/hello_kernel` 时被调用。

用户执行：

```
cat /proc/hello_kernel
```

背后会先触发 `open()`，然后进入这个函数。

参数：

```
struct inode *inode
```

表示该 proc 节点的 inode 信息，本实验未使用。

```
struct file *file
```

表示这次打开操作对应的文件对象。

核心语句：

```
single_open(file, hello_kernel_show, NULL);
```

含义是：
 这个 proc 文件内容很简单，只需要一次 `show()` 就能生成完整输出，因此使用 `single_open()` 把 `hello_kernel_show()` 绑定到这次文件读取流程中。

------

## 2.3 `hello_kernel_proc_ops`

```
static const struct proc_ops hello_kernel_proc_ops = {
    .proc_open    = hello_kernel_open,
    .proc_read    = seq_read,
    .proc_lseek   = seq_lseek,
    .proc_release = single_release,
};
```

作用：
 定义 `/proc/hello_kernel` 支持哪些文件操作。

它相当于一张操作表：

| 操作    | 对应函数            | 作用                 |
| ------- | ------------------- | -------------------- |
| open    | `hello_kernel_open` | 打开 proc 节点时调用 |
| read    | `seq_read`          | 读取内容时调用       |
| lseek   | `seq_lseek`         | 支持文件偏移         |
| release | `single_release`    | 关闭文件时释放资源   |

这里没有自己写 read 函数，而是复用了 `seq_file` 框架提供的 `seq_read()`。

------

## 2.4 `hello_kernel_init()`

```
static int __init hello_kernel_init(void)
{
    proc_create("hello_kernel", 0444, NULL, &hello_kernel_proc_ops);
    pr_info("hello_kernel: /proc/hello_kernel created\n");

    return 0;
}
```

作用：
 在内核启动过程中创建 `/proc/hello_kernel` 节点。

核心语句：

```
proc_create("hello_kernel", 0444, NULL, &hello_kernel_proc_ops);
```

参数含义：

| 参数                     | 含义                                    |
| ------------------------ | --------------------------------------- |
| `"hello_kernel"`         | proc 节点名称                           |
| `0444`                   | 只读权限                                |
| `NULL`                   | 父目录为空，表示创建在 `/proc` 根目录下 |
| `&hello_kernel_proc_ops` | 该节点对应的操作表                      |

`pr_info()` 用来打印内核日志，可以通过：

```
dmesg | grep hello_kernel
```

验证初始化函数是否执行。

------

## 2.5 `fs_initcall()`

```
fs_initcall(hello_kernel_init);
```

作用：
 把 `hello_kernel_init()` 注册到内核启动初始化流程中。

它告诉内核：

> 在文件系统初始化阶段，调用 `hello_kernel_init()`。

所以不是手动调用 `hello_kernel_init()`，而是内核启动过程中自动调用它。

------

# 3. 为什么只新建 `.c` 文件还不够

新建：

```
fs/proc/hello_kernel.c
```

只是放代码。

要真正生效，必须同时满足：

```
1. hello_kernel.c 里调用 proc_create()
2. fs/proc/Makefile 里加入 hello_kernel.o
3. 重新编译并启动新内核
```

例如 Makefile 中需要加入：

```
obj-y += hello_kernel.o
```

否则这个 `.c` 文件不会被编进内核，运行时也不会出现 `/proc/hello_kernel`。

------

# 4. 用户读取 `/proc/hello_kernel` 时内核做了什么

用户执行：

```
cat /proc/hello_kernel
```

整体流程如下：

```
cat 程序启动
  ↓
open("/proc/hello_kernel")
  ↓
进入 procfs
  ↓
找到 hello_kernel 节点
  ↓
调用 hello_kernel_proc_ops.proc_open
  ↓
执行 hello_kernel_open()
  ↓
single_open() 绑定 hello_kernel_show()
  ↓
cat 调用 read()
  ↓
进入 hello_kernel_proc_ops.proc_read
  ↓
执行 seq_read()
  ↓
seq_read() 调用 hello_kernel_show()
  ↓
hello_kernel_show() 使用 seq_printf() 生成内容
  ↓
内容返回给用户态
  ↓
cat 显示结果
  ↓
close()
  ↓
调用 single_release() 释放资源
```

------

# 5. procfs 和 `/proc` 的关系

## procfs

procfs 是内核提供的虚拟文件系统。

它不对应真实磁盘文件，而是由内核动态生成内容。

## `/proc`

`/proc` 是用户态看到的挂载点。

通过：

```
mount -t proc proc /proc
```

把内核中的 procfs 挂载到用户态的 `/proc` 目录。

所以：

```
procfs = 内核里的虚拟文件系统机制
/proc  = 用户态看到的挂载入口
```

------

# 6. 本实验的意义

这个实验完成了从 `printk` 到用户态接口的过渡。

以前的 `printk` 是：

```
内核主动打印日志
用户被动查看 dmesg
```

而 procfs 是：

```
用户主动读取 /proc/hello_kernel
内核动态返回信息
```

这说明已经开始理解：

- 内核如何注册用户态可访问接口
- 用户态如何通过文件接口访问内核信息
- `/proc` 这类虚拟文件系统不是磁盘文件，而是内核动态生成内容
- `seq_file` 是实现 proc 只读输出的常用框架

------

# 7. 一句话总结

本实验实现了一个最小 procfs 只读节点：
 **通过 `proc_create()` 在内核中注册 `/proc/hello_kernel`，通过 `seq_file` 框架在用户读取时动态生成内容，从而完成用户态主动读取内核信息的完整闭环。**

# procfs和/proc的区别

**procfs** 是内核里的一个**虚拟文件系统类型**。
 **`/proc`** 是用户态看到的一个**挂载点目录**。

可以这样理解：

```
procfs = 内核提供的一套虚拟文件系统机制
/proc  = 把这套机制挂出来以后，用户看到的目录
```

举个例子：

```
mount -t proc proc /proc
```

这句话的意思是：

```
把 procfs 这个虚拟文件系统，挂载到 /proc 这个目录上
```

所以：

| 名称    | 是什么            | 位置               |
| ------- | ----------------- | ------------------ |
| procfs  | 文件系统类型/机制 | 内核里             |
| `/proc` | 挂载点目录        | 用户态根文件系统里 |

`/proc` 里的很多“文件”不是真实磁盘文件，比如：

```
/proc/cpuinfo
/proc/meminfo
/proc/version
/proc/uptime
```

它们是 procfs 动态生成的。用户执行：

```
cat /proc/version
```

实际流程是：

```
cat 读取 /proc/version
        ↓
进入 procfs
        ↓
调用内核对应函数
        ↓
内核动态生成内容
        ↓
返回给用户态
```

你这次实验里：

```
proc_create("hello_kernel", 0444, NULL, &hello_kernel_proc_ops);
```

是在 **procfs 里注册一个新节点**。

而：

```
mount -t proc proc /proc
```

是把 **procfs 挂载到 `/proc` 目录**，让用户态能看到它。

一句话总结：

**procfs 是内核提供的“虚拟文件系统”；`/proc` 是它在用户空间里的入口。**



