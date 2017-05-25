---
title: 数据库使用bytea类型存取二进制文件
comments: true
date: 2017-05-25 21:03:14
tags: [数据库, Oracle, OCI]
categories: [技术分享]
---


> 数据库除了存取数字、文本等数据类型之外，还经常用来存取二进制文件类型。下面用OCI接口举例如何用数据库来存储图片。可以存取二进制文件类型的可以用char、text、bytea、lob等数据类型。正常要使用lob类型来存放二进制文件，因为lob类型支持随机访问，但是需要比较复杂的接口调用。这里用postresql的bytea数据类型来存放二进制文件。



表结构
------------------
```sql
create table sample (
a text primary key,            --name of file
b bytea);                      --data of file
```

图片导入到数据库
-------------------
<center>![image_1bfrp85vf1j851v581vo4dt882p9.png-14.4kB][1]</center>

数据库导出到图片
-------------------
<center>![image_1bfrp8lv01oac1n8r1q021g0t5om.png-16.3kB][2]</center>
<center>![image_1bfrp950m1v0710a710i6gd5tlt13.png-82.5kB][3]</center>


程序源码
-------------------
```c
#include "oci.h" //OCI接口头文件
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

OCIEnv *envhp;      //环境句柄
OCIError *errhp;	//错误句柄
OCISvcCtx *svchp;	//上下文句柄
OCIStmt *stmthp = (OCIStmt *) NULL;//语句句柄

sword finish_demo(boolean loggedon, OCIEnv *envhp, OCISvcCtx *svchp,
                   OCIError *errhp, OCIStmt *stmthp);
sword init_handles(OCIEnv **envhp, OCIError **errhp);
sword log_on(OCIEnv *envhp, OCIError *errhp,
       text *uid, text *pwd, text *DbName);
char *file_to_binary_string(char *filename);
void binary_string_to_file(char *filename, char * str);
void store(char * filename);
void fetch(char * filename);



void
report_error(OCIError *errhp);

//初始化各种句柄
sword
init_handles(OCIEnv **envhp, OCIError **errhp)
{
	printf("Environment setup ....\n");

	/* 初始化OCI应用环境 */
	if (OCIInitialize(OCI_DEFAULT, (dvoid *)0,
		(dvoid * (*)(dvoid *, size_t))0,
		(dvoid * (*)(dvoid *, dvoid *, size_t))0,
		(void (*)(dvoid *, dvoid *))0))
	{
		printf("FAILED: OCIInitialize()\n");
		return OCI_ERROR;
	}

	/* 初始化环境句柄 */
	if (OCIEnvInit((OCIEnv **)envhp, (ub4)OCI_DEFAULT,
		(size_t)0, (dvoid **)0))
	{
		printf("FAILED: OCIEnvInit()\n");
		return OCI_ERROR;
	}

	/* 从环境句柄上分配一个错误信息句柄 */
	if (OCIHandleAlloc((dvoid *)*envhp, (dvoid **)errhp,
		(ub4)OCI_HTYPE_ERROR, (size_t)0, (dvoid **)0))
	{
		printf("FAILED: OCIHandleAlloc() on errhp\n");
		return OCI_ERROR;
	}

	return OCI_SUCCESS;
}

//使用给定的用户名和口令登录到指定的数据库服务上
sword
log_on(OCIEnv *envhp, OCIError *errhp,
       text *uid, text *pwd, text *DbName)
{
	printf("Logging on as %s  ....\n", uid);

	/* 连接数据库，在 OCILogon 中分配 Service handle */
	if (OCILogon(envhp, errhp, &svchp, uid, (ub4)strlen((char*)uid),
		pwd, (ub4)strlen((char*)pwd), DbName, (ub4)strlen((char*)DbName)))
	{
		printf("FAILED: OCILogon()\n");
		return OCI_ERROR;
	}
	printf("%s logged on.\n", uid);

	return OCI_SUCCESS;
}

//断开连接，并释放各种句柄
sword
finish_demo(boolean loggedon, OCIEnv *envhp, OCISvcCtx *svchp,
                   OCIError *errhp, OCIStmt *stmthp)
{
	if (stmthp)
		OCIHandleFree((dvoid *)stmthp, (ub4)OCI_HTYPE_STMT);

	printf("logoff ...\n");
	if (loggedon)
		OCILogoff(svchp, errhp);

	printf("Freeing handles ...\n");
	if (svchp)
		OCIHandleFree((dvoid *) svchp, (ub4)OCI_HTYPE_SVCCTX);
	if (errhp)
		OCIHandleFree((dvoid *) errhp, (ub4)OCI_HTYPE_ERROR);
	if (envhp)
		OCIHandleFree((dvoid *) envhp, (ub4)OCI_HTYPE_ENV);

	return OCI_SUCCESS;
}
//打印错误信息
void
report_error(OCIError *errhp)
{
	text  msgbuf[512];
	sb4   errcode = 0;

	OCIErrorGet((dvoid *)errhp, (ub4)1, (text *)NULL, &errcode,
		msgbuf, (ub4)sizeof(msgbuf), (ub4)OCI_HTYPE_ERROR);
	printf("ERROR CODE = %d", errcode);
	printf("%.*s\n", 512, msgbuf);

	return;
}

//on error return NULL
//on success return the binary string
char *file_to_binary_string(char *filename)
{
	FILE *fp = fopen(filename,"r");
	char *buffer = NULL;
	size_t pos = 0; // file read position
	size_t size = 1024; //how much mem allocated
	size_t nread = -1;
	unsigned char ch; // hold byte read from file


	if (fp == NULL)
	{
		printf("error opening %s for reading\n", filename);
		return NULL;
	}

	buffer = (char *)malloc(size);

	// read until end of file
	while( 0 != (nread = fread(&ch,1,1,fp)))
	{
		char tmp[3]={0}; //hold the string form of binary data, i,e: FF

		sprintf(tmp,"%02x",ch);
		buffer[pos++]=tmp[0];
		if (pos==size)
		{
			buffer = (char *)realloc(buffer, size*2);
			size = size*2;
		}
		buffer[pos++]=tmp[1];
		if (pos==size)
		{
			buffer = (char *)realloc(buffer, size*2);
			size = size*2;
		}
	}
	buffer[pos]='\0'; // add tailing zero

	fclose(fp);

	// the buffer should be freed by the caller
	return buffer;
}

void binary_string_to_file(char *filename, char *str)
{
	FILE *fp = fopen(filename,"w");
	unsigned char table['F'+1];
	int len = strlen(str);
	int i;

	table['0']=0;
	table['1']=1;
	table['2']=2;
	table['3']=3;
	table['4']=4;
	table['5']=5;
	table['6']=6;
	table['7']=7;
	table['8']=8;
	table['9']=9;
	table['A']=10;
	table['B']=11;
	table['C']=12;
	table['D']=13;
	table['E']=14;
	table['F']=15;

	if (fp == NULL)
	{
		printf("error opening %s for writing\n", filename);
		return;
	}

	for(i = 0; i < len; i+=2)
	{
		unsigned char bin;
		bin = 16*table[str[i]] + table[str[i+1]];
		fprintf(fp, "%c", bin);
	}

	fclose(fp);
}


//store a file in to database
//should have a table like this:
//
//create table sample ( a text primary key, -- filename
//                      b bytea );          -- data
//
void store(char * filename)
{
	char *str = NULL;
	char *sql = NULL;

	str = file_to_binary_string(filename);

	if(str==NULL)
	{

	}

	sql = (char *)malloc(strlen(str)+100); // more space for insert ...
	memset(sql,0,strlen(str)+100);         // more space for insert ...

	sprintf(sql, "insert into sample values('%s', '%s');", filename, str);

	if (OCIStmtPrepare(stmthp, errhp, (unsigned char *)sql, (ub4)strlen((char *)sql),
		(ub4)OCI_NTV_SYNTAX, (ub4)OCI_DEFAULT))
	{
		printf("FAILED: OCIStmtPrepare() insert\n");
		finish_demo(TRUE, envhp, svchp, errhp, stmthp);
	}

	//非查询类型语句，iters参数必需>=1
	if (OCIStmtExecute(svchp, stmthp, errhp, (ub4)1, (ub4)0,
		(CONST OCISnapshot*)0, (OCISnapshot*)0,
		(ub4)OCI_DEFAULT))
	{
		printf("FAILED: OCIStmtExecute() insert\n");
		finish_demo(TRUE, envhp, svchp, errhp, stmthp);
	}


	free(str);
	free(sql);

	printf("ok!\n");
}

void fetch(char * filename)
{
	char sql[1024] = {0}; // should be enough
	OCIParam     *mypard = (OCIParam *) 0;
	ub2          dtype;
	text         *col_name;
	ub4          counter, col_name_len, char_semantics;
	ub2          col_width;
	ub4 dsize = 100*1024*1024; //100M should be enough
	char *str = NULL;
	OCIDefine *bndhp;
	ub4 stmrow = 0;

	str=(char *)malloc(dsize+1);
	memset(str,0,dsize+1);

	sprintf(sql, "select b from sample where a = '%s'", filename);

	//准备查询语句
	if (OCIStmtPrepare(stmthp, errhp, sql, (ub4)strlen((char *)sql),
		(ub4)OCI_NTV_SYNTAX, (ub4)OCI_DEFAULT))
	{
		printf("FAILED: OCIStmtPrepare() select\n");
		report_error(errhp);
		return ;
	}

	//执行查询语句
	if (OCIStmtExecute(svchp, stmthp, errhp, (ub4)0, (ub4)0,
		(CONST OCISnapshot*)0, (OCISnapshot*)0,
		(ub4)OCI_DEFAULT))
	{
		printf("FAILED: OCIStmtExecute() select\n");
		report_error(errhp);
		return ;
	}


	// bind
	if (OCIDefineByPos(stmthp, &bndhp, errhp, 1,
		(dvoid *)str, (sb4)dsize, (ub2)SQLT_STR,
		(dvoid *)0, (ub2 *)0, (ub2 *)0, (ub4)OCI_DEFAULT))
	{
		printf("FAILED: OCIDefineByPos()\n");
		report_error(errhp);
		return ;
	}

	if (OCIStmtFetch(stmthp, errhp, 1, OCI_FETCH_NEXT, 0) == OCI_ERROR)
	{

		printf("FAILED: OCIStmtFetch() select\n");
		report_error(errhp);
		return ;

	}

	OCIAttrGet(stmthp, OCI_HTYPE_STMT, &stmrow, 0, OCI_ATTR_ROW_COUNT, errhp);

	if(stmrow == 0)//没有新行被获取，那么跳出
	{

		printf("no rows fetched\n");
		return ;
	}

	printf("提取到了%d行记录\n", stmrow);


	binary_string_to_file(filename, str);

	free(str);
	printf("ok!\n");
}


//主程序入口
int
main()
{
	text *DbName =(text*)"Kingbase"; //配置文件 sys_service.conf 中配置的数据源名
	text *username = (text *)"SYSTEM"; //用户名
	text *password = (text *)"MANAGER"; //口令
	int logged_on = FALSE;
	int choice = 0;
	char filename[1024] = {0};

	//初始化各种句柄
	if (init_handles(&envhp, &errhp))
	{
		printf("FAILED: init_handles()\n");
		return finish_demo(logged_on, envhp, svchp, errhp, stmthp);
	}

	//登录到数据库服务
	if (log_on(envhp, errhp, username, password, DbName))
	{
		printf("FAILED: log_on()\n");
		return finish_demo(logged_on, envhp, svchp, errhp, stmthp);
	}
	logged_on = TRUE;
	if (OCIHandleAlloc((dvoid *)envhp, (dvoid **)&stmthp,
		(ub4)OCI_HTYPE_STMT, (CONST size_t)0, (dvoid **) 0))
	{
		printf("FAILED: alloc statement handle\n");
		return finish_demo(logged_on, envhp, svchp, errhp, stmthp);
	}

	printf("\n1)store a file."
	       "\n2)fetch a file."
	       "\nyour choice : ");
	scanf("%d", &choice);

	switch(choice)
	{
		case 1:
			printf("FileName : ");
			scanf("%s", filename);
			store(filename);
		break;
		case 2:
			printf("FileName : ");
			scanf("%s", filename);
			fetch(filename);
		break;
		default:
			printf("bad choice!\n");
		break;
	}

	//登出数据库服务，并清理各种句柄
	return finish_demo(logged_on, envhp, svchp, errhp, stmthp);
}

```


  [1]: http://static.zybuluo.com/shenyuflying/8rpikj01xb3qbp621akbb9lo/image_1bfrp85vf1j851v581vo4dt882p9.png
  [2]: http://static.zybuluo.com/shenyuflying/h9fs74rea0iba2cnvcngyygo/image_1bfrp8lv01oac1n8r1q021g0t5om.png
  [3]: http://static.zybuluo.com/shenyuflying/jm5w06dsy85czxswribdbjk7/image_1bfrp950m1v0710a710i6gd5tlt13.png



如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
