---
title: hexo博客增加菜单栏目
comments: true
date: 2017-01-16 03:32:36
tags: [网站, hexo]
categories: 技术分享
---

> hexo中的yelee主题默认提供了`主页`、`归档`、`标签`、`关于`这几个菜单条目。那么如何增加新的条目呢？

## 新建页面
```
hexo new page "message"
```
新建的页面会在`source/message/index.md`这里，打开之后可以输入内容。
## 加入到菜单栏目
```
vim themes/yelee/_config.yml
```
在`menu`下面加入新建立的`message`
```
menu:
  主页: /
  归档: /archives/
  标签: /tags/
  导航: /map/
  留言: /message/  ##加在这里
  关于: /about/

```

## 更改css

博客增加了两个栏目：`导航`和`留言`。然后发现地方不够了，最后的`关于`被下面的图标覆盖了。怎么增加导航栏上下的高度呢？




```
vim ./themes/yelee/source/css/_partial/main.styl
```

```

    .header-menu{
        menu-line-height = (28/16)rem
        font-weight: 550;
        line-height: menu-line-height;
        font-family: inherit;
        cursor: pointer;
        text-transform: uppercase;
        float: none;
        min-height: @line-height * 5; ## 增加这个，控制了菜单的最小高度。
        max-height: @line-height * 7; ## 增加这个，控制了菜单的最大高度。
        overflow: visible;
        text-align: center;
        display: -webkit-box;
        -webkit-box-orient: horizontal;
        -webkit-box-pack: center;
        -webkit-box-align: center;
        li{
            cursor: default;
            a{
                font-size: (14/16)rem;
                min-width: left-col-width;
            }
        }
    }

```

最后的效果如下：

<center>![image_1b6ijg51it7m1512m9m12331h8k9.png-125.7kB][1]</center>

  [1]: http://static.zybuluo.com/shenyuflying/9wmd7ty8zmzysvltpy2o0p3s/image_1b6ijg51it7m1512m9m12331h8k9.png



如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
