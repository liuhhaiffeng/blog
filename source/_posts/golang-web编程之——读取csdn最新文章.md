---
title: golang web编程之——读取csdn最新文章
comments: true
date: 2016-10-29 06:35:08
tags: [编程语言,golang]
categories: 技术分享
---


> 利用go语言内置的各种网络包可以方便的进行web编程。本文章利用了csdn的[开放API](http://open.csdn.net/wiki/apis)实现读取最新文章的需求。演示了go语言发起http get请求和json的umarshing特性。


简单的http GET请求
-------------------

go语言内置了`net/http`包，采用`http.Get`能够方便的发起GET请求

```go
package main

import (
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
)

func main() {

	resp, err := http.Get("http://api.csdn.net/blog/getnewarticlelist?client_id=????&page=1&size=10")
	if err != nil {
		log.Fatal(err)
	}
	contents, err := ioutil.ReadAll(resp.Body)
	resp.Body.Close()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("%s", contents)
}
```

看到返回的内容为json格式。

```
{"page":1,"count":12733,"size":10,"list":[{"ArticleId":52963517,"BlogId":5022393,"UserName":"u013410747","Title":"linux修改默认的编辑器","Description":"sudo select-editor    选择vim 搞定。。","PostTime":"\/Date(1477712141000)\/","UpdateTime":"\/Date(1477712177908)\/","Digg":0,"Bury":0,"ChannelId":2,"Type":1,"Status":0,"ViewCount":0,"CommentCount":0,"CommentAuth":2,"IsTop":false,"Level":0,"OutlinkCount":0,"Note":null,"IP":null,"Categories":null,"Tags":[],"ColumnAlias":null,"ColumnTitle":null,"MarkDownContent":null,"MarkDownDirectory":null,"ArticleEditType":0,"ArticleMore":null},...umnAlias":null,"ColumnTitle":null,"MarkDownContent":null,"MarkDownDirectory":null,"ArticleEditType":0,"ArticleMore":null}]}成功: 进程退出代码 0.

```




JSON转为内部结构体
----------------------

那么如何把json格式的内容转为golang的内部结构体呢？这需要利用`json.Unmarshal`

```go
package main

import (
	"encoding/json"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
)

type Article struct {
	ArticleId   int
	BlogId      int
	UserName    string
	Title       string
	Description string
	PostTime    string
	UpdateTime  string
	ViewCount    int
	CommentCount int
	Categories  int
	ColumnAlias bool
	Url         string
}
type Articles struct {
	Page  int `json:"page,int"`
	Count int
	Size  int
	List  []*Article
}

func main() {
	resp, err := http.Get("http://api.csdn.net/blog/getnewarticlelist?client_id=????&page=0&size=20")
	if err != nil {
		log.Fatal(err)
	}
	str, err := ioutil.ReadAll(resp.Body)
	resp.Body.Close()
	if err != nil {
		log.Fatal(err)
	}

	var contents Articles
	if err := json.Unmarshal(str, &contents); err != nil {
		log.Fatal("Json unmarshing failed: ", err)
	}

	for k, v := range contents.List {
		fmt.Printf("%v. %v\n", k, v.Title)
	}
}

```

最后，我们完成了用csdn的[开放API](http://open.csdn.net/wiki/apis)实现读取最新文章的需求。

```
0. 1048. Find Coins (25)解题报告
1. 1049. Counting Ones (30)解题报告
2. java实习第二天
3. 1060. 爱丁顿数(25)
4. 剑指offer-数组中出现次数超过一半的数字
5. jsonp跨域访问案例
6. framework jar包MAKEFILE示例
7. Unity3d 乱序之惑
8. CodeForces 445C DZY Loves Physics
9. LeetCode #424: Longest Repeating Character Replacement
10. 面板显示private变量用标签[SerializeField]
11. PostgreSQL问题解决--连接数过多
12. 多维高斯分布及多维条件高斯分布
13. 王朝  都要学C
14. CharacterController.Move 实现角色移动
15. bootargs
16. 不用输入法输自己的名字!!!!
17. mybatis mbg自动生成的selectByExample按条件查询不出来值。
18. 系统乔迁留念贴
19. java中Proxy(代理与动态代理)
```


 
