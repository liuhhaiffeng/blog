---
title: Here Document
date: 2016-10-02 11:49:52
tags: ["linux","shell"]
---


我们经常需要在shell中新建文件并输入内容，通常的做法有2种。
第一种： 用vim打开一个文件，输入内容，保存并退出。
第二种： 用echo输出重定向：echo "hello world" > a.out。
但是这两种方法都有一定弊端，比如第一种无法在shell脚本里面实现往文件中输入内容。第二种，无法输入多行内容。
如果我们需要在shell脚本中对文件中输入多行内容，这时候我们Here Document就派上用场了。

## 什么是Here Document?
Here Document 是在Linux Shell 中的一种特殊的重定向方式，它的基本的形式如下
```shell
cmd << delimiter
  Here Document Content
delimiter
```
它的作用就是将两个 delimiter 之间的内容(Here Document Content 部分) 传递给cmd 作为输入参数。

## 在shell脚本中对文件输入多行内容
基本方法是用here document把输入重定向给cat然后输出重定向到a.out文件。
```shell
yshen@yshen-ThinkPad-X201 ~ $ cat <<EOF > a.out
> this is line1
> this is line2
> this is line3
> ...
> this is lineN
> EOF
yshen@yshen-ThinkPad-X201 ~ $ cat a.out 
this is line1
this is line2
this is line3
...
this is lineN

```

这样，我们就使用Here Document在shell脚本中完成了多行文本的输入。
