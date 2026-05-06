# 🧪 实验名称

## **procfs 可读写接口实验（Module + proc write）**

------

# 🎯 实验目标

实现一个内核接口：

```
/proc/hello_module
```

具备能力：

```
cat /proc/hello_module        → 读取内核变量
echo 100 > /proc/hello_module → 修改内核变量
```

------

# 🧭 一、整体流程（你必须记住这张图）

## 🔹 模块加载

```
insmod hello_module.ko
        ↓
hello_module_init()
        ↓
proc_create()
        ↓
创建 /proc/hello_module
```

------

## 🔹 读流程（cat）

```
cat /proc/hello_module
        ↓
open → hello_module_open
        ↓
read → seq_read
        ↓
hello_module_show
        ↓
seq_printf 输出 myvalue
```

------

## 🔹 写流程（echo ⭐重点）

```
echo 100 > /proc/hello_module
        ↓
write()
        ↓
hello_module_write()
        ↓
copy_from_user()
        ↓
kstrtoint()
        ↓
myvalue = 100
```

------

## 🔹 模块卸载

```
rmmod hello_module
        ↓
hello_module_exit()
        ↓
remove_proc_entry()
        ↓
删除 /proc/hello_module
```

------

# 🧩 二、关键代码（精简版）

## 1️⃣ 内核变量

```
static int myvalue = 0;
```

👉 内核状态

------

## 2️⃣ 读函数

```
static int hello_module_show(struct seq_file *m, void *v)
{
    seq_printf(m, "myvalue: %d\n", myvalue);
    return 0;
}
```

👉 控制 `cat` 输出

------

## 3️⃣ 写函数（核心）

```
static ssize_t hello_module_write(struct file *file,
                                  const char __user *buffer,
                                  size_t count,
                                  loff_t *ppos)
{
    char buf[32] = {0};
    int value;

    if (count >= sizeof(buf))
        return -EINVAL;

    if (copy_from_user(buf, buffer, count))
        return -EFAULT;

    if (kstrtoint(buf, 10, &value))
        return -EINVAL;

    myvalue = value;

    pr_info("hello_module: myvalue updated to %d\n", myvalue);

    return count;
}
```

👉 核心三步：

```
拿数据 → 解析 → 改变量
```

------

## 4️⃣ proc 操作表

```
static const struct proc_ops hello_module_proc_ops = {
    .proc_open    = hello_module_open,
    .proc_read    = seq_read,
    .proc_write   = hello_module_write,
    .proc_lseek   = seq_lseek,
    .proc_release = single_release,
};
```

👉 把用户操作和函数绑定

------

## 5️⃣ 模块加载

```
proc_create(PROC_NAME, 0666, NULL, &hello_module_proc_ops);
```

👉 创建接口

------

## 6️⃣ 模块卸载

```
remove_proc_entry(PROC_NAME, NULL);
```

👉 删除接口

------

# 🔑 三、实验关键点（一定要理解）

## 1️⃣ `/proc` 不是文件

```
/proc/hello_module ≠ 磁盘文件
                 = 内核接口
```

------

## 2️⃣ 文件操作 = 内核函数调用

```
cat   → read → show()
echo  → write → write()
```

👉 这是最核心认知

------

## 3️⃣ 用户态 → 内核态的桥

```
echo 100
   ↓
write()
   ↓
copy_from_user()
   ↓
内核变量改变
```

👉 你第一次真正“让用户控制内核”

------

## 4️⃣ seq_file 是读接口模板

```
single_open + seq_read + seq_printf
```

👉 标准 proc 读模型

------

## 5️⃣ write 回调是驱动入口雏形

```
write() = 控制命令入口
```

👉 后面 char device / sysfs 都是同一模式

------

# 🧠 四、这个实验的本质

你完成的是一个**最小驱动控制模型**：

```
用户态
  ↓
文件接口
  ↓
内核函数
  ↓
内核状态
```

------

# 🚀 五、实验意义（非常关键）

## 1️⃣ 从“读”到“控制”

之前：

```
cat → 只能看
```

现在：

```
echo → 可以改
```

👉 从观察者变成操作者

------

## 2️⃣ 真实驱动的基础模型

几乎所有驱动都在做：

```
用户输入 → 内核解析 → 修改状态
```

------

## 3️⃣ 为后面铺路

你现在已经具备：

```
module ✔
proc ✔
read ✔
write ✔
```

👉 下一步自然是：

```
/dev 设备（字符设备）
```

------

# 📌 六、一句话总结

👉 **这个实验实现了一个可读写的内核接口，让用户通过文件操作直接控制内核变量，是驱动开发最基础的交互模型。**

------

# 🧭 我给你一个方向（很关键）

你现在不要急着学新 API，先巩固这个模型：

如果你能随口说出：

```
echo → write → copy_from_user → kstrtoint → 修改变量
```

说明你已经真正理解了。