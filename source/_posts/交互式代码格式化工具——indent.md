---
title: 交互式代码格式化工具——indent
comments: true
date: 2016-10-21 09:20:49
tags: [编程语言,"Shell"]
categories: [技术分享,开源社区]
---


> 有时候在编码过程中很多细节不注意，造成了代码不符合规范。如果人工来检查，比较繁琐，还容易遗漏，不如交给工具做。在之前一片文章[《利用sed生成规范的c代码 》](http://shenyu.wiki/2016/10/20/%E5%88%A9%E7%94%A8sed%E7%94%9F%E6%88%90%E8%A7%84%E8%8C%83%E7%9A%84c%E4%BB%A3%E7%A0%81/)基础上集合shell+vimdiff实现了交互式代码检查。

## 先睹为快

一份不规范的代码test.c运行工具检查一下：
```
./indent test.c
```
效果如下图所示：
![](http://static.zybuluo.com/shenyuflying/qv3omp4cp2ycm123vqscxob5/image_1avj8a4sm1fvr2s1e9dgsjrgs9.png)

左面是例子中的那份不规范的代码，右面是提示的规范代码。不规范的代码用红色标出。如果接受修改，移动到红色处用do来接受修改。]c移动到下一处。

修改完了之后wqa退出。

## 实现
这个小工具是用shell+sed+vimdiff写的。可以看到linux的强大之处就是可以融合多个工具来实现一个更强大的工具。
源码放在了github上：https://github.com/shenyuflying/indent

