---
title: 利用sed生成规范的c代码
comments: true
date: 2016-10-20 10:15:38
tags: [sed,c]
---


> 有时候在编码过程中很多细节不注意，造成了代码不符合规范。一行行人工来修改，太麻烦了，不如交给工具做。鼎鼎大名的sed就是用来帮助你做这些繁重工作的。

我们准备了一个不规范的代码
```c
$ cat equal.c 
if(a==b && b == c)
{
	/* some blank char follow */      
	if (c== d)
	{
		func(a,b, c)
		func(d,e, f)
		func(a, b, c)
	}

}
```
其中有这几种不规范的情况

1. if后面不加空格
2. ==前后不加空格
3. 逗号后面不加空格
4. 行尾有空白

所以看起来十分乱。

废话不多说，先给出格式化脚本：

```sed
#!/bin/sed -f                                                               

# 处理==两边加空格情况
# a== b to a == b
s/\([^ ]\)\(==\)/\1 \2/g
# a ==b to a == b
s/\(==\)\([^ ]\)/\1 \2/g

# 处理&&两边加空格情况
# a&& b to a && b
s/\([^ ]\)\(&&\)/\1 \2/g
# a &&b to a && b
s/\(&&\)\([^ ]\)/\1 \2/g

# 处理!=两边加空格情况
# a!= b to a != b
s/\([^ ]\)\(!=\)/\1 \2/g
# a !=b to a != b
s/\(!=\)\([^ ]\)/\1 \2/g

# 处理if后面加空格情况
# if(...) to if (...)
s/if(/if (/g

# 处理for后面加空格情况
# for(...) to for (...)
s/for(/for (/g

# 处理逗号后面加空格情况
# (a,b,c) to (a, b, c) 
s/\(,\)\([^ \t]\)/, \2/g

# 处理行尾空格情况
# remove tailing blanks
s/[ \t]*$//g

```


用上面我们的sed脚本格式化一下，看到几个不规范的情况都修正了。
```c
$ ./ident.sed   ./equal.c 
if (a == b && b == c)
{
	/* some blank char follow */
	if (c == d)
	{
		func(a, b, c)
		func(d, e, f)
		func(a, b, c)
	}

}
```

然后测试了一个比较大的c文件，没有发现啥bug。但是字符串里面的也会格式化，需要注意一下。




