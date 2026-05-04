---
slug: arch-linux-nvidia-driver-troubleshooting
title: ArchLinux 安装 Nvidia 驱动过程中我所踩过的坑
date: 2026-04-30
excerpt: eGPU 场景的核心问题是“驱动加载时序”与“链路稳定性”
tags:
  - arch
  - nvidia
  - egpu
  - linux
status: published
updatedAt: 2026-05-1
---
## 省流
通过安装 bolt，和调整 nvidia 服务与 SDDM 启动顺序来解决 nvidia-smi 检测不到显卡的问题，但是显卡直通显示器目前还做不到。

## 后日谈

- 2026.4.30，nvidia-smi 已经可以正常检测显卡了，但是 Wayland 和 eGPU 的问题始终无法解决，可能过几天尝试一下 X11 的方案。

## 一个偶然的想法
前天晚上突然想配点环境玩玩，于是准备在 Fedora 和 ArchLinux 中选一个。开始是打算配一个 WSL 系统玩玩得了，但是发现 Arch 好像没有现成的 WSL 内核，问了问别人，最终打算还是从硬盘划 200G 出来把玩一下。

安装过程和心得过两天写一下，这里先把紧要的先写下来（甚至这里的不少内容都是当时处理完直接让 AI 固定下来的），后面再看输入法、声卡驱动、显示器 HDR 之类的问题，也当是写一个小小的备忘录了。

我自己用的是一套比较特殊的显示方案：笔记本屏幕使用核显，然后通过雷电口连接一个雷蛇 Core X V2，然后显卡直通一个 HKC 显示屏（HDR 问题还没有解决）。

## 先说系统环境

- 系统: Arch Linux
- 内核: 6.19.14-arch1-1 (含 LTS)
- eGPU: Razer Core X V2 (USB4)
- 显卡: NVIDIA GeForce RTX 5060 Ti (GB206)
- 驱动: nvidia-open-dkms 595.58.03 (仓库替代 nvidia-dkms)

## 我的思路
我先是安装了 nvidia-open-dkms，一个用于较新 GPU 型号的驱动包，这个包会在安装的时候自动根据你的内核版本编译驱动。

```bash
sudo pacman -S nvidia-open-dkms
```

但是 lspci 和 nvidia-smi 一直没有反应，然后还会关机会卡在 "Reached target: Graphical Interface"。
此时我打开了《部落冲突：皇室战争》，和室友来了几把酣畅淋漓的 2v2 对战。我玩的时候，突然想到显卡坞，会不会需要额外的驱动（毕竟声卡驱动都是要独立安装的），然后安装了 bolt。

```bash
sudo pacman -S bolt
```

## 求助 AI
然后就发现 fastfetch 和 lspci 识别到了 RTX 5060Ti。但是事情没这么简单，nvidia-smi 仍然显示 "No devices were found"（可能先前显示的是驱动未就绪之类，我不太记得了）。翻阅 Arch Wiki 无果之后，就用 opencode 问 GPT 5.2 Codex 怎么解决，它给出的最终解决方法是：

1. 清空 initramfs 预加载模块
   - 文件: /etc/mkinitcpio.conf
   - 修改: MODULES=()

2. 重建 initramfs
   - 命令: mkinitcpio -P

3. 新增 systemd 服务，在 Thunderbolt 之后加载 NVIDIA
   - 文件: /etc/systemd/system/nvidia-egpu.service
   - 作用: bolt.service 后 modprobe nvidia 系列模块

4. 让 SDDM 等待 eGPU 模块加载完成
   - 文件: /etc/systemd/system/sddm.service.d/override.conf
   - 内容: After/Requires nvidia-egpu.service（当需要热插拔时去掉Requires！）

相关的文件改动有：

- /etc/systemd/system/nvidia-egpu.service

```ini
[Unit]
Description=Load NVIDIA modules after Thunderbolt
After=bolt.service
Wants=bolt.service

[Service]
Type=oneshot
ExecStart=/usr/bin/modprobe nvidia
ExecStart=/usr/bin/modprobe nvidia_modeset
ExecStart=/usr/bin/modprobe nvidia_uvm
ExecStart=/usr/bin/modprobe nvidia_drm
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

- /etc/systemd/system/sddm.service.d/override.conf

```ini
[Unit]
After=nvidia-egpu.service
# 当不需要热插拔时，取消下一行的注释
# Requires=nvidia-egpu.service
```

- /etc/modprobe.d/nvidia-egpu.conf

```conf
options nvidia NVreg_EnableMSI=0
```

## 关键日志与结论
### 1) 设备可见但驱动无法完成初始化
dmesg 关键错误:

- NVRM: Xid 79, GPU has fallen off the bus
- pciehp: Slot Link Down / Card not present
- nvidia-modeset: Error while waiting for GPU progress

结论: eGPU 在驱动初始化阶段掉线，属于链路/初始化时序问题。

### 2) MSI-X 失败导致显示设置卡死
后续日志显示:

- NVRM: Failed to enable MSI-X
- RmInitAdapter failed

结论: MSI/MSI-X 中断路径在 eGPU 上不稳定，触发显示重配时崩溃。

### 3) MSI-X 失败 -> 禁用 MSI
问题: 调整多显示器时卡死
措施:

1. 禁用 MSI
   - 文件: /etc/modprobe.d/nvidia-egpu.conf
   - 内容: options nvidia NVreg_EnableMSI=0
2. 重建 initramfs
   - 命令: mkinitcpio -P

### 4) 临时 GRUB 参数（无效的参数）
尝试:

- pci=realloc
- pcie_aspm=off


## 经验总结

- eGPU 场景的核心问题是“驱动加载时序”与“链路稳定性”
- initramfs 过早加载会导致 GPU 掉线
- 将驱动加载延后到 Thunderbolt 稳定后，是最有效的修复手段
- MSI/MSI-X 在 eGPU 上可能不稳定，必要时禁用
