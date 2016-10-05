---
title: hexo静态博客修改侧边栏
comments: true
date: 2016-10-04 11:53:33
tags: ["hexo","博客"]
---


> hexo主题默认的侧边栏只有`首页`、`归档`、`标签`这几项。调研了几个比较优秀的博客之后发现，还需要添加`关于`、`搜索`、`阅读`、`旅行`等等几个侧边栏项目，那么如何在博客里面添加新的侧边栏项目呢？

## 添加关于、留言项目
关于、留言这两个项目实际上是利用了hexo的page页面功能。

### Step1 新建关于页面
```
hexo new page about
INFO  Created: ~/blog/source/about/index.md
```
打开修改为如下
```
---
title: about
date: 2016-09-17 13:21:20
comments: false   添加这行关闭评论
---
here is something about me
```


### Step2 新建留言页面
```
hexo new page message

```
打开修改为如下
```
---
title: message
date: 2016-09-17 13:21:20
comments: true   打开评论
---
有事请留言
```

### Step3 把关于、留言添加到侧边栏中

打开主题配置文件`./themes/next/_config.yml`，可以看到侧边栏Menu Settings有两项：
第一项是menu，控制侧边栏显示的项目；
第二项是menu_icons，控制显示项目的图标；
```
menu:
  home: /
  books: /categories/阅读
  travel: /categories/旅行
  archives: /archives
  tags: /tags
  message: /message #添加这一行
  about: /about     #添加这一行

menu_icons:
  enable: true
  home: home
  about: user     #添加这一行
  categories: th
  books: book
  travel: plane
  tags: tags
  archives: archive
  commonweal: heartbeat
  message: comment #添加这一行

```
添加我们需要的阅读、旅行两个项目。
然后配置项目的图标，可以参考http://fontawesome.io/icons/

### Step4 生成
```
hexo g
```
我们看到，生成了关于、评论的页面
```
/public/about/index.html
/public/message/index.html
```


## 添加阅读、旅行项目
阅读、旅行这两个项目实际上是利用了hexo主题的categories分类功能。
### Step1  模板添加分类categories

打开` ./scaffolds/post.md`添加一行`categories:`
```
---
title: {{ title }}
date: {{ date }}
tags:
categories:
comments: true
---
```

### Step2 把分类添加到侧边栏中

打开主题配置文件`./themes/next/_config.yml`，可以看到侧边栏Menu Settings有两项：
第一项是menu，控制侧边栏显示的项目；
第二项是menu_icons，控制显示项目的图标；
```
menu:
  home: /
  books: /categories/阅读
  travel: /categories/旅行
  archives: /archives
  tags: /tags
  message: /message
  about: /about

menu_icons:
  enable: true
  home: home
  about: user
  categories: th
  books: book
  travel: plane
  tags: tags
  archives: archive
  commonweal: heartbeat
  message: comment

```
添加我们需要的阅读、旅行两个项目。
然后配置项目的图标，可以参考http://fontawesome.io/icons/

### Step3 新建文章

```
$ hexo new "我推荐的编程书籍"
INFO  Created: ~/blog/source/_posts/我推荐的编程书籍.md
$ vim ./source/_posts/我推荐的编程书籍.md 

```
新建的文章自动添加了categories标签。
```
---
title: 我推荐的编程书籍
comments: true
date: 2016-10-04 11:19:10
tags: 阅读
categories:
---
## 《代码大全》 Code Complete
。。。该书籍的介绍。。。
```

### Step4 生成
```
hexo g
```
我们看到，生成了书籍分类的页面index.html
```
/home/yshen/blog/public/categories/书籍/index.html
```


## 效果图
最后，看一下最终效果：
![效果图](http://static.zybuluo.com/shenyuflying/nsteo6fk5y9ss50gz1z22gny/side_bar.png)


