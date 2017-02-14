---
title: 如何阅读程序源码之——call graph
comments: true
date: 2017-02-14 01:58:20
tags: [工具, valgrind, kcachegrind, gprof, doxygen, cflow, egypt, cctree, codeviz]
categories: 技术分享
---

> 多阅读大型开源项目能够显著提升自己的能力，但是面对几万甚至几十万行的代码是不是感觉无从下手。理解大型工程的一个诀窍就是能够理清函数之间的调用关系。这篇文章介绍几个好用的工具，用来帮助我们生成代码的调用关系，以便更好的理解项目代码。

说到函数调用关系（call graph）相信大家都不陌生，函数调用关系其实就是程序的执行流程。生成函数调用关系的方法一般来说有2个，即动态方法和静态方法。



动态方法
-------------------------
动态方法就是在程序执行过程中，把函数的调用情况记录下来，一些程序性能分析工具常常都有这个功能，比如`gprof`。相对于静态方法来说，动态方法准确的反映了程序执行的过程。但是一些没有实际执行到的分支没有显示出来。
常用的工具有：
1. [gprof](https://sourceware.org/binutils/docs/gprof/)：可以输出每个函数的调用次数，每个函数消耗的处理器时间，函数之间的调用关系。通过在编译和链接程序的时候（使用 -pg 编译和链接选项），gcc 在你应用程序的每个函数中都加入了一个名为mcount函数，也就是说你的应用程序里的每一个函数都会调用mcount，而mcount会在内存中保存一张函数调用图，并通过函数调用堆栈的形式查找子函数和父函数的地址。这张调用图也保存了所有与函数相关的调用时间，调用次数等等的所有信息。在程序结束后会生成gmon.out文件，然后再用gprof来分析gmon.out文件。但是gprof不支持多线程应用，多线程下只能采集主线程性能数据。原因是gprof采用ITIMER_PROF信号，在多线程内只有主线程才能响应该信号。如果需要支持多线程程序那么需要程序员自己处理线程信号。另外gprof只能输出文本形式的调用关系。
2. [KCachegrind](http://kcachegrind.sourceforge.net/html/Home.html)：利用calgrind生成的结果，绘制调用关系图形。使用也非常简单,首先用valgrind来执行程序`valgrind --tools=callgrind ./prog`，程序结束后在目录下生成`callgrind.out.pid`文件，然后用kcachegrind打开这个文件即可。

静态方法
----------------------
静态方法就是直接分析程序源码文件，把函数调用关系记录下来。静态方法能够更为全面的反映函数的调用情况（不管实际是否能调用）。相对于动态方法来说，静态方法不用编译，速度较快节约了大量时间。但是，静态方法的函数调用关系不一定能够反映程序的实际执行情况。
常用的工具有：
1. [doxygen](http://www.doxygen.nl/manual/index.html)：可由代码以生成html等各种格式的文档，需要编码时按照一定的规范，十分适合大型工程，可以看[postgresql](https://doxygen.postgresql.org/)的在线文档。但是已有工程由于不符合doxygen的规范，不适合使用。
2. [cflow](http://www.gnu.org/software/cflow/)：直接解析C源代码，用lex&yacc来生成文本格式的调用关系图。不需要任何改动。但是只能生成文本格式，不能生成图像。
3. [egypt](http://www.gson.org/egypt/egypt.html)：用gcc的-fdump-rtl-expand选项生成调用关系，然后用graphviz生成调用图图像。需要简单改动一下Makefile,或者是make CFLAGS="-fdump-rtl-expand"。
4. [CCTree](http://www.vim.org/scripts/script.php?script_id=2368)：这个是vim的一个插件，利用cscope的输出来生成交叉引用的信息。和SourceInsight的交叉引用很像。
5. [codeviz](http://freecode.com/projects/codeviz/) :其基本原理是给 GCC 打个补丁，让它在编译时每个源文件时 dump 出其中函数的 call graph，然后用 Perl 脚本收集并整理调用关系，转交给Graphviz绘制图形。这种方法还需要重新编译gcc，比较麻烦。

动态方法和静态方法各有好处，在实际使用过程中需要两者相结合才会发挥更大力量。下面是他们生成的调用关系效果图：
 

1. gprof
```
$ gprof -b ./node2dot ./gmon.out 
Flat profile:

Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name    
100.22      0.03     0.03        8     3.76     3.76  add_node
  0.00      0.03     0.00       54     0.00     0.00  get_one_name
  0.00      0.03     0.00       32     0.00     0.00  add_item
  0.00      0.03     0.00       16     0.00     0.00  pop
  0.00      0.03     0.00       16     0.00     0.00  push
  0.00      0.03     0.00        8     0.00     0.00  add_link
  0.00      0.03     0.00        1     0.00     0.00  print_body
  0.00      0.03     0.00        1     0.00     0.00  print_footer
  0.00      0.03     0.00        1     0.00     0.00  print_header


			Call graph


granularity: each sample hit covers 2 byte(s) for 33.26% of 0.03 seconds

index % time    self  children    called     name
                0.03    0.00       8/8           main [2]
[1]    100.0    0.03    0.00       8         add_node [1]
-----------------------------------------------
                                                 <spontaneous>
[2]    100.0    0.00    0.03                 main [2]
                0.03    0.00       8/8           add_node [1]
                0.00    0.00      54/54          get_one_name [3]
                0.00    0.00      32/32          add_item [4]
                0.00    0.00      16/16          push [6]
                0.00    0.00      16/16          pop [5]
                0.00    0.00       8/8           add_link [7]
                0.00    0.00       1/1           print_header [10]
                0.00    0.00       1/1           print_footer [9]
                0.00    0.00       1/1           print_body [8]
-----------------------------------------------
                0.00    0.00      54/54          main [2]
[3]      0.0    0.00    0.00      54         get_one_name [3]
-----------------------------------------------
                0.00    0.00      32/32          main [2]
[4]      0.0    0.00    0.00      32         add_item [4]
-----------------------------------------------
                0.00    0.00      16/16          main [2]
[5]      0.0    0.00    0.00      16         pop [5]
-----------------------------------------------
                0.00    0.00      16/16          main [2]
[6]      0.0    0.00    0.00      16         push [6]
-----------------------------------------------
                0.00    0.00       8/8           main [2]
[7]      0.0    0.00    0.00       8         add_link [7]
-----------------------------------------------
                0.00    0.00       1/1           main [2]
[8]      0.0    0.00    0.00       1         print_body [8]
-----------------------------------------------
                0.00    0.00       1/1           main [2]
[9]      0.0    0.00    0.00       1         print_footer [9]
-----------------------------------------------
                0.00    0.00       1/1           main [2]
[10]     0.0    0.00    0.00       1         print_header [10]
-----------------------------------------------


Index by function name

   [4] add_item                [3] get_one_name            [9] print_footer
   [7] add_link                [5] pop                    [10] print_header
   [1] add_node                [8] print_body              [6] push

```
2. kcachegrind
![image_1b8t50qjn1p7f9i0bjs1qud1cqc9.png-253.8kB][2]
3. egypt 
![image_1b8renepf1ckbd1d1qjk1ikv4a0m.png-42.6kB][3]
4. CCTree
![image_1b8rei332c4v1ppefci182muf49.png-128.3kB][4]

PS:以上分析的源码来自我的两个开源项目：
pgNodeGraph: http://github.com/shenyuflying/pgNodeGraph
codeRunner: http://github.com/shenyuflying/codeRunner

  [1]: http://static.zybuluo.com/shenyuflying/4n143s2us4sxph95uhh5tbij/image_1b8renepf1ckbd1d1qjk1ikv4a0m.png
  [2]: http://static.zybuluo.com/shenyuflying/bz18v8q1z4vif7o5h9lpepra/image_1b8t50qjn1p7f9i0bjs1qud1cqc9.png
  [3]: http://static.zybuluo.com/shenyuflying/4n143s2us4sxph95uhh5tbij/image_1b8renepf1ckbd1d1qjk1ikv4a0m.png
  [4]: http://static.zybuluo.com/shenyuflying/rt5utk4711iyfaoo77c2anem/image_1b8rei332c4v1ppefci182muf49.png




如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
