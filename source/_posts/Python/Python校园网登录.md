---
title: Python校园网登录
categories:
  - Python
tags:
  - Python
  - 爬虫
abbrlink: e58f6b32
date: 2020-10-09 21:48:46
updated: 2020-10-09 21:48:46
---
#Python #爬虫
# 前言

  

做个开机自动登录WiFi脚本

  

# 配置环境

  

-   Python 3.7
-   Pycharm
-   Fiddler Everywhere

<!-- more -->

# 抓包分析

  

**登陆界面**

  

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602160396411-a02cc83c-51c5-454c-b675-326a3a0b24bf.png)

  

**请求地址**

  

http://211.69.15.33:9999/portalAuthAction.do

  

**请求数据**

  

![](https://cdn.nlark.com/yuque/0/2020/png/2658344/1602160396415-a3c20c20-e56b-4e5b-ad92-d14bbbe31acc.png)

  

其中关键的只有wlanuserip，wlanacIp，userid，useridtemp，passwd。

  

wlanuserip，wlanacIp可以通过ipconfig获取，也可以访问校园网界面，获取参数，这里选择后者。

  

# Python代码

  

import requests
from bs4 import BeautifulSoup
from win10toast import ToastNotifier
import webbrowser


class HAITWIFI:

    def __init__(self):
        self.username = '用户名'
        self.password = '密码'
        # 移动@gxyyd  联通@gxylt  电信@gxydx
        self.operator = {'1': '@gxyyd', '2': '@gxylt', '3': '@gxydx'}.get('1', '@gxyyd')

    def getIP(self):
        # 获取动态参数
        url = '校园网url'
        r = requests.get(url)
        soup = BeautifulSoup(r.content, 'html5lib')
        wlanuserip = soup.find('input', attrs={'name': 'wlanuserip'}).attrs['value']
        wlanacip = soup.find('input', attrs={'name': 'wlanacIp'}).attrs['value']
        return wlanuserip, wlanacip

    def login(self):
        # 登录模块
        ip = self.getIP()
        login_url = "登录url"
        headers = {
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) '
                          'Chrome/85.0.4183.121 Safari/537.36',
            'Referer': '校园网url',
        }
        post_data = {'wlanuserip': ip[0],
                     'wlanacname': 'HAIT-SR8808',
                     'chal_id': '',
                     'chal_vector': '',
                     'auth_type': 'PAP',
                     'seq_id': '',
                     'req_id': '',
                     'wlanacIp': ip[1],
                     'ssid': '',
                     'vlan': '',
                     'mac': '',
                     'message': '',
                     'bank_acct': '',
                     'isCookies': '',
                     'version': '0',
                     'authkey': '88----89',
                     'url': '',
                     'usertime': '0',
                     'listpasscode': '0',
                     'listgetpass': '0',
                     'getpasstype': '0',
                     'randstr': '7150',
                     'domain': '',
                     'isRadiusProxy': 'true',
                     'usertype': '0',
                     'isHaveNotice': '0',
                     'times': 12,
                     'weizhi': 0,
                     'smsid': '',
                     'freeuser': '',
                     'freepasswd': '',
                     'listwxauth': '0',
                     'templatetype': '1',
                     'tname': 'gxy_pc_portal',
                     'logintype': '0',
                     'act': '',
                     'is189': 'false',
                     'terminalType': '',
                     'checkterminal': 'true',
                     'portalpageid': '23',
                     'listfreeauth': '0',
                     'viewlogin': '1',
                     'userid': self.username + self.operator,
                     'authGroupId': '',
                     'smsoperatorsflat': '',
                     'useridtemp': self.username + self.operator,
                     'passwd': self.password,
                     'operator': self.operator}
        r = requests.post(url=login_url, data=post_data, headers=headers)
        bs = BeautifulSoup(r.content, 'html5lib')
        toaster = ToastNotifier()
        if bs.find_all('input', id='loginOut') != []:
            toaster.show_toast('WIFI登录', '登录成功', icon_path='D:\\Program Files\\Python37\\DLLs\\py.ico', duration=2, threaded=True)
        else:
            toaster.show_toast('WIFI登录', '登录失败', icon_path='D:\\Program Files\\Python37\\DLLs\\py.ico', duration=2, threaded=True)
            webbrowser.open(
                'http://url/portalReceiveAction.do?wlanuserip=' + ip[0] + '&wlanacname=HAIT-SR8808')


HAITlogin = HAITWIFI()
HAITlogin.login()

  

# 添加Win10弹窗提醒

  

这里我使用的是win10toast模块，创建对象后，使用`show_toast()`即可成功弹窗。

  

# 开机自启

  

-   方案一 加入Windows 服务  
    首先使用Pyinstaller将程序编译为exe，然后使用[WinSW](https://github.com/winsw/winsw/releases)，将程序添加为Windows 服务。  
    优点：B格很高  
    缺点：有BUG，可能是win10toast的问题，也可能是python打包文件的问题，虽然能完成登录，但无法弹窗。
-   方案二 添加到`启动`文件夹  
    这种很简单，写一个bat，运行脚本，最后把bat放到`AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup`文件夹中即可。  
    优点：简单，实用  
    缺点：B格不够，运行太慢，会弹黑框框