---
title: linux下配置IP地址子网掩码和网关
comments: true
date: 2016-10-05 11:09:42
tags: linux
---

## 命令配置

```
ifconfig eth0 ipaddr 192.12.12.12 netmask 255.255.255.0 
route add default gw 192.12.12.1
ifconfig eth0 down
ifconfig eht0 up
```

该方法重启过之后就失效了，如果要永久生效，方法有：
1. 把以上配置命令加入到启动脚本里面
2. 修改网络配置文件


