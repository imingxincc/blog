---
title: "MySQL Binlog 详解"
date: 2021-04-06T15:41:47+08:00
draft: true
target: ["MySQL binlog", "binlog 日志分析", "MySQL binlog 详解"]
categories: ["MySQL binlog 详解"]
---

最近通过 MySQL binlog 日志分析，我为公司找到了一个千万级的大漏洞。觉得 binlog 日志真的是很强大的，所以想着好好整理一下，希望可以帮助需要的人。

<!--more-->

## 前情提要

事情是这样的，一天晚上领导突然打电话说让我看一下有几个用户余额异常增加。我一看他给的数据，好家伙小几千万充值到用户余额里面去了。这个问题肯定严重了，我赶紧去查后台日志。居然没有？！！难道数据库被人误操作了？？难道被人搞了？？满脑子问号，不管了先暂停余额相关交易，再修改数据库密码。至少保证资金不会外流，然后从服务器端下载下当天的所有 binlog 日志。经过一番分析后发下，这是一个程序漏洞，立马联系相关平台负责人排查测试，最终 BUG 复现并修复。好了，问题到此终于解决了。

有些时候单从程序方面排查寻找漏洞是很费力的，业务太复杂、入口太多等等。这时候通过数据库日志分析得到历史执行 SQL，根据 SQL 执行逻辑来确认代码位置，进而确定是否存在 BUG 或者是误操作就很方便了。

## 什么是 binlog ？

MySQL 有四种类型的日志：

1. Error Log
Error Log 是错误日志，记录 mysqld 的一些错误。

2. General Query Log
General Query Log 是一般查询日志，记录 mysqld 正在做的事情，比如客户端的连接和断开、来自客户端每条 Sql Statement 记录信息；如果你想准确知道客户端到底传了什么玩意儿给服务端，这个日志就非常管用了，不过它非常影响性能。

3. Binary Log
binlog 是 MySQL Server 层记录的二进制日志文件，用于记录 MySQL 的数据更新或者潜在更新（比如 DELETE 语句执行删除而实际并没有符合条件的数据）并以"事务"的形式保存在磁盘中，像 select 或 show 等不会修改数据的操作则不会记录在 binlog 中。那么 binlog 就有了两个重要的用途：复制和备份恢复。比如主从表的复制或备份恢复什么的。

4. Slow Query Log
Slow Query Log 是慢查询日志，记录一些查询比较慢的 SQL 语句。这种日志非常常用，主要是给开发者调优用的。

## binlog 管理

在 my.cnf 配置中设置：log-bin="存放日志路径" 来开启 binlog 日志。

```MySQL
# binlog 信息查询，binlog 开启后可以在配置文件中查看其位置信息，也可以在 MySQL 命令行中查看：
mysql> show variables like "%log_bin%";
+---------------------------------+----------------------------------+
| Variable_name                   | Value                            |
+---------------------------------+----------------------------------+
| log_bin                         | ON                               |
| log_bin_basename                | /www/server/data/mysql-bin       |
| log_bin_index                   | /www/server/data/mysql-bin.index |
| log_bin_trust_function_creators | OFF                              |
| log_bin_use_v1_row_events       | OFF                              |
| sql_log_bin                     | ON                               |
+---------------------------------+----------------------------------+
6 rows in set (0.00 sec)

# 开启 binlog 后，会在数据目录（默认）生产 host-bin.n （具体 binlog 信息）文件及 host-bin.index 索引文件（记录 binlog 文件列表）。当 binlog 日志写满（binlog 大小 max_binlog_size，默认1G）或者数据库重启才会生产新文件，但是也可通过手工进行切换让其重新生成新的文件（flush logs）；另外，如果正使用大的事务，由于一个事务不能横跨两个文件，因此也可能在 binlog 文件未满的情况下刷新文件。
# 查看 binlog 文件列表
mysql> show binary logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000407 |    234493 |
| mysql-bin.000408 |    209942 |
| mysql-bin.000409 |    416793 |
| mysql-bin.000410 |    222345 |
| mysql-bin.000411 |     39557 |
| mysql-bin.000412 |      6847 |
| mysql-bin.000413 |       154 |
| mysql-bin.000414 |       177 |
| mysql-bin.000415 |       154 |
+------------------+-----------+
9 rows in set (0.00 sec)
```

```MySQL
# 查看 binlog 的状态：show master status 可以查看当前二进制日志文件的状态信息，显示当前正在写入的二进制文件及当前 position。
mysql> show master status;
+------------------+----------+--------------+------------------+--------------------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                                |
+------------------+----------+--------------+------------------+--------------------------------------------------+
| mysql-bin.000415 |      154 |              |                  | e7ae08a7-d6a4-11e8-b2a9-00163e0eebcc:1-953144183 |
+------------------+----------+--------------+------------------+--------------------------------------------------+
1 row in set (0.00 sec)

# 清空 binlog 日志文件
mysql> reset master;
Query OK, 0 rows affected (0.01 sec)

mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

mysql> show binary logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       154 |
+------------------+-----------+
1 row in set (0.00 sec)
```

## binlog 查看

默认情况下 binlog 日志是二进制格式，无法直接查看。可以使用下面两种方式进行查看:

1. show binlog events

这种方式可以解析制定的 binlog 日志，但不宜提取大量日志，速度很慢，不建议使用。

```MySQL
# SHOW BINLOG EVENTS [IN 'log_name'] [FROM pos] [LIMIT [offset,] row_count]
mysql> SHOW BINLOG EVENTS IN 'mysql-bin.000001' FROM 154 LIMIT 5\G;
*************************** 2. row ***************************
   Log_name: mysql-bin.000001
        Pos: 219
 Event_type: Query
  Server_id: 1
End_log_pos: 310
       Info: BEGIN
*************************** 3. row ***************************
   Log_name: mysql-bin.000001
        Pos: 310
 Event_type: Intvar
  Server_id: 1
End_log_pos: 342
       Info: INSERT_ID=1
*************************** 4. row ***************************
   Log_name: mysql-bin.000001
        Pos: 342
 Event_type: Query
  Server_id: 1
End_log_pos: 602
       Info: use `test`; INSERT INTO `users` (`name`, `email`, `password`, `remember_token`, `created_at`, `updated_at`) VALUES ('test', 'test@gmail.com', '123456', '123456', '2021-03-31 17:29:53', '2021-03-31 17:53:27')
```

2. mysqlbinlog

mysqlbinlog 是 MySQL 原生自带的 binlog 解析工具，速度快而且可以配合管道命令过滤数据，适合解析大量 binlog 文件，建议使用。

```bash
# 参数说明：
# /data/mysql-bin.000108 需要解析的 binlog 日志文件。
# --database 指定数据库名。
# --base64-output=decode-rows -vv 显示具体 SQL 语句
# --skip-gtids=true 忽略 GTID 显示
# > 000108.log 将结果导入到指定文件，方便查看。
# 也可使用–read-from-remote-server从远程服务器读取二进制日志，
# 还可使用--start-position --stop-position、--start-time --stop-time精确解析 binlog 日志。
mysqlbinlog /data/mysql-bin.000108 --database dbname_test --base64-output=decode-rows -vv --skip-gtids=true > 000108.log

mysqlbinlog --start-datetime='2020-09-10 00:00:00' --stop-datetime='2020-09-11 00:00:00' --database dbname_test -vv mysql-bin.000108 > 000108.log

mysqlbinlog --start-postion=107 --stop-position=1000 --database dbname_test -vv mysql-bin.000108 > 000108.log

mysqlbinlog -u username -p password -hremote_host_address -P3306 --read-from-remote-server --start-datetime='2020-09-10 00:00:00' --stop-datetime='2020-09-11 00:00:00' -vv mysql-bin.000108 > 000108.log
```

## 数据恢復

```MySQL
# 1. 首先看下当前 binlog 的位置
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |     1584 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

# 2. 向 users 表中插入两条数据
mysql> INSERT INTO `users` (`name`, `email`, `password`, `remember_token`, `created_at`, `updated_at`) VALUES ('test01', 'test01@example.com', '123456', '', '2021-03-31 17:29:53', '2021-03-31 17:29:53');
mysql> INSERT INTO `users` (`name`, `email`, `password`, `remember_token`, `created_at`, `updated_at`) VALUES ('test02', 'test02@example.com', '123456', '', '2021-03-31 17:29:53', '2021-03-31 17:29:53');

# 3. 再次查看下当前 binlog 的位置
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |     2584 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

# 4. 查询新插入的数据
mysql> SELECT * FROM `users` WHERE name = 'test01';
+----+--------+--------------------+----------+----------------+---------------------+---------------------+
| id | name   | email              | password | remember_token | created_at          | updated_at          |
+----+--------+--------------------+----------+----------------+---------------------+---------------------+
|  6 | test01 | test01@example.com | 123456   |                | 2021-03-31 17:29:53 | 2021-03-31 17:29:53 |
+----+--------+--------------------+----------+----------------+---------------------+---------------------+
1 row in set (0.00 sec)

# 5. 删除一条数据
mysql> DELETE FROM `users` WHERE name = 'test01' OR name = 'test02';
Query OK, 1 row affected (0.01 sec)

# 6. binlog 恢復（指定 pos 点恢復/部分恢復）
sudo /www/server/mysql/bin/mysqlbinlog --start-position=1584 --stop-position=2584 '/www/server/data/mysql-bin.000003' > /data/test.sql

# 7. 数据恢復
mysql> source /data/test.sql;

# 8. 查看恢復后的数据
mysql> SELECT * FROM `users` WHERE name = 'test01' OR name = 'test02';
+----+--------+--------------------+----------+----------------+---------------------+---------------------+
| id | name   | email              | password | remember_token | created_at          | updated_at          |
+----+--------+--------------------+----------+----------------+---------------------+---------------------+
|  6 | test01 | test01@example.com | 123456   |                | 2021-03-31 17:29:53 | 2021-03-31 17:29:53 |
|  7 | test02 | test02@example.com | 123456   |                | 2021-03-31 17:29:53 | 2021-03-31 17:29:53 |
+----+--------+--------------------+----------+----------------+---------------------+---------------------+
2 rows in set (0.00 sec)
```

恢复，就是让 MySQL 将保存在 binlog 日志中指定段落区间的 SQL 语句逐个重新执行一次而已。

## 参考文章
* [MySQL binlog 日志解析](https://opensource.actionsky.com/20200807-mysql/ "MySQL binlog 日志解析")
* [MySQL Binlog 解析](https://blog.csdn.net/u013256816/article/details/53020335 "MySQL Binlog 解析")
* [腾讯工程师带你深入解析 MySQL binlog](https://zhuanlan.zhihu.com/p/33504555 "腾讯工程师带你深入解析 MySQL binlog")


