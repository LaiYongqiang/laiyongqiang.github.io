---
layout: post
title: Github 博客创建备忘录
description: 记录通过Github基于BeiYuu的工程创建本博客的主要步骤与需要注意的地方
category: blog
---

这个博客是根据[BeiYuu][]的博客建立起来的，我是根据BeiYuu的一遍博文[使用Github Pages建独立博客][]，clone了他在[beiyuu.github.com][]上的[工程][]来一步一步搭建起来的，在这里记录一下博客的建立的一些步骤： 

## 1.创建[Github][]账号
在创建用户名的时候要注意一下，你的用户名将会成为你的博客的域名的一部分，比如我的Github账号是`LaiYongqiang`，我的博客域名就成了`http://laiyongqiang.github.io`，具体可见[Github Page的基本建立教程][].

## 2.创建代码仓
在这一步，我们可以还可以参考这位雨知大大的博文[通过GitHub Pages建立个人站点(详细步骤)],在这个阶段主要步骤以及是：
- clone [beiyuu.github.com][]工程
- 在代码仓的设置里面![Github_Settings](http://ylai.qiniudn.com/bloggithub_settings.png?imageView/2/w/640/h/960)将代码仓的名字改为为：`username`.github.io 或者`username`.github.com 
- 在项目的settings里面查看是否已经显示`Your site is published at http://username.github.io.`,设置后到生效需要一段时间
  ![Published](http://ylai.qiniudn.com/blogpublished.png?? imageView2/1/q/85|watermark/2/text/bGFpeW9uZ3FpYW5nLmdpdGh1Yi5pbw==/font/5a6L5L2T/fontsize/500/fill/I0VGRUZFRg==/dissolve/100/gravity/SouthEast/dx/10/dy/10) 
- 打开浏览器，通过http://`username`.github.`io`或者http://`username`.github.`com`访问你的博客
 
## 3. 修改代码
在这一步，主要的工作有：
> 1. 替换BeiYuu的相关内容
> 2. 更新左上角`+`号处的联系方式
> 3. 删除或者更新右上角的新浪关注和评论处的分享到微博
> 4. 删除已有的博文
> 5. 删除统计代码
> 6. 创建自己的评论[Disqus][]，注意要在设置中把语言改成中文
> 7. 更新README.md  

在你完成这些工作之后，你的博客已经基本上建立完成了，剩下的就是开始写博文，充实你的博客吧


[Github]: https://github.com/ "Github"
[BeiYuu]:    http://beiyuu.com  "BeiYuu"
[使用Github Pages建独立博客]: http://beiyuu.com/github-pages/
[beiyuu.github.com]:https://github.com/beiyuu/beiyuu.github.com
[工程]:https://github.com/beiyuu/beiyuu.github.com
[Github Page的基本建立教程]: http://pages.github.com/ "Github Pages"
[通过GitHub Pages建立个人站点(详细步骤)]: http://www.cnblogs.com/purediy/archive/2013/03/07/2948892.html
[Disqus]: http://disqus.com/ "Disqus"
