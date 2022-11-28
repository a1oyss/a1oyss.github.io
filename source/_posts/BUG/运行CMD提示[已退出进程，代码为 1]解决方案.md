---
title: '运行CMD提示[已退出进程，代码为 1]解决方案'
categories:
  - BUG
tags:
  - BUG
abbrlink: 10e60d97
date: 2021-05-27 10:25:33
updated: 2021-05-27 10:25:33
---

今天准备使用npm时遇到了这个问题，npm install没有反应，npm也没反应，但是nvm确确实实是有安装好nodejs的。

然后一步步找，突然看到windows下的npm命令是调用npm.cmd

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2658344/1622081898067-f4afff9f-cbdd-4099-8f5a-52377ffc6fd1.png#align=left&display=inline&height=100&margin=%5Bobject%20Object%5D&name=image.png&originHeight=200&originWidth=559&size=17226&status=done&style=none&width=279.5)

接着就试着直接运行，发现一个黑框一闪而过。

然后再尝试打开cmd，也是一个黑框一闪而过，感觉问题应该就是出在这。

使用windows terminal打开cmd，结果就出现了

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2658344/1622082027615-7a316adb-8f69-4aa8-9f7c-21c68a935138.png#align=left&display=inline&height=108&margin=%5Bobject%20Object%5D&name=image.png&originHeight=216&originWidth=754&size=13853&status=done&style=none&width=377)

网上搜索到的注册表，gpedit，全都没有效果，后来直接去stackoverflow搜，找到了一个看起来很像的帖子：[https://stackoverflow.com/questions/66335300/cmd-crashes-with-exit-code-1-after-uninstalling-anaconda/](https://stackoverflow.com/questions/66335300/cmd-crashes-with-exit-code-1-after-uninstalling-anaconda/)

看到后面有个anaconda，突然想起来之前我好像也装过一次，帖子描述也和我一样：

直接打开cmd无效，但是从powershell中打开cmd /d就可以正常进入。

下面给出了一条解决方法：

```

C:\Windows\System32\reg.exe DELETE "HKCU\Software\Microsoft\Command Processor" /v AutoRun /f

```

使用其他终端运行命令后，问题解决。