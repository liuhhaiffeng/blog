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
cd codeRunner
make
make install
```

默认是安装在了`/usr/local/bin/codeRunner`这里。

一个小例子：
<center>![](/uploads/codeRunner-eg1.gif)</center>

另外，codeRunner还支持循环等等各种C语言的数据类型、流程控制等等。

<center>![](/uploads/codeRunner-eg2.gif)</center>

下面一个例子输出乘法口诀表：

<center>![](/uploads/codeRunner-eg3.gif)</center>

如果有错误，会马上提示，纠正错误之后，可以再次运行：

<center>![](/uploads/codeRunner-eg4.gif)</center>

还有更复杂的用法，比如通过`程序模板`来支持`函数`等等。详细的用法可以看README哦。

