---
title: 多个进程处于Sending Data状态导致整个MySQL实例查询缓慢
categories:
- [MySQL]
tags:
- MySQL
top: 1
---

## 问题场景

数据量级只有十几万的数据表查询居然耗时 30s 以上，而且只是一个很简单的 join 联表查询。

```sql
SELECT
	esa.ID,
	esa.EVENT_CODE,
	esa.HAPPEN_TIME,
	esa.EVENT_LEVEL,
	esa.HOST_NAME,
	esa.SYSTEM_NAME,
	si.SYSTEM_NAME AS source,
	esa.ALARM_GROUP,
	esa.EVENT_DETAIL,
	esa.POINT_STATUS,
	esa.HANDLE_STATUS,
	esa.CONFIRM_STATUS,
	esa.IS_SHIELD,
	esa.POINT_ID,
	esa.HAS_EXPERT_STORE,
	concat( '/img', esa.PIC_URL ) PIC_URL,
	esa.IS_SIMULATE,
	esa.ASSET_ID,
	esa.IS_CONTACT_ASSET 
FROM
	event_signal_actual esa
	LEFT JOIN subsystem_info si ON esa.source = si.SYSTEM_CODE 
WHERE
	esa.IS_SIMULATE != 1 
	AND esa.IS_SHIELD = 0 
	AND si.SYSTEM_CODE != '0011' 
ORDER BY
	esa.HAPPEN_TIME DESC 
	LIMIT 100
```

IS_SIMULATE 和 IS_SHIELD 是 int 类型，只有 0 和 1；SYSTEM_CODE 和 SOURCE 是 varchar(32) 类型。event_signal_actual 表数据量级为十几万，subsystem_info 为十几条数据。

> 数据量级这么少，而且在测试服执行耗时是毫秒级别的，只是在正式服耗时 30s 以上。其实在这一步就应该可以认为是正式服 mysql 数据库实例出现了问题，但由于当时对 mysql 优化不够熟练，还是按照流程一步步排查。

## SQL 优化思路

大概思路就是：EXPLAIN 分析 ---》SHOW PROFILE 分析 ---〉从 mysql 实例找原因。

1. EXPLAIN 分析 SQL 语句，从 SQL 自身和索引方面优化。
2. SHOW PROFILE 分析 SQL 执行中的各个阶段。
3. 从 MySQL 实例找原因，例如全局参数 innodb_buffer_pool_size 是否合理等等。
4. SHOW PROCESSLIST 打印 MySQL 实例上当前正在执行的查询、连接状态和相关信息，从而进行性能监控、问题排查和优化。例如是否有大量查询卡在 Sending data 阶段。

### EXPLAIN查询执行计划

在 MySQL 中，`EXPLAIN` 是一个用于查看查询执行计划的命令。可以帮助分析查询是如何在数据库内部执行的，以及如何使用索引来提高性能。

| id   | select_type | table | partitions | type  | possible_keys   | key             | key_len | ref                | rows  | filtered | Extra                                              |
| ---- | ----------- | ----- | ---------- | ----- | --------------- | --------------- | ------- | ------------------ | ----- | -------- | -------------------------------------------------- |
| 1    | SIMPLE      | esa   |            | range | IDX_SOURCE      | IDX_SOURCE      | 99      |                    | 22948 | 9.00     | Using index condition; Using where; Using filesort |
| 1    | SIMPLE      | si    |            | ref   | IDX_SYSTEM_CODE | IDX_SYSTEM_CODE | 99      | pcs9700.esa.SOURCE | 1     | 100.00   | null                                               |

看到 Extra 中的 Using filesort 就知道需要优化 order by 排序语句。新增 HAPPEN_TIME 字段的索引。

| id   | select_type | table | partitions | type  | possible_keys | key             | key_len | ref  | rows | filtered | Extra       |
| ---- | ----------- | ----- | ---------- | ----- | ------------- | --------------- | ------- | ---- | ---- | -------- | ----------- |
| 1    | SIMPLE      | esa   |            | index | IDX_SOURCE    | idx_happen_time | 8       |      | 586  | 1.53     | Using where |

再次 EXPLAIN 分析可以看到查询使用的索引已经变更为 idx_happen_time ，Extra 优化信息显示 Using where ，Using filesort 的问题已经解决。

**但是 sql 在正式服执行仍然耗时 30s 以上**。

> 这时认为是 Using where 的优化程度不够高，因此拆解 sql 。

```sql
SELECT
	esa.ID
FROM
	event_signal_actual esa
ORDER BY
	esa.HAPPEN_TIME DESC 
	LIMIT 100
```

EXPLAIN 分析如下：

| id   | select_type | table | partitions | type  | possible_keys | key             | key_len | ref  | rows | filtered | Extra       |
| ---- | ----------- | ----- | ---------- | ----- | ------------- | --------------- | ------- | ---- | ---- | -------- | ----------- |
| 1    | SIMPLE      | esa   |            | index | IDX_SOURCE    | idx_happen_time | 8       |      | 100  | 100.00   | Using index |

可以看到，Extra 优化信息显示 Using index ，表示查询使用覆盖索引，不需要回表查询。**但是 sql 在正式服执行仍然耗时 30s 以上**。

> 这时基本确定 sql 语句自身是没有问题。

### SHOW PROFILE分析

`SHOW PROFILE` 是一个用于查看查询执行过程中详细信息的命令。它可以显示查询中各个阶段的耗时、资源使用情况等。`SHOW PROFILE` 命令需要在查询执行之后使用，它会返回一个包含多个不同阶段的性能数据集。

1. 先执行要分析的 sql 语句。

2. 执行 SHOW PROFILES 查看会话中最近执行的查询和语句的性能分析数据，找到刚刚执行的 sql 语句。

   ![image-20230816115937162](https://file.liuzx.com.cn/docsify-pic/202308161159255.png)

3. 执行 SHOW PROFILE FOR QUERY 3 （3 为要分析的 sql 的 Query_ID）。

   ![image-20230816120324323](https://file.liuzx.com.cn/docsify-pic/202308161203344.png)

   可以看到，sql 执行过程中最耗时的就是 Sending data 阶段，官方解释如下：

   > 官方文档：https://dev.mysql.com/doc/refman/5.7/en/general-thread-states.html
   >
   > `Sending data`
   >
   > The thread is reading and processing rows for a [`SELECT`](https://dev.mysql.com/doc/refman/5.7/en/select.html) statement, and sending data to the client. Because operations occurring during this state tend to perform large amounts of disk access (reads), it is often the longest-running state over the lifetime of a given query.
   >
   > 该线程正在读取和处理 SELECT 语句的行，并将数据发送到客户端。由于在此状态期间发生的操作往往会执行大量磁盘访问（读取），因此它通常是给定查询生命周期中运行时间最长的状态。

>这时认为 sql 语句本身是没有问题的，把注意力放在 mysql 实例上。

### Buffer Pool

Buffer Pool 是一个内存区域，用于缓存数据库表的数据和索引，以减少对磁盘的频繁读取操作。

从上一步骤可以知道整个 sql 执行过程就是卡在 Sending data 阶段，而在此状态期间发生的操作往往会执行大量磁盘访问。所以接下来分析 mysql 的 buffer pool 配置，通过优化 buffer pool 配置，将磁盘数据页加载到缓存中，减少磁盘 IO 次数。

```sql
mysql> SHOW GLOBAL VARIABLES LIKE 'innodb_buffer%';
+-------------------------------------+----------------+
| Variable_name                       | Value          |
+-------------------------------------+----------------+
| innodb_buffer_pool_chunk_size       | 134217728      |
| innodb_buffer_pool_dump_at_shutdown | ON             |
| innodb_buffer_pool_dump_now         | OFF            |
| innodb_buffer_pool_dump_pct         | 25             |
| innodb_buffer_pool_filename         | ib_buffer_pool |
| innodb_buffer_pool_instances        | 1              |
| innodb_buffer_pool_load_abort       | OFF            |
| innodb_buffer_pool_load_at_startup  | ON             |
| innodb_buffer_pool_load_now         | OFF            |
| innodb_buffer_pool_size             | 134217728      |
+-------------------------------------+----------------+
10 rows in set (0.01 sec)
```

mysql 实例的 innodb_buffer_pool_size 默认值为 128M 。

官方建议如下：

> 官方文档：https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_buffer_pool_size
>
> A larger buffer pool requires less disk I/O to access the same table data more than once. On a dedicated database server, you might set the buffer pool size to 80% of the machine's physical memory size. Be aware of the following potential issues when configuring buffer pool size, and be prepared to scale back the size of the buffer pool if necessary.
>
> 更大的缓冲池需要更少的磁盘 I/O 来多次访问相同的表数据。在专用数据库服务器上，您可以将缓冲池大小设置为计算机物理内存大小的 80%。

正式服 mysql 实例独享服务器资资源，有 60 多 G 的内存，而 innodb_buffer_pool_size 却只设置了默认的 128M 。到了这一步，我基本上已经认为只要增大 innodb_buffer_pool_size 大小就可以解决该问题。

```sql
SET GLOBAL innodb_buffer_pool_size = 40 * 1024 * 1024 * 1024
SHOW STATUS LIKE 'innodb_buffer%';
```

> 但现实是无论设置 innodb_buffer_pool_size 为 20G、30G、40G，并且确定配置已生效，sql 语句执行还是耗时 30s 以上。

### SHOW PROCESSLIST

`SHOW PROCESSLIST` 是 MySQL 提供的一个用于查看当前活动连接和查询的命令。它可以帮助您了解数据库服务器上当前正在执行的查询、连接状态和相关信息，从而进行性能监控、问题排查和优化。

分析到上一步骤，已经绝望。这时 SHOW PROCESSLIST ，发现大量查询命令卡在 Sending data 阶段。

查询卡在 Sending data 阶段的进程：

```sql
SELECT * FROM information_schema.PROCESSLIST WHERE STATE='Sending data' ORDER BY TIME desc;
```

![image-20230816154635675](https://file.liuzx.com.cn/docsify-pic/202308161546703.png)

可以看到大量关于表 xxl_job_log 的查询卡在 Sending data 阶段，发现单个 xxl_job_log 数据表就有 1600w 多数据，占用 16G 磁盘空间，而且这是一个 order by 排序查询语句。这时感觉是大量 xxl_job_log 数据需要加载在内存排序，占用大量 buffer_pool 内存空间，导致整个 mysql 实例加载速度慢，这也解释了无论设置 innodb_buffer_pool_size 为 20G、30G、40G，sql 查询执行依旧那么慢。

接下来是打算清理 xxl_job_log ，排查发现这是一个 xxljob 调度日志数据表，配置 3 天的日志保留时间不生效导致数据积累到千万量级，经该模块负责人同意可以直接 truncate 整个数据表（效率比 delete 快）。无需使用 delete 来删除并保留一周内数据。

>delete 在 where 子句中指定时间删除千万量级的数据表耗时非常长，会生成事务日志，需要耐心等待。我在 delete 执行期间不耐烦直接 kill 掉进程，导致需要耗费更多时间去回滚数据（进程一只处于 killed 状态，直到回滚完成）。truncate 执行期间会加 system lock ，可能会阻塞数据库所有操作。

清理完所有千万数据量级的 xxl_job_log 表后，执行 SHOW PROCESSLIST 可以看到所有卡在 Sending data 阶段的命令都消失了。

> 至此，整个优化流程完成，该 sql 语句执行耗时优化到毫秒级别，而且整个 mysql 实例也流畅了。

## EXPLAIN参数详解

1. **id**：

   SQL 执行的顺序表示，id 从大到小执行。id 相同时，执行顺序由上至下

2. **select_type**： 描述了查询的类型。常见的值包括：

   | 类型     | 说明                                                         |
   | -------- | ------------------------------------------------------------ |
   | SIMPLE   | 简单查询，不包含子查询或 UNION。                             |
   | PRIMARY  | 主查询，包含 union 或者子查询（相关子查询），最外层的部分标记为 primary 。 |
   | SUBQUERY | 子查询，通常出现在 `SELECT` 列表或 `WHERE` 子句中。          |
   | DERIVED  | 派生表，临时表，例如在 `FROM` 子句中的子查询。               |
   | UNION    | 联合查询，位于 union 中第二个及其以后的子查询被标记为 union，第一个就被标记为 primary ，如果是union 位于 from 中则标记为 derived 。 |

   官方文档解释如下：

   | `select_type` Value                                          | JSON Name                    | Meaning                                                      |
   | :----------------------------------------------------------- | :--------------------------- | :----------------------------------------------------------- |
   | `SIMPLE`                                                     | None                         | Simple [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) (not using [`UNION`](https://dev.mysql.com/doc/refman/8.0/en/union.html) or subqueries) |
   | `PRIMARY`                                                    | None                         | Outermost [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) |
   | [`UNION`](https://dev.mysql.com/doc/refman/8.0/en/union.html) | None                         | Second or later [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) statement in a [`UNION`](https://dev.mysql.com/doc/refman/8.0/en/union.html) |
   | `DEPENDENT UNION`                                            | `dependent` (`true`)         | Second or later [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) statement in a [`UNION`](https://dev.mysql.com/doc/refman/8.0/en/union.html), dependent on outer query |
   | `UNION RESULT`                                               | `union_result`               | Result of a [`UNION`](https://dev.mysql.com/doc/refman/8.0/en/union.html). |
   | [`SUBQUERY`](https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html#optimizer-hints-subquery) | None                         | First [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) in subquery |
   | `DEPENDENT SUBQUERY`                                         | `dependent` (`true`)         | First [`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html) in subquery, dependent on outer query |
   | `DERIVED`                                                    | None                         | Derived table                                                |
   | `DEPENDENT DERIVED`                                          | `dependent` (`true`)         | Derived table dependent on another table                     |
   | `MATERIALIZED`                                               | `materialized_from_subquery` | Materialized subquery                                        |
   | `UNCACHEABLE SUBQUERY`                                       | `cacheable` (`false`)        | A subquery for which the result cannot be cached and must be re-evaluated for each row of the outer query |
   | `UNCACHEABLE UNION`                                          | `cacheable` (`false`)        | The second or later select in a [`UNION`](https://dev.mysql.com/doc/refman/8.0/en/union.html) that belongs to an uncacheable subquery (see `UNCACHEABLE SUBQUERY`) |

3. **table**： 指示查询操作的表。

4. **partitions**： 如果查询涉及到分区表，这个字段会显示被访问的分区。

5. **type**： 描述了 MySQL 在表中找到所需行的方式。常见的值包括：

   | 类型   | 说明                                                         |
   | ------ | ------------------------------------------------------------ |
   | All    | 全表扫描，最坏的情况                                         |
   | index  | 和全表扫描一样。只是扫描表的时候按照索引次序进行而不是行。主要优点就是避免了排序,  但是开销仍然非常大。如在 Extra 列看到 Using  index，说明正在使用覆盖索引，只扫描索引的数据，它比按索引次序全表扫描的开销要小很多。 |
   | range  | 使用索引范围匹配多行数据，一个有限制的索引扫描。key  列显示使用了哪个索引。当使用 =、<>、>、>=、<、<=、IS NULL、<=>、BETWEEN、LIKE 或 IN() 运算符中的任何一个，将键列与常量进行比较时，会使用 range 。 |
   | ref    | 使用非唯一索引来匹配数据，**返回所有匹配某个单个值的行**。。此类索引访问只有当使用非唯一性索引或唯一性索引非唯一性前缀时才会发生。这个类型跟 eq_ref 不同的是，它用在关联操作只使用了索引的最左前缀，或者索引不是 UNIQUE 和PRIMARY  KEY。ref 可以用于使用 = 或 <=> 操作符的带索引的列。 |
   | eq_ref | 使用等值连接匹配一行数据，**最多只返回一条符合条件的记录**。当 join 连接时使用索引的所有部分并且索引是 PRIMARY KEY 或 UNIQUE NOT NULL 索引时，使用它。（通常用于 join 连接操作） |
   | const  | 该表最多有一个匹配行，该行在查询开始时读取。由于只有一行，因此该行中的列的值可以被优化器的其余部分视为常量。 const 表非常快，因为它们只被读取一次。将 PRIMARY KEY 或 UNIQUE 索引的所有部分与常量值进行比较时，将使用 const。（通常用于单表常量查询） |
   | system | 该表只有一行。                                               |
   | Null   | 意味说 mysql 能在优化阶段分解查询语句，在执行阶段甚至用不到访问表或索引（高效） |

6. **possible_keys**： 显示可能被用于查询的索引。

7. **key**： 显示实际被用于查询的索引。

8. **key_len**： 显示被使用的索引的长度。在不损失精确性的情况下，长度越短越好 。

9. **ref**： 显示哪个列或常量与索引进行比较。

10. **rows**： 指示 MySQL 认为它必须检查才能执行查询的行数。

11. **filtered**： 表示在 `WHERE` 子句中被过滤掉的行的百分比。

12. **Extra**： 包含关于查询优化的其他信息。

    | 类型                  | 说明                                                         |
    | --------------------- | ------------------------------------------------------------ |
    | Using index           | 表示查询使用了覆盖索引（Covering Index），即查询所需的数据可以直接从索引中获取，而不需要回表查询。 |
    | Using where           | 表示查询在检索数据时需要进一步的过滤操作，即需要在 MySQL 层面应用 `WHERE` 子句中的条件。"Using where" 表示在查询执行过程中，优化器可能使用了索引来匹配部分查询条件，但是还需要进一步的操作来满足剩余的条件，这些条件可能不在索引中。常见情况例如回表查询，即查询需要返回的列不全部包含在使用的索引中，因此查询还需要回表查询来获取缺失的数据列。 |
    | Using temporary       | 表示查询需要创建临时表来处理某些操作，如排序或分组。         |
    | Using filesort        | 表示查询需要进行文件排序操作，即 MySQL 需要将数据写入临时文件并对其进行排序，通常与 `ORDER BY` 子句相关。可以通过选择合适的索引来改进性能，用索引来为查询结果排序。 |
    | Distinct              | 表示查询需要对结果进行去重操作。                             |
    | Not exists            | 表示查询使用了 `NOT EXISTS` 子查询。                         |
    | Dependent subquery    | 表示查询使用了相关子查询（dependent subquery），即子查询依赖于外部查询。 |
    | Cached                | 表示查询结果来自缓存。                                       |
    | Using index condition | 表示查询的一部分或全部查询条件可以在索引中进行求值，而不需要通过回表查询实际的行数据。这可以降低数据访问的开销，从而提高查询性能。 |

## 如何确定 buffer_pool 大小

执行 `SHOW STATUS LIKE 'innodb_buffer%';` 

**Innodb_buffer_pool_read_requests** 表示读请求的次数。

**Innodb_buffer_pool_reads** 表示从物理磁盘中读取数据的请求次数。

所以buffer pool的命中率就可以这样得到：

```
buffer pool 命中率 = 1 - (Innodb_buffer_pool_reads/Innodb_buffer_pool_read_requests) * 100%
```

## 参考

MySQL EXPLAIN 官方文档：https://dev.mysql.com/doc/refman/8.0/en/explain-output.html

MySQL SHOW PROFILE 官方文档：https://dev.mysql.com/doc/refman/5.7/en/general-thread-states.html

一次mysql order by desc 慢的排查：https://juejin.cn/post/6844903874839445517

https://z.itpub.net/article/detail/D14C1032130798C5D2E0B00C10905150