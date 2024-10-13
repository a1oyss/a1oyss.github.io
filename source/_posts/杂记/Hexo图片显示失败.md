---
title: Hexo图片显示失败
tags:
  - 随笔记录
  - Hexo
categories:
  - 杂记
abbrlink: 8759
date: 2021-03-28 00:09:09
updated: 2021-03-28 00:09:09
---
#Hexo
# Part 1

  

今天提交文章的时候，上去博客看了一眼，发现图片全都显示不出来，

  

看了下图片url，全都是

  

file://image/123.jpg

  

这种形式的。

  

因为我使用的typora来写markdown，自动插入图片默认的是本地路径，去设置里更改为使用相对路径即可

  

# part 2

  

原本以为大功告成，结果发现还是显示错误，去网上查了下，有些插件没有装，_config.yml里的选项也没有开。。。

  

安装`hexo-asset-image`

  

npm install https://github.com/CodeFalling/hexo-asset-image --save

  

_config.yml配置

  

`post_asset_folder: false`改为`post_asset_folder: true`

  

# part 3

  

再次以为大功告成，结果发现还是错误，之前`hexo g -d` 的时候没有注意，这次执行的时候突然发现有几条记录有点奇怪

  

update link as:-->/.io//06/01/vim/1561905818946.png
update link as:-->/.io//06/01/vim/1561905818946.png

  

这个`.io`不知道是怎么来的，不管怎么修改图片的路径，这个`.io`总是有

  

最终查来查去发现是`hexo-asset-image`这个插件的问题，hexo 3.0以上与hexo 3.0以下获取url的方式不同，结果就导致获取到了`.io`这种奇怪的域名。

  

# 解决问题

  

参考[hexo引用本地图片无法显示](https://blog.csdn.net/xjm850552586/article/details/84101345)

  

将`/node_modules/hexo-asset-image/index.js`里面的内容修改为

  

'use strict';
var cheerio = require('cheerio');

// http://stackoverflow.com/questions/14480345/how-to-get-the-nth-occurrence-in-a-string
function getPosition(str, m, i) {
  return str.split(m, i).join(m).length;
}

var version = String(hexo.version).split('.');
hexo.extend.filter.register('after_post_render', function(data){
  var config = hexo.config;
  if(config.post_asset_folder){
    var link = data.permalink;
	if(version.length > 0 && Number(version[0]) == 3)
	   var beginPos = getPosition(link, '/', 1) + 1;
	else
	   var beginPos = getPosition(link, '/', 3) + 1;
	// In hexo 3.1.1, the permalink of "about" page is like ".../about/index.html".
	var endPos = link.lastIndexOf('/') + 1;
    link = link.substring(beginPos, endPos);

    var toprocess = ['excerpt', 'more', 'content'];
    for(var i = 0; i < toprocess.length; i++){
      var key = toprocess[i];
 
      var $ = cheerio.load(data[key], {
        ignoreWhitespace: false,
        xmlMode: false,
        lowerCaseTags: false,
        decodeEntities: false
      });

      $('img').each(function(){
		if ($(this).attr('src')){
			// For windows style path, we replace '\' to '/'.
			var src = $(this).attr('src').replace('\\', '/');
			if(!/http[s]*.*|\/\/.*/.test(src) &&
			   !/^\s*\//.test(src)) {
			  // For "about" page, the first part of "src" can't be removed.
			  // In addition, to support multi-level local directory.
			  var linkArray = link.split('/').filter(function(elem){
				return elem != '';
			  });
			  var srcArray = src.split('/').filter(function(elem){
				return elem != '' && elem != '.';
			  });
			  if(srcArray.length > 1)
				srcArray.shift();
			  src = srcArray.join('/');
			  $(this).attr('src', config.root + link + src);
			  console.info&&console.info("update link as:-->"+config.root + link + src);
			}
		}else{
			console.info&&console.info("no src attr, skipped...");
			console.info&&console.info($(this));
		}
      });
      data[key] = $.html();
    }
  }
});

  

接着再去`hexo g -d`一下就能显示成功了。

  

hexo 3.0以上用户应该也可以选择直接卸载`hexo-asset-image`插件，直接使用官方的`相对路径引用的标签插件`

  

[资源文件夹](https://hexo.io/zh-cn/docs/asset-folders.html)

通过常规的 markdown 语法和相对路径来引用图片和其它资源可能会导致它们在存档页或者主页上显示不正确。在Hexo 2时代，社区创建了很多插件来解决这个问题。但是，随着Hexo 3 的发布，许多新的标签插件被加入到了核心代码中。这使得你可以更简单地在文章中引用你的资源。

`{% asset_path slug %}`

`{% asset_img slug [title] %}`

`{% asset_link slug [title] %}`

比如说：当你打开文章资源文件夹功能后，你把一个 `example.jpg` 图片放在了你的资源文件夹中，如果通过使用相对路径的常规 markdown 语法 `![](/example.jpg)` ，它将 _不会_ 出现在首页上。（但是它会在文章中按你期待的方式工作）

正确的引用图片方式是使用下列的标签插件而不是 markdown ：

`{% asset_img example.jpg This is an example image %}`

通过这种方式，图片将会同时出现在文章和主页以及归档页中。