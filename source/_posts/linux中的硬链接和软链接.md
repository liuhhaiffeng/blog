---
title: linux中的硬链接和软链接
comments: true
date: 2016-10-10 06:03:07
tags: ["操作系统",Linux,Linux编程]
categories: [技术分享]
--


## 建立硬链接和软链接
在linux中我们通过ln命令来建立硬链接和软链接，默认是建立硬链接，加上-s参数那么建立的就是软链接。
```
$ echo "Hello world" > a
$ ln a a_hard_link
$ ln -s a a_soft_link
```
建立好之后，用ls命令看一下，记得加上-i这样我们就可以看到inode号了
```
$ls -ali
19796575 -rw-rw-r--  2 yshen yshen   12 10月 10 11:33 a
19796575 -rw-rw-r--  2 yshen yshen   12 10月 10 11:33 a_hard_link
19796574 lrwxrwxrwx  1 yshen yshen    1 10月 10 11:34 a_soft_link -> a

```
**inode**
科普一下：inode全称是index node即索引节点。可见inode中并没有存放文件的实际内容，而是存放了索引。这个索引指向的是磁盘中存放文件内容的物理位置。
**目录**
在linux中目录也有一个inode，inode指向记录目录实际内容的磁盘物理位置。

## 什么是硬链接和软链接
再回到我们的例子里，ls的结果第一列是该文件的inode号，可以看到硬链接的inode号和文件原来的inode号是一样的。说明指向的物理位置是一样的。
我们看软链接的inode号不同，说明是一个新的文件，该文件指向的是另外的一个物理位置。该物理位置里面存放着链接目标的路径。

## 区别
让我们删除连接目标文件a
```
$ rm a
$ ls -ali
总用量 12
19796573 drwxrwxr-x  2 yshen yshen 4096 10月 10 11:35 .
58720267 drwxrwxr-x 43 yshen yshen 4096 10月 10 11:33 ..
19796575 -rw-rw-r--  1 yshen yshen   12 10月 10 11:33 a_hard_link
19796574 lrwxrwxrwx  1 yshen yshen    1 10月 10 11:34 a_soft_link -> a
```
硬链接和软链接现在还能访问么？
```
$ cat a_hard_link 
Hello world
$ cat a_soft_link 
cat: a_soft_link: 没有那个文件或目录
```
硬链接还能访问，软链接现在已经失效了！根据定义我们不难找出原因：
(1)硬链接指向的是物理位置和连接对象指向的物理位置一样，该物理位置有一个引用计数（ls的第三列），删除链接对象后，引用计数从原来的2变为1，所以物理上文件并没有删除，如果为0的话采用unlink删除。所以硬链接依然可以访问。
(2)软链接其实是一个文件，文件指向的路径已经失效了，所以会显示没有那个文件或目录。

我们总结一下区别：
(1) 链接目标失效之后，硬链接依然可以访问。软链接不能访问。
(2) 硬链接只能指向和链接目标同一个分区，软链接没有限制，甚至可以指向网络地址。





