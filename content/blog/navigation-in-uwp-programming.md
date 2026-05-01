---
slug: navigation-in-uwp-programming
title: UWP 中的页面导航
date: 2020-05-03
excerpt: 通过一个空白应用示例，演示 UWP 中新增页面与 Frame 导航的基本做法。
tags:
  - uwp
  - navigation
  - windows
status: published
updatedAt: 2020-05-03
---

在阅读这一篇文章时，如果您没有安装 UWP，请参照我写的：

[开始学习UWP](https://laipuran.github.io/2020/05/21/Start_to_learn_uwp.html)

OK, let's start working.

首先，创建一个空白应用，进入后，在左上角项目中新建项目。

![UWP加入新页面](https://laipuran.github.io/blog-img/UWP%E5%8A%A0%E5%85%A5%E6%96%B0%E9%A1%B5%E9%9D%A2.png)

在 **新建的页面** 加入内容，如：

```xml
<TextBlock FontSize="48">Another Page!<\TextBlock>
```

在 `MainPage.xaml` 的 **grid** 内加入：

```xml
<Button Name="NavigationButton" Click="NavigationButton_Click">Go to next page<\Button>
<Frame x:Name""MyFrame><\Frame>
```

将输入位置点到 **NavigationButton_Click**，按 F12 导航到 `MainPage.xaml.cs` 函数处，加入：

```cs
MyFrame.Navigate(typeof(你新建页面的名字));
```

这可以在你设置的 Frame 位置转移到另外一个页面。

按 **Ctrl** + **F5** 运行代码，查看结果。

[下载实例](https://laipuran.github.io/blog-img/Navigate.rar)
