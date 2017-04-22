---
title: 修改hosts来访问谷歌
comments: true
date: 2017-01-18 07:41:42
categories: 其他
---


> 想用google查东西，可以通过修改hosts方式,非常简单。

用root用户执行

```
git clone https://github.com/racaljk/hosts
cp /etc/hosts /etc/hosts.bak
cat hosts/hosts > /etc/hosts
rm -rf hosts
```

[这个网址](https://github.com/racaljk/hosts)上面有人在维护最新的hosts。如果hosts过期，可以通过这个脚本定时执行，来确保总是最新的。

<center>![image_1b6o85n6c14021lv0s2bh0b190c9.png-212.2kB][1]</center>

这样就可以来搜索需要的东西。比如我的博客就可以通过google搜到哦。

[1]: http://static.zybuluo.com/shenyuflying/5jgl6gv0e53fdfne1o0ofyjh/image_1b6o85n6c14021lv0s2bh0b190c9.png



如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
