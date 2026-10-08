---
slug: vivado-2018-3-arch-installation
title: 在 Arch Linux 上原生安装 Vivado 2018.3 HL WebPACK
date: 2026-09-10
excerpt: Vivado 2018.3 的网页安装器已经失效，只能在滚动更新的 Arch 上用全量包安装。本文记录补齐旧版兼容库、安装 WebPACK、配置电缆驱动，以及过程中踩到的坑。
tags:
  - fpga
  - vivado
  - xilinx
  - arch
  - linux
status: published
updatedAt: 2026-09-10
---

## 省流

Vivado 2018.3 的网页安装器已被官方下架，只能用约 19G 的全量包安装。在滚动更新的 Arch Linux 上，只要补齐 `libtinfo.so.5` 等旧版兼容库、用全量包装 WebPACK、再手动装好电缆驱动，就能原生跑起来，不需要分区装 Windows 或开虚拟机。

## 背景

最近需要用到 Xilinx 的 FPGA 工具链，目标版本是 2018.3，WebPACK 免费版就够用。手头的机器是 Arch Linux（滚动更新），而 Vivado 官方只支持 Ubuntu / RHEL / SLES 等固定发行版，并不支持 Arch。

一开始考虑过分区装 Windows 或者开虚拟机，但缩小 btrfs 分区风险太高，最终决定直接在 Arch 上原生安装。

## 环境

- 系统：Arch Linux（滚动更新），内核 7.2.x，glibc 2.44
- 目标：Vivado HL WebPACK 2018.3
- 下载器：Xilinx Platform Cable USB
- 安装位置：`/opt/Xilinx`

## 一个重要的前提：网页安装器已经失效

2018.3 的 Linux 网页安装器（`Xilinx_Vivado_SDK_Web_..._Lin64.bin`）现在已无法使用——AMD 下架了旧的组件源，跑起来会下不到东西。只能下载全量包 `Xilinx_Vivado_SDK_2018.3_1207_2324.tar.gz`（约 19G）。

可能的捷径：登录 AMD 账号后，直接在下面的 URL 末尾拼接文件名即可触发下载：

```
https://account.amd.com/en/forms/downloads/xef-vivado.html?filename=Xilinx_Vivado_SDK_2018.3_1207_2324.tar.gz
```

## 安装步骤

### 1. 补齐依赖

Arch 的库普遍偏新，Vivado 需要一些旧版 ABI 的库：

```bash
sudo pacman -S --needed inetutils libxi libxtst xorg-xlsclients rlwrap jre8-openjdk ncurses5-compat-libs
yay -S --needed fxload
```

其中比较关键的是：

- `ncurses5-compat-libs`（来自 archlinuxcn 源）提供 `libtinfo.so.5` / `libncurses.so.5`，缺了安装器会卡在 “Generating installed devices list” 这一步。
- `libxcrypt-compat` 提供 `libcrypt.so.1`，新系统一般已经预装。
- `fxload` 用于 Platform Cable USB 的固件加载。

事后复盘，`jre8-openjdk` 和 `rlwrap` **可能非必需**：安装器和 Vivado 本身都自带 JRE 9.0.4，自带的 rlwrap 也能正常工作。

### 2. 安装

解压全量包后进入目录（目录里含 `xsetup` 和 `payload/`），直接用 `-e/-l` 指定版本和路径，跳过交互式的 ConfigGen：

```bash
sudo ./xsetup -a XilinxEULA,3rdPartyEULA,WebTalkTerms -b Install \
     -e "Vivado HL WebPACK" -l /opt/Xilinx
```

> 坑：`xsetup -b ConfigGen` 是**交互式**的，会等待你输入版本编号。如果在没有输入的环境里运行（脚本、管道等），它会一直循环并疯狂刷日志——我这边一度生成了 4.1G 的日志文件。真要用的话，请务必 `printf '1\n' | xsetup -b ConfigGen`。

安装完约 28G，`/opt/Xilinx/Vivado/2018.3/bin/vivado` 就位。

### 3. 安装电缆驱动

```bash
cd /opt/Xilinx/Vivado/2018.3/data/xicom/cable_drivers/lin64/install_script/install_drivers
sudo ./install_drivers
sudo udevadm control --reload-rules && sudo udevadm trigger
```

脚本会安装 `/etc/udev/rules.d/52-xilinx-*.rules`，其中 Platform Cable USB 对应 `52-xilinx-pcusb.rules`（vendor `03fd`）。装完把电缆拔插一次以应用规则。

### 4. 配置环境变量

```bash
echo 'source /opt/Xilinx/Vivado/2018.3/settings64.sh' >> ~/.zshrc
```

### 5. 验证

```bash
vivado -version
# Vivado v2018.3 (64-bit)
```

## 踩坑记录

1. **ConfigGen 交互死循环**：见上，无输入会刷出几个 G 的日志。用 `-e/-l` 直接安装最省心。
2. **升级内核后未重启导致设备不识别**：运行中的内核版本和磁盘上的模块目录不一致时，`uas` / `usb-storage` 等模块加载不了，插上移动硬盘会认不到。做硬件、存储相关操作前，先重启到当前内核最稳妥。
3. **`rlwrap`、`jre8-openjdk` 可能非必需**：自带 JRE 9.0.4 与自带 rlwrap 都能用。

## 清理与卸载

安装完成后，全量包和解压出来的目录都可以删掉（约 20G）。若要彻底卸载：

```bash
# 卸载 Vivado
cd /opt/Xilinx/.xinstall/Vivado_2018.3 && sudo ./xsetup -b Uninstall
```

## 参考资料

**AMD / Xilinx 官方**

- 2018.3 下载页：https://www.amd.com/en/support/downloads/adaptive-socs-and-fpgas/development-tools/2018-3.html
- UG973 发布说明 / 安装 / 许可：https://docs.amd.com/r/en-US/ug973-vivado-release-notes-install-license
- 安装器与 OS 支持矩阵：https://www.amd.com/en/support/adaptive-socs-and-fpgas/installer-info-general.html
- `libtinfo.so.5` 说明（AR 76585）：https://adaptivesupport.amd.com/s/article/76585

**Arch 社区**

- ArchWiki Xilinx Vivado：https://wiki.archlinux.org/title/Xilinx_Vivado
- ArchLinuxCN Xilinx Vivado：https://wiki.archlinuxcn.org/wiki/Xilinx_Vivado
- AUR vivado 包：https://aur.archlinux.org/packages/vivado
- AUR PKGBUILD（含一些安装技巧）：https://aur.archlinux.org/cgit/aur.git/plain/PKGBUILD?h=vivado
- archlinuxcn 仓库（`ncurses5-compat-libs` 来源）：https://github.com/archlinuxcn/repo
- AUR fxload：https://aur.archlinux.org/packages/fxload
