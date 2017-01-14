---
title: c和汇编混合编程
comments: true
date: 2016-11-14 12:42:51
tags: [编程,c,assembly]
categories: 技术分享
---

> 在各种高级语言大行其道的今天为什么要用汇编呢？其实主要的原因有：第一，在C语言在关键地方嵌入汇编可以获得最大的性能提升，比如说一些关键算法；第二，实现硬件相关的功能（这点嵌入式开发经常用到）。第三，不能用C语言实现的特性可以用汇编实现，比如说可以利用lock指令来实现原子操作。
本文介绍了如何把汇编语言嵌入到c语言中的基础，然后给了2个例子。

<!--more-->

在数据库中，为了实现一些特殊的操作，比如[tas锁](http://shenyu.wiki/2016/11/13/PostgreSQL%E4%B8%AD%E7%9A%84spinlock%E5%AE%9E%E7%8E%B0%E7%9A%84%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/)，需要借助汇编来完成。比如PostgreSQL中的tas锁就是用汇编写的。那么如何用c和汇编混合编程呢？先介绍一下基本语法，c语言中嵌入汇编的语法如下：
```
     asm [volatile] ( AssemblerTemplate
                      : OutputOperands
                      [ : InputOperands
                      [ : Clobbers ] ])
```
`asm`：标志一段嵌入式汇编的开始
`volatile`：关闭编译器优化，这样就就不会把某些编译器认为无效的汇编删掉
`AssemblerTemplate`：用于生成汇编的模板，里面包含了汇编代码和一些占位符
`OutputOperands`：在汇编中修改的c程序中的变量，用逗号分隔
`InputOperands`：在汇编中读取的c程序中的变量，用逗号分隔
`Clobbers`：除了在OutputOperands列出的之外被汇编修改的register或其他的值

下面分别介绍一下上述内容

AssemblerTemplate
-------------------
汇编模板实际上是一个字符串，比如
```
        "   cmpb    $0,%1   \n"
        "   jne     1f      \n"
        "   lock            \n"
        "   xchgb   %0,%1   \n"
```
里面包含了对`OutputOperands`、`InputOperands`的引用。接下来编译器会把汇编魔板中的引用等等变换为实际的汇编指令。在汇编模板中除了输入汇编指令之外，还可以有一些特殊符号，特殊符号表示了其他的含义：
`%n`：引用`OutputOperands`、`InputOperands`中的第几个值 
`%%`：在汇编中输出一个`%`符号
`%=`：输出一个数字，每次都是不同的。一般用来作为goto标签。
`%{`：输出`{`
`%|`：输出`|`
`%}`：输出`}`

OutputOperands
-----------------
`OutputOperands`的语法如下：
```
[ [asmSymbolicName] ] constraint (cvariablename)
```
### asmSymbolicName
`asmSymbolicName`asm符号名称，如果asm模板里面用`%[Value]`来指定了一个名称。那么可以在`OutputOperands`里面用`[ [asmSymbolicName] ] constraint (cvariablename)`映射到具体的c程序变量名。比如：
```
    int a,b;
    /* a=1;b=2;*/
    asm("mov $1,%[a]\n"
        "mov $2,%[b]\n"
        :[a]"+rm"(a),[b]"+rm"(b)
        :/* no output */
        :"cc");
    printf("a=%d\nb=%d\n",a,b);
```
如果没有用`asm符号`名称的话，用0下标开始的变量名`%0`,`%1`来引用。比如：
```
    int a,b;
    /* a=1;b=2;*/
    asm("mov $1,%0\n"
        "mov $2,%1\n"
        :"+rm"(a),"+rm"(b)
        :/* no output */
        :"cc");
    printf("a=%d\nb=%d\n",a,b);
```
### constraint
一个字符串用来当做界定符。界定符用来说明变量放置的位置
`=` 该变量覆盖已存在的变量
`+` 变量用于读写
`r` 把变量放在register寄存器中
`m` 把变量放在memory中

### cvariablename
引用c程序上下文中的变量名


InputOperands
------------------
正如字面意思所讲，`InputOperands`表示输入的变量，这些变量将被汇编使用。

Clobbers
-----------------
`cc`以上汇编代码改动了标志位
`memory`以上汇编代码进行了内存读写
    
好了，解释了基本语法之后，简单看几个例子：


例子1--赋值
-------------
最简单的一个例子，用了汇编的`mov`指令，用于赋值：
```c
#include<stdio.h>
#include<stdlib.h>
int main()
{
    int a,b;
    __asm__("mov $1,%0\n"
            "mov $2,%1\n"
            :"+rm"(a),"+rm"(b)
            :/* no output */
            :"cc");

    printf("a=%d\nb=%d\n",a,b);

}

```
可以看到结果，a，b已经赋值了。
```
$ ./main 
a=1
b=2

```
实际生成的汇编：
```
0000000000400526 <main>:
  400526:   55                      push   %rbp
  400527:   48 89 e5                mov    %rsp,%rbp
  40052a:   48 83 ec 10             sub    $0x10,%rsp
  40052e:   8b 55 f8                mov    -0x8(%rbp),%edx
  400531:   8b 45 fc                mov    -0x4(%rbp),%eax

  400534:   ba 01 00 00 00          mov    $0x1,%edx
  400539:   b8 02 00 00 00          mov    $0x2,%eax

  40053e:   89 55 f8                mov    %edx,-0x8(%rbp)
  400541:   89 45 fc                mov    %eax,-0x4(%rbp)
  400544:   8b 55 fc                mov    -0x4(%rbp),%edx
  400547:   8b 45 f8                mov    -0x8(%rbp),%eax
  40054a:   89 c6                   mov    %eax,%esi
  40054c:   bf f4 05 40 00          mov    $0x4005f4,%edi
  400551:   b8 00 00 00 00          mov    $0x0,%eax
  400556:   e8 a5 fe ff ff          callq  400400 <printf@plt>
  40055b:   b8 00 00 00 00          mov    $0x0,%eax
  400560:   c9                      leaveq
  400561:   c3                      retq
  400562:   66 2e 0f 1f 84 00 00    nopw   %cs:0x0(%rax,%rax,1)
  400569:   00 00 00 
  40056c:   0f 1f 40 00             nopl   0x0(%rax)

```

例子2--交换
--------------
一般在c语言中，交换a，b的值需要3条语句：
```c
temp=a;
a=b;
b=temp;
```
利用汇编的`xchg`指令，可以一条指令就完成a，b交换，而且不用中间变量
```c
#include<stdio.h>
#include<stdlib.h>
int main()
{
    int a=2,b=1;
    /* a=b;b=a;*/
    asm("xchg %0,%1\n"
        :"+rm"(a),"+rm"(b)
        :/* no output */
        :"cc");

    printf("a=%d\nb=%d\n",a,b);

}

```
可以看到结果，a，b已经赋值了。
```
$ ./main 
a=1
b=2

```

例子3--tas锁
-----------
tas锁求整个操作只由一条指令完成，并且将总线锁住，以保证操作的“原子性”。相比之下，C语句在编译之后到底有几条指令是没有保证的，也无法要求在计算过程中对总线加锁。这时候就要借助`lock`指令来完成这个工作。
```c
static __inline__ int
tas(volatile lock_t *lock)
{
    register lock_t _res = 1;

    __asm__ __volatile__(
        "   cmpb    $0,%1   \n"
        "   jne     1f      \n"
        "   lock            \n"
        "   xchgb   %0,%1   \n"
        "1: \n"
:       "+q"(_res), "+m"(*lock)
:       /* no inputs */
:       "memory", "cc");
    return (int) _res;
}
```
如上代码中的`lock`指令确保了接下来的`xchgb`指令交换`*lock`和`_res`的操作是原子性的。从而一下完成了获取`*lock`的值作为返回值和把`*lock`设为1这两个动作一下完成。

如果感兴趣可以进一步阅读：
gcc官方文档：https://gcc.gnu.org/onlinedocs/gcc-6.2.0/gcc/Extended-Asm.html#Extended-Asm
Linux汇编语言开发指南：http://www.ibm.com/developerworks/cn/linux/l-assembly
一篇好的博文：http://blog.csdn.net/hu3167343/article/details/37660593

