---
title: linux查找文件内容
comments: true
date: 2016-10-04 18:55:36
tags: linux shell
---

> 有时候我们需要在一堆文件中查找包含`某单词`的文件。比如在一堆源码文件中找出来调用一个函数的地方。

可以用find，xargs，grep三个命令联合起来实现。
```bash
find ./ -type f -name *.c | xargs grep "word"
```
find找出当前目录下所有.c的普通文件
xargs将文件名变成参数传入到grep
grep打开该文件并找出文件中包含word的行

## 一个脚本
为了方便使用，避免每次都输入很长的命令，写出了下面一个脚本。
```bash
#!/bin/bash                                                                 
#递归查找当前目录文件中的内容
#参数1： 文件后缀，可以省略
#参数2： 要查找的字符串

F_ARG_SUFFIX=""
F_ARG_PATTEN=""

if [ $# == 0 ] ; then
    echo "usage: f [file suffix] pattern"
    exit 0
fi

if [ $# == 1 ] ; then
    F_ARG_PATTEN="$1"
fi


if [ $# == 2 ] ; then
    F_ARG_SUFFIX="-name *.$1"
    F_ARG_PATTEN="$2"
fi

find ./ -type f $F_ARG_NAME | xargs grep $F_ARG_PATTEN
```
保存在/bin目录下，并加上可执行权限

## 例子
比如找出postgreSQL源码中, c语言文件使用到heap_open函数的地方：
```bash
$ f c heap_open
./backend/commands/tablecmds.c:         rel = heap_openrv(rv, AccessExclusiveLock);
./backend/commands/tablecmds.c:                         rel = heap_open(childrelid, NoLock);
...

$ f c heap_open | wc -l
587

```

一共使用了587次，很多啊！


