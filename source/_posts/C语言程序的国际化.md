---
title: C语言程序的国际化
comments: true
date: 2016-11-02 11:37:12
tags: [编程, c]
categories: 技术分享
---


> c语言程序写的时候字符串用英语，基本固定了。那么如果想让别的语言的人使用（比如中文），就得把程序里面的字符串挨个翻译，最后再重新编译一次程序维护起来特别麻烦。那么有什么办法能够让程序显示中文？GNU的gettext项目就是用来做这个的。这篇文章以c语言为例介绍了国际化的gettext。



术语
---------
.po文件： po，Portable Object，里面记录了翻译内容。
.mo文件： mo，Machine Object，其实是一个二进制文件，用来提供给程序使用。
xgettext：一个工具，根据c文件生成po文件
msgfmt： 一个工具，根据po文件生成mo文件
msgmerge：更新po文件

基本过程
---------
```
        xgettext              翻译        msgfmt          INSTALL
.c文件----------->.pot文件---------->.po---------->.mo------------>./bin/po/zh/LC_MESSAGES/.mo

         gcc                              INSTALL
.c文件----------->可执行文件-------------------------------------->./bin
```

准备
---------

在程序中添加这么几行
```
#include <libintl.h>
#include<locale.h>
#define _(String) gettext (String)
#define gettext_noop(String) String
#define N_(String) gettext_noop (String)

int main()
{
...
    setlocale (LC_ALL, "");
    bindtextdomain (PACKAGE, LOCALEDIR);
    textdomain (PACKAGE);
...
}

```
在需要翻译的字符串使用`_()`包括进来。比如如下的一个例子：
```
#include<stdio.h>                                                               
#include<stdlib.h>
#include<locale.h>
#include <libintl.h>
#define _(String) gettext (String)
#define gettext_noop(String) String
#define N_(String) gettext_noop (String)

#define PACKAGE "main"
#define LOCALEDIR "po"

int main(int argc, char *argv[])
{
    setlocale (LC_ALL, "");
    bindtextdomain (PACKAGE, LOCALEDIR);
    textdomain (PACKAGE);
    printf(_("Hello world!\n"));

    return 0;
}

```
用xgettext程序来从.c文件中找到所有可以翻译的字符串并生成.pot模板文件
```
xgettext -k_ -o main.pot main.c 
cp main.pot main.po
```
翻译po文件中的字符串，po文件的格式详见http://www.gnu.org/software/gettext/manual/html_node/PO-Files.html#PO-Files
```
# SOME DESCRIPTIVE TITLE.                                                       
# Copyright (C) YEAR THE PACKAGE'S COPYRIGHT HOLDER
# This file is distributed under the same license as the PACKAGE package.
# FIRST AUTHOR <EMAIL@ADDRESS>, YEAR.
#
#, fuzzy
msgid ""
msgstr ""
"Project-Id-Version: PACKAGE VERSION\n"
"Report-Msgid-Bugs-To: \n"
"POT-Creation-Date: 2016-11-02 17:31+0800\n"
"PO-Revision-Date: YEAR-MO-DA HO:MI+ZONE\n"
"Last-Translator: FULL NAME <EMAIL@ADDRESS>\n"
"Language-Team: LANGUAGE <LL@li.org>\n"
"Language: \n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"

#: main.c:18
#, c-format
msgid "Hello world!\n"
msgstr "你好世界！\n"

```
用msgfmt从po文件生成mo文件

```
$ msgfmt ./main.po -o main.mo
```

把mo文件放到正确的地方，注意这里的`po`文件夹是main.c里面`PACKAGE`对应的文件夹。
```
$ mkdir -p po/zh_CN/LC_MESSAGES/
$ cp ./main.mo  ./po/zh_CN/LC_MESSAGES/main.mo
```
接下来要gcc编译这个程序。
```
gcc main.c -o main
```
最后是这样的
```
$ tree
.
├── main
├── main.c
├── main.mo
├── main.po
├── main.pot
└── po
    └── zh_CN
        └── LC_MESSAGES
            └── main.mo

3 directories, 6 files
```

运行一下，看到已经是翻译成中文了。
```
$ ./main 
你好世界！
```

维护
---------

如果以后修改了.c文件，比如新加了一行
```
...
    printf(_("Hello world!\n"));
    printf(_("Hello again!\n"));
...
```
那么用msgmerge就可以更新原来的po文件，不需要再从头来过
```
$ xgettext --add-comments --keyword=_ main.c -o main.pot --from-code=UTF-8
$ mv main.pot main.po
$ msgmerge main.po.old main.po
main.po: 警告： 字符集“CHARSET”不是可移植的编码名称。
                将消息转换为用户字符集可能不工作。
.... 完成。
# SOME DESCRIPTIVE TITLE.
# Copyright (C) YEAR THE PACKAGE'S COPYRIGHT HOLDER
# This file is distributed under the same license as the PACKAGE package.
# FIRST AUTHOR <EMAIL@ADDRESS>, YEAR.
#
#, fuzzy
msgid ""
msgstr ""
"Project-Id-Version: PACKAGE VERSION\n"
"Report-Msgid-Bugs-To: \n"
"POT-Creation-Date: 2016-11-02 18:30+0800\n"
"PO-Revision-Date: YEAR-MO-DA HO:MI+ZONE\n"
"Last-Translator: FULL NAME <EMAIL@ADDRESS>\n"
"Language-Team: LANGUAGE <LL@li.org>\n"
"Language: \n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"

#: main.c:18
#, c-format
msgid "Hello world!\n"
msgstr "你好世界！\n"

#: main.c:19
#, fuzzy, c-format
msgid "Hello again!\n"
msgstr "你好世界！\n"

```

可以看到Hello again被翻译成了你好世界，上面注释有fuzzy是标示这个翻译是模糊匹配的，需要人工再翻译一下。翻译过后去掉fuzzy即可。弄完之后用msgfmt再生成mo文件并拷贝到指定目录即可。当然这些都可以用Makefile来维护。稍后再写一篇PostgreSQL国际化的内容。
