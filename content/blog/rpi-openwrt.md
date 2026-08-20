---
slug: rpi-openwrt
title: 树莓派 Docker OpenWrt/Kwrt 旁路由配置
date: 2026-06-20
excerpt: 在树莓派 4B 上用 Docker 运行 OpenWrt/Kwrt 容器作为旁路由的配置记录。
tags:
  - raspberry-pi
  - openwrt
  - docker
  - network
status: published
updatedAt: 2026-06-20
---
## 未解决的问题

在用神秘插件的时候，第一次加载界面总是非常慢，要大概七八秒。

## 起因

考数电前一天，看到树莓派能不用刷机，跑一个 Docker 的 OpenWrt/Kwrt 容器作为旁路由。想着这个可玩性应该不错，就花了半天时间配了一下。

## 环境

- 机型：Raspberry Pi 4B
- Docker 镜像网址：https://openwrt.ai/

## 网络配置

```text
config interface 'lan'
        option device 'eth0'
        option proto 'static'
        option ipaddr '192.168.31.2'
        option netmask '255.255.255.0'
        option gateway '192.168.31.1'
        option dns '192.168.31.1 114.114.114.114'
```

Kwrt 的设置：开启 NAT，关闭 DHCP（作为旁路由）

## 启动容器

```bash
docker run -d \
  --name openwrt \
  --restart always \
  --network macnet \
  --ip 192.168.31.2 \
  --privileged \
  my-openwrt:v1 \
  /bin/sh -c "ip link set eth0 up && ip route add default via 192.168.31.1 dev eth0 ; /sbin/init"
```

## 注意事项

- 网卡混杂模式要写入 systemd 服务中，或者 NetworkManager 配置里面，不能写在 rc.local，因为后者已经被弃用。
- 防火墙记得设置
- 如果要用 PassWall 插件，记得全部选项都看一遍

## 待解决的问题

DNS 是目前这套方案的时间瓶颈，目前还没有去解决。

## 其他说明

不使用官方 Docker 镜像的理由：绝对不是我拉不下来镜像😡

剩下的步骤问 AI 应该就行，网站也有教程：https://openwrt.ai/docker%E7%89%88openwrt%E6%97%81%E8%B7%AF%E7%94%B1%E5%AE%89%E8%A3%85%E8%AE%BE%E7%BD%AE%E6%95%99%E7%A8%8B/