---
layout: post
title: '修改 JEB 的字体大小'
subtitle: '查阅官方文档寻找解决方案'
date: 2019-12-19
categories: 技术
cover: '../../../assets/img/post_bg/20191216.jpeg'
tags: android 工具
---


## 1. 关于 JEB

&#160; &#160; &#160; &#160;JEB 全称 [JEB Decompiler](https://www.pnfsoftware.com)，是一款强大的 android 应用分析工具，它能够解析出其他工具无法解析出的代码，因此基本是分析 android 应用必备的一款工具。（该工具的高级功能是付费的，网上可以找到[破解版](https://github.com/WuFengXue/android-reverse)，__如果手头比较宽裕，建议支持正版。__）

## 2. 修改字体大小

&#160; &#160; &#160; &#160;将 JEB 升级到 3.0 后，发现字体变得很小，搜了一圈没有找到修改方法，就自己翻官方文档，最终找到了[（官方文档）](https://www.pnfsoftware.com/jeb2/manual/settings/#styles-and-fonts)，如下（Mac 上）：

```doc
顶部【编辑】菜单 -> 【Fonts and styles...】 -> 【流程序字体】下的【选择字体】
 -> 弹出的新窗口中调整大小 -> 关闭新窗口 -> 点击【确定】
```

&#160; &#160; &#160; &#160;也可在帮助中找到，如下：

```doc
顶部【帮助】菜单 -> 搜索框输入【font】 -> 选择【菜单项 Fonts and styles...】
 -> 同上
```

## 3. 总结

&#160; &#160; &#160; &#160;当搜索引擎找不到解答的时候，可以尝试在官方文档或者工具自带的帮助选项中查找。

