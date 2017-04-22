---
title: soname
comments: true
date: 2017-02-09 09:06:12
tags: [编程,c,linux]
categories: 技术分享
---


客户反映一个动态库改名为后，编译时链接也是正确的。但在编译出来的应用做依赖分析时(ldd)，发现加载的还是改名之前的动态库。经过分析，是由于编译时带了`-Wl,-soname -Wl,libclntsh.so`选项：
```
gcc -shared  .libs/bind.o ... .libs/kociapi.o  -L/lib  -Wl,--version-script=koci.lds -Wl,-soname -Wl,libclntsh.so -o .libs/libclntsh.so
```
那么这样在连接时传给`ld`的参数就会是`ld -soname libclntsh.so`
生成的动态库也就带上了`SONAME`这个字段。
<center> ![image_1b8gve8oi15h91nd0qjsb5eaju9.png-52.2kB][1] </center>
关于这个选项手册`man ld`里面解释是：
```
-hname
-soname=name
    When creating an ELF shared object, set the internal DT_SONAME field  to  the  specified
    name.   When  an  executable is linked with a shared object which has a DT_SONAME field,
    then when the executable is run the dynamic linker  will  attempt  to  load  the  shared
    object specified by the DT_SONAME field rather than the using the file name given to the
    linker.
```
意思是如创建动态链接库的时候，如果加了`soname`按么生成的ELF文件中就会有`SONAME`这个字段。之后可执行文件加载这个动态链接库的时候，就会按照`SONAME`这个字段来找对应的文件，而不是当初链接可执行文件时候指定的动态链接库的文件名。这就是问题发生的原因。

那么`soname`有什么作用呢？
[google](https://en.wikipedia.org/wiki/Soname)上的解释是：
```
The soname is often used to provide version backwards-compatibility information. For instance, if versions 1.0 through 1.9 of the shared library libx provide identical interface, they would all have the same soname, e.g. libx.so.1. If the system only includes version 1.3 of that shared object, with filename libx.so.1.3, the soname field of the shared object tells the system that it can be used to fill the dependency for a binary which was originally compiled using version 1.2.
```
也就是说`soname`是为了向下兼容的目的，比如老版本的名字是lib1.2，升级之后lib1.3。如果没有加-soname那么系统还会找lib1.2，也就没有达到升级的目的。如果-soname lib1那么系统认为lib1.2和lib1.3是兼容的，找的时候文件名只要是匹配lib1这个soname前缀就可以了。

考虑到目前我们的动态链接库目前都是叫libclntsh.so，并没有名字的区分。最后的解决方法很简单，就是增加编译选项: `--disable-soname`在编译的时候不加`-soname`：
```
      ./configure --disable-soname
       make && make install
```


  [1]: http://static.zybuluo.com/shenyuflying/lklzdw5gskd5dmuzlnpazkl8/image_1b8gve8oi15h91nd0qjsb5eaju9.png


如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
