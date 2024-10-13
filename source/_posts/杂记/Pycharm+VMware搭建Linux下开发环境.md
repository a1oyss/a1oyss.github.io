---
title: Pycharm+VMware搭建Linux下开发环境
tags:
  - 随笔记录
categories:
  - 杂记
abbrlink: 58699
date: 2020-10-08 18:44:09
updated: 2020-10-08 18:44:09
---
#IDE 
# 前言

  

弄了一天的vim+xshell，结果还是不尽人意，spacevim要求终端支持真彩色，但是xshell显然⑧行，将`enable_guicolors`设置为false后才勉强能看。

  

于是突发奇想，将pycharm通过ssh连接到虚拟机当中的python环境，能不能直接在主机上面写脚本调试，查了下还真有这个功能，不过需要pro版本。

  

# 准备工作

  

1.  PyCharm Pro
2.  VMware Workstation
3.  科学上网

  

# 配置

  

## 主题

  

File->Settings..->Plugins，安装Material Theme UI，接着按照自己喜欢的来配就行

  

## 配置SSH Interpreter

  

安装好PyCharm Pro之后，创建一个项目，创建好之后找到File->Settings..->Project: ctf->Project Interpreter配置python解释器

  

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602153037485-394af4f2-19fa-4fc5-bf3d-39ae5070cc1e.png)

  

选择Add..->SSH Interpreter

  

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602153038413-b7c8f816-bc23-4b7d-8bd8-b942a9a3bc7a.png)

  

将虚拟机的ip地址和用户名填写到相应的地方，然后NEXT。

  

填写密码

  

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602153037359-53ad4c7a-e476-4f78-a933-0c20d1a5599a.png)

  

设置解释器路径和文件同步位置，然后FINISH

  

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602153037610-0c30b0ee-f187-4e09-b062-efe90dbd551b.png)

  

FINISH之后PyCharm会先进行一次同步，同步python库，并且把一些配置文件什么的给传过去，时间略长。

  

PyCharm的同步还有版本对比功能

  

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602153039519-485d68c3-bdd4-4077-b6e2-803a96d51c72.png)

  

不得不说比预想中还要好用。

  

## 配置hexo

  

File->Settings..->Tools->Terminal中可以设置PyCharm的终端

  

将Start directory设置为hexo根目录

  

shell path可以直接填写终端名称，例如powershell.exe

  

Tab name，标签名称，随便修改。

  

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602153037444-308883fb-4fa7-4e19-b28f-8dc1e5bc9836.png)

  

这样就可以随时进行hexo的相关操作了。

  

## 配置SSH Terminal

  

File->Settings..->Tools->SSH Terminal

  

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602153037542-9d00c3ac-9446-4e4a-87fe-aabca5182ef0.png)

  

Deloyment server中选择root@xxx.xxx.xxx.xxx:22

  

这样每次打开就不需要再选一次了。

  

# FAQ

  

## PyCharm中进行调试后，console中输出□=

  

应该是PyCharm的问题，有两种解决办法：

  

1.  Run->Edit Configurations..->Templates->Python->Enviroment variables中添加环境变量

PWNLIB_NOTERM=True

2.  修改`/usr/local/lib/python2.7/dist-packages/pwnlib/args.py`最后一行，注释掉`term.init()`

  

# 总结

  

pro版本还是好用，配置好之后，直接在一个窗口中进行所有操作，不需要几个窗口之间来回切了，方便了不少。