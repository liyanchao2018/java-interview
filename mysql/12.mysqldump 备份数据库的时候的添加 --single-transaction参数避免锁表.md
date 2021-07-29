# mysqldump --single-transaction参数避免锁表

> [mysqldump 备份数据库的时候的添加 --single-transaction参数避免锁表](https://www.shiqidu.com/d/417)

## 1.mysqldump不用--single-transaction的问题

​		在使用`mysqldump`备份数据库的时候发现数据无法查询了，查询资料后得知可以在执行`mysqldump`的时候会执行`FLUSH TABLES WITH READ LOCK`，这会关闭所有打开的表，同时对于所有数据库中的表都加一个读锁，直到显示地执行unlock tables，该操作常常用于数据备份的时候。

**FLUSH TABLES WITH READ LOCK做了什么操作**

> **FTWRL主要包括3个步骤：**

- 上全局读锁(lock_global_read_lock)

- 清理表缓存(close_cached_tables)

- 上全局COMMIT锁(make_global_read_lock_block_commit)

​		第一步的作用是堵塞更新，备份时，我们期望获取此时数据库的一致状态，不希望有更多的更新操作进来。对于innodb引擎而言,其自身的MVCC机制，可以保证读到老版本数据，因此第一步对它使多余的。

​		第二步，清理表缓存，这个操作对于myisam有意义，关闭myisam表时，会强制要求表的缓存落盘，这对于物理备份myisam表是有意义的，因为物理备份是直接拷贝物理文件。对于innodb表，则无需这样，因为innodb有自己的redolog，只要记录当时LSN，然后备份LSN以后的redolog即可。

​		第三步，主要是保证能获取一致性的binlog位点，这点对于myisam和innodb作用是一样的。

所以总的来说，FTWRL对于innodb引擎而言，最重要的是获取一致性位点，前面两个步骤是可有可无的，因此如果业务表全部是innodb表，这把大锁从原理上来讲是可以拆的。

所以如果你的表引擎是innodb的话，你不想在备份的时候全局读锁导致数据查询失败。你可以加上 `--single-transaction`。



## 2.[MySQLdump之single-transaction详解](https://www.cnblogs.com/zhangshengdong/p/9196128.html)

### single-transaction

- **开启general log选项**

1. 查看目前general log的情况

~~~mysql
mysql> show variables like '%general_log%';
+------------------+--------------------------------------------+
| Variable_name    | Value                                      |
+------------------+--------------------------------------------+
| general_log      | OFF                                        |
| general_log_file | /data/mysqldata/3306/general_statement.log |
+------------------+--------------------------------------------+
2 rows in set (0.00 sec)
~~~

2.开启general log的选项

~~~mysql
mysql> set global general_log=on;
~~~

3.使用mysqldump命令

~~~mysql
[mysql@racnode1 ~]$ /usr/local/mysql/bin/mysqldump -uroot -p'zsd@7101' -S /data/mysqldata/3306/mysql.sock --single-transaction --default-character-set=utf8 zdemo student > /tmp/studentbackup.sql
~~~

其中使用了两个参数

- --single-transaction
    此选项会将隔离级别设置为：REPEATABLE READ。并且随后再执行一条START TRANSACTION语句，让整个数据在dump过程中保证数据的一致性，这个选项对InnoDB的数据表很有用，且不会锁表。但是这个不能保证MyISAM表和MEMORY表的数据一致性。
    为了确保使用`--single-transaction`命令时，保证dump文件的有效性。需没有下列语句`ALTER TABLE, CREATE TABLE, DROP TABLE, RENAME TABLE, TRUNCATE TABLE`，因为一致性读不能隔离上述语句。所以如果在dump过程中，使用上述语句，可能会导致dump出来的文件数据不一致或者不可用。
    如何验证上述的过程呢，可以开启general log看看过程是否如上述所说。
- --default-character-set=utf8
  导出的dump文件字符集为uft8，检验文件字符集的命令可以使用`file -i`

通用查询日志文件如下:

~~~mysql
2018-06-18T11:42:31.035205Z      9163 Query     /*!40100 SET @@SQL_MODE='' */
2018-06-18T11:42:31.036090Z      9163 Query     /*!40103 SET TIME_ZONE='+00:00' */
2018-06-18T11:42:31.036905Z      9163 Query     /*!80000 SET SESSION information_schema_stats_expiry=0 */
2018-06-18T11:42:31.037521Z      9163 Query     SET SESSION NET_READ_TIMEOUT= 700, SESSION NET_WRITE_TIMEOUT= 700
2018-06-18T11:42:31.038398Z      9163 Query     SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ
2018-06-18T11:42:31.038977Z      9163 Query     START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT */
2018-06-18T11:42:31.039859Z      9163 Query     SHOW VARIABLES LIKE 'gtid\_mode'
2018-06-18T11:42:31.058093Z      9163 Query     UNLOCK TABLES
中间日志省略
......
2018-06-18T11:42:31.084432Z      9163 Query     SAVEPOINT sp
2018-06-18T11:42:31.087632Z      9163 Query     show create table `student`
2018-06-18T11:42:31.088094Z      9163 Query     SET SESSION character_set_results = 'utf8'
2018-06-18T11:42:31.088407Z      9163 Query     show fields from `student`
2018-06-18T11:42:31.092360Z      9163 Query     show fields from `student`
2018-06-18T11:42:31.094718Z      9163 Query     SELECT /*!40001 SQL_NO_CACHE */ * FROM `student`
2018-06-18T11:42:32.815435Z      9163 Query     ROLLBACK TO SAVEPOINT sp
2018-06-18T11:42:32.815546Z      9163 Query     RELEASE SAVEPOINT sp
~~~

从上述日志分析:
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ 设置隔离级别为REPEATABLE READ
START TRANSACTION 开启了事务
  事务的实现是通过InnoDB存储引擎的MVCC机制事项，细节如下:InnoDB是一个多版本控制的存储引擎。它可以对已修改的行保留一个旧版本的数据信息。用于支持事务特性。例如：并发和数据回滚。这个信息保留在数据结构中的表空间中，这个表空间称之为rollback segment回滚段。(在Oracle中也有一种类似的数据结构)。
  当事务需要回滚的时候，InnoDB会使用回滚段的信息，用于执行undo操作。对于某一行，InnoDB会用早先版本的信息来保障读一致性(consistent read)。
  Undo日志在回滚段(rollback segment)中被分为两部分，一部分叫做插入undo日志(insert undo logs)，另外一部分叫做更新undo日志(update undo logs)。插入undo日志只用于事务回滚，如事务一旦提交，那么日志就可以被丢弃。更新undo日志用在读一致性，InnoDB会指定一个数据快照，这个快照的构建来自于更新undo日志中数据行的早期版本。通过数据行早期版本的快照来保证读一致性，如果不需要这种事务数据保护的时候，这个日志可以被丢弃。

### 保存点的日志分析

~~~mysql
SAVEPOINT SP
......中间日志省略...
SELECT /*!40001 SQL_NO_CACHE */ * FROM `student`
......中间日志省略...
ROLLBACK TO SAVEPOINT sp
RELEASE SAVEPOINT sp
~~~

可以看到通过REPEATABLE READ事务，保证数据一致性数据，然后
设置保存点sp，当读取了所有数据的快照，就回退这个保存点sp。可以比喻为游戏中存档之后，然后取档备份成一个游戏外的文件，删除这个档。可以当作这个档在这个游戏内不存在。

### 查看当前会话级别

~~~mysql
## 会话级当前事务级别
mysql> show variables like '%isolation%';
+-----------------------+-----------------+
| Variable_name         | Value           |
+-----------------------+-----------------+
| transaction_isolation | REPEATABLE-READ |
+-----------------------+-----------------+
1 row in set (0.03 sec)
## 系统全局级当前事务级别
mysql> show global variables like '%isolation%';
+-----------------------+-----------------+
| Variable_name         | Value           |
+-----------------------+-----------------+
| transaction_isolation | REPEATABLE-READ 
+-----------------------+-----------------+
1 row in set (0.11 sec)
## 修改全局级事务级别
mysql>set global transaction_isolation='read-committed'
~~~

就算修改了全局事务级别，Mysqldump导出时也会设定隔离事务级别为：REPEATABLE READ。用于保证数据的读一致性。

### 导出文件的字符集类型

~~~mysql
[mysql@racnode1 tmp]$ file -i studentbackup.sql 
studentbackup.sql: text/plain; charset=utf-8
~~~

