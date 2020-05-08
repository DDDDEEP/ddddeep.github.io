---
layout: post
title:      "整理 MySQL 数据库备份与恢复方法"
subtitle:   ""
date:       2019-03-02 00:00:00
author:     "DEEP"
header-img: "img/home-bg.jpg"
catalog: true
tags:
    -   数据库
---

## 序言

一直比较担心以后不小心删库后会手忙脚乱不知所措，因此需要提前做好功课。

下文拼接整理各方博客文章的方法，成果难免出现错误或不太成熟。

如果有大大发现文章错误或有更好的流程方案，希望能跟博主分享一下～

## 备份

### 开启 bin-log 日志

bin-log 日志可将数据库运行过程中的操作记录以特殊结构记录下来。

当发生数据误删等行为时，可通过该日志逐步还原到之前某一时间点的状态。

在 Ubuntu 下 MySQL 的配置文件位于 `/etc/mysql/mysql.conf.d/mysqld.cnf`。

找到下面代码并将 server-id、log_bin 的注释去掉：

```ini
# The following can be used as easy to replay backup logs or for replication.
# note: if you are setting up a replication slave, see README.Debian about
# other settings you may need to change.
server-id=1
log_bin=/var/log/mysql/mysql-bin.log
expire_logs_days=10
max_binlog_size=100M
# binlog_do_db= include_database_name
# binlog_ignore_db= include_database_name
```

然后重启 MySQL，执行 SQL 语句 `show variables like '%log_bin%';`：

```
+---------------------------------+--------------------------------+
| Variable_name                   | Value                          |
+---------------------------------+--------------------------------+
| log_bin                         | ON                             |
| log_bin_basename                | /var/log/mysql/mysql-bin       |
| log_bin_index                   | /var/log/mysql/mysql-bin.index |
| log_bin_trust_function_creators | OFF                            |
| log_bin_use_v1_row_events       | OFF                            |
| sql_log_bin                     | ON                             |
+---------------------------------+--------------------------------+
```

可见 log_bin 已经启用，日志存储位置为 `/var/log/mysql`。

日志相关 SQL 命令如下：

```sql
flush logs; -- 新建最新的 bin-log 日志
show binary logs; -- 查看日志信息
show binlog events; -- 查看日志的详细事务信息
show master status; -- 查看最后一个日志的相关信息
reset master; -- 清空日志`
```

下面使用 `show binlog events` 查看新增一条记录时增加的事务信息：

```
| Pos  | Event_type     | Server_id | End_log_pos | Info                                 |
| 1133 | Anonymous_Gtid |         1 |        1198 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS' |
| 1198 | Query          |         1 |        1274 | BEGIN                                |
| 1274 | Table_map      |         1 |        1326 | table_id: 126 ( test_bin.test2 ) |
| 1326 | Write_rows     |         1 |        1366 | table_id: 126 flags: STMT_END_F      |
| 1366 | Xid            |         1 |        1397 | COMMIT /* xid=630 */                 |
```

在事务开始时，会运行 `SET @@SESSION.GTID_NEXT= 'ANONYMOUS'`。

然后在进行一系列事务后，最后通过 Xid 事务提交该操作包含的事务。

### 自动生成 SQL 备份文件

最简单的方法是写个脚本然后用 crontab 自动执行。

但为了方便还是用别人的现成轮子吧： automysqlbackup，适合轻量的环境使用。

安装后脚本位于 `/usr/sbin/automysqlbackup`，配置文件位于 `/etc/default/automysqlbackup`。

默认备份目录位于 `/var/lib/automysqlbackup`。

该工具可对每个数据库进行每日、每周或每月的自动备份，并可发送包含备份的邮件。

同时也会发送一些可能出现的警告或报错信息。

但鉴于该脚本是将配置项的用户密码变量直接代入 `mysqldump` 语句中。

这会使其每次备份输出警告信息，并会将该警告信息发送给预定邮箱：

```bash
Warning: Using a password on the command line interface can be insecure.
```

使用 mysql_config_editor 定制的配置文件作为参数运行命令则不会出现警告。

但该工具并未提供该功能，所以只好手动对脚本和配置文件进行改动了。

首先使用 mysql_config_editor 新建配置项：

```bash
mysql_config_editor set --login-path=root --host=localhost --user=root --password
```

 在配置文件中加上 login-path 的配置项：

```bash
# 使用 --login-path
LOGINPATH=root

# Username to access the MySQL server e.g. dbuser
USERNAME=root

# Username to access the MySQL server e.g. password
PASSWORD=root
```

此配置文件还包括发送的邮箱地址和备份数据库等相关的配置，可自定义设置。

然后找到脚本中执行 `mysqldump` 的位置，修改代码：

```bash
    if [ -n "${LOGINPATH}" ] ; then
        mysqldump --login-path=local $NEWOPT $1 > $2
    elif [ -z "${USERNAME}" -o -z "${PASSWORD}" ] ; then
        mysqldump --defaults-file=/etc/mysql/debian.cnf $NEWOPT $1 > $2
    else
        mysqldump --user=$USERNAME --password=$PASSWORD --host=$DBHOST $NEWOPT $1 > $2
    fi
```

修改后，脚本执行 `mysqldump` 时就不会出现警告信息，因此其也不会再发送报告该警告的邮件。

## 恢复数据

### SQL 备份

如果有完整的 SQL 备份的话，那恢复就非常方便了，怎样来都可以 (

### bin-log 日志

到此步骤时，应该是不小心删了些重要数据，且 SQL 备份并未记录这些重要数据的情况。

如果有上次的 SQL 备份与完整的 bin-log 日志的话，那还是很好恢复的。

首先马上关掉数据库，新建 bin-log 日志以截断旧日志，防止新的数据污染日志。

然后将所有数据和日志导出拉到本地操作，由本地还原数据后并生成新的完整 SQL 备份。

下面编写一个简单的例子来模拟还原过程，初始数据表如下：

```bash
mysql> use test_bin;
mysql> select * from test;
+----+
| id |
+----+
|  1 |
|  2 |
|  3 |
+----+
3 rows in set ( 0.00 sec )
```

在生产使用前，为初始的数据表生成了一个 SQL 备份。

然后在生产过程时，符合预期的插入记录 4，但不小心删除了记录 2，然后因来不及关闭数据库还不合预期的插入了记录 5：

```bash
mysql> select * from test;
+----+
| id |
+----+
|  1 |
|  3 |
|  4 |
|  5 |
+----+
4 rows in set ( 0.00 sec )
```

此时马上导出 SQL 数据，利用 `flush logs;` 新建日志防止新数据污染，并关掉数据库。

然后复制所有日志与备份到本地进行还原，现在我们要还原到删除记录 2 前的状态。

首先导入最新的 SQL 备份，然后分析日志。

利用时间或其它信息找出最新 SQL 备份后对应的位置和误删操作的位置。

确认对应位置后，便可利用 mysqlbinlog 工具筛选出两个位置间的事务，常用参数有如下：

| 参数             | 意义       |
| ---------------- | ---------- |
| --database       | 指定数据库 |
| --start-datetime | 开始时间   |
| --stop-datetime  | 结束时间   |
| --start-position | 起始位置   |
| --end-position   | 结束位置   |

利用 mysqlbinlog 工具配上对应参数，即可过滤出对应区间的二进制 SQL 语句。

然后通过管道传入 mysql 中即可实现事务重做，下面为本例的对应命令：

```bash
mysqlbinlog --no-defaults --database=test_bin --start-datetime="2019-03-31 11:43:44" --stop-datetime="2019-03-31 11:43:50" mysql-bin.000001 | mysql -uroot -proot
```

成功后即可在本地导出正确的 SQL 备份，并将其导入到服务器上。

然后服务器进行新的 SQL 备份并刷新日志，即可完成恢复数据流程。

### ibdata1

 [参考博文](<https://blog.51cto.com/hcymysql/1552917>) 待填坑。

### ib_logfile 事务日志

该日志是用于数据库崩溃时的恢复的。

不过好像也可以根据数据的特殊格式，编写脚本提取出误删的数据。

待填坑。

### 恢复 ibdata1、ib_logfile

该场景是误删掉 `ibdata1` 和 `ib_logfile` 了，此时千万不要停掉数据库！

 [参考博文](<http://www.cnblogs.com/gomysql/p/3702216.html>) 待填坑。

## 结语

备份 SQL + bin-log 搭配应该可以处理大部分误删的情况。

但以前是没了解过就瞎折腾的，这两件事一件都没做过，还好没捅出乱子。

剩下没填的坑也是不小的话题，感觉提高下数据库的姿势水平再搞比较好，就只好先这么放着了。

~~ ( 咕咕咕 ) ~~
