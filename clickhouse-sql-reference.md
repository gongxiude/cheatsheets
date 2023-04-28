---
title: ClickHouse SQL Reference
updated: 2023-04-28
layout: 2017/sheet
category: Databases
---

{: .-one-column}
## 数据库操作SQL

ClickHouse 目前支持 5 种类型的数据库引擎, 其中`Atomic`为默认数据库引擎：

- `Atomic`: 支持非阻塞的DROP TABLE和RENAME TABLE查询和原子的EXCHANGE TABLES t1 AND t2查询。默认情况下使用Atomic数据库引擎。
- `MySQL`：MySQL引擎用于将远程的MySQL服务器中的表映射到ClickHouse中，并允许您对表进行INSERT和SELECT查询，以方便您在ClickHouse与MySQL之间进行数据交换。
- `MaterializedMySQL`: 创建ClickHouse数据库，包含MySQL中所有的表，以及这些表中的所有数据。
- `Lazy`：日志引擎，此类数据库下只能使用 Log 系列的表引擎。
- `PostgreSQL`: 允许连接到远程PostgreSQL服务。支持读写操作(SELECT和INSERT查询)，以在ClickHouse和PostgreSQL之间交换数据,和mysql引擎类似。
- `MaterializedPostgreSQL`: 使用PostgreSQL数据库表的初始数据转储创建ClickHouse数据库，并启动复制过程，即执行后台作业，以便在远程PostgreSQL数据库中的PostgreSQL数据库表上发生新更改时应用这些更改。
- `Replicated`: 该引擎基于Atomic引擎。它支持通过将DDL日志写入ZooKeeper并在给定数据库的所有副本上执行的元数据复制。
- `SQLite`: 允许连接到SQLite数据库，并支持ClickHouse和SQLite交换数据， 执行INSERT和SELECT查询。

参照官方文档 [Database Engines](https://clickhouse.com/docs/en/engines/database-engines)了解更多。

{: .-two-column}
### Create / Open / Delete Database

#### CREATE DATABASE

```sql
CREATE DATABASE [IF NOT EXISTS] db_name [ON CLUSTER cluster] [ENGINE = engine(...)] [COMMENT 'Comment']
```

example: 

```
CREATE DATABASE DatabaseName;
CREATE DATABASE IF NOT EXISTS db_name[ENGINE=engine];
```

#### DELETE DATABASE

```
DROP DATABASE [IF EXISTS] db [ON CLUSTER cluster] [SYNC]
```

example:

```
DROP DATABASE DatabaseName;
```

#### SHOW DATABASE

查看创建数据库语句
```
SHOW CREATE DATABASE <database>
```

查看数据库
```
SHOW DATABASES;
```

{: .-one-column}
## 数据表操作

云数据库ClickHouse支持的表引擎分为MergeTree、Log、Integrations和Special四个系列。本文主要对这四类表引擎进行概要介绍，并通过示例介绍常用表引擎的功能。

{: .-two-column}
### Create / Delete / Modify Table

#### Create


#### Drop


#### Alter


#### Change field order


### Keys


### Users and Privileges
