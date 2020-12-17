---
title: Windows To Go 安装与 WSL 的使用
date: 2018-10-19 19:28:02
tags: [WSL, Windows To Go]
---

迫于之前的电脑机械硬盘的速度实在是无法忍受，正好又赶上了固态降价，于是入了一块固态硬盘做移动硬盘，准备将系统安装到上面。

<!--more-->

## 材料

- 光威 SSD 240G
- Windows10 1803 企业版 ISO

## Windows To Go

### 准备镜像

首先准备好系统镜像，可以在 [itellyou](https://msdn.itellyou.cn/) 上下载，要求必须为企业版。WTG 安装的系统无法升级，而 WSL 要求 Win10 1706 以上的系统版本，因此直接选择新版镜像。

### 格式化硬盘

准备好硬盘盒，通过 USB 接口连接到电脑。此时电脑虽然识别到了设备，但是却无法使用。

右键此电脑 - 管理 - 存储 - 磁盘管理，找到移动硬盘对应的磁盘，然后将其格式化。

### 安装 Windows To Go

使用软件（UltraISO 等）将下载的系统镜像解压到移动硬盘里。

进入设置，搜索 Windows To Go，打开后会自动识别到你的移动硬盘。选择企业版进行安装，安装完成后重启，修改启动项从移动硬盘启动。

### 系统安装

接下来就是要等待了，系统会重启几次，设置用户密码等信息，进入系统。

刚安装完的系统是没有驱动的，分辨率可能会有问题，也可能无法使用无线网络。此时需要安装一个驱动人生（注意不是驱动精灵）来安装驱动，驱动安装完成之后就可以过河拆桥把驱动人生卸载了。

此时已经完成了安装流程，接下来就是安装各种软件等操作了。

## WSL

WSL 的全称为 Windows Subsystem for Linux，可以在 Windows 下模拟使用 Linux 的系统。在 Win10 中安装还是很简单的，只要打开 Microsoft Store 搜索 WSL 安装即可。要注意系统版本要高于 1706，否则无法安装，我就因此重新装了一遍 WTG。

现在 Windows 下很多软件都对 WSL 有不错的支持，比如 VSCode，可以自动检测到 WSL 并将终端修改为 WSL。

经我这阵子的观察，WSL 还是很好用的。只是还是有一些与原生的 Linux 不同，比如 nmap 等软件无法使用（Windows 下有个替代版，直接 apt 安装的无法使用）。但总体来说，WSL 还是非常强大的。