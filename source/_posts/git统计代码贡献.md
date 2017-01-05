---
title: git统计代码贡献
comments: true
date: 2017-01-05 09:09:02
tags: git
---


> 辛苦了一年，到年终总结的时候啦。想统计一下今年贡献了多少代码。那么怎么用git来统计某个作者贡献代码行数呢？

思路是采用git log打出来每次提交的信息。可以加上`--author`参数来指明是哪个作者的。
```
$ git log   --pretty=format:'%an %Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr)%Creset' --abbrev-commit --date=relative --shortstat --author="Shen" 
```
打出来的结果如下所示：

<center> ![image_1b5jmt36dc6f1tj412mi173i1kcr9.png-116.2kB][1] </center>

能够看到每次提交的代码变更行数，接下来祭出grep和awk神器来做下统计吧！
```
$ git log   --pretty=format:'%Cred%h%Creset %an-%C(yellow)%d%Creset %s %Cgreen(%cr)%Creset' --abbrev-commit --date=relative --shortstat --author="Shen" |grep "insertions" |awk '{sum += $4 + $6} END {print sum}'
7185
```
在这个分支上贡献了7185行代码，再接再厉！

[1]: http://static.zybuluo.com/shenyuflying/x2n5fklgimkamh9mulxlcj61/image_1b5jmt36dc6f1tj412mi173i1kcr9.png



如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
