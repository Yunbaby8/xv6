# 模块动态创建 proc 节点实验笔记

## 1. 实验目标

本实验在可加载内核模块中创建一个 procfs 节点：

```
/proc/hello_module
```

实现效果：

```
insmod hello_module.ko  -> /proc/hello_module 出现
cat /proc/hello_module  -> 读取模块导出的内核信息
rmmod hello_module      -> /proc/hello_module 消失
```

------

## 2. 实验流程

### Step 1：进入模块目录

```
cd /home/yunhe/allcode/kernel-lab/modules/hello_module
```

### Step 2：修改模块源码

文件：

```
hello_module.c
```

### Step 3：重新编译模块

```
make clean
make
```

生成：

```
hello_module.ko
```

### Step 4：复制模块到 initramfs

```
mkdir -p /home/yunhe/allcode/kernel-lab/initramfs/root
cp hello_module.ko /home/yunhe/allcode/kernel-lab/initramfs/root/
```

### Step 5：重新打包 initramfs

```
cd /home/yunhe/allcode/kernel-lab
./scripts/pack.sh
```

### Step 6：启动 QEMU

```
qemu-system-x86_64 \
  -kernel /home/yunhe/allcode/linux-6.6.87-build/arch/x86/boot/bzImage \
  -initrd /home/yunhe/allcode/kernel-lab/initramfs.cpio \
  -append "console=ttyS0 rdinit=/init" \
  -nographic
```

### Step 7：加载模块并验证

```
insmod /root/hello_module.ko
ls /proc/hello_module
cat /proc/hello_module
dmesg | grep hello_module
```

### Step 8：卸载模块并验证

```
rmmod hello_module
ls /proc/hello_module
dmesg | grep hello_module
```

卸载后 `/proc/hello_module` 应该不存在。

------

## 3. 具体代码

```
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/jiffies.h>
#include <linux/utsname.h>

#define PROC_NAME "hello_module"

static int hello_module_show(struct seq_file *m, void *v)
{
    seq_printf(m, "Hello from hello_module\n");
    seq_printf(m, "jiffies: %lu\n", jiffies);
    seq_printf(m, "release: %s\n", init_uts_ns.name.release);

    return 0;
}

static int hello_module_open(struct inode *inode, struct file *file)
{
    return single_open(file, hello_module_show, NULL);
}

static const struct proc_ops hello_module_proc_ops = {
    .proc_open    = hello_module_open,
    .proc_read    = seq_read,
    .proc_lseek   = seq_lseek,
    .proc_release = single_release,
};

static int __init hello_module_init(void)
{
    if (!proc_create(PROC_NAME, 0444, NULL, &hello_module_proc_ops)) {
        pr_err("hello_module: failed to create /proc/%s\n", PROC_NAME);
        return -ENOMEM;
    }

    pr_info("hello_module: /proc/%s created\n", PROC_NAME);
    return 0;
}

static void __exit hello_module_exit(void)
{
    remove_proc_entry(PROC_NAME, NULL);
    pr_info("hello_module: /proc/%s removed\n", PROC_NAME);
}

module_init(hello_module_init);
module_exit(hello_module_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Eric Zhou");
MODULE_DESCRIPTION("A hello module with procfs entry");
```

------

## 4. 代码分析

### 4.1 头文件

```
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/jiffies.h>
#include <linux/utsname.h>
```

含义：

- `linux/init.h`：提供 `__init`、`__exit`、`module_init()`、`module_exit()`
- `linux/module.h`：提供模块相关机制和模块元信息宏
- `linux/kernel.h`：提供 `pr_info()`、`pr_err()` 等内核日志函数
- `linux/proc_fs.h`：提供 `proc_create()`、`remove_proc_entry()`、`struct proc_ops`
- `linux/seq_file.h`：提供 `seq_file` 输出框架
- `linux/jiffies.h`：提供 `jiffies`
- `linux/utsname.h`：提供内核版本信息

------

### 4.2 节点名称

```
#define PROC_NAME "hello_module"
```

定义 proc 节点名字。
 最终用户态看到的是：

```
/proc/hello_module
```

------

### 4.3 `hello_module_show()`

```
static int hello_module_show(struct seq_file *m, void *v)
{
    seq_printf(m, "Hello from hello_module\n");
    seq_printf(m, "jiffies: %lu\n", jiffies);
    seq_printf(m, "release: %s\n", init_uts_ns.name.release);

    return 0;
}
```

作用：
 定义用户读取 `/proc/hello_module` 时显示什么内容。

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

------

### 4.4 `hello_module_open()`

```
static int hello_module_open(struct inode *inode, struct file *file)
{
    return single_open(file, hello_module_show, NULL);
}
```

作用：
 用户打开 `/proc/hello_module` 时调用。

参数：

```
struct inode *inode
```

表示该 proc 节点的 inode 信息，本实验未使用。

```
struct file *file
```

表示这次打开操作对应的文件对象。

核心：

```
single_open(file, hello_module_show, NULL);
```

含义是：
 这个 proc 文件内容比较简单，只需要调用一次 `show()` 就能生成完整输出，因此用 `single_open()` 把 `hello_module_show()` 绑定到这次读取流程中。

------

### 4.5 `hello_module_proc_ops`

```
static const struct proc_ops hello_module_proc_ops = {
    .proc_open    = hello_module_open,
    .proc_read    = seq_read,
    .proc_lseek   = seq_lseek,
    .proc_release = single_release,
};
```

作用：
 定义 `/proc/hello_module` 的文件操作。

| 操作    | 对应函数            | 作用           |
| ------- | ------------------- | -------------- |
| open    | `hello_module_open` | 打开 proc 节点 |
| read    | `seq_read`          | 读取 proc 内容 |
| lseek   | `seq_lseek`         | 支持偏移操作   |
| release | `single_release`    | 关闭时释放资源 |

这里没有自己写 read 函数，而是复用 `seq_file` 框架的 `seq_read()`。

------

### 4.6 `hello_module_init()`

```
static int __init hello_module_init(void)
{
    if (!proc_create(PROC_NAME, 0444, NULL, &hello_module_proc_ops)) {
        pr_err("hello_module: failed to create /proc/%s\n", PROC_NAME);
        return -ENOMEM;
    }

    pr_info("hello_module: /proc/%s created\n", PROC_NAME);
    return 0;
}
```

作用：
 模块加载时执行，创建 `/proc/hello_module`。

关键语句：

```
proc_create(PROC_NAME, 0444, NULL, &hello_module_proc_ops)
```

参数含义：

| 参数                     | 含义                                    |
| ------------------------ | --------------------------------------- |
| `PROC_NAME`              | 节点名，即 `hello_module`               |
| `0444`                   | 只读权限                                |
| `NULL`                   | 父目录为空，表示创建在 `/proc` 根目录下 |
| `&hello_module_proc_ops` | 该节点的操作表                          |

如果创建失败，返回：

```
-ENOMEM
```

表示内存不足或资源申请失败。

------

### 4.7 `hello_module_exit()`

```
static void __exit hello_module_exit(void)
{
    remove_proc_entry(PROC_NAME, NULL);
    pr_info("hello_module: /proc/%s removed\n", PROC_NAME);
}
```

作用：
 模块卸载时执行，删除 `/proc/hello_module`。

关键语句：

```
remove_proc_entry(PROC_NAME, NULL);
```

表示从 `/proc` 根目录下删除 `hello_module` 节点。

这一步非常重要。
 模块卸载时必须清理自己创建的资源，否则会留下无效接口或导致异常。

------

### 4.8 `module_init()` 和 `module_exit()`

```
module_init(hello_module_init);
module_exit(hello_module_exit);
```

作用：

- `module_init()`：告诉内核模块加载时调用哪个函数
- `module_exit()`：告诉内核模块卸载时调用哪个函数

对应关系：

```
insmod hello_module.ko -> hello_module_init()
rmmod hello_module     -> hello_module_exit()
```

------

## 5. 内核操作流程

### 5.1 加载模块时

用户执行：

```
insmod /root/hello_module.ko
```

内核大致做这些事：

```
读取 hello_module.ko
  ↓
解析模块格式
  ↓
检查内核版本和符号依赖
  ↓
把模块代码加载到内核空间
  ↓
完成符号重定位
  ↓
调用 hello_module_init()
  ↓
执行 proc_create()
  ↓
在 procfs 中注册 hello_module 节点
  ↓
/proc/hello_module 出现
```

------

### 5.2 用户读取 `/proc/hello_module` 时

用户执行：

```
cat /proc/hello_module
```

内核大致做这些事：

```
cat 调用 open("/proc/hello_module")
  ↓
procfs 找到 hello_module 节点
  ↓
调用 hello_module_proc_ops.proc_open
  ↓
执行 hello_module_open()
  ↓
single_open() 绑定 hello_module_show()
  ↓
cat 调用 read()
  ↓
调用 hello_module_proc_ops.proc_read
  ↓
执行 seq_read()
  ↓
seq_read() 调用 hello_module_show()
  ↓
hello_module_show() 用 seq_printf() 生成内容
  ↓
内容返回给用户态
  ↓
cat 显示出来
```

------

### 5.3 卸载模块时

用户执行：

```
rmmod hello_module
```

内核大致做这些事：

```
检查模块是否正在被使用
  ↓
如果引用计数为 0，允许卸载
  ↓
调用 hello_module_exit()
  ↓
执行 remove_proc_entry()
  ↓
从 procfs 中删除 hello_module 节点
  ↓
释放模块占用的内核资源
  ↓
模块从内核中移除
```

卸载后：

```
ls /proc/hello_module
```

应该提示不存在。

------

## 6. 实验意义

这个实验把前面两个实验连起来了：

```
procfs 内核信息导出实验
        +
内核模块加载与卸载实验
        ↓
模块动态创建 proc 节点实验
```

它的意义是：

1. 理解模块生命周期不只是打印日志，还可以管理内核资源。
2. 理解 `insmod` 可以动态增加用户态可访问的内核接口。
3. 理解 `rmmod` 时必须清理模块创建的资源。
4. 掌握驱动开发中常见的模式：

```
模块加载 -> 注册接口
模块卸载 -> 注销接口
```

这就是很多驱动的基本结构。后面做 `sysfs`、字符设备、I2C 驱动时，都会反复用到这个思想。

------

## 7. 一句话总结

本实验实现了一个由内核模块动态创建和删除的 procfs 节点：加载模块时通过 `proc_create()` 注册 `/proc/hello_module`，用户态通过 `cat /proc/hello_module` 主动读取模块导出的内核信息，卸载模块时通过 `remove_proc_entry()` 删除该节点，从而完成了“模块生命周期管理用户态接口”的完整闭环。