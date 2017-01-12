---
title: PostgreSQL逻辑备份还原
comments: true
date: 2017-01-12 09:27:58
tags: PostgreSQL
---

主要内容
=======

## pg_dump
1. pg_dump 参数解析
2. pg_dump 基本流程
3. pg_dump 中的事务

## pg_restore
4. pg_restore 参数解析
5. pg_restore 基本流程
6. pg_restore 中的事务

## 补充内容
7. 时间紧迫的情况下如何尽快会恢复
8. 并行备份/恢复
9. 其他的备份方式
10. 思考
11. FAQ

什么是逻辑(Logical)备份还原
===========

## 逻辑备份
逻辑备份是在联机状态下通过读取数据库中用户创建的数据对象信息，通过SQL方式，从数据库中抽取出各个对象的定义和数据信息到备份文件中。

## 逻辑还原
逻辑还原是通过执行备份文件中SQL语句，重建数据库对象。


pg_dump 参数解析
================

## help
```shell
$ ./pg_dump --help
```

```
pg_dump dumps a database as a text file or to other formats.

Usage:
  pg_dump [OPTION]... [DBNAME]

General options:
  -f, --file=FILENAME          output file or directory name
  -F, --format=c|d|t|p         output file format (custom, directory, tar,
                               plain text (default))
  -j, --jobs=NUM               use this many parallel jobs to dump
  -v, --verbose                verbose mode
  -V, --version                output version information, then exit
  -Z, --compress=0-9           compression level for compressed formats
  --lock-wait-timeout=TIMEOUT  fail after waiting TIMEOUT for a table lock
  -?, --help                   show this help, then exit

Options controlling the output content:
  -a, --data-only              dump only the data, not the schema
  -b, --blobs                  include large objects in dump
  -B, --no-blobs               exclude large objects in dump
  -c, --clean                  clean (drop) database objects before recreating
  -C, --create                 include commands to create database in dump
  -E, --encoding=ENCODING      dump the data in encoding ENCODING
  -n, --schema=SCHEMA          dump the named schema(s) only
  -N, --exclude-schema=SCHEMA  do NOT dump the named schema(s)
  -o, --oids                   include OIDs in dump
  -O, --no-owner               skip restoration of object ownership in
                               plain-text format
  -s, --schema-only            dump only the schema, no data
  -S, --superuser=NAME         superuser user name to use in plain-text format
  -t, --table=TABLE            dump the named table(s) only
  -T, --exclude-table=TABLE    do NOT dump the named table(s)
  -x, --no-privileges          do not dump privileges (grant/revoke)
  --binary-upgrade             for use by upgrade utilities only
  --column-inserts             dump data as INSERT commands with column names
  --disable-dollar-quoting     disable dollar quoting, use SQL standard quoting
  --disable-triggers           disable triggers during data-only restore
  --enable-row-security        enable row security (dump only content user has
                               access to)
  --exclude-table-data=TABLE   do NOT dump data for the named table(s)
  --if-exists                  use IF EXISTS when dropping objects
  --inserts                    dump data as INSERT commands, rather than COPY
  --no-security-labels         do not dump security label assignments
  --no-synchronized-snapshots  do not use synchronized snapshots in parallel jobs
  --no-tablespaces             do not dump tablespace assignments
  --no-unlogged-table-data     do not dump unlogged table data
  --quote-all-identifiers      quote all identifiers, even if not key words
  --section=SECTION            dump named section (pre-data, data, or post-data)
  --serializable-deferrable    wait until the dump can run without anomalies
  --snapshot=SNAPSHOT          use given snapshot for the dump
  --strict-names               require table and/or schema include patterns to
                               match at least one entity each
  --use-set-session-authorization
                               use SET SESSION AUTHORIZATION commands instead of
                               ALTER OWNER commands to set ownership

Connection options:
  -d, --dbname=DBNAME      database to dump
  -h, --host=HOSTNAME      database server host or socket directory
  -p, --port=PORT          database server port number
  -U, --username=NAME      connect as specified database user
  -w, --no-password        never prompt for password
  -W, --password           force password prompt (should happen automatically)
  --role=ROLENAME          do SET ROLE before dump

If no database name is supplied, then the PGDATABASE environment
variable value is used.

Report bugs to <pgsql-bugs@postgresql.org>.


```

## 备份为sql形式

```
默认是备份为sql的形式，输出为stdout，可以把输出指向另一个postgres实例来实现数据库迁移。这样实现类似远程同步的功能。
                              pg_dump
postgres1  ------------------------------------------> postgres2
$ ./pg_dump -hlocalhost -p5432  postgres
```

```sql
$ ./pg_dump -hlocalhost -p5432  postgres
--
-- PostgreSQL database dump
--

-- Dumped from database version 10devel
-- Dumped by pg_dump version 10devel

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SET check_function_bodies = false;
SET client_min_messages = warning;
SET row_security = off;

--
-- Name: postgres; Type: COMMENT; Schema: -; Owner: yshen
--

COMMENT ON DATABASE postgres IS 'default administrative connection database';


--
-- Name: plpgsql; Type: EXTENSION; Schema: -; Owner: 
--

CREATE EXTENSION IF NOT EXISTS plpgsql WITH SCHEMA pg_catalog;


--
-- Name: EXTENSION plpgsql; Type: COMMENT; Schema: -; Owner: 
--

COMMENT ON EXTENSION plpgsql IS 'PL/pgSQL procedural language';


SET search_path = public, pg_catalog;

SET default_tablespace = '';

SET default_with_oids = false;

--
-- Name: tb1; Type: TABLE; Schema: public; Owner: yshen
--

CREATE TABLE tb1 (
    a integer
);


ALTER TABLE tb1 OWNER TO yshen;

--
-- Name: tb2; Type: TABLE; Schema: public; Owner: yshen
--

CREATE TABLE tb2 (
    a integer
);


ALTER TABLE tb2 OWNER TO yshen;

--
-- Data for Name: tb1; Type: TABLE DATA; Schema: public; Owner: yshen
--

COPY tb1 (a) FROM stdin;
1
2
\.


--
-- Data for Name: tb2; Type: TABLE DATA; Schema: public; Owner: yshen
--

COPY tb2 (a) FROM stdin;
1
2
\.


--
-- PostgreSQL database dump complete
--


```

## 备份为二进制形式
```
$ ./pg_dump -Fc -f dump.dmp  postgres
```
备份为二进制，那么还原的时候就可以很方便的指定还原选项，比如说只还原某个表。
另外也可以把二进制格式用pg_restore导出为sql格式。
```
              pg_dump                    pg_restore
postgres1--------------------> custom-------------------->postgres2
```

pg_dump基本流程
============

## 基本流程
1. 收集备份对象信息（查询系统表）
2. 拓扑排序
3. 输出（拼接定义，生成toc_entry）

## 基本流程
```
根据备份类型（二进制-F c 即custom|文本-F p 即plain text）来初始化ArchiveHandle结果保存在g_fout全局变量里
调用ConnectDatabase连接数据库，把结果保存在g_conn全局变量中
调用setup_connection来设置很多的连接参数，比如编码等等
BEGIN
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE
	获取备份对象的属性 getXXX相关函数
	对表加锁: LOCK TABLE "PUBLIC"."TB1" IN ACCESS SHARE MODE （备份过程中不能drop）
    依赖关系拓扑排序
    获取对象数据并输出：COPY "PUBLIC"."TB1" ("VALUE")  TO stdout;  dumpXXX相关函数
COMIT
```

## pg_dump的log


```
$ ./pg_dump -hlocalhost -p5432 --verbose  -Fc  -f ./dmp.dmp  postgres
```

```
pg_dump: last built-in OID is 16383
pg_dump: reading extensions
pg_dump: identifying extension members
pg_dump: reading schemas
pg_dump: reading user-defined tables
pg_dump: reading user-defined functions
pg_dump: reading user-defined types
pg_dump: reading procedural languages
pg_dump: reading user-defined aggregate functions
pg_dump: reading user-defined operators
pg_dump: reading user-defined access methods
pg_dump: reading user-defined operator classes
pg_dump: reading user-defined operator families
pg_dump: reading user-defined text search parsers
pg_dump: reading user-defined text search templates
pg_dump: reading user-defined text search dictionaries
pg_dump: reading user-defined text search configurations
pg_dump: reading user-defined foreign-data wrappers
pg_dump: reading user-defined foreign servers
pg_dump: reading default privileges
pg_dump: reading user-defined collations
pg_dump: reading user-defined conversions
pg_dump: reading type casts
pg_dump: reading transforms
pg_dump: reading table inheritance information
pg_dump: reading partition information
pg_dump: reading event triggers
pg_dump: finding extension tables
pg_dump: finding inheritance relationships
pg_dump: finding partition relationships
pg_dump: reading column info for interesting tables
pg_dump: finding the columns and types of table "public.tb1"
pg_dump: finding the columns and types of table "public.tb2"
pg_dump: flagging inherited columns in subtables
pg_dump: reading indexes
pg_dump: reading constraints
pg_dump: reading triggers
pg_dump: reading rewrite rules
pg_dump: reading policies
pg_dump: reading row security enabled for table "public.tb1"
pg_dump: reading policies for table "public.tb1"
pg_dump: reading row security enabled for table "public.tb2"
pg_dump: reading policies for table "public.tb2"
pg_dump: reading partition key information for interesting tables
pg_dump: reading large objects
pg_dump: reading dependency data
pg_dump: saving encoding = UTF8
pg_dump: saving standard_conforming_strings = on
pg_dump: saving database definition
pg_dump: dumping contents of table "public.tb1"
pg_dump: dumping contents of table "public.tb2"

```

## 备份过程中数据库端的log


```sql
--进行了一些基本的设置
LOG:  statement: SELECT pg_catalog.pg_is_in_recovery()
LOG:  statement: SET DATESTYLE = ISO
LOG:  statement: SET INTERVALSTYLE = POSTGRES
LOG:  statement: SET extra_float_digits TO 3
LOG:  statement: SET synchronize_seqscans TO off
LOG:  statement: SET statement_timeout = 0
LOG:  statement: SET lock_timeout = 0
LOG:  statement: SET idle_in_transaction_session_timeout = 0
LOG:  statement: SET row_security = off
--开启一个事物
LOG:  statement: BEGIN
LOG:  statement: SET TRANSACTION ISOLATION LEVEL REPEATABLE READ, READ ONLY
LOG:  statement: SET search_path = pg_catalog
--开始获取备份对象信息，对应的是pg_dump中的getXXX系列函数
LOG:  statement: SELECT x.tableoid, x.oid, x.extname, n.nspname, x.extrelocatable, x.extversion, x.extconfig, x.extcondition FROM pg_extension x JOIN pg_namespace n ON n.oid = x.extnamespace
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT classid, objid, refobjid FROM pg_depend WHERE refclassid = 'pg_extension'::regclass AND deptype = 'e' ORDER BY 3
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT n.tableoid, n.oid, n.nspname, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = nspowner) AS rolname, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(n.nspacl,pg_catalog.acldefault('n',n.nspowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('n',n.nspowner)))) as foo) as nspacl, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('n',n.nspowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(n.nspacl,pg_catalog.acldefault('n',n.nspowner)))) as foo) as rnspacl, NULL as initnspacl, NULL as initrnspacl FROM pg_namespace n LEFT JOIN pg_init_privs pip ON (n.oid = pip.objoid AND pip.classoid = 'pg_namespace'::regclass AND pip.objsubid = 0) 
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT c.tableoid, c.oid, c.relname, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(c.relacl,pg_catalog.acldefault(CASE WHEN c.relkind = 'S' THEN 's' ELSE 'r' END::"char",c.relowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault(CASE WHEN c.relkind = 'S' THEN 's' ELSE 'r' END::"char",c.relowner)))) as foo) AS relacl, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault(CASE WHEN c.relkind = 'S' THEN 's' ELSE 'r' END::"char",c.relowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(c.relacl,pg_catalog.acldefault(CASE WHEN c.relkind = 'S' THEN 's' ELSE 'r' END::"char",c.relowner)))) as foo) as rrelacl, NULL AS initrelacl, NULL as initrrelacl, c.relkind, c.relnamespace, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = c.relowner) AS rolname, c.relchecks, c.relhastriggers, c.relhasindex, c.relhasrules, c.relhasoids, c.relrowsecurity, c.relforcerowsecurity, c.relfrozenxid, c.relminmxid, tc.oid AS toid, tc.relfrozenxid AS tfrozenxid, tc.relminmxid AS tminmxid, c.relpersistence, c.relispopulated, c.relreplident, c.relpages, CASE WHEN c.reloftype <> 0 THEN c.reloftype::pg_catalog.regtype ELSE NULL END AS reloftype, d.refobjid AS owning_tab, d.refobjsubid AS owning_col, (SELECT spcname FROM pg_tablespace t WHERE t.oid = c.reltablespace) AS reltablespace, array_remove(array_remove(c.reloptions,'check_option=local'),'check_option=cascaded') AS reloptions, CASE WHEN 'check_option=local' = ANY (c.reloptions) THEN 'LOCAL'::text WHEN 'check_option=cascaded' = ANY (c.reloptions) THEN 'CASCADED'::text ELSE NULL END AS checkoption, tc.reloptions AS toast_reloptions, EXISTS (SELECT 1 FROM pg_attribute at LEFT JOIN pg_init_privs pip ON (c.oid = pip.objoid AND pip.classoid = 'pg_class'::regclass AND pip.objsubid = at.attnum)WHERE at.attrelid = c.oid AND ((SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(at.attacl,pg_catalog.acldefault('c',c.relowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('c',c.relowner)))) as foo) IS NOT NULL OR (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('c',c.relowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(at.attacl,pg_catalog.acldefault('c',c.relowner)))) as foo) IS NOT NULL OR NULL IS NOT NULL OR NULL IS NOT NULL))AS changed_acl FROM pg_class c LEFT JOIN pg_depend d ON (c.relkind = 'S' AND d.classid = c.tableoid AND d.objid = c.oid AND d.objsubid = 0 AND d.refclassid = c.tableoid AND d.deptype = 'a') LEFT JOIN pg_class tc ON (c.reltoastrelid = tc.oid) LEFT JOIN pg_init_privs pip ON (c.oid = pip.objoid AND pip.classoid = 'pg_class'::regclass AND pip.objsubid = 0) WHERE c.relkind in ('r', 'S', 'v', 'c', 'm', 'f', 'P') ORDER BY c.oid
--对表加ACESS SHARE锁，防止在备份的时候drop，alter等操作
LOG:  statement: LOCK TABLE public.tb1 IN ACCESS SHARE MODE
LOG:  statement: LOCK TABLE public.tb2 IN ACCESS SHARE MODE
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT p.tableoid, p.oid, p.proname, p.prolang, p.pronargs, p.proargtypes, p.prorettype, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(p.proacl,pg_catalog.acldefault('f',p.proowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('f',p.proowner)))) as foo) AS proacl, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('f',p.proowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(p.proacl,pg_catalog.acldefault('f',p.proowner)))) as foo) AS rproacl, NULL AS initproacl, NULL AS initrproacl, p.pronamespace, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = p.proowner) AS rolname FROM pg_proc p LEFT JOIN pg_init_privs pip ON (p.oid = pip.objoid AND pip.classoid = 'pg_proc'::regclass AND pip.objsubid = 0) WHERE NOT proisagg
depend WHERE classid = 'pg_proc'::regclass AND objid = p.oid AND deptype = 'i')
pg_namespace WHERE nspname = 'pg_catalog')form D )initprivs)
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT t.tableoid, t.oid, t.typname, t.typnamespace, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(t.typacl,pg_catalog.acldefault('T',t.typowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('T',t.typowner)))) as foo) AS typacl, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('T',t.typowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(t.typacl,pg_catalog.acldefault('T',t.typowner)))) as foo) AS rtypacl, NULL AS inittypacl, NULL AS initrtypacl, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = t.typowner) AS rolname, t.typelem, t.typrelid, CASE WHEN t.typrelid = 0 THEN ' '::"char" ELSE (SELECT relkind FROM pg_class WHERE oid = t.typrelid) END AS typrelkind, t.typtype, t.typisdefined, t.typname[0] = '_' AND t.typelem != 0 AND (SELECT typarray FROM pg_type te WHERE oid = t.typelem) = t.oid AS isarray FROM pg_type t LEFT JOIN pg_init_privs pip ON (t.oid = pip.objoid AND pip.classoid = 'pg_type'::regclass AND pip.objsubid = 0) 
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT l.tableoid, l.oid, l.lanname, l.lanpltrusted, l.lanplcallfoid, l.laninline, l.lanvalidator, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(l.lanacl,pg_catalog.acldefault('l',l.lanowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('l',l.lanowner)))) as foo) AS lanacl, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('l',l.lanowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(l.lanacl,pg_catalog.acldefault('l',l.lanowner)))) as foo) AS rlanacl, NULL AS initlanacl, NULL AS initrlanacl, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = l.lanowner) AS lanowner FROM pg_language l LEFT JOIN pg_init_privs pip ON (l.oid = pip.objoid AND pip.classoid = 'pg_type'::regclass AND pip.objsubid = 0) WHERE l.lanispl ORDER BY l.oid
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT p.tableoid, p.oid, p.proname AS aggname, p.pronamespace AS aggnamespace, p.pronargs, p.proargtypes, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = p.proowner) AS rolname, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(p.proacl,pg_catalog.acldefault('f',p.proowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('f',p.proowner)))) as foo) AS aggacl, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('f',p.proowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(p.proacl,pg_catalog.acldefault('f',p.proowner)))) as foo) AS raggacl, NULL AS initaggacl, NULL AS initraggacl FROM pg_proc p LEFT JOIN pg_init_privs pip ON (p.oid = pip.objoid AND pip.classoid = 'pg_proc'::regclass AND pip.objsubid = 0) WHERE p.proisagg AND (p.pronamespace != (SELECT oid FROM pg_namespace WHERE nspname = 'pg_catalog') OR p.proacl IS DISTINCT FROM pip.initprivs)
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT tableoid, oid, oprname, oprnamespace, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = oprowner) AS rolname, oprkind, oprcode::oid AS oprcode FROM pg_operator
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT tableoid, oid, amname, amtype, amhandler::pg_catalog.regproc AS amhandler FROM pg_am
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT tableoid, oid, opcname, opcnamespace, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = opcowner) AS rolname FROM pg_opclass
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT tableoid, oid, opfname, opfnamespace, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = opfowner) AS rolname FROM pg_opfamily
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT tableoid, oid, prsname, prsnamespace, prsstart::oid, prstoken::oid, prsend::oid, prsheadline::oid, prslextype::oid FROM pg_ts_parser
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT tableoid, oid, tmplname, tmplnamespace, tmplinit::oid, tmpllexize::oid FROM pg_ts_template
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT tableoid, oid, dictname, dictnamespace, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = dictowner) AS rolname, dicttemplate, dictinitoption FROM pg_ts_dict
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT tableoid, oid, cfgname, cfgnamespace, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = cfgowner) AS rolname, cfgparser FROM pg_ts_config
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT f.tableoid, f.oid, f.fdwname, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = f.fdwowner) AS rolname, f.fdwhandler::pg_catalog.regproc, f.fdwvalidator::pg_catalog.regproc, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(f.fdwacl,pg_catalog.acldefault('F',f.fdwowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('F',f.fdwowner)))) as foo) AS fdwacl, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('F',f.fdwowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(f.fdwacl,pg_catalog.acldefault('F',f.fdwowner)))) as foo) AS rfdwacl, NULL AS initfdwacl, NULL AS initrfdwacl, array_to_string(ARRAY(SELECT quote_ident(option_name) || ' ' || quote_literal(option_value) FROM pg_options_to_table(f.fdwoptions) ORDER BY option_name), E',
n_data_wrapper f LEFT JOIN pg_init_privs pip ON (f.oid = pip.objoid AND pip.classoid = 'pg_foreign_data_wrapper'::regclass AND pip.objsubid = 0) 
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT f.tableoid, f.oid, f.srvname, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = f.srvowner) AS rolname, f.srvfdw, f.srvtype, f.srvversion, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(f.srvacl,pg_catalog.acldefault('S',f.srvowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('S',f.srvowner)))) as foo) AS srvacl, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('S',f.srvowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(f.srvacl,pg_catalog.acldefault('S',f.srvowner)))) as foo) AS rsrvacl, NULL AS initsrvacl, NULL AS initrsrvacl, array_to_string(ARRAY(SELECT quote_ident(option_name) || ' ' || quote_literal(option_value) FROM pg_options_to_table(f.srvoptions) ORDER BY option_name), E',
n_server f LEFT JOIN pg_init_privs pip ON (f.oid = pip.objoid AND pip.classoid = 'pg_foreign_server'::regclass AND pip.objsubid = 0) 
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT oid, tableoid, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = defaclrole) AS defaclrole, defaclnamespace, defaclobjtype, defaclacl FROM pg_default_acl
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT tableoid, oid, collname, collnamespace, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = collowner) AS rolname FROM pg_collation
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT tableoid, oid, conname, connamespace, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = conowner) AS rolname FROM pg_conversion
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT tableoid, oid, castsource, casttarget, castfunc, castcontext, castmethod FROM pg_cast ORDER BY 3,4
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT tableoid, oid, trftype, trflang, trffromsql::oid, trftosql::oid FROM pg_transform ORDER BY 3,4
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT inhrelid, inhparent FROM pg_inherits WHERE inhparent NOT IN (SELECT oid FROM pg_class WHERE relkind = 'P')
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT inhrelid as partrelid, inhparent AS partparent,		 pg_get_expr(relpartbound, inhrelid) AS partbound FROM pg_class c, pg_inherits WHERE c.oid = inhrelid AND c.relispartition
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT e.tableoid, e.oid, evtname, evtenabled, evtevent, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = evtowner) AS evtowner, array_to_string(array(select quote_literal(x)  from unnest(evttags) as t(x)), ', ') as evttags, e.evtfoid::regproc as evtfname FROM pg_event_trigger e ORDER BY e.oid
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT conrelid, confrelid FROM pg_constraint JOIN pg_depend ON (objid = confrelid) WHERE contype = 'f' AND refclassid = 'pg_extension'::regclass AND classid = 'pg_class'::regclass;
LOG:  statement: SET search_path = public, pg_catalog
LOG:  statement: SELECT a.attnum, a.attname, a.atttypmod, a.attstattarget, a.attstorage, t.typstorage, a.attnotnull, a.atthasdef, a.attisdropped, a.attlen, a.attalign, a.attislocal, pg_catalog.format_type(t.oid,a.atttypmod) AS atttypname, array_to_string(a.attoptions, ', ') AS attoptions, CASE WHEN a.attcollation <> t.typcollation THEN a.attcollation ELSE 0 END AS attcollation, pg_catalog.array_to_string(ARRAY(SELECT pg_catalog.quote_ident(option_name) || ' ' || pg_catalog.quote_literal(option_value) FROM pg_catalog.pg_options_to_table(attfdwoptions) ORDER BY option_name), E',
alog.pg_attribute a LEFT JOIN pg_catalog.pg_type t ON a.atttypid = t.oid WHERE a.attrelid = '16384'::pg_catalog.oid AND a.attnum > 0::pg_catalog.int2 ORDER BY a.attnum
LOG:  statement: SET search_path = public, pg_catalog
LOG:  statement: SELECT a.attnum, a.attname, a.atttypmod, a.attstattarget, a.attstorage, t.typstorage, a.attnotnull, a.atthasdef, a.attisdropped, a.attlen, a.attalign, a.attislocal, pg_catalog.format_type(t.oid,a.atttypmod) AS atttypname, array_to_string(a.attoptions, ', ') AS attoptions, CASE WHEN a.attcollation <> t.typcollation THEN a.attcollation ELSE 0 END AS attcollation, pg_catalog.array_to_string(ARRAY(SELECT pg_catalog.quote_ident(option_name) || ' ' || pg_catalog.quote_literal(option_value) FROM pg_catalog.pg_options_to_table(attfdwoptions) ORDER BY option_name), E',
alog.pg_attribute a LEFT JOIN pg_catalog.pg_type t ON a.atttypid = t.oid WHERE a.attrelid = '16388'::pg_catalog.oid AND a.attnum > 0::pg_catalog.int2 ORDER BY a.attnum
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT tableoid, oid, rulename, ev_class AS ruletable, ev_type, is_instead, ev_enabled FROM pg_rewrite ORDER BY oid
LOG:  statement: SET search_path = public, pg_catalog
LOG:  statement: SELECT oid, tableoid, pol.polname, pol.polcmd, pol.polpermissive, CASE WHEN pol.polroles = '{0}' THEN NULL ELSE    pg_catalog.array_to_string(ARRAY(SELECT pg_catalog.quote_ident(rolname) from pg_catalog.pg_roles WHERE oid = ANY(pol.polroles)), ', ') END AS polroles, pg_catalog.pg_get_expr(pol.polqual, pol.polrelid) AS polqual, pg_catalog.pg_get_expr(pol.polwithcheck, pol.polrelid) AS polwithcheck FROM pg_catalog.pg_policy pol WHERE polrelid = '16384'
LOG:  statement: SET search_path = public, pg_catalog
LOG:  statement: SELECT oid, tableoid, pol.polname, pol.polcmd, pol.polpermissive, CASE WHEN pol.polroles = '{0}' THEN NULL ELSE    pg_catalog.array_to_string(ARRAY(SELECT pg_catalog.quote_ident(rolname) from pg_catalog.pg_roles WHERE oid = ANY(pol.polroles)), ', ') END AS polroles, pg_catalog.pg_get_expr(pol.polqual, pol.polrelid) AS polqual, pg_catalog.pg_get_expr(pol.polwithcheck, pol.polrelid) AS polwithcheck FROM pg_catalog.pg_policy pol WHERE polrelid = '16388'
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: WITH RECURSIVE w AS ( SELECT d1.objid, d2.refobjid, c2.relkind AS refrelkind FROM pg_depend d1 JOIN pg_class c1 ON c1.oid = d1.objid AND c1.relkind = 'm' JOIN pg_rewrite r1 ON r1.ev_class = d1.objid JOIN pg_depend d2 ON d2.classid = 'pg_rewrite'::regclass AND d2.objid = r1.oid AND d2.refobjid <> d1.objid JOIN pg_class c2 ON c2.oid = d2.refobjid AND c2.relkind IN ('m','v') WHERE d1.classid = 'pg_class'::regclass UNION SELECT w.objid, d3.refobjid, c3.relkind FROM w JOIN pg_rewrite r3 ON r3.ev_class = w.refobjid JOIN pg_depend d3 ON d3.classid = 'pg_rewrite'::regclass AND d3.objid = r3.oid AND d3.refobjid <> w.refobjid JOIN pg_class c3 ON c3.oid = d3.refobjid AND c3.relkind IN ('m','v') ) SELECT 'pg_class'::regclass::oid AS classid, objid, refobjid FROM w WHERE refrelkind = 'm'
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT l.oid, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = l.lomowner) AS rolname, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(l.lomacl,pg_catalog.acldefault('L',l.lomowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('L',l.lomowner)))) as foo) AS lomacl, (SELECT pg_catalog.array_agg(acl) FROM (SELECT pg_catalog.unnest(coalesce(pip.initprivs,pg_catalog.acldefault('L',l.lomowner))) AS acl EXCEPT SELECT pg_catalog.unnest(coalesce(l.lomacl,pg_catalog.acldefault('L',l.lomowner)))) as foo) AS rlomacl, NULL AS initlomacl, NULL AS initrlomacl FROM pg_largeobject_metadata l LEFT JOIN pg_init_privs pip ON (l.oid = pip.objoid AND pip.classoid = 'pg_largeobject'::regclass AND pip.objsubid = 0) 
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT classid, objid, refclassid, refobjid, deptype FROM pg_depend WHERE deptype != 'p' AND deptype != 'e' ORDER BY 1,2
LOG:  statement: SET search_path = pg_catalog
LOG:  statement: SELECT tableoid, oid, (SELECT rolname FROM pg_catalog.pg_roles WHERE oid = datdba) AS dba, pg_encoding_to_char(encoding) AS encoding, datcollate, datctype, datfrozenxid, datminmxid, (SELECT spcname FROM pg_tablespace t WHERE t.oid = dattablespace) AS tablespace, shobj_description(oid, 'pg_database') AS description FROM pg_database WHERE datname = 'postgres'
LOG:  statement: SELECT provider, label FROM pg_catalog.pg_shseclabel WHERE classoid = 'pg_database'::pg_catalog.regclass AND objoid = 12289
LOG:  statement: SELECT description, classoid, objoid, objsubid FROM pg_catalog.pg_description ORDER BY classoid, objoid, objsubid
LOG:  statement: SELECT label, provider, classoid, objoid, objsubid FROM pg_catalog.pg_seclabel ORDER BY classoid, objoid, objsubid
LOG:  statement: SET search_path = public, pg_catalog
LOG:  statement: SET search_path = public, pg_catalog
--开始备份数据
LOG:  statement: SET search_path = public, pg_catalog
LOG:  statement: COPY public.tb1 (a) TO stdout;
LOG:  statement: SET search_path = public, pg_catalog
LOG:  statement: COPY public.tb2 (a) TO stdout;

```

## 对应pg_dump中的getXXX系列函数

```
$ cat src/bin/pg_dump/common.c | grep get[a-zA-Z].* | grep -v "*"
	extinfo = getExtensions(fout, &numExtensions);
	getExtensionMembership(fout, extinfo, numExtensions);
	nspinfo = getNamespaces(fout, &numNamespaces);
	tblinfo = getTables(fout, &numTables);
	getOwnedSeqs(fout, tblinfo, numTables);
	funinfo = getFuncs(fout, &numFuncs);
	typinfo = getTypes(fout, &numTypes);
	getProcLangs(fout, &numProcLangs);
	getAggregates(fout, &numAggregates);
	oprinfo = getOperators(fout, &numOperators);
	getAccessMethods(fout, &numAccessMethods);
	getOpclasses(fout, &numOpclasses);
	getOpfamilies(fout, &numOpfamilies);
	getTSParsers(fout, &numTSParsers);
	getTSTemplates(fout, &numTSTemplates);
	getTSDictionaries(fout, &numTSDicts);
	getTSConfigurations(fout, &numTSConfigs);
	getForeignDataWrappers(fout, &numForeignDataWrappers);
	getForeignServers(fout, &numForeignServers);
	getDefaultACLs(fout, &numDefaultACLs);
	collinfo = getCollations(fout, &numCollations);
	getConversions(fout, &numConversions);
	getCasts(fout, &numCasts);
	getTransforms(fout, &numTransforms);
	inhinfo = getInherits(fout, &numInherits);
	partinfo = getPartitions(fout, &numPartitions);
	getEventTriggers(fout, &numEventTriggers);
	getTableAttrs(fout, tblinfo, numTables);
	getIndexes(fout, tblinfo, numTables);
	getConstraints(fout, tblinfo, numTables);
	getTriggers(fout, tblinfo, numTables);
	getRules(fout, &numRules);
	getPolicies(fout, tblinfo, numTables);
	getTablePartitionKeyInfo(fout, tblinfo, numTables);

```


pg_dump 中的事务
=============

## pg_dump 中的事务 1
备份是在一个事务里面进行的：
```sql
BEGIN
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE
	lock all tables in access shared lock --锁住了全部的表
    dump table1
	dump table2
	…
    dump tablen
commit
```
## pg_dump 中的事务 2
在dump的同时，别的会话可以select，insert update delete。但是不能有drop，alter，truncate这些，否者会被阻塞。
```
编 号	  类 型	    对应操作	    互 斥
1	Access Share	Select	8
2	Row Share	Select For Update, Select For Share	7,8
3	Row Exclusive	Insert, Update, Delete	5,6,7,8
4	Share Update Exclusive	Vacuum(Non-Full), Analyze, Create Index Concurrently	4,5,6,7,8
5	Share	Create Index(Without Concurrently)	3,4,6,7,8
6	Share Row Exclusive	 	3,4,5,6,7,8
7	Exclusive	 	2,3,4,5,6,7,8
8	Access Exclusive	Alter Table, Drop Table, Vacuum Full	1,2,3,4,5,6,7,8
```


pg_dump 源码解析
==================

## 源码文件
```
ls src/bin/pg_dump
```

```
common.c   --获取Catalog元信息，比如很多getXXX函数来读取系统表信息
compress_io.c
compress_io.h
dumputils.c --ACL相关
dumputils.h
Makefile
nls.mk
parallel.c  --并行备份还原的框架
parallel.h
pg_backup_archiver.c  --备份API和初始化不同备份类型
pg_backup_archiver.h
pg_backup_custom.c --custom备份类型的实现
pg_backup_db.c  --连数据库的相关函数
pg_backup_db.h
pg_backup_directory.c --dir备份类型的实现
pg_backup.h
pg_backup_null.c --null备份类型的实现
pg_backup_tar.c --tar备份类型的实现
pg_backup_tar.h
pg_backup_utils.c --pg_dump和pg_restore共享的一些工具函数，比如报错函数等等
pg_backup_utils.h
pg_dumpall.c   --最后生成一个pg_dumpall工具，调用pg_dump来备份一个实例上的所有databases
pg_dump.c --pg_dump的main入口
pg_dump.h
pg_dump_sort.c
pg_restore.c --pg_restore的main入口


```

## 数据结构
1. DumpableObject：记录了备份对象的type，id，name，dependencies...
其他备份对象DumpableObjectType继承DumpableObject，比如TableInfo的第一个成员就是DumpableObject，再加上特有的表的信息。
2. Archive *g_fout : 全局变量，记录了数据库版本，编码，报错个数...由CreateArchive建立
3. ArchiveHandle ：继承Archive，增加了备份文件格式，和对应格式的dump函数指针，全局变量等。
4. ArchiveHandle 中的   struct _tocEntry *toc; 是每个备份对象的信息：比如表定义，oid等等
3. KCIConnection      *g_conn;  当前的数据库链接

## 主要流程--初始化

从pg_dump.c这里开始
```
pg_dump.c --pg_dump的main入口
参数解析和判断相容性
CreateArchive 初始化g_fout
    初始化对应格式的回调函数指针
        InitArchiveFmt_Custom 
            一般我们用的就是这种模式，都在一个文件里面
        InitArchiveFmt_Files
            toc信息在单独main文件中，每个对象都备份为单独文件，可用于并行备份
        InitArchiveFmt_Null
            导出为sql脚本
        InitArchiveFmt_Tar 
            和files类型很像但是把输出指向一个tar归档，所以备份的是一个文件

```
## 主要流程--重要的函数指针
每种归档格式都需要初始化相应的回调函数指针
1. ArchiveEntryPtr： 每个备份对象TOC创建时调用
2. StartDataPtr：表数据调备份前调用
3. WriteDataPtr：备份表表数据时调用
4. EndDataPtr：表数据备份完成调用
5. WriteBytePtr：向输出写数据（1字节）
6. ReadBytePtr：读数据（1字节）
7. WriteBufPtr：写数据
8. ReadBufPtr：读数据
9. _CloseArchive：开始实际写入数据，最后关闭文件

等等，具体可以看`struct _archiveHandle`

## 主要流程--连接到数据库

```
ConnectDatabase ()连接到数据库。
setup_connection()接下来做一些基本的数据库设置：
    clientEncoding
    SET DATESTYLE = ISO
    SET use_std_cast = off
    BEGIN  --开始一个事务
    SET TRANSACTION ISOLATION LEVEL SERIALIZABLE --隔离级别：可串行化
    SET extra_float_digits TO 2
```

## 主要流程--处理include和exclude入参
1. 入参schema_include_patterns转为schema_include_oids
2. 入参schema_exclude_patterns转为schema_exclude_oids
3. 参数table_include_patterns转为table_include_oids
4 .使用的函数为expand_schema_name_patterns并在转的过程中检查对应的名称是否存在，如果不存在就会报错。

## 主要流程--获取备份对象信息
```
getSchemaData()
    getNamespaces()
    getTables()
    getFuncs()
    getXXX() ---查询系统表获取有那些备份对象

```

## 主要流程--拓扑排序
```
getDependencies() --从pg_depend获取依赖信息
getDumpableObjects() --创建指针数组，指向备份对象，用于随后的排序。
sortDumpableObjects() --拓扑排序
```

## 主要流程--开始归档(ArchiveEntry)
```
dumpTable()
dumpTableData()
dumpXXX系列函数获取单个备份对象的信息
接下来调用
    ArchiveEntry()
      分配TocEntry空间，TOC=TableOfContent
      TocEntry里面存放了对象的基本信息（比如表定义）以及
      dataDumper回调函数用于最后输出使用（比如copy语句）
```
## 主要流程--处理选项
这一步根据备份选项来处理每一个TOC
看函数_tocEntryRequired
```
typedef enum
{
    REQ_SCHEMA = 0x01,          /* want schema */
    REQ_DATA = 0x02,            /* want data */
    REQ_SPECIAL = 0x04          /* for special TOC entries */
} teReqs;

```
## 主要流程--开始输出
```
if (plainText)
    RestoreArchive(fout); --输出为sql形式
        restore_toc_entry --输出每个备份对象
            _PrintTocData
                dumpTableData_copy --实际的输出函数

    CloseArchive(fout); --输出为custom形式
        _CloseArchive() --回调函数

```

pg_restore 参数解析
=============

## help
```
$ ./pg_restore --help
```

```
pg_restore 从一个 pg_dump 备份的二进制文件恢复到数据库或者文本文件.

$ ./pg_restore    --help
pg_restore restores a PostgreSQL database from an archive created by pg_dump.

Usage:
  pg_restore [OPTION]... [FILE]

General options:
  -d, --dbname=NAME        connect to database name
  -f, --file=FILENAME      output file name
  -F, --format=c|d|t       backup file format (should be automatic)
  -l, --list               print summarized TOC of the archive
  -v, --verbose            verbose mode
  -V, --version            output version information, then exit
  -?, --help               show this help, then exit

Options controlling the restore:
  -a, --data-only              restore only the data, no schema
  -c, --clean                  clean (drop) database objects before recreating
  -C, --create                 create the target database
  -e, --exit-on-error          exit on error, default is to continue
  -I, --index=NAME             restore named index
  -j, --jobs=NUM               use this many parallel jobs to restore
  -L, --use-list=FILENAME      use table of contents from this file for
                               selecting/ordering output
  -n, --schema=NAME            restore only objects in this schema
  -O, --no-owner               skip restoration of object ownership
  -P, --function=NAME(args)    restore named function
  -s, --schema-only            restore only the schema, no data
  -S, --superuser=NAME         superuser user name to use for disabling triggers
  -t, --table=NAME             restore named relation (table, view, etc.)
  -T, --trigger=NAME           restore named trigger
  -x, --no-privileges          skip restoration of access privileges (grant/revoke)
  -1, --single-transaction     restore as a single transaction
  --disable-triggers           disable triggers during data-only restore
  --enable-row-security        enable row security
  --if-exists                  use IF EXISTS when dropping objects
  --no-data-for-failed-tables  do not restore data of tables that could not be
                               created
  --no-security-labels         do not restore security labels
  --no-tablespaces             do not restore tablespace assignments
  --section=SECTION            restore named section (pre-data, data, or post-data)
  --strict-names               require table and/or schema include patterns to
                               match at least one entity each
  --use-set-session-authorization
                               use SET SESSION AUTHORIZATION commands instead of
                               ALTER OWNER commands to set ownership

Connection options:
  -h, --host=HOSTNAME      database server host or socket directory
  -p, --port=PORT          database server port number
  -U, --username=NAME      connect as specified database user
  -w, --no-password        never prompt for password
  -W, --password           force password prompt (should happen automatically)
  --role=ROLENAME          do SET ROLE before restore

The options -I, -n, -P, -t, -T, and --section can be combined and specified
multiple times to select multiple objects.

If no input file name is supplied, then standard input is used.

Report bugs to <pgsql-bugs@postgresql.org>.

```
##根据对象列表文件还原1

还原时，可以先获取备份文件中备份对象的列表，然后在列表中选择需要还原的对象。
首先，执行如下命令，获取备份文件中对象的列表；列表文件包含了备份文件中所有对象的信息：
```
	pg_restore -l -f dump.lst dumpfile.dmp
```
生成的对象列表文件"dump.lst"中，会有如下的对象信息：
```
	3; 2445711 TABLE C_HISTORY SYSTEM
	4; 2609506 TABLE HISTORY SYSTEM
	5; 2445711 TABLE DATA C_HISTORY SYSTEM
	6; 2609506 TABLE DATA HISTORY 
```
##根据对象列表文件还原2
然后，在生成的列表文件中，对不需要还原的对象，在行首标注'；'，或将对象所在的行删除。另外，还可以重新排列各对象在列表文件中顺序，从而指定还原顺序。
最后，通过指定修改后的对象列表文件，还原数据库：
```
pg_restore  -L dump.lst dumpfile.dmp
```


pg_restore基本流程
=================


## 打开归档文件并读取TOC
```
(gdb) bt
#0  ReadToc (AH=0x62c3e0) at pg_backup_archiver.c:2520
#1  0x000000000040ee1e in InitArchiveFmt_Custom (AH=0x62c3e0) at pg_backup_custom.c:190
#2  0x0000000000408b81 in _allocAH (FileSpec=0x7fffffffe2d1 "./test.sql",
    fmt=archUnknown, compression=0, mode=archModeRead,
    setupWorkerPtr=0x4042cb <setupRestoreWorker>) at pg_backup_archiver.c:2334
#3  0x0000000000404363 in OpenArchive (FileSpec=0x7fffffffe2d1 "./test.sql",
    fmt=archUnknown) at pg_backup_archiver.c:219
#4  0x0000000000403dd9 in main (argc=3, argv=0x7fffffffdf28) at pg_restore.c:383

```

## 恢复数据
```
(gdb) bt
#0  restore_toc_entry (AH=0x62c3e0, te=0x631ce0, is_parallel=0 '\000')
    at pg_backup_archiver.c:711
#1  0x000000000040500e in RestoreArchive (AHX=0x62c3e0) at pg_backup_archiver.c:653
#2  0x0000000000403ea3 in main (argc=3, argv=0x7fffffffdf28) at pg_restore.c:422
(gdb) 
```

## 例子：
```
1. 向服务器发送`copy table from stdin;`
2. 紧接着从归档文件中读取数据
#0  ReadDataFromArchive (AH=0x62c3e0, compression=-1, readF=0x40ff0e <_CustomReadFunc>)
    at compress_io.c:162
#1  0x000000000040f4e1 in _PrintData (AH=0x62c3e0) at pg_backup_custom.c:514
#2  0x000000000040f465 in _PrintTocData (AH=0x62c3e0, te=0x632fe0)
    at pg_backup_custom.c:494
#3  0x00000000004057f9 in restore_toc_entry (AH=0x62c3e0, te=0x632fe0,
    is_parallel=0 '\000') at pg_backup_archiver.c:892
#4  0x000000000040500e in RestoreArchive (AHX=0x62c3e0) at pg_backup_archiver.c:653
#5  0x0000000000403ea3 in main (argc=3, argv=0x7fffffffdf28) at pg_restore.c:422
```


pg_restore 事务
=================

## 默认是每个对象开启一个事务
```
begin;
copy xxx from stdin;
commit;
begin;
copy xxx from stdin;
commit;
...
```

## 单事务还原


如果加上--single-transaction那么是整个还原过程是在一个事务里面
```
begin;
copy xxx from stdin;
copy xxx from stdin;
...
commit;
```





如果还想看到更多此类文章，请移步到[小宇的博客](http://shenyu.wiki)。
