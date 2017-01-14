---
title: mysql数据处理
date: 2015-02-17 18:10:00
tags: [数据库, "MySQL"]
categories: 技术分享
---

## 任务

> 将近10天的温度数据，9个温度测点，采样间隔为1分钟，共计12万行数据，需要导出每小时的温度数据。

ps: 现在有很多工具，比如mysql workbench可以很方便完成数据导入的工作。这里用ｃ语言把数据转为insert语句，是为了更好的解释其中的工作原理。
## 数据格式

数据格式如下：有3列，分别是日期，时间，温度。文件名是测点编号。

![ddd](http://img.blog.csdn.net/20141125135942138?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc2hlbnl1Zmx5aW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## 数据库设计为如下

```sql
    create database sensorDB;  
    use sensorDB;  
    create table sensor  
    (  
        ID smallint ,  
        dt Date,  
        tm Time,  
        temp float,  
        primary key (ID,dt,tm)  
    );  
```
建立一个视图把dt和tm两个字段合并 
```sql
create view temperature  
as  
select ID, cast(CONCAT(dt,' ',tm) as datetime) '时间' , temp from sensor   ;  
```

## txt格式文件导入数据库

用C语言写了一个小工具，可以把txt转成sql语句【见后面的源代码】        运行后，输入txt的文件名，要插入的表名，和传感器的编号即可。   程序运行完毕之后，自动生成sql语句如下：
![d](http://img.blog.csdn.net/20141125140008127?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc2hlbnl1Zmx5aW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
然后直接运行sql语句，数据就导入到数据库中了。

## 按指定间隔查询数据

利用了timestampdiff（）来计算时间间隔 ，并利用%来逐个判断
 
```sql
select * from temperature where ID = 9 and  timestampdiff(MINUTE,时间,'2014-04-19 15:00:00')%60 =0;
```


如下便是间隔1小时的温度数据

![dd](http://img.blog.csdn.net/20141125140019422?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc2hlbnl1Zmx5aW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## 附件：txt转sql源代码 C

```c
#include<stdlib.h>  
#include<stdio.h>  
#include<string.h>  
  
#define BUFFER_LEN 2048  
const char sp[] = { ' ', '\t', ';','\r','\n' };  
  
   
void error(char *msg)  
{  
    printf("ERROR:%s",msg);  
    fflush(stdin);  
    getchar();  
    exit(1);  
}  
int isSeperator(char  ch)  
{  
      
    for (int i = 0; i < sizeof(sp) / sizeof(char); i++)  
    {  
        if ( ch == sp[i])  
        {  
            return 1;  
        }  
    }  
    return 0;  
}  
int main()  
{  
    printf("=============================\n");  
    printf("          txt2sql\n");  
    printf("=============================\n");  
    printf("data fielname: ");  
    fflush(stdin);  
    char filename_in[256], filename_out[256];  
    scanf("%s",filename_in);  
    strcpy(filename_out, filename_in);  
    strcat(filename_out, "_sql.txt");  
    strcat(filename_in, ".txt");  
    printf("table name: ");  
    fflush(stdin);  
    char table_name[256];  
    scanf("%s", table_name);  
  
    printf("sensor ID: ");  
    fflush(stdin);  
    char sensor_ID[256];  
    scanf("%s", sensor_ID);  
  
  
    FILE *fin = fopen(filename_in, "r");  
    FILE *fout = fopen(filename_out, "w");  
    if (fin == NULL || fout == NULL)  
    {  
        error("cannot open file");  
    }  
    char buffer[BUFFER_LEN];  
    while (fgets(buffer, BUFFER_LEN, fin) != 0)  
    {  
        //除去多余的分隔符  
        char elem[BUFFER_LEN];  
        int k = 0;  
        memset(elem, 0, sizeof(elem));  
  
        for (int i = 0; i < strlen(buffer); i++)  
        {  
            if (buffer[i] == '\n' || buffer[i] == '\r' || buffer[i]=='\0') buffer[i] = ' ';  
        }  
  
  
        for (int i = 0; i < strlen(buffer); i++)  
        {  
            if (!isSeperator(buffer[i]))  
            {  
                elem[k++] = buffer[i];  
            }  
            else  
            {  
                //如果最后一个是分隔符则跳过  
                if (i == strlen(buffer) && isSeperator(buffer[i]))  
                    continue;  
                //如果下一个还是分隔符，则跳过。  
                if (i != strlen(buffer) - 1 && isSeperator(buffer[i + 1]))  
                    continue;  
                if (i != strlen(buffer) - 1 && !isSeperator(buffer[i + 1]))  
                                elem[k++] = buffer[i];  
            }  
        }  
        /*printf("\"%s\"",elem);*/  
        //经过处理的elem只有包含一个分隔符在一起的情况，最后没有空元素  
  
        fprintf(fout, "INSERT INTO %s VALUES ('%s',", table_name,sensor_ID);  
        int left = 0;  
        for (int i = 0; i < strlen(elem); i++)  
        {  
            if (isSeperator(elem[i]))  
            {  
                  
                fprintf(fout, "\'");  
                for (int j = left; j < i; j++)  
                {  
                    fprintf(fout, "%c", elem[j]);  
                }  
                fprintf(fout, "\',");  
                left = i + 1;  
            }  
        }  
        //the last one  
        fprintf(fout, "\'");  
        for (int j = left; j < strlen(elem); j++)  
        {  
            fprintf(fout, "%c", elem[j]);  
        }  
        fprintf(fout, "\');\n");  
           
    }  
      
  
  
    fclose(fin);   
    fclose(fout);  
  
    printf("Done!");  
    fflush(stdin);  
    getchar();  
    exit(0);  
}  
```
