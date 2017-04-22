---
title: wget下载网站供离线浏览
date: 2016-10-02 16:44:45
tags: [工具, wget]
categories: 技术分享
---


> 我们经常使用wget命令来下载文件，那么能不能使用wget来下载整个网站呢？

看wget的man说明，可以发现wget有几个特点：
1. 非交互式，可以工作在后台。
2. 可以追踪html中的链接，在下载过程中把连接自动转换为本地路径，从而创建本地网站镜像。
3. 健壮性高，在失败之后会不断尝试，支持断线重连。

发现wget果然强大，有网站下载这个功能，而且还可以控制网站下载的方式，下面就是wget进行网站下载的命令：

```
$ wget \
     --recursive \
     --no-clobber \
     --page-requisites \
     --html-extension \
     --convert-links \
     --restrict-file-names=windows \
     --domains shenyu.wiki \
     --no-parent \
         shenyu.wiki
```

上面的命令会下载 http://www.shenyu.wiki 下面所有的页面。

上面用到的wget网站下载选择解释：

    --recursive: 下载整个网站
    --domains shenyu.wiki: 不要下载指定域名之外的网页。
    --no-parent: 仅仅下载当前目录结构下的文件。
    --page-requisites: 现在网页包括的所有内容(images, CSS and so on).
    --html-extension: 将网页保存为html文件。
    --convert-links: 将连接转换为本地连接
    --restrict-file-names=unix: 文件名保存为unix格式,如果是要保存windows格式的，这里写成windows
    --no-clobber: 不要覆盖已有文件，在下载中断后继续下载。

下载完成之后，生成了`shenyu.wiki`文件夹，进去可以看到该网站的内容都下载好了。可以打开index.html文件来离线浏览网站。

```
$ ll
total 220
drwxr-xr-x 6 yshen yshen   4096 10月  2 16:41 ./
drwxr-xr-x 3 yshen yshen   4096 10月  2 16:41 ../
drwxr-xr-x 4 yshen yshen   4096 10月  2 16:41 2016/
-rw-r--r-- 1 yshen yshen  21632 10月  2 13:08 about.html
-rw-r--r-- 1 yshen yshen  27364 10月  2 13:08 archives.html
drwxr-xr-x 2 yshen yshen   4096 10月  2 16:41 css/
-rw-r--r-- 1 yshen yshen 101642 10月  2 13:08 index.html
-rw-r--r-- 1 yshen yshen  19836 10月  2 13:08 message.html
-rw-r--r-- 1 yshen yshen  20598 10月  2 13:08 tags.html
drwxr-xr-x 2 yshen yshen   4096 10月  2 16:41 uploads/
drwxr-xr-x 4 yshen yshen   4096 10月  2 16:41 vendors/
```




