---
title: linux把文件放置到剪贴板
comments: true
date: 2016-11-20 03:01:08
tags: ["操作系统",Linux,Linux使用]
categories: 技术分享
---

> 如果想要把一个文件中的内容放置到剪贴板，通常的做法是用vim打开文件，然后复制粘贴。有时候文章很长，那么需要多次操作才可以。能不能用一个命令把文件中的内容放置到剪贴板呢？


## install
```
sudo apt-get install xsel
```

## how to
```
cat xxx.txt | xsel -b -i
```



如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
