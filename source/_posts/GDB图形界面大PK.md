---
title: GDB图形界面大PK
comments: true
date: 2016-12-29 12:07:25
tags: [工具, gdb]
categories: 技术分享 
---

> gdb虽然很强大，但是却略显单调。其实gdb有很多前端图形界面。那么选哪个呢？请看GDB图形界面大PK。

TUI
--------

<center>![image_1b54hipghpqg176a1vd5133l1p3r13.png-27.6kB][1]</center>

gdb原生的图形模式，支持gdb所有的特性。

**使用方法：**
```
gdb attach pid 之后ctrl+x a
```

cgdb
--------
http://cgdb.github.io/

<center>![image_1b54jn7f11bincfncgpqjs1vbt34.png-26.1kB][2]</center>

可以认为是TUI模式的增强版，具有代码高亮、查找等等功能。同时也支持gdb所有的特性。
如果之前对gdb和vim比较熟悉的话，用这个就会很顺手。

**使用方法：**
```
cgdb attach pid
```


ddd
--------

<center>![image_1b54hghu7ibvrhn1g27ra1cq2m.png-39kB][3]</center>

采用gdb作为后端，强大之处在于能够图形化显示程序中的结构体。
**使用方法：**
```
ddd [options...] executable-file [core-file | process-id]
```

insight
--------

<center>![image_1b54hg2kn170q15ebk4q1pr1ji89.png-144.1kB][4]</center>

Nemiver
-----------
<center>![image_1b54iglhg1vq6cog15oqnd61in51g.png-137kB][5]</center>

界面比上面几个好看点，但是不知道如何输入gdb命令

**使用方法：**
```
nemiver --attach=<pid|process name> 
```


KDevelop
------------
https://www.kdevelop.org/

<center>![image_1b54mesls1d1u10okgksqoo19l23h.png-178.3kB][8]</center>

**使用方法：**
还没搞明白如何attach到进程。

总结
-----------
一般使用gdb的TUI模式就可以，这个每台机器上都有，也可以远程用终端调试。最简单的就是最强大的。如果觉得TUI模式太单调的话，可以用cgdb。

参考
----------

https://sourceware.org/gdb/wiki/GDB%20Front%20Ends

  [1]: http://static.zybuluo.com/shenyuflying/beiufda6on7jxe25kf5nqlxg/image_1b54hipghpqg176a1vd5133l1p3r13.png
  [2]: http://static.zybuluo.com/shenyuflying/xrtnvqcgnjc5u5s92fdxie4f/image_1b54jn7f11bincfncgpqjs1vbt34.png
  [3]: http://static.zybuluo.com/shenyuflying/6wybyzwxwgsrs94nipn6mjar/image_1b54hghu7ibvrhn1g27ra1cq2m.png
  [4]: http://static.zybuluo.com/shenyuflying/qe9ek0yb52h43pl4df9z2ei1/image_1b54hg2kn170q15ebk4q1pr1ji89.png
  [5]: http://static.zybuluo.com/shenyuflying/i67fi7vfxpy0st8m1pspgvdb/image_1b54iglhg1vq6cog15oqnd61in51g.png
  [6]: http://static.zybuluo.com/shenyuflying/v9aiyxjswa0kpvb86eu762ws/image_1b54jdr4j1fvjdkj1mfo1jg01j52n.png
  [7]: http://static.zybuluo.com/shenyuflying/osrg99thuvckljo6xz3zc1rx/image_1b54jb06t1ol415rt110fi4d1nnb2a.png
  [8]: http://static.zybuluo.com/shenyuflying/hr7xsop7me5qv8s65isaydj0/image_1b54mesls1d1u10okgksqoo19l23h.png
  [9]: http://static.zybuluo.com/shenyuflying/so0nlb0ay6wq3eg53uonzboo/image_1b54j5a02tiq1eom1jjg1jg5oqv1t.png


如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
