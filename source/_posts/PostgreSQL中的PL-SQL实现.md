---
title: PostgreSQL中的PL/SQL实现
comments: true
date: 2017-01-13 01:15:30
tags: PostgreSQL
---


## 基本流程
1. 函数创建
1.1 create function的语法解析
1.2 创建plsql函数 ProcessUtility->CreateFunction->CreateProcedure
2. 调用plsql函数 
2.1 编译
2.2 执行


## 函数创建

### create function的语法解析
创建plsql存储过程的基本语法：
```c
	create [or replace] function <fname>
			[(<type-1> { , <type-n>})]
			returns <type-r>
			as <filename or code in language as appropriate>
			language <lang> [with parameters]
```c
对应到实现gram.y:6770 中有
```c
CreateFunctionStmt:
			CREATE opt_or_replace FUNCTION func_name func_args_with_defaults
			RETURNS func_return createfunc_opt_list opt_definition
				{
					CreateFunctionStmt *n = makeNode(CreateFunctionStmt);
					n->replace = $2;
					n->funcname = $4;
					n->parameters = $5;
					n->returnType = $7;
					n->options = $8;
					n->withClause = $9;
					$$ = (Node *)n;
```c
可以看到是主要解析如下内容：
1. func_name 函数名
2. fuc_args 参数
3. fuc_return 返回值
4. createfunc_opt_list 内容，对于plsql来说

### 插入系统表
解析完了之后调用CreateFunction函数来实现实际的创建，然后调用ProcedureCreate来插入系统表pg_proc中
```c
ObjectAddress
CreateFunction(CreateFunctionStmt *stmt,  //入参就是语法解析返回的结构体
                const char *queryString)
{
...做一些检查和转换
ProcedureCreate() //来完成实际的创建工作，其实就是插入到了系统表pg_proc
}
```c
注意：此时只是把plsql的文本存下来了，并没有编译。编译会在后续进行


## 调用plsql函数
plpgsql语言对应的处理函数，已经在系统初始化时通过脚本加载。
```c
./backend/catalog/postgres.bki:insert ( "plpgsql" t t "plpgsql_call_handler" "plpgsql_inline_handler" "plpgsql_validator" "$libdir/plpgsql" _null_ )
```c
所以，当调用plsql函数就会调用相应的处理程序。相关函数在pl_handler.c文件中。
在该文件中我们看到如下函数：

_PG_init  系统启动加载plpgsql.so动态链接库之后，即调用该函数完成一系列初始化工作
plpgsql_call_handler 编译执行plsql函数
plpgsql_inline_handler 编译执行plsql匿名块
plpgsql_validator 在Create Function时检验函数的有效性

可以看到，当用户调用plsql函数时就会进入plpgsql_call_handler函数中,经过精简的函数如下：

```c

Datum
plpgsql_call_handler(PG_FUNCTION_ARGS)
{
...
	SPI_connect()  // SPI 用于实际执行sql语句

	func = plpgsql_compile(fcinfo, false); // 编译plsql
...
	retval = plpgsql_exec_function(func, fcinfo, NULL); //执行plsql
...
	SPI_finish() // 断开SPI
	return retval;
```c

## 编译


```c
PLpgSQL_function *
plpgsql_compile(FunctionCallInfo fcinfo, bool forValidator)
{
...
先看hash表里面有没缓存，以及缓存是否失效，如果有效则直接返回，不用再编译啦。
否则需要编译，调用do_compile()
}
```c

```c
static PLpgSQL_function *
do_compile(FunctionCallInfo fcinfo,
		   HeapTuple procTup,
		   PLpgSQL_function *function,
		   PLpgSQL_func_hashkey *hashkey,
		   bool forValidator)
{
	plpgsql_scanner_init(proc_source); // 这里的proc_source就是plsql函数文本
	plpgsql_ns_init(); // 初始化name stack
	plpgsql_ns_push(NameStr(procStruct->proname), PLPGSQL_LABEL_BLOCK); //name stack 中的第一个值就是proname
	plpgsql_start_datums(); // 初始化本地变量，默认先分配128个变量空间
	plpgsql_resolve_polymorphic_argtypes(); //解析函数参数
	plpgsql_build_datatype(); // 建立对应的数据类型
	plpgsql_build_variable(); // 分配入参的存储，加入变量空间，加入命名空间
    plpgsql_build_variable("found"... // 建立以后要用到的found变量
 	parse_rc = plpgsql_yyparse(); // 词法、语法解析
 	if (plpgsql_DumpExecTree)
		plpgsql_dumptree(function);  // 调试模式:打出编译树
	plpgsql_HashTableInsert(function, hashkey); // 加到编译缓存里面

}
```c
ps：调试模式很有用，可以通过下面方式打开：

```c
CREATE FUNCTION less_than(a text, b text) RETURNS boolean AS $$
# option dump
BEGIN
RETURN a < b;
END;
$$ LANGUAGE plpgsql;
```c
在函数体里面加上# option dump即可。执行会显示出来
```c
STATEMENT:  select less_than (1 ,2);

Execution tree of successfully compiled PL/pgSQL function less_than(text,text):

Function's data area:
    entry 0: VAR $1               type text (typoid 25) atttypmod -1
    entry 1: VAR $2               type text (typoid 25) atttypmod -1
    entry 2: VAR found            type bool (typoid 16) atttypmod -1

Function's statements:
  3:BLOCK <<*unnamed*>>
  4:  RETURN 'SELECT a < b'
    END -- *unnamed*

End of execution tree of function less_than(text,text)

```c
### 语法解析
postgreSQL 9.5的用户文档
https://www.postgresql.org/docs/9.5/static/plpgsql.html
可以对照用法来看实现
一个plsql形如
```c
[ <<label>> ]         -- [标签，可选]
[ DECLARE             -- [DELARE开头
    declarations ]    -- 中间是变量的声明，这部分是可选的]
BEGIN                 -- BEGIN
    statements        -- 各语句
END [ label ];        -- END [标签，可选]
```c
对应到实现
```c
pl_block		: decl_sect K_BEGIN proc_sect exception_sect K_END opt_label
					{
						PLpgSQL_stmt_block *new;

						new = palloc0(sizeof(PLpgSQL_stmt_block));
...各种成员填充
						check_labels($1.label, $6, @6);
						plpgsql_ns_pop();

						$$ = (PLpgSQL_stmt *)new;
					}
				;
```c
可以看到是按如下顺序解析的
1. decl_sec 声明部分   --重点
2. prc_sec  语句部分   --重点
3. exception 异常部分
4. opt_label 标签部分
解析完成之后PLpgSQL_stmt_block强制转为一个PLpgSQL_stmt类型的结构返回，可以认为一个declare...begin...end对应一个PLpgSQL_stmt结构。

### 声明部分解析
用户声明一个变量语法：
```c
name [ CONSTANT ] type [ COLLATE collation_name ] [ NOT NULL ] [ { DEFAULT | := | = } expression ];
```c
例子：
```c
quantity integer DEFAULT 32;
url varchar := 'http://mysite.com';
user_id CONSTANT integer := 10;
```c

先介绍下面两个全局变量用来存放plsql中的变量
```c
PLpgSQL_datum **plpgsql_Datums;  //变量数组
int			plpgsql_nDatums;     //数组长度
```c
在pl_gram.y：478解析变量，并加入全局变量数组
```c

decl_statement	: decl_varname decl_const decl_datatype decl_collate decl_notnull decl_defval
		{
			PLpgSQL_variable	*var;
...
			var = plpgsql_build_variable($1.name, $1.lineno,
													 $3, true);
...
```c

对于解析的变量，加到变量数组里面
```c
PLpgSQL_variable *
plpgsql_build_variable(const char *refname, int lineno, PLpgSQL_type *dtype,
					   bool add2namespace)
{
	PLpgSQL_variable *result;

	switch (dtype->ttype)
	{
		case PLPGSQL_TTYPE_SCALAR:
			{
				/* Ordinary scalar datatype */
				PLpgSQL_var *var;
...
				plpgsql_adddatum((PLpgSQL_datum *) var);
				if (add2namespace)
					plpgsql_ns_additem(PLPGSQL_NSTYPE_VAR,
									   var->dno,
									   refname);
				result = (PLpgSQL_variable *) var;
...
```c
变量解析重点关注
decl_varname    -- 变量名解析
decl_datatype   -- 变量类型解析



### 语句部分解析
可以看到如下，是按每个语句规则匹配，不再详细展开了。

```c
proc_stmt		: pl_block ';'
						{ $$ = $1; }
				| stmt_assign
						{ $$ = $1; }
				| stmt_if
						{ $$ = $1; }
				| stmt_case
						{ $$ = $1; }
				| stmt_loop
						{ $$ = $1; }
				| stmt_while
						{ $$ = $1; }
				| stmt_for
						{ $$ = $1; }
				| stmt_foreach_a
						{ $$ = $1; }
				| stmt_exit
						{ $$ = $1; }
				| stmt_return
						{ $$ = $1; }
				| stmt_raise
						{ $$ = $1; }
				| stmt_assert
						{ $$ = $1; }
				| stmt_execsql
						{ $$ = $1; }
				| stmt_dynexecute
						{ $$ = $1; }
				| stmt_perform
						{ $$ = $1; }
				| stmt_getdiag
						{ $$ = $1; }
				| stmt_open
						{ $$ = $1; }
				| stmt_fetch
						{ $$ = $1; }
				| stmt_move
						{ $$ = $1; }
				| stmt_close
						{ $$ = $1; }
				| stmt_null
						{ $$ = $1; }
				;
```c

## 执行

编译完了之后我们回到plpgsql_call_handler看到接下来要进入这个函数 plpgsql_exec_function来实际执行编译好的语句。

```c
Datum
plpgsql_exec_function(PLpgSQL_function *func, FunctionCallInfo fcinfo,
					  EState *simple_eval_estate)
{

	plpgsql_estate_setup();//填充PLpgSQL_execstate estat结构体，该结构体里面保存了执行状态的基本信息。
	for (i = 0; i < estate.ndatums; i++)
		estate.datums[i] = copy_plpgsql_datum(func->datums[i]); //注意：这里把func里面的本地变量挨个拷贝到了estate  里面
	...
	把入参的值赋给对应的本地变量
    ...
    rc = exec_stmt_block(&estate, func->action); //实际执行阶段
    
    // 对返回值进行处理

```c

接下来看看 exec_stmt_block 这个函数

```c
static int
exec_stmt_block(PLpgSQL_execstate *estate, PLpgSQL_stmt_block *block)
{

    // 先对本地变量进行初始化
    rc = exec_stmts(estate, block->body); //执行语句
}

```c


```c
static int
exec_stmts(PLpgSQL_execstate *estate, List *stmts)
{
	ListCell   *s;

	foreach(s, stmts)  // 遍历每个语句
	{
		PLpgSQL_stmt *stmt = (PLpgSQL_stmt *) lfirst(s);
		int			rc = exec_stmt(estate, stmt);   //执行一个单独的语句

		if (rc != PLPGSQL_RC_OK)
			return rc;
	}

	return PLPGSQL_RC_OK;
}

```c
exec_stmt 针对每种类型的语句来实际执行，可以看下列函数的源码，不再赘述。

```c

static int
exec_stmt(PLpgSQL_execstate *estate, PLpgSQL_stmt *stmt)
{
	PLpgSQL_stmt *save_estmt;
	int			rc = -1;

	save_estmt = estate->err_stmt;
	estate->err_stmt = stmt;

	/* Let the plugin know that we are about to execute this statement */
	if (*plpgsql_plugin_ptr && (*plpgsql_plugin_ptr)->stmt_beg)
		((*plpgsql_plugin_ptr)->stmt_beg) (estate, stmt);

	CHECK_FOR_INTERRUPTS();

	switch ((enum PLpgSQL_stmt_types) stmt->cmd_type)
	{
		case PLPGSQL_STMT_BLOCK:
			rc = exec_stmt_block(estate, (PLpgSQL_stmt_block *) stmt);
			break;

		case PLPGSQL_STMT_ASSIGN:
			rc = exec_stmt_assign(estate, (PLpgSQL_stmt_assign *) stmt);
			break;

		case PLPGSQL_STMT_PERFORM:
			rc = exec_stmt_perform(estate, (PLpgSQL_stmt_perform *) stmt);
			break;

		case PLPGSQL_STMT_GETDIAG:
			rc = exec_stmt_getdiag(estate, (PLpgSQL_stmt_getdiag *) stmt);
			break;

		case PLPGSQL_STMT_IF:
			rc = exec_stmt_if(estate, (PLpgSQL_stmt_if *) stmt);
			break;

		case PLPGSQL_STMT_CASE:
			rc = exec_stmt_case(estate, (PLpgSQL_stmt_case *) stmt);
			break;

		case PLPGSQL_STMT_LOOP:
			rc = exec_stmt_loop(estate, (PLpgSQL_stmt_loop *) stmt);
			break;

		case PLPGSQL_STMT_WHILE:
			rc = exec_stmt_while(estate, (PLpgSQL_stmt_while *) stmt);
			break;

		case PLPGSQL_STMT_FORI:
			rc = exec_stmt_fori(estate, (PLpgSQL_stmt_fori *) stmt);
			break;

		case PLPGSQL_STMT_FORS:
			rc = exec_stmt_fors(estate, (PLpgSQL_stmt_fors *) stmt);
			break;

		case PLPGSQL_STMT_FORC:
			rc = exec_stmt_forc(estate, (PLpgSQL_stmt_forc *) stmt);
			break;

		case PLPGSQL_STMT_FOREACH_A:
			rc = exec_stmt_foreach_a(estate, (PLpgSQL_stmt_foreach_a *) stmt);
			break;

		case PLPGSQL_STMT_EXIT:
			rc = exec_stmt_exit(estate, (PLpgSQL_stmt_exit *) stmt);
			break;

		case PLPGSQL_STMT_RETURN:
			rc = exec_stmt_return(estate, (PLpgSQL_stmt_return *) stmt);
			break;

		case PLPGSQL_STMT_RETURN_NEXT:
			rc = exec_stmt_return_next(estate, (PLpgSQL_stmt_return_next *) stmt);
			break;

		case PLPGSQL_STMT_RETURN_QUERY:
			rc = exec_stmt_return_query(estate, (PLpgSQL_stmt_return_query *) stmt);
			break;

		case PLPGSQL_STMT_RAISE:
			rc = exec_stmt_raise(estate, (PLpgSQL_stmt_raise *) stmt);
			break;

		case PLPGSQL_STMT_ASSERT:
			rc = exec_stmt_assert(estate, (PLpgSQL_stmt_assert *) stmt);
			break;

		case PLPGSQL_STMT_EXECSQL:
			rc = exec_stmt_execsql(estate, (PLpgSQL_stmt_execsql *) stmt);
			break;

		case PLPGSQL_STMT_DYNEXECUTE:
			rc = exec_stmt_dynexecute(estate, (PLpgSQL_stmt_dynexecute *) stmt);
			break;

		case PLPGSQL_STMT_DYNFORS:
			rc = exec_stmt_dynfors(estate, (PLpgSQL_stmt_dynfors *) stmt);
			break;

		case PLPGSQL_STMT_OPEN:
			rc = exec_stmt_open(estate, (PLpgSQL_stmt_open *) stmt);
			break;

		case PLPGSQL_STMT_FETCH:
			rc = exec_stmt_fetch(estate, (PLpgSQL_stmt_fetch *) stmt);
			break;

		case PLPGSQL_STMT_CLOSE:
			rc = exec_stmt_close(estate, (PLpgSQL_stmt_close *) stmt);
			break;

		default:
			estate->err_stmt = save_estmt;
			elog(ERROR, "unrecognized cmdtype: %d", stmt->cmd_type);
	}

	/* Let the plugin know that we have finished executing this statement */
	if (*plpgsql_plugin_ptr && (*plpgsql_plugin_ptr)->stmt_end)
		((*plpgsql_plugin_ptr)->stmt_end) (estate, stmt);

	estate->err_stmt = save_estmt;

	return rc;
}
```c


如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
