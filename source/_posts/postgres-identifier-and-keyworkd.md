---
title: postgres identifier and keyword
comments: true
date: 2017-05-10 10:17:58
tags: [数据库, PostgreSQL, PostgreSQL内核, 语言层]
categories: 技术分享
---


identifier
-----------
identifier is defined in scan.l just like below

<center>![image_1bfnvt1dsi911hs41pml1qmm1sf2m.png-21.2kB][1]</center>

it is a string start with `letter` or `-` and might following with numbers.

keyword
--------
keywords is defined in `kwlist.h` just like below

<center>![image_1bfo0372n16ohq9165i18k518sl13.png-91.6kB][2]</center>

the name of keyword is the string scanned by the lexer.
the value is defined in the lexer `%token <keyword> ` session and later convert to an enum value.
we have four categories of keyword:

- reserved key workd: keywords that are usable only as ColLabel( select a as col1 ...)
- unreserved keyword : keyword can be used as any type of name.
- column keyword: keywords that can be column, table, etc, names
- function name keyword: keywords that can be function names

relation of identifier and keyword
----------------------------------
in short, keyword is a subset of identifiers. in the lex scanner, when we have an identifier matched, we first check to see if it is a keyword by `ScanKeywordLookup`. the function take a binary search in the keyword list. if a keyword is found the enum value of the keyword is returned to the parser.

if the identifier is not a keyword, we truncate the string in to NAMEDATELEN(which is 64 byte long) and convert into lower case. after we have done the casting, we store the string value of the identifier into `yylval->str` and return `IDENT` to the parser.

<center>![image_1bfnvkmf2kog7pd1ihf29j7ca9.png-99.5kB][3]</center>

more about key word categories
------------------------------
the reason why we have four categories of keyword is to allow keyword to use in more cases. suppose we only have one category keyword, the reserved keyword. the keyword, for example `NUMERIC` might not be used as colnames.

by putting `NUMERIC` into unreserved keyword category we allow it to be used as column names.

<center>![image_1bfo1o0g11nq44chrtqa9f1ane1t.png-34.7kB][4]</center>

however, the `SELECT` is a reserved keyword which cannot be used as column names, else we might have a sql query like this
```
select select from select;
```
this will make the parser confused.

add new key word
-----------------
if you want to make any keyword changes, for example add a new keyword, you can do it like this:
1. add it in `kwlist.h` and fill in the right category
2. add it in `gram.y` `%token` section so a enum value for it can be generated
3. add your keyword into the right place of the rules in `gram.y`
4. complie and test

  [1]: http://static.zybuluo.com/shenyuflying/r9ofw87pnanbqn41qm0zib7e/image_1bfnvt1dsi911hs41pml1qmm1sf2m.png
  [2]: http://static.zybuluo.com/shenyuflying/ikrcc22t1eko062lv1l818dd/image_1bfo0372n16ohq9165i18k518sl13.png
  [3]: http://static.zybuluo.com/shenyuflying/a147jgs1l59oz60obsmlbbjd/image_1bfnvkmf2kog7pd1ihf29j7ca9.png
  [4]: http://static.zybuluo.com/shenyuflying/ywt2xxzhx52c2wra5o189cx3/image_1bfo1o0g11nq44chrtqa9f1ane1t.png

如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
