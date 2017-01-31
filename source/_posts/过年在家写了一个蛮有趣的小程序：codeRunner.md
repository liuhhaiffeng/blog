---
title: 过年在家写了一个蛮有趣的小程序：codeRunner
comments: true
date: 2017-01-31 12:20:30
tags: [编程,c]
categories: 技术分享
---

> 在工作中经常需要临时试验一小段代码，比如想要知道sizeof(int)的大小是多少？对于C语言这样的编译型语言来说，需要进行`编辑`,`编译`,`运行`等等系列操作，如果出现错误就需要重新来过，非常的耗费时间。针对这个痛点，借鉴一些解释型语言能够逐行运行的特点，输入一行就能马上看到输出，写了`codeRunner`这个程序，用C语言模仿解释型语言逐行运行的这种特性，方便临时开发调试。

代码已经开源在了[github](https://github.com/shenyuflying/codeRunner)上，有兴趣的同学可以试用一下。

```
git clone https://github.com/shenyuflying/codeRunner
make
make install
```

默认是安装在了`/usr/local/bin/codeRunner`这里。

![](/uploads/codeRunner-eg1.gif)



如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
