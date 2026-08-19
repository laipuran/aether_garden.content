---
slug: how-to-build-a-website
title: 如何自建一个网站——Jekyll篇
date: 2022-12-16
excerpt: 一份基于 Jekyll 的个人网站搭建记录，从环境准备到本地运行。
tags:
  - jekyll
  - website
status: published
updatedAt: 2026-08-19
---

### 1. 准备工作

- 使用任意 Linux 发行版
- 可以选择换成 [清华的源](https://mirrors.tuna.tsinghua.edu.cn/)

### 2. 安装 Ruby 和 Ruby-dev

这里以 Debian 系发行版为例：
```bash
sudo apt install ruby -y
sudo apt install ruby-dev -y
```

### 3. 接着安装 gem 包

先换 gem 源：

```bash
gem sources --add https://mirrors.tuna.tsinghua.edu.cn/rubygems/ --remove https://rubygems.org/
gem sources -l
```

然后安装 bundler, jekyll 两个包：

```bash
# 安装 bundler
sudo gem install bundler
# 换 bundler 源
bundle config mirror.https://rubygems.org https://mirrors.tuna.tsinghua.edu.cn/rubygems
# 安装 jekyll
sudo gem install jekyll
```

### 4. 切换到你想创建网页的目录，创建网站

```bash
cd ./...
```

接着新建网站：

```bash
jekyll new my awesome-website
```

### 5. ~~像本篇一样~~弄好头信息

```yaml
---
layout: post
title: 如何自建一个网站
date: 2022-12-16 16:04:28 +0800
---
```

使用 markdown 写好文章内容，然后就可以。

### 6. 运行

```bash
bundle exec jekyll s
```

Done!

### 7. 视频教程

如何制作一个像这样的网站：

<iframe src="//player.bilibili.com/player.html?aid=678557527&bvid=BV1nm4y1f7sq&cid=496593480&page=1" scrolling="yes" border="0" frameborder="no" framespacing="0" allowfullscreen="true" width="640" height="360"> </iframe>
