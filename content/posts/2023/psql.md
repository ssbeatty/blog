---
title: 使用psql命令行工具
date: 2023-04-17T15:25:05+08:00
lastmod: 2023-04-17T15:25:05+08:00
author: sasaba
cover: /img/使用psql命令行工具.jpg
images:
- /img/使用psql命令行工具.jpg
categories:
  - 工具 
tags:
  - Postgres
draft: true
---

使用psql命令行工具。

<!--more-->

## PostgreSQL简介

PostgreSQL是一个功能强大的开源对象关系型数据库系统。它具有超过30年的开发历史，得到了广泛的认可和使用。PostgreSQL的一个关键特性是其丰富的扩展性，可以通过用户自定义的函数、数据类型和操作符来轻松扩展其功能。

## psql命令行工具

psql是PostgreSQL的交互式终端工具，允许用户执行SQL查询和管理PostgreSQL数据库。以下是psql的一些基本用法及高级功能：

### 连接到数据库

要使用psql连接到PostgreSQL数据库，请在命令行中输入以下命令：

```sh
psql -h hostname -p port -U username -d databasename
```

其中，hostname是数据库服务器的地址，port是服务器的端口号（默认为5432），username是要连接的用户，databasename是要连接的数据库名称。

### 执行SQL查询
在psql提示符下，您可以直接输入SQL查询并按Enter键执行。例如，要查看数据库中所有的表，可以执行以下命令：

```text
1、列举数据库：\l
2、选择数据库：\c 数据库名
3、查看该某个库中的所有表：\dt
4、切换数据库：\c interface
5、查看某个库中的某个表结构：\d 表名
6、退出psgl：\q
```

要执行更复杂的查询，例如从某个表中查询数据，可以输入如下命令：

```text
SELECT * FROM tablename;
```
请注意，在SQL查询的末尾添加分号;是必须的。

### 查询历史记录与命令补全
psql提供了命令历史记录功能，您可以使用上下箭头键在以前执行过的命令之间进行导航。同时，psql还支持命令补全功能，可以在输入命令时按Tab键自动补全命令和表名等。

### 事务处理
PostgreSQL支持事务处理，允许您在一个事务中执行一组操作，确保要么所有操作都成功执行，要么所有操作都回滚。要开始一个新的事务，使用BEGIN命令：

```text
BEGIN;
```
要提交事务中的更改，使用COMMIT命令：

```text
COMMIT;
```
要回滚事务中的更改，使用ROLLBACK命令：

```text
ROLLBACK;
```
### 导入和导出数据
psql还允许您导入和导出数据。要将数据从CSV文件导入到表中，可以使用以下命令：

```text
COPY tablename FROM '/path/to/your/csvfile.csv' WITH (FORMAT csv, HEADER);
```
要将表中的数据导出到CSV文件，可以使用以下命令：

```text
COPY tablename TO '/path/to/your/csvfile.csv' WITH (FORMAT csv, HEADER);
```

## 一些操作

### VACUUM 

VACUUM是PostgreSQL中的一个重要命令，它用于清理数据库中的无用数据。VACUUM命令会删除数据库中的无用数据，释放磁盘空间，并重新组织表中的数据，以便更有效地使用磁盘空间。VACUUM命令还会更新数据库中的统计信息，这些统计信息用于优化查询计划。

VACUUM命令有两种模式：FULL和FREEZE。FULL模式会清理数据库中的所有无用数据，而FREEZE模式只会清理已经被删除的数据。默认情况下，VACUUM命令会使用FULL模式。要使用FREEZE模式，可以在VACUUM命令后面添加FREEZE选项：

```text
VACUUM TABLE tablename FREEZE;
```
要使用FULL模式，可以在VACUUM命令后面添加FULL选项：

```text
VACUUM TABLE tablename FULL;
```

### TRUNCATE

清空表中的数据，但不删除表本身。

```text
TRUNCATE TABLE tablename;
```

### 删除XLOG

之前有一次遇到VACUUM FULL的时候，由于表太大，导致VACUUM FULL的时候，数据库的XLOG文件太大，导致数据库无法启动，所以需要删除XLOG文件。

#### 关闭数据库

```text
[postgres@localhost pg_xlog]$ pg_stop
waiting for server to shut down...... done
server stopped
```

#### 查看版本控制信息
```text
[postgres@localhost ~]$ pg_controldata
pg_control version number:            903
Catalog version number:               201105231
Database system identifier:           5741432812849707775
Database cluster state:               shut down
pg_control last modified:             Thu 10 Jan 2013 07:56:36 AM CST
Latest checkpoint location:           2/38000020
Prior checkpoint location:            2/34009190
Latest checkpoint's REDO location:    2/38000020
Latest checkpoint's TimeLineID:       4
Latest checkpoint's NextXID:          0/1888
Latest checkpoint's NextOID:          65595
Latest checkpoint's NextMultiXactId:  1
Latest checkpoint's NextMultiOffset:  0
Latest checkpoint's oldestXID:        1792
Latest checkpoint's oldestXID's DB:   1
Latest checkpoint's oldestActiveXID:  0
Time of latest checkpoint:            Thu 10 Jan 2013 07:56:35 AM CST
```

注意这两个参数：
Latest checkpoint's NextXID: 0/ 1888

Latest checkpoint's NextOID: 65595

#### 重置，删除 pg_xlog
```text
[postgres@localhost ~]$ pg_resetxlog -o 65595 -x1888 -f /database/pgdata/
Transaction log reset
```

注意在pg10以上的版本中命令为`pg_resetwal`

