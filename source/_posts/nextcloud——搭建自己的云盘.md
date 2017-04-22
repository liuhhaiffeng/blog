---
title:  nextcloud——搭建自己的云盘
comments: true
date: 2017-03-05 18:50:22
tags: [linux, nextcloud, nas]
categories: 技术分享
---
 
> 最近多家云盘相继关停，费了很多时间才把上面的东西下载到本地，而且各大云盘都限速500KB/s，尤其是百度云上的小电影都变成了8秒。技术宅岂能容忍？是时候搭建自己的私有云盘了！

搭建自己的私有云有什么好处呢？首先没有什么容量、下载速度的限制，而且本地访问速度很快。然后可以和本地的cifs、ftp配合使用来实现多个设备文件共享：比如可以在电视、手机等等智能设备上挂载云盘中的文件来实现播放电影、看照片、听歌等需求。最后可以防止泄密和和谐^_^

说到私有云，其实有很多现成的产品可以使用，比如群晖、铁威马、西数等。买过来，插上一块硬盘就可以用，十分适合小白。但是成本略高，仅仅主机就需要1000多元，再加上一块硬盘，这种解决方案的成本一般都要超过2000元。自己搭建私有云的话，不仅成本很低，而且可以自己定制很多功能，比如在线笔记、邮件等等功能。但是需要会折腾linux哦！

自己搭建私有云其实很简单，首先需要一台主机，然后需要选择一个私有云软件（比如ownCloud、nextCloud、seafile）。在这里我还是用我的树莓派作为主机，虽然IO能力差了点，大概上传下载为2MB/s（无线），但是足够自己日常使用而且很省电。在对比几个不同的私有云软件之后，最终采用了nextCloud，感觉这个功能更为强大。具体如何搭建和配置nextCloud，它的[官网](https://nextcloud.com/)上说的很清楚，在这里就不再赘述了。搭建很快，最后的效果如下：

<center>![image_1baf0hai9veg123ib61dbe7blm.png-250.6kB][1]</center>
<center>登录界面</center>


<center>![image_1baf0gev81c3bhhl1mae3a21uh49.png-32.1kB][2]</center>
<center>首页</center>

nextCloud提供了一个大气的web界面，实现了基本的文件上传、下载、图片视频预览等基本功能。然而其强大之处就是可以安装[apps](https://apps.nextcloud.com/)来实现云盘的功能扩展。

外部存储
------------
在这里我用到了**外部存储app**来挂载诸多本地的资源，比如移动硬盘、smb/cifs等。
<center>![image_1baf176hj7qr12ltkum1c0vfj213.png-70.7kB][3]</center>

在线笔记
------------
另外用到了**在线笔记app**，这样可以有自己的在线笔记了，支持markdown语法而且是存储在本机的哦！
<center>![image_1baf1g8jfbl1934gecen2apm1g.png-23.1kB][4]</center>

markdown editor
------------------
最后可以**markdown editor**实现在线写博客，这样不用每次麻烦ssh登录到服务器来发布博客了。
<center>![image_1baf1m16ql8t1nnpgmmuac1u8a1t.png-238.3kB][5]</center>

功能是不是很强大，nextCloud你值得拥有！



  [1]: http://static.zybuluo.com/shenyuflying/iqi9hampq8pdkmqfoptmoszr/image_1baf0hai9veg123ib61dbe7blm.png
  [2]: http://static.zybuluo.com/shenyuflying/vn1xzj23galuj8mihi3tm3hd/image_1baf0gev81c3bhhl1mae3a21uh49.png
  [3]: http://static.zybuluo.com/shenyuflying/s0ksgdqnowh9nws8l67wp544/image_1baf176hj7qr12ltkum1c0vfj213.png
  [4]: http://static.zybuluo.com/shenyuflying/t7ok5ieds41aa3tdkmddvjmu/image_1baf1g8jfbl1934gecen2apm1g.png
  [5]: http://static.zybuluo.com/shenyuflying/m0off98icgxcnlfqjdo5gki2/image_1baf1m16ql8t1nnpgmmuac1u8a1t.png