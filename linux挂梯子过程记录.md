# 一、你的目标

你想在这台刚装好的 **Ubuntu Server 笔记本** 上把代理挂起来，目的主要是：

- 让 Linux 本机能科学上网
- 方便访问 GitHub、raw、release 资源
- 后面装工具、拉源码、做内核实验更顺

------

# 二、你一开始走的路线

你最开始不是直接装 `mihomo`，而是先尝试：

1. 在 Linux 上装 **mihoro**
2. 用你的 **Clash 订阅** 去初始化
3. 希望 `mihoro` 帮你：
   - 下载内核
   - 管理配置
   - 启动服务

这条路表面上合理，但实际走下来，问题很多。

------

# 三、一路上遇到的主要问题

## 1. `mihoro: command not found`

### 现象

你一开始执行 `mihoro init`，系统提示：

```
mihoro: command not found
```

### 原因

不是命令坏了，而是：

- `mihoro` 根本还没装上
- `~/.local/bin` 目录都还不存在
- 所以不是 PATH 问题，是**安装压根没成功**

### 你做的处理

你后来重新安装了 `mihoro`，并且把它放进了：

```
~/.local/bin
```

最后通过：

```
mihoro --version
```

看到了：

```
mihoro 0.13.0
```

这一步说明 `mihoro` 本体终于装好了。

------

## 2. GitHub 下载很慢，安装脚本卡住

### 现象

你执行安装脚本时，看起来“没反应”，或者卡很久。

### 原因

不是脚本死了，而是：

- Linux 访问 GitHub、raw、release 资源很慢
- 安装脚本跑到下载阶段时，看起来像卡住

### 你做的处理

你后来改成了：

- 不再盲目 `curl ... | sh`
- 先下载脚本
- 再手动下载 release 包
- 再手动解压安装

这个决策是对的。
 因为一旦网络差，**自动脚本最难排查**，手动下载最清楚。

------

## 3. 订阅格式不对，`mihoro init` 报错

### 现象

你把订阅链接喂给 `mihoro` 后，报错大意是：

- `remote config: invalid type: string`
- `expected MihomoYamlConfig`

### 原因

你喂进去的不是 **Clash/Mihomo YAML 配置**，而是：

- `vless://...`
- `hysteria2://...`
- 或这些节点串的 Base64 形式

也就是说，你给的是**节点分享格式**，不是 `mihoro` 想要的 **YAML 订阅**。

### 你做的处理

你后面做了几个重要判断：

1. 识别出 `clash://install-config?...` 只是**外层包装**
2. 识别出打开订阅后看到的 `vless://...` 依然不对
3. 最后去 **Clash Verge 本地配置目录** 里找到了真正的 YAML 配置文件

这个转向非常关键。

------

## 4. 你以为“当前目录是空的”，其实是目录搞混了

### 现象

你执行 `ls` 后看不到 `/proc` 之类的目录，以为系统有问题。

### 原因

你当时在：

```
/home/yunhe
```

也就是你的家目录，不是在根目录 `/`。

### 结论

这是 Linux 初学时很正常的坑，不是系统没装好。

------

## 5. 你以为 `mihoro` 能直接吃本地 `config.yaml`

### 现象

你已经拿到了：

```
~/.config/mihoro/config.yaml
```

而且这个文件内容是标准 YAML，但执行：

- `mihoro apply`
- `mihoro start`
- `mihoro status`

还是报错：

```
remote_config_url undefined
```

### 原因

这一步是整个过程里最容易绕晕的地方。

`mihoro` 的设计不是“本地 YAML 启动器”，而是：

- 先要求你有 `remote_config_url`
- 下载远程 YAML
- 再用 `mihoro.toml` 覆盖部分选项
- 再管理 systemd

所以你把 `remote_config_url` 删掉后，`mihoro` 自己就不会工作了。

### 结论

这时候你终于看明白了：

**`mihoro` 不适合你现在这条“我已经有本地 YAML 文件”的路线。**

这就是后面你成功的核心转折点。

------

## 6. `mihomo` 本体其实没真正装好

### 现象

虽然你已经有了：

- `~/.config/mihomo/config.yaml`
- `~/.config/mihomo/ui`
- `country.mmdb`

但执行：

```
ls -l ~/.local/bin/mihomo
```

发现没有。

### 原因

说明之前 `mihoro init` 虽然部分下载了：

- 配置
- UI
- geodata

但**真正负责跑代理的 `mihomo` 二进制没有正确落盘**。

### 你做的处理

你最后决定：

**不再依赖 `mihoro` 自动装内核，直接手动装 `mihomo`。**

这是你整个流程里最正确的一次换路。

------

# 四、你做出的关键决策变化

## 决策变化 1：从“自动脚本安装”改成“手动下载”

原因：

- GitHub 访问慢
- 自动脚本不好定位错误

结果：

- 可控性更强
- 每一步都能看到到底卡在哪

------

## 决策变化 2：放弃继续硬怼订阅链接

原因：

- 订阅链接返回的是节点串，不是 YAML
- `mihoro` 明确报配置类型不对

结果：

- 不再浪费时间研究各种包装链接
- 直接从 Clash Verge 本地拿现成可用配置

------

## 决策变化 3：从“继续用 mihoro 管理一切”改成“直接运行 mihomo”

原因：

- `mihoro` 强依赖 `remote_config_url`
- 你已经有本地 `config.yaml`
- 再用 `mihoro` 只会绕远路

结果：

- 直接用真正的代理内核 `mihomo`
- 用本地配置文件启动
- 路线一下变直了

------

# 五、最后成功的方法是什么

你最后真正成功的方案，不是：

- `mihoro + 原始订阅链接`

而是：

## 最终成功路线

### 1. 从 Clash Verge 本地导出/拿到 YAML 配置

你在 Windows 上找到了：

```
clash-verge.yaml
```

并确认里面是标准 Mihomo/Clash YAML，比如有：

- `mode: rule`
- `mixed-port: 7897`
- `dns:`
- `proxies:`

### 2. 把它放到 Linux

你最终放到了：

```
~/.config/mihomo/config.yaml
```

### 3. 手动下载并安装 `mihomo`

你成功手动下载了正确的 release 包，解压后装到了：

```
~/.local/bin/mihomo
```

并且验证版本成功。

### 4. 直接启动 `mihomo`

你执行了：

```
~/.local/bin/mihomo -d ~/.config/mihomo
```

这一步才是真正把代理跑起来的关键。

### 5. 验证端口监听成功

你确认了：

- `127.0.0.1:7897`
- `127.0.0.1:9097`

都已经监听。

### 6. 设置环境变量让当前 shell 走代理

你后来加了：

- `proxy_on`
- `proxy_off`

这两个 bash 函数，本质上是给当前 shell 设置：

```
http_proxy
https_proxy
all_proxy
```

### 7. 最终通过 curl 验证出网成功

你成功执行了：

```
curl -I https://www.google.com
curl -I https://github.com
```

而且都返回了正常响应头。

这一步说明：

**代理不仅启动了，而且真的能用。**

------

# 六、最终你到底解决了哪些问题

你最后实际上解决了这几个层次的问题：

## 1. 系统层

- Ubuntu Server 装好了
- SSH 能连
- 合盖不休眠也处理了

## 2. 软件层

- `mihoro` 装上了，但你看清了它的适用边界
- `mihomo` 真正装上并能运行

## 3. 配置层

- 找到了正确的 YAML 配置来源
- 不再误用 `vless://` / base64 节点串

## 4. 网络层

- 代理端口起来了
- 本机程序能通过代理访问 Google / GitHub

## 5. 使用层

- 你已经能通过 `proxy_on` 一键给当前 shell 开代理

------

# 七、整个过程的核心经验

如果把这次折腾压缩成几条最值钱的经验，就是这些：

## 1. `vless://`、`hysteria2://` 节点串，不等于 Clash/Mihomo YAML

这是你这次最大的格式坑。

## 2. `mihoro` 和 `mihomo` 不是一回事

- `mihoro` 是管理器
- `mihomo` 才是真正跑代理的内核

## 3. 当你已经有可用的本地 YAML 时，直接跑 `mihomo` 最直接

别再被 `mihoro` 的远程订阅逻辑绑住。

## 4. 自动脚本一旦遇到网络慢，最好的办法就是拆开做

- 手动下载
- 手动解压
- 手动验证

## 5. 先让东西“跑起来”，再考虑自启、服务化、优化

你这次最后能成功，就是因为你后面不再纠结“优雅”，而是先让它真的工作。

------

# 八、最后给你一句压缩总结

**你这次挂梯子的成功路线，不是“把订阅链接喂给 mihoro”，而是“从 Clash Verge 拿到现成可用的 YAML 配置，手动安装 mihomo 内核，再直接用本地配置启动”。**