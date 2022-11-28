---
title: VScode+XDebug远程调试ThinkPHP RCE
categories:
  - 杂记
tags:
  - CTF
  - PHP
  - IDE
abbrlink: 5cdca07e
date: 2020-12-02 08:29:14
updated: 2020-12-02 08:29:14
---
#CTF #PHP #VSCode
# 环境准备

-   VScode
-   VMware

# 宿主机配置

1.  将ThinkPHP 5.0.5（或者其他源码）下载到宿主机当中，使用VScode打开项目
2.  安装PHP Debug
3.  点击左侧栏中的运行，选择创建launch.json文件，写入配置

{
            "name": "Listen for XDebug",
            "type": "php",
            "request": "launch",
            "pathMappings": {
                "虚拟机当中的源码位置": "${workspaceRoot}"
            },
            "port": 9000 //默认XDebug端口
}

# 虚拟机配置

1.  首先安装XDebug，[官网](https://xdebug.org/docs/install)有非常详细的安装教程。
2.  修改XDebug.ini配置，这里建议去写一个phpinfo()文件，在这里面查看

Additional .ini files parsed

/etc/php/7.4/apache2/conf.d/20-xdebug.ini

编辑此配置文件，一般第一行`zend_extension`是已经写好的，如果没有的话加上xdebug的路径就行

接着在下面写入其他配置

[XDebug]
xdebug.remote_enable = 1
xdebug.remote_autostart = 1
xdebug.remote_port = 9000 //端口可以自定义，与vscode的保持一致就行
xdebug.remote_connect_back = 1
xdebug.auto_trace = 1
xdebug.collect_includes = 1
xdebug.collect_params = 1
; xdebug.remote_log = /var/log/xdebug.log //可选

关于配置以及xdebug原理之后会详细说。

3.  重启服务，我的是apache，使用`service apache2 restart`重启服务。

# 开始调试

VScode中，在需要调试的地方下好断点，然后在点击运行中的开始调试

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1606658034288-172a33e3-3c50-4f7b-a9f6-e53fa05e6fbf.png)

vscode就会开始监听9000端口，等待xdebug的连接，然后去访问会触发断点的网页即可。

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1606658143202-4c2ad032-9691-4188-9ccb-0f5908b2034d.png)可以看到成功触发断点。

# XDebug原理

首先XDebug需要客户端以及服务端。

服务端就是我们在VMware中安装的XDebug，而客户端一般来说像PHPStorm这样的IDE自带，或者VScode自己去安装PHP Debug。

当进行XDebug调试时，首先客户端会开始监听端口，等待XDebug的连接，连接成功则开始调试。如图所示：

![](https://cdn.nlark.com/yuque/0/2020/gif/2658344/1606658438431-fb756647-266f-43e9-b346-9ada6c464e83.gif)

图片来自[http://guojianxiang.com/posts/2015-09-06-PHP_Debug_Tool-Xdebug.html](http://guojianxiang.com/posts/2015-09-06-PHP_Debug_Tool-Xdebug.html)

这里面是将`xdebug.remote_host`设为一个固定值，也就是说只会朝`remote_host`指向的IP发送调试请求。

但是如果将`xdebug.remote_connect_back`的值设为1，即可取消此限制。

设置这个值之后，xdebug会从http请求的头部获取客户端的ip当作remote_host，而原本的remote_host即使设置了，也会忽略。