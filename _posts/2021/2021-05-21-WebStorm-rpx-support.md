---
layout: post
title: 'WebStorm 支持微信小程序的 rpx 单位'
subtitle: '支持 uni-app 的 vue 文件'
date: 2021-05-21
categories: 技术
cover: '../../../assets/img/post_bg/20210521.jpeg'
tags: 小程序 工具
---


## 1. 前言

&#160; &#160; &#160; 最近在参与公司的小程序项目，项目里的前端同学大部分直接使用 [vs code](https://code.visualstudio.com)，在体验了一小段时间后，基于对 [Android Studio](https://developer.android.google.cn/studio/releases/index.html) 的熟悉度，决定还是先切换到 [WebStorm](https://www.jetbrains.com/webstorm/) （都是 [JetBrains](https://www.jetbrains.com/?var=1) 家的 IDE）试试，实在不行再切回去。

&#160; &#160; &#160; 配置 + 插件后，工作基本可以正常开展了，除了恼人的 rpx 单位问题。

## 2. rpx 单位问题

&#160; &#160; &#160; [rpx（responsive pixel）](https://developers.weixin.qq.com/miniprogram/dev/framework/view/wxss.html) 是微信小程序为了适配不同屏幕分辨率推出的一种尺寸单位。因为是国内的自定义单位，所以 WebStorm 无法识别，进而导致两个问题：

* IDE 会报【Mismatched property value 】错误并标红

![](../../../assets/img/2021/05/WebStorm-rpx/mismatched-property-value.png)

* 执行代码格式化之后，rpx 和前面的数值之间会多出一个空格

![](../../../assets/img/2021/05/WebStorm-rpx/reformat-code-exception.png)




## 3. 过渡方案

&#160; &#160; &#160;在网上找了一圈，找到一种勉强可以接受的方案（具体见[《让webstorm编写微信小程序时支持rpx》](https://blog.csdn.net/lulitianyu/article/details/83240864)），大致思路如下：

* 关闭【Invalid CSS property value】检测项（解决 IDE报错标红）
  * 缺点：关闭是一刀切的，对于其他单位的检测也被关掉了
* 添加文件修改监听，借助 [sed](https://www.runoob.com/linux/linux-comm-sed.html) 命令将多出的空格去掉（解决代码格式化问题）
  * 缺点1：特定场景，比如 rpx2px 之类的方法调用之前的空格也会被移除
  * 缺点2：低配电脑或运行程序比较多时，编辑过程会弹出内存与磁盘内容不一致的对话框

## 4. 新方案

&#160; &#160; &#160;因为使用的是个人的老电脑，经常性地开关文件监听（为了解决缺点2，只在要格式化时打开）感觉比较麻烦，所以就又找起了解决方案。

&#160; &#160; &#160;这次改成直接在插件市场里面搜索 uni-app、vue、rpx 和 wx 关键字，还真的找到了两款：

* [Wxapp Support](https://plugins.jetbrains.com/plugin/12539-wxapp-support)
* [Wechat mini program support](https://plugins.jetbrains.com/plugin/13396-wechat-mini-program-support)



### 4.1 Wxapp Support

&#160; &#160; &#160;直接在 WebStorm 设置中的插件市场搜索安装，未能生效。查看其 [github 项目](https://github.com/zxj5470/wxapp-intellij)，发现市场的版本是旧的，虽然作者提供了最新体验版的下载链接，但是因为已经过了缓存期限，下载不了。

&#160; &#160; &#160;本来自己克隆后想自己编译的，后面找到解决方案了，就放弃了。



### 4.2 wechat-miniprogram-plugin（最终方案）

&#160; &#160; &#160;直接在 WebStorm 设置中的插件市场搜索安装，也是未能生效。查看插件文档介绍，发现有类似需求的评论。

![](../../../assets/img/2021/05/WebStorm-rpx/rating-and-reviews.png)

&#160; &#160; &#160;查看其 [gitee 项目](https://gitee.com/zxy_c/wechat-miniprogram-plugin)，最终在一个[已关闭的 issue](https://gitee.com/zxy_c/wechat-miniprogram-plugin/issues/I1J7PN) 中找到了解决方案：

* WebStorm -> Preferences... -> 左侧搜索框输入 wechat -> Languages & Frameworks -> Wechat Mini Program -> 右侧的菜单中，将【小程序支持】切换到【启用】-> Apply -> OK -> 等待1至2分钟，标红自动消失，格式化代码不再添加空格

![](../../../assets/img/2021/05/WebStorm-rpx/open-wechat-mini-program.png)



## 5 总结

* 你遇到的问题，别人可能已经遇到过并解决了；如果还没有人解决，可以尝试成为第一个解决的人
* 搜索引擎搜索不到结果的时候，可以尝试直接在 IDE 插件市场或 github 中进行搜索
* 可以尝试搜索或翻阅开源项目的 issues 栏目，说不定可以找到方案或灵感



## 6 附：方案汇总及个人 uni-app 开发使用的插件分享



&#160; &#160; &#160;__方案汇总：__

* 安装 wechat-miniprogram-plugin 插件
* 按 4.2 节的方式，配置插件



&#160; &#160; &#160;__个人 uni-app 开发使用的插件：__

* [Git Commit Template](https://plugins.jetbrains.com/plugin/9861-git-commit-template)：git 提交模板
* [IDE Eval Reset](https://zhile.io/2020/11/18/jetbrains-eval-reset-da33a93d.html)：Webstorm 试用期限重置（有能力的建议还是付费支持一下）
* [Rainbow Brackets](https://plugins.jetbrains.com/plugin/10080-rainbow-brackets)：不同层级的括号展示不同的颜色
* [Vuesion Theme](https://plugins.jetbrains.com/plugin/12226-vuesion-theme)：漂亮的颜色主题
* [Wechat mini program support](https://plugins.jetbrains.com/plugin/13396-wechat-mini-program-support)：小程序开发支持，目前主要解决 rpx 问题

