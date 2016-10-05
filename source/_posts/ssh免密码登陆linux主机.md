---
title: ssh免密码登陆linux主机
comments: true
date: 2016-10-05 10:07:14
tags: linux
---

> 比如想登陆一台服务器但是每次都需要输密码，这时候我们可以用ssh的authorized_keys文件来记录允许登陆的主机。

## 配置hosts
比如我们有2台主机，分别是host1和host2
```
cat /etc/hosts
192.168.0.101 host1
192.168.0.102 host2
```
## 生成key
```
[root@host1 ~]# ssh-keygen -t dsa -f ~/.ssh/id_dsa -N ""
Generating public/private dsa key pair.
Your identification has been saved in /root/.ssh/id_dsa.
Your public key has been saved in /root/.ssh/id_dsa.pub.
The key fingerprint is:
91:09:5c:82:5a:6a:50:08:4e:b2:0c:62:de:cc:74:44 root@host1.clusterlabs.org
The key's randomart image is:
+--[ DSA 1024]----+
|==.ooEo..        |
|X O + .o o       |
| * A    +        |
|  +      .       |
| .      S        |
|                 |
|                 |
|                 |
|                 |
+-----------------+
```

## 新建authorized_keys文件

```
[root@host1 ~]# cp ~/.ssh/id_dsa.pub ~/.ssh/authorized_keys
```


## 把authorized_keys文件部署到远程主机
```
[root@host1 ~]# scp -r ~/.ssh/authorized_keys host2:～/.ssh/

```

## 免密码登陆远程主机
```
[root@host1 ~]# ssh host2 
```

其实就是把本机生成的*.pub文件内容copy到另外一台主机~/.ssh/authorized_keys文件里面就ok！
