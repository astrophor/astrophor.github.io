---
layout: post
title: blog like a hacker
categories: [technology, summary]
description: 想开个博客了，越来越觉得纯靠记忆是不行的，还是写点东西记录一下自己的生活比较好。网上免费的博客平台很多，无意中看到这篇文章后，决定自己也折腾一下，在github上搭建一个博客，顺便了解一下git的用法，这里简单记录下
---

想开个博客了，越来越觉得纯靠记忆是不行的，还是写点东西记录一下自己的生活比较好。

网上免费的博客平台很多，无意中看到[这篇文章][1]后，决定自己也折腾一下，在github上搭建一个博客，顺便了解一下git的用法，这里简单记录下。

github提供pages功能，本意是给项目提供web页面功能，但是geek却发现这个功能还能用来写博客，甚至写书，估计github设计人员都没想到吧。

具体的介绍可以见[这里][2]，github可以提供2种形式的页面：

1. 个人用户、企业用户的页面
2. 项目页面

这2种页面区别是，对于前者，一个账号只能有一个，而且域名固定，比如我的账号名是astrophor，那么我的主页地址就是`astrophor.github.com`，但是后者就没有个数限制，每个项目都可以有一个，名字形式就是`astrophor.github.com/projectname`。

同时，两者在github上的开发也不太一样：对于前者，需要在github上创建一个名字为`astrophor.github.com`的项目，不能是其他名字，页面代码需要在主干上开发。这样，当博客发布时，github就去`astrophor.github.com`项目的主干下面，寻找`index.html`文件。而对于后者，需要在页面文档需要在一个名为`ph-pages`的分支下面开发，同时该分支必须是孤儿节点，其他的同前者一样。这样规定我理解主要是便于github寻找页面地址。

项目创建好以后就可以部署blog代码了，github是通过jekyll来生成页面的，因此你需要了解如何使用jekyll的使用方式，具体参见[这里][3]。具体使用方法上面的连接讲的很详细了，照着做就好，网站上面还有很多例子，是个很好的参考。接下来的工作就是网页设计内容了，可以参考bootstrap，做页面很方便。至于页面模板，可以参考jekyll主页中提供的一些样例，或者自己设计就好。

最后，博客文件格式支持markdown和textile，学习成本都不高。

当然，完整自己做一个还是挺麻烦的，还好现在已经有一些这种风格的博客平台：

* [简书][4]
* [farbox][5]

它们使用起来都很容易，只需要你懂markdown语法就好。

最后说一点不相关的，感觉现在已经不时兴写博客了，都更愿意发微博，或者改签名，也许跟浮躁的氛围有关。不过自己最近是越来越觉得还是得写博客，一方面可以记录自己的生活和想法，另一方面，写也是理清自己思路的一个方式，通过写博客我才发现原来知道一个事情，和自己能够清晰的将其表达出来，这两者完全不是一个概念。

所以，还是适时放慢一下前行的脚步，慢慢回味一下曾经走过的路，脑海中飘过的思绪，沉淀下来，自己才能积累、提高。



[1]:http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html
[2]:https://help.github.com/articles/user-organization-and-project-pages
[3]:http://jekyllrb.com/
[4]:http://jianshu.io/
[5]:http://www.farbox.com/