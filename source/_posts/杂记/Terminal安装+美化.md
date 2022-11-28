---
title: Terminal安装+美化
categories:
  - 杂记
tags:
  - Terminal
abbrlink: 2ff2acab
date: 2022-05-25 15:48:26
updated: 2022-05-25 15:48:26
---
#Terminal
# 前言

Powershell这体验着实是不行，乱码，颜色无法显示。是时候该换Windows Terminal了

# 安装

## Github安装

下载页面[Github](https://github.com/microsoft/terminal/releases)，下载下来安装就可以了，值得注意的是，如果从 GitHub 安装，终端将不会自动更新为新版本。

## Microsoft Store安装（推荐）

直接在开始菜单中打开Microsoft Store，搜索Windows Terminal，安装即可

# 美化

虽然刚安装好的Terminal看起来还可以，不过还是不够美观，继续优化

## Powerline

Powerline 提供自定义的命令提示符体验，提供 Git 状态颜色编码和提示符。

### 安装字体

安装 Powerline 字体，[Github下载](https://github.com/microsoft/cascadia-code/releases)

### 安装Posh-Git 和 Oh-My-Posh

打开Powershell，执行下列命令，执行途中全部选Yes

Install-Module posh-git -Scope CurrentUser
Install-Module oh-my-posh -Scope CurrentUser

### 自定义 PowerShell 提示符

找到Powershell的配置文件，当前用户的配置文件位置在`C:\Users\用户名\Documents\WindowsPowerShell\`,一般叫`profile.ps1`或者`Microsoft.PowerShell_profile.ps1`，打开配置文件，将以下内容添加到末尾：

Import-Module posh-git
Import-Module oh-my-posh
Set-Theme Paradox

### 设置字体

打开Windows Terminal的配置文件，找到Powershell的配置，即`name`为`Windows PowerShell`的项，添加`"fontFace": "Cascadia Code PL"`，这样就大功告成了。

## 配色方案以及背景

这里就直接放上配置文件了

{
    "$schema": "https://aka.ms/terminal-profiles-schema",
    "defaultProfile": "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",
    
    // 全局配置
    // 关闭多个选项卡时是否需要确认
    "confirmCloseAllTabs": false,
    // 主题
    "theme":"dark",
    // If enabled, selections are automatically copied to your clipboard.
    "copyOnSelect": false,
    // If enabled, formatted data is also copied to your clipboard
    "copyFormatting": false,
    // 配置文件
    "profiles":
    {
        "defaults":
        {
            // Put settings here that you want to apply to all profiles.
            // 配色方案
            "colorScheme": "Gruvbox Dark",
            // 指针样式
            "cursorShape": "vintage"
        },
        "list":
        [
            {
                // Powershell配置
                "guid": "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",
                "name": "Windows PowerShell",
                "commandline": "powershell.exe",
                // 启动时位置
                "startingDirectory": "D:/",
                // 字体
                "fontFace": "Cascadia Code PL",
                // 启用 acrylic 背景，这里自行调整
                "useAcrylic": true,
                // 设置 Acrylic 不透明度
                "acrylicOpacity": 0.5,
                // 背景图片
                "backgroundImage": "背景图片位置",
                // 背景图片不透明度
                "backgroundImageOpacity":0.6,
                "hidden": false
            },
            {
                // cmd配置
                "guid": "{0caa0dad-35be-5f56-a8ff-afceeeaa6101}",
                "name": "命令提示符",
                "commandline": "cmd.exe",
                "hidden": false
            },
            {
                "guid": "{b453ae62-4e3d-5e58-b989-0a998ec441b8}",
                "hidden": false,
                "name": "Azure Cloud Shell",
                "source": "Windows.Terminal.Azure"
            }
        ]
    },
    // 配色方案
    "schemes": [
        {
            "name": "Gruvbox Dark",
            "black": "#1e1e1e",
            "red": "#be0f17",
            "green": "#868715",
            "yellow": "#cc881a",
            "blue": "#377375",
            "purple": "#a04b73",
            "cyan": "#578e57",
            "white": "#978771",
            "brightBlack": "#7f7061",
            "brightRed": "#f73028",
            "brightGreen": "#aab01e",
            "brightYellow": "#f7b125",
            "brightBlue": "#719586",
            "brightPurple": "#c77089",
            "brightCyan": "#7db669",
            "brightWhite": "#e6d4a3",
            "background": "#1e1e1e",
            "foreground": "#e6d4a3"
        }
    ],
    // Add custom keybindings to this array.
    // To unbind a key combination from your defaults.json, set the command to "unbound".
    // To learn more about keybindings, visit https://aka.ms/terminal-keybindings
    "keybindings":
    [
        // Copy and paste are bound to Ctrl+Shift+C and Ctrl+Shift+V in your defaults.json.
        // These two lines additionally bind them to Ctrl+C and Ctrl+V.
        // To learn more about selection, visit https://aka.ms/terminal-selection
        { "command": {"action": "copy", "singleLine": false }, "keys": "ctrl+c" },
        { "command": "paste", "keys": "ctrl+v" },
        // Press Ctrl+Shift+F to open the search box
        { "command": "find", "keys": "ctrl+shift+f" },
        // Press Alt+Shift+D to open a new pane.
        // - "split": "auto" makes this pane open in the direction that provides the most surface area.
        // - "splitMode": "duplicate" makes the new pane use the focused pane's profile.
        // To learn more about panes, visit https://aka.ms/terminal-panes
        { "command": { "action": "splitPane", "split": "auto", "splitMode": "duplicate" }, "keys": "alt+shift+d" }
    ]
}

# 最终效果

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602153212576-1418edf9-4933-4a4f-a6e6-3e1772744a97.png)

舒服了