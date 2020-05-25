---
layout:       post
title:        Visual Studio 安装被阻止
category:     tech
description:  Visual Studio 安装被阻止，安装的版本早于已安装的版本。
---

想更新一下 Visual Studio 的版本，更新了 Vsisual Studio 2013.4，没有问题，

继续更新 Visual Studio 2013.5 安装后，发现使用的一个插件无法使用。

找了许久暂时没有找到解决原因，又担心影响使用，就暂时卸载了。

卸载 2013.5 后，担心卸载的同时卸载了之前的安装内容，就想着把 2013.4 重新安装一遍，

结果重新安装的时候就提示了 Setup Blocked.  给出的警告消息：

**The product version that you are trying to set up is earlier than the version already installed on this computer.**

Google 一下找到一篇文章，按照提供的方法解决了问题。

#### 解决步骤如下

1. 查询关键字 **Detected related bundle**，复制后面紧跟的一串 GUID 字符串，例如 {17551f85-1d1c-4142-a83f-bbd18a3522c2}

2. 打开注册表去搜索这串 GUID: 17551f85-1d1c-4142-a83f-bbd18a3522c2，找到对应的项去修改 **BundleVersion** 的值，一般按照格式修改比你要安装的版本小就可以了。

3. Try again.

参考：<a href="https://johndelizo.wordpress.com/2013/12/23/visual-studio-2013-setup-blocked-the-product-version-that-you-are-trying-to-set-up-is-earlier-than-the-version-already-installed-on-this-computer-fixed/" target="_blank">Visual Studio 2013 Setup Blocked: The product version that you are trying to set up is earlier than the version already installed on this computer. [FIXED]</a>
