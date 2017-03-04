---
title:  科学上网——搭建自己的DNS服务器
comments: true
date: 2017-02-16 08:16:22
tags: [dns,dnsmasq,树莓派,web]
categories: 技术分享
---


> DNS（Domain Name Server）故名思议就是把把域名解析为IP地址。为啥要自己搭DNS服务器呢？好处主要有以下几个：（1）通过缓存来提高域名解析的速度。（2）通过域名指向正确的IP地址能够访问被wall的网站。（3）通过域名指向无效的IP地址来过滤网站的广告。那么这篇文章就带领你搭建自己的DNS服务器来实现上述功能。

首先你需要一台主机来做DNS服务器：可以用普通电脑，或者树莓派，或者一个路由器刷openwrt系统。其设置都是大同小异。这里我用树莓派，比较省电，可以7x24小时开机、性能也还不错。

<center> ![image_1b92v70v7a93gac1ofog7k1no29.png-172.7kB][1] </center>

安装dnsmasq
------------
dnsmasq是linux下的一个轻量级DHCP和DNS服务器。在这里我们只用到它的DNS服务。那我们先把它装好：
```
sudo apt install dnsmasq -y
```
把如下配置项加入`/etc/dnsmasq.conf`里
```
listen-address=127.0.0.1,192.168.1.10
cache-size=15000
```
这个地址`192.168.1.10`就是树莓派的固定IP。
设置好之后我们通过`service dnsmasq status`能看到服务是否跑起来了。

设置hosts
---------
[这里](https://github.com/racaljk/hosts)提供了能访问很多被wall网站的hosts
[这里](https://github.com/vokins/yhosts)提供了能去广告的hosts
下面我们需要把这些hosts加入到本机的`/etc/hosts`文件里面，我写了一个脚本：
```
#!/bin/sh

# script to update the hosts file

# Must be run as root
if [[ `whoami` != "root" ]]
then
  echo "This install must be run as root or with sudo."
  exit
fi

DATE=`date`

cp /etc/hosts /etc/hosts.bak
echo "" > /etc/hosts

# for google, wiki, etc.
git clone https://github.com/racaljk/hosts
cat hosts/hosts >> /etc/hosts
rm -rf hosts

# wipe out ads
git clone https://github.com/vokins/yhosts
cat yhosts/hosts >> /etc/hosts
rm -rf yhosts

# delete some entry: taobao alipay ...
sed -i '/taobao/d' /etc/hosts
sed -i '/alipay/d' /etc/hosts

# restart the dns server
service dnsmasq restart

echo "updated at "$DATE >> update_hosts.log
```

接下来把这个脚本加入定时任务`crontab -e`，设为每周执行一次来获取最新的hosts。
```
# m h  dom mon dow   command
0 0 * * 1 /root/update_hosts.sh
```

配置路由器
-------
接下来需要把路由器的首选DNS服务器设置为我们刚刚搭建的DNS服务器的IP地址`192.168.1.10`。设置完之后你的手机、平板、电脑等设备就会在每次接入路由器的时候从路由器获取DNS。当然也可以挨个设置每个设备的DNS。很显然，因此如果你有多个设备比如手机平板的话，在路由器统一设置比较方便。

<center> ![image_1b8uf1bb49o41hh6eakk0ugn9.png-63.3kB][2] </center>



测试
---------



|        网站      |        网址               |     是否能访问    |
|------------------|---------------------------|------------------|
|谷歌              | https://www.google.com.hk | OK               |
|YouTube           |https://www.youtube.com    | OK               |
|维基百科           |https://www.wikipedia.org |  OK               |
|Facebook          |https://www.facebook.com   | OK                |
|twitter          |https://www.facebook.com    | OK                |
有些网站需要用https协议才能正常访问，所以需要输入https打头的地址。

另外好多原来被wall的app都可以用了，好多视频网站的广告也消失了。


参考
--------
https://wiki.archlinux.org/index.php/Dnsmasq
http://debugo.com/dnsmasq/
http://www.heystephenwood.com/2013/06/use-your-raspberry-pi-as-dns-cache-to.html


[1]: http://static.zybuluo.com/shenyuflying/dn5f81hxruayw9fcy79ox4u6/image_1b92v70v7a93gac1ofog7k1no29.png
[2]: http://static.zybuluo.com/shenyuflying/o40fxl4rt87oui5ikbqonsl6/image_1b8uf1bb49o41hh6eakk0ugn9.png

如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
