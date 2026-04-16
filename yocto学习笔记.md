# 第一阶段

这是一个多 layer 组织的系统

BitBake 会按 recipe 干活

最终目标是 image

image 是针对 machine 出的

改源码只是其中一步，能不能进镜像还得看元数据链路

### 名词解释：

- Yocto：Yocto 是一套用于构建定制 Linux 系统的工具和方法，OpenBMC 用它来管理配置并生成 BMC 镜像。
- OpenEmbedded：OpenEmbedded 更偏底层的 metadata 和构建生态，Yocto 项目是在这套体系之上提供工具、流程和文档支持。
- BitBake：BitBake 是执行构建任务的引擎，会根据 recipe 和 layer 中的 metadata 去完成构建流程。
- layer：layer 是组织 metadata 的边界，用来隔离不同类型的定制；OpenBMC 中很多 layer 以 `meta-*` 目录形式出现。
- recipe：recipe 描述一个软件怎样被获取、构建、安装和打包，是 BitBake 执行构建的直接依据。
- image：image 负责把需要的包和配置组合成最终系统镜像。
- machine：OpenBMC 本质上是基于 Yocto/OE 体系构建出来的 BMC Linux 发行版。它通过多个 layer 组织 recipes、配置和平台差异，再由 BitBake 构建成面向具体 machine 的镜像。

### 代码结构：

打开 OpenBMC 仓库，数一数你看到的 `meta-*` 目录，并猜它们大致分工。

**Openbmc的功能是如何进入固件的：**

OpenBMC 的功能要进入最终固件，通常需要先有对应的源码或配置，再由某个 layer 中的 BitBake recipe 描述其获取、构建、安装和打包方式；BitBake 依据这些 metadata 生成包和安装内容；随后 image 决定是否将该功能纳入系统，而 machine 则决定目标硬件平台，最终产出对应平台的 OpenBMC 固件镜像。