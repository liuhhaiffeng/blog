---
title: linux中的本地化
comments: true
date: 2016-10-27 10:03:21
tags: linux
---


>linux中显示乱码了怎么办？这时候需要设置对本地化。知其然还要知其所以然，下面一步步来为你截开linux中的本地化的神秘面纱。


从locale说起
------------

locale翻译过来是`本地`的意思。linux中的`locale`工具能够输出当前本地化信息，或者输出所有支持的本地化、编码信息。
在linux中执行`locale`命令可以显示出当前的本地化信息。
```
$ locale
LANG=zh_CN.UTF-8
LANGUAGE=zh_CN:zh
LC_CTYPE="zh_CN.UTF-8"
LC_NUMERIC="zh_CN.UTF-8"
LC_TIME="zh_CN.UTF-8"
LC_COLLATE="zh_CN.UTF-8"
LC_MONETARY="zh_CN.UTF-8"
LC_MESSAGES="zh_CN.UTF-8"
LC_PAPER="zh_CN.UTF-8"
LC_NAME="zh_CN.UTF-8"
LC_ADDRESS="zh_CN.UTF-8"
LC_TELEPHONE="zh_CN.UTF-8"
LC_MEASUREMENT="zh_CN.UTF-8"
LC_IDENTIFICATION="zh_CN.UTF-8"
LC_ALL=
```
本地化信息包含了13个变量

1. LC_CTYPE
用于字符分类和字符串处理（大小写转换），控制所有字符的处理方式，包括字符编码，字符是单字节还是多字节，如何打印等。是最重要的一个环境变量。
2. LC_COLLATE
字符的比较和排序规则。
3. LC_MONETARY
货币格式。
4. LC_NUMERIC
非货币的数字显示格式。
5. LC_TIME
时间和日期格式。
6. LC_MESSAGES
提示信息的语言。另外还有一个LANGUAGE参数，它与LC_MESSAGES相似，但如果该参数一旦设置，则LC_MESSAGES参数就会失效。LANGUAGE参数可同时设置多种语言信息，如LANGUANE="zh_CN.GB18030:zh_CN.GB2312:zh_CN"。
7. LANG
LC_*的默认值，是最低级别的设置，如果LC_*没有设置，则使用该值。类似于 LC_ALL。如果LANG设置了，别的变量也设置成这个值。
8. LC_ALL
它是一个宏，如果该值设置了，则该值会覆盖所有LC_*的设置值。注意，LANG的值不受该宏影响。
9. LC_PAPER
纸张大小。
10. LC_NAME
名称的格式
11. LC_ADDRESS
地址的格式
12. LC_TELEPHONE
电话的格式
13. LC_MEASUREMENT
度量单位


可以看各种变量的值都是`zh_CN.UTF-8`,它表示什么意思呢？
分为2个部分:
`zh_CN` : 是语言.地点信息
`UTF-8` : 是字符编码信息
其书写格式是`语言[_地域[.字符集]]`
比如`zh_CN.UTF-8`说明现在用中文，地处中华人民共和国，用的是UTF-8字符编码。
比如`zh_TW.BIG5`说名现在用中文，地处台湾，用的是大五码字符集
可以通过`locale -a`命令查看支持的地点
```
$ locale -a
C           #最早期简单的C语言环境
C.UTF-8
en_US.utf8  #美国
POSIX       #posix标准
zh_CN.utf8  #中国
zh_SG.utf8  #新加坡

```
可以通过`locale -m`命令查看支持的编码
```
$ locale -m | grep BIG
BIG5
BIG5-HKSCS
UTF-8
GB18030
GB2312
GBK
GB_1988-80
```

有关这些信息都放在`/usr/share/i18n`文件夹下：
```
/usr/share/i18n
|-- SUPPORTED
|-- charmaps
|   |-- ANSI_X3.110-1983.gz
|   |-- ANSI_X3.4-1968.gz
|   |-- ARMSCII-8.gz
|   |-- ASMO_449.gz
|   |-- BIG5-HKSCS.gz
|   |-- BIG5.gz
|   |-- UTF-8.gz
|   |-- VIDEOTEX-SUPPL.gz
|   |-- VISCII.gz
|   `-- WINDOWS-31J.gz
`-- locales
    |-- POSIX
    |-- aa_DJ
    |-- aa_ER
    |-- aa_ER@saaho
    |-- aa_ET
    |-- af_ZA
    |-- zh_CN
    |-- zh_HK
    |-- zh_SG
    |-- zh_TW
    `-- zu_ZA

```
locales文件夹下面其实都是可编辑的文本文件。有兴趣的可以打开试试。

有时候因为编码的不同，显示出来的是乱码。那么如何设置呢？

如何设置编码
-------------
设定locale就是设定12大类的locale分类属性，即13个`LC_*`。除了这13个变量可以设定以外，为了简便起见，还有两个变量：`LC_ALL`和`LANG`。它们之间有一个优先级的关系：
```
LC_ALL > LANG
```
可以这么说，LC_ALL是最上级设定或者强制设定，而LANG是默认设定值。

1、如果你设定了`LC_ALL＝zh_CN.UTF-8`，那么不管`LC_*`和`LANG`设定成什么值，它们都会被强制服从`LC_ALL`的设定，成为 `zh_CN.UTF-8`。

2、假如你设定了`LANG＝zh_CN.UTF-8`，并且没有设定`LC_ALL`的话，那么系统的locale设定以`LC_*=zh_CN.UTF-8`。

PS: 除了环境变量设置对，如果使用的是图形化终端，还需要在终端设置相应的编码。

