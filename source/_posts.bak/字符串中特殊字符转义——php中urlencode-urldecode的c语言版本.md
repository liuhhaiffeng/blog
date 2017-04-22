---
title: 字符串中特殊字符转义——php中urlencode urldecode的c语言版本
comments: true
date: 2016-10-09 05:54:58
tags: c
---


> 有时候字符串中的特殊字符会影响到字符串解析，比如在某些情况中不允许字符串中有特殊字符，但是用户输入的文本中可能包含一些特殊字符。这时我们需要对特殊字符进行转义。在这里我们模仿php里面的urlencode和urldecode函数，给出其C语言实现。

例子
---------
```
$ ./str_encode 
 select * from hello_world; = [+select+%2A+from+hello_world%3B]

```

源码
------
<!--more-->
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>


static char *str_encode(char *s, int len, int *new_length);
static int str_decode(char *str, int len);
static int htoi(char *s);


int main(int argc, char *argv[])
{

	char * msg = "hello world!";
	char * msg_encode;
	char  msg_decode[1024] ;

	msg_encode = str_encode(msg, strlen(msg), NULL);

	/* decode modify the str inplace , so have to make a copy of it */
	strcpy(msg_decode, msg_encode);
	str_decode(msg_decode, strlen(msg_decode));

	printf("%s = [%s]\n", msg_decode, msg_encode);

}

static char *str_encode(char *s, int len, int *new_length)
{
    register unsigned char c;
    unsigned char *to, *start;
    unsigned char const *from, *end;
    static unsigned char hexchars[] = "0123456789ABCDEF";
    from = (unsigned char *)s;
    end  = (unsigned char *)s + len;
    start = to = (unsigned char *) malloc((3*len+1)*sizeof(char));

    while (from < end)
    {
        c = *from++;

        if (c == ' ')
        {
            *to++ = '+';
        }
        else if ((c < '0' && c != '-' && c != '.') ||
                 (c < 'A' && c > '9') ||
                 (c > 'Z' && c < 'a' && c != '_') ||
                 (c > 'z'))
        {
            to[0] = '%';
            to[1] = hexchars[c >> 4];
            to[2] = hexchars[c & 15];
            to += 3;
        }
        else
        {
            *to++ = c;
        }
    }
    *to = 0;
    if (new_length)
    {
        *new_length = to - start;
    }
    return (char *) start;
}


static int str_decode(char *str, int len)
{
    char *dest = str;
    char *data = str;

    while (len--)
    {
        if (*data == '+')
        {
            *dest = ' ';
        }
        else if (*data == '%' && len >= 2 && isxdigit((int) *(data + 1)) && isxdigit((int) *(data + 2)))
        {
            *dest = (char) htoi(data + 1);
            data += 2;
            len -= 2;
        }
        else
        {
            *dest = *data;
        }
        data++;
        dest++;
    }
    *dest = '\0';
    return dest - str;
}




static int htoi(char *s)
{
    int value;
    int c;

    c = ((unsigned char *)s)[0];
    if (isupper(c))
        c = tolower(c);
    value = (c >= '0' && c <= '9' ? c - '0' : c - 'a' + 10) * 16;

    c = ((unsigned char *)s)[1];
    if (isupper(c))
        c = tolower(c);
    value += c >= '0' && c <= '9' ? c - '0' : c - 'a' + 10;

    return (value);
}
```





