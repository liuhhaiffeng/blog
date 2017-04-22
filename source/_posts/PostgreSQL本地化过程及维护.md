---
title: PostgreSQL本地化过程及维护
comments: true
date: 2016-11-03 07:47:59
tags: [数据库, PostgreSQL, PostgreSQL内核, 本地化]
categories: 技术分享
---


> PostgreSQL如果要支持本地化，那么./configure的时候要加上--enable-nls构建选项。加了该选项之后通过设置相应的locale之后，PostgreSQL显示的信息就可以是中文了。那么其中的过程是什么呢？如何在添加了新的程序或加了新的翻译字符串之后维护本地化内容呢？

[TOC]

术语
---------
.po文件： po，Portable Object，里面记录了翻译内容。
.mo文件： mo，Machine Object，其实是一个二进制文件，用来提供给程序使用。
xgettext：一个工具，根据c文件生成po文件
msgfmt： 一个工具，根据po文件生成mo文件
msgmerge：更新po文件
nls.mk：翻译包的配置，比如翻译哪些c文件

基本过程
---------
```
        xgettext            人工翻译        msgfmt          INSTALL
.c文件----------->.pot文件---------->.po---------->.mo------------>./bin/po/zh/LC_MESSAGES/.mo

         gcc                              INSTALL
.c文件----------->可执行文件-------------------------------------->./bin
```
基本过程如上图，首先需要xgettext工具根据.c文件生成.pot文件，接下来需要人工翻译。翻译过后用msgfmt工具把翻译好的.po文件生成.mo文件。最后把.mo文件放到制定位置即可。但是不需要我们自己直接调这些工具来完成本地化的工作，其过程都写在了`nls-global.mk`文件里面。我们只要`make init-po`、`make update-po`、`make install-po`这几个步骤即可。

最后生成的目录结构如下
```
$ tree
.
├── psql.c
...
├── Makefile
├── nls.mk
├── po
│   ├── cs.mo
│   ├── cs.po
...
│   ├── zh_CN.mo
│   ├── zh_CN.po
│   ├── zh_TW.mo
│   └── zh_TW.po

├── prompt.c
...

1 directory, 65 files

```

本地化详细过程
-------------------
./configure的时候加上--enable-nls构建选项之后，构建过程会发生如下变化：
1. gcc就会有-DENABLE_NLS的宏定义。从而gettext相关的代码就会被编译进去。
2. make就会处理相关国际化的内容。相关过程包含了nls.mk和nls-global.mk文件里面。

每个工具比如src/bin/psql目录面都有一个nls.mk文件用来说明目录下对应工具的翻译包的生成规则，另外src文件夹下面还有一个nls-global.mk文件里面写了如何利用gettext相关工具生成翻译和安装翻译包。下面看一下他们中的内容。

<!--more-->

### nls.mk
```
CATALOG_NAME     = psql
AVAIL_LANGUAGES  = cs de es fr it ja pl pt_BR ru zh_CN zh_TW
GETTEXT_FILES    = command.c common.c copy.c help.c input.c large_obj.c \
                   mainloop.c psqlscanslash.c startup.c \
                   describe.c sql_help.h sql_help.c \
                   tab-complete.c variables.c \
                   ../../fe_utils/print.c ../../fe_utils/psqlscan.c \
                   ../../common/exec.c ../../common/fe_memutils.c ../../common/username.c \
    GETTEXT_TRIGGER../../common/wait_error.c
GETTEXT_TRIGGERS = N_ psql_error simple_prompt write_error
GETTEXT_FLAGS    = psql_error:1:c-format write_error:1:c-format
```
下面来解释每个变量：
`CATALOG_NAME`是指对应的程序，这里是psql
`AVAIL_LANGUAGES`是指的可用的翻译，比如这里有多种。
`GETTEXT_FILES`是指需要翻译的c文件，一般把程序c文件中带输出的c文件都加到这里
`GETTEXT_TRIGGERS`是除了默认的指包含可翻译字符串的函数，默认的有`_`、`gettext`。
`GETTEXT_FLAGS`是指`GETTEXT_TRIGGERS`列出的对应函数的第几个参数是待翻译的字符串


### nls-global.mk

这个文件比较长，我们分开来看。
第一部分都是注释
```
# src/nls-global.mk                                 
# Common rules for Native Language Support (NLS)
#
# If some subdirectory of the source tree wants to provide NLS, it
# needs to contain a file 'nls.mk' with the following make variable
# assignments:
#
# CATALOG_NAME          -- name of the message catalog (xxx.po); probably
#                          name of the program
# AVAIL_LANGUAGES       -- list of languages that are provided/supported
# GETTEXT_FILES         -- list of source files that contain message strings
# GETTEXT_TRIGGERS      -- (optional) list of functions that contain
#                          translatable strings
# GETTEXT_FLAGS         -- (optional) list of gettext --flag arguments to mark
#                          function arguments that contain C format strings
#                          (functions must be listed in TRIGGERS and FLAGS)
#
# That's all, the rest is done here, if --enable-nls was specified.
#
# The only user-visible targets here are 'init-po', to make an initial
# "blank" catalog from program sources, and 'update-po', which is to
# be called if the messages in the program source have changed, in
# order to merge the changes into the existing .po files.
```
是说如果这个文件夹下的程序需要本地化NLS那么需要包含一个nls.mk的文件，以及它的格式。nls.mk文件的内容上面讲过。
还说了用户只需要用`make init-po`来初始化一个空的`catalog`语言包然后`make update-po`来维护翻译就行了。另外的用户可以不关心。其实还提供了如下命令
`make install-po`来安装本地化文件
`make clean-po`来删除一些无用的中间文件比如.pot .po.new .mo
```
# existence checked by Makefile.global; otherwise we won't get here
include $(srcdir)/nls.mk
```
包含了这个目录下的nls.mk文件，即一些规则。其实这样只需要维护nls.mk文件就行。nls-global.mk文件一般不需要改动。
```
# If user specified the languages he wants in --enable-nls=LANGUAGES,
# filter out the rest.  Else use all available ones.
ifdef WANTED_LANGUAGES
LANGUAGES = $(filter $(WANTED_LANGUAGES), $(AVAIL_LANGUAGES))
else
LANGUAGES = $(AVAIL_LANGUAGES)
endif
```
这几行说了在./configure的时候加了`--enable-nls=LANGUAGES`就是说只需要支持一种语言，那么make就会用`filter`函数过滤掉不需要的语言，否则就会生成所有语言的翻译。
```
PO_FILES = $(addprefix po/, $(addsuffix .po, $(LANGUAGES)))
ALL_PO_FILES = $(addprefix po/, $(addsuffix .po, $(AVAIL_LANGUAGES)))
MO_FILES = $(addprefix po/, $(addsuffix .mo, $(LANGUAGES)))
```
这几行是说接下来需要生成哪些po文件和mo文件。比如LANGUAGES是`zh_CN`那么addsuffix会加上`.po`后缀生成`zh_CN.po`接下来addprefix会加上`po/`前缀生成`po/zh_CN.po`类似的也会生成`po/zh_CN.mo`

```
ifdef XGETTEXT
XGETTEXT += -ctranslator --copyright-holder='PostgreSQL Global Development Group' --msgid-bugs-address=pgsql-bugs@postgresql.org --no-wrap --sort-by-file --package-name='$(CATALOG_NAME) (PostgreSQL)' --package-version='$(MAJORVERSION)'
endif

ifdef MSGMERGE
MSGMERGE += --no-wrap --previous --sort-by-file
endif

```
这几行是在准备xgettext和msgerge命令需要的参数。

```
# _ is defined in c.h, so it's global
GETTEXT_TRIGGERS += _
GETTEXT_FLAGS    += _:1:pass-c-format
```
这几行对比nls.mk文件中对应的选项，其实就是加了默认的`_()`函数。
```
# common settings that apply to backend and all backend modules
BACKEND_COMMON_GETTEXT_TRIGGERS = \
    errmsg errmsg_plural:1,2 \
    errdetail errdetail_log errdetail_plural:1,2 \
    errhint \
    errcontext \
    XactLockTableWait:4 \
    MultiXactIdWait:6 \
    ConditionalMultiXactIdWait:6
BACKEND_COMMON_GETTEXT_FLAGS = \
    errmsg:1:c-format errmsg_plural:1:c-format errmsg_plural:2:c-format \
    errdetail:1:c-format errdetail_log:1:c-format errdetail_plural:1:c-format errdetail_plural:2:c-format \
    errhint:1:c-format \
    errcontext:1:c-format

```
接下来又加了另外的一些函数。这些可以认为是多个语言包默认都要支持的，就不用挨个加在nls.mk文件里面了。
```
all-po: $(MO_FILES)

%.mo: %.po
    $(MSGFMT) $(MSGFMT_FLAGS) -o $@ $<
```
这里是生成规则，是说最后需要生成`.mo`文件。生成`.mo`文件首先需要先成`.po`文件。`.mo`来生成。
```
ifeq ($(word 1,$(GETTEXT_FILES)),+)
po/$(CATALOG_NAME).pot: $(word 2, $(GETTEXT_FILES)) $(MAKEFILE_LIST)                                                                                                                                             
ifdef XGETTEXT
    $(XGETTEXT) -D $(srcdir) -n $(addprefix -k, $(GETTEXT_TRIGGERS)) $(addprefix --flag=, $(GETTEXT_FLAGS)) -f $<
else
    @echo "You don't have 'xgettext'."; exit 1
endif
else # GETTEXT_FILES
po/$(CATALOG_NAME).pot: $(GETTEXT_FILES) $(MAKEFILE_LIST)
# Change to srcdir explicitly, don't rely on $^.  That way we get
# consistent #: file references in the po files.
ifdef XGETTEXT
    $(XGETTEXT) -D $(srcdir) -n $(addprefix -k, $(GETTEXT_TRIGGERS)) $(addprefix --flag=, $(GETTEXT_FLAGS)) $(GETTEXT_FILES)
else
    @echo "You don't have 'xgettext'."; exit 1
endif
endif # GETTEXT_FILES
    @$(MKDIR_P) $(dir $@)
    sed -e '1,18 { s/SOME DESCRIPTIVE TITLE./LANGUAGE message translation file for $(CATALOG_NAME)/;s/PACKAGE/PostgreSQL/g;s/VERSION/$(MAJORVERSION)/g;s/YEAR/'`date +%Y`'/g; }' messages.po >$@
    rm messages.po


```
这一段是最重要的，根据规则`po/$(CATALOG_NAME).pot: $(GETTEXT_FILES) $(MAKEFILE_LIST)`来看是生成`pot`文件依赖`GETTEXT_FILES`和`MAKEFILE_LIST`这段比较复杂，就是用xgettext生成po文件。下面是一些安装的规则，这里就略去不讲了。具体语法可以参考make的文档。

ok，明白了基本的原理，那么如何维护呢？下面介绍一些维护的内容。


维护
---------
### 添加了新的程序
从其他目录下拷贝一个nls.mk文件过来，可以参考psql的那个nls.mk：
执行如下命令从c文件生成po文件
```
make init-po
```
接下来翻译po文件即可，翻译过后把fuzzy去掉。需要注意占位符的位置是一致的。
最后
```
make install
```
生成mo文件并安装到对应的目录

### 更改或加入了新的翻译字符串

当我们改动代码加入或修改了需要翻译的字符串时需要执行命令
```
make update-po
```
接下来翻译po文件即可，翻译过后把fuzzy去掉。
最后
```
make install
```

### 更改或加入了新的c文件
修改nls.mk在`GETTEXT_FILES`加上新加的c文件



