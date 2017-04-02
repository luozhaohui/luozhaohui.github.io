---
layout: post
title: "Subline Text 2 安装 Markdown 插件"
date: 2014-10-05 22:35:02 +0800
comments: true
categories: [工具软件]
tags: [工具]
description: Subline Text 2 安装 Markdown 插件
keywords: Markdown, Subline Text 
---

##简介
本文介绍如何在 Subline Text 2 中安装 Markdown 插件。

<!--more-->

## 步骤

1. 按 ctrl + ` 调出 console；  

2. 粘贴以下代码到底部命令行并回车： 

``` python
    import urllib2,os;pf='Package Control.sublime-package';ipp=sublime.installed_packages_path();os.makedirs(ipp) if not os.path.exists(ipp) else None;open(os.path.join(ipp,pf),'wb').write(urllib2.urlopen('http://sublime.wbond.net/'+pf.replace(' ','%20')).read())
```

3. 稍作等待，若在 output 窗口输出如下信息，表示安装成功：  
> Please restart Sublime Text to finish installation   

4. 然后重启 Subline Text 2，这时在菜单 Perferences -> package settings 中会出现 package control 项；  

5. 点击 package control 项(或 Mac 系统下按 cmd + shift + p，Windows 系统下按 ctrl + shift + p)，接着输入 install package，回车，稍作等待（需要连接网络）；  

6. 在上方的输入窗口中输入 markdown，就会列出一堆与 Markdown 相关的插件；

7. 依次安装 Markdown Build 和 Markdown Preview 两个插件即可；

8. 新建后缀为 .markdown 的文件，按照 [Markdown 语法](http://wowubuntu.com/markdown/#blockquote) 规则添加内容；  

9. 选择菜单 Tools -> Build System -> Markdown，然后选择 Tools -> Build 编译，就会在与 .markdown 文件同目录下生成相应的 .html 文件；

10. Mac 系统下按 cmd + shift + p (Windows 系统下按 ctrl + shift + p)，输入 markdown，在自助提示列表中选择 Markdown Preview: Preview in browser，则会在浏览器中预览其效果。 

---
补充：

1. Mac 系统下可以安装Markdown编辑工具 [Macdown](http://macdown.uranusjr.com/) ；  

2. 或直接使用在线Markdown编辑工具：[mahua](http://mahua.jser.me/) 。
