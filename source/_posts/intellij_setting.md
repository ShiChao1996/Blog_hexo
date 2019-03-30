---
title: Intellij 自定义注释
cover: /images/intellij/intellij.jpeg
author: 
  nick: Lovae
tags: 
    - Intellij 
date: 2018-4-15

# 首页每篇文章的子标题
subtitle: 在 intellij 产品中高亮显示自定义注释
categories: other

---
> 有时候在读好的代码的时候想要自己加上一些注释，并且为了和原来的注释区分开，想让它高亮显示，应该怎么设置呢？
#### steps
* 打开 preference --> editor --> TODO 
* 在 Patterns 里添加一行, 填写 \b**name**\b.*, 其中 name 就是你的自定义注释，选择颜色，然后就 OK 啦！
* 使用的时候按照 todo 那种注释写，不过把 todo 换成你自定义的就好啦

如图：
![](/images/intellij/example.jpg)