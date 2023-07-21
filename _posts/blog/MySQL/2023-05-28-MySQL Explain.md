---
title: MySQL Explain
categories:
  - Blog
  - MySQL
tags:
  - MySQL
  - SQL 优化
---

MySQL 官方文档 [8.8.2 EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html)

EXPLAIN 输出列：

| Column                                                       | JSON Name       | Meaning                                        |
| :----------------------------------------------------------- | :-------------- | :--------------------------------------------- |
| [`id`](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html#explain_id) | `select_id`     | The `SELECT` identifier                        |
| [`select_type`](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html#explain_select_type) | None            | The `SELECT` type                              |
| [`table`](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html#explain_table) | `table_name`    | The table for the output row                   |
| [`partitions`](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html#explain_partitions) | `partitions`    | The matching partitions                        |
| [`type`](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html#explain_type) | `access_type`   | The join type                                  |
| [`possible_keys`](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html#explain_possible_keys) | `possible_keys` | The possible indexes to choose                 |
| [`key`](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html#explain_key) | `key`           | The index actually chosen                      |
| [`key_len`](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html#explain_key_len) | `key_length`    | The length of the chosen key                   |
| [`ref`](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html#explain_ref) | `ref`           | The columns compared to the index              |
| [`rows`](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html#explain_rows) | `rows`          | Estimate of rows to be examined                |
| [`filtered`](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html#explain_filtered) | `filtered`      | Percentage of rows filtered by table condition |
| [`Extra`](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html#explain_extra) | None            | Additional information                         |

## 1. `id`

id 列的编号是 select 的序列号，有几个 select 就有几个 id，并且 id 是按照 select 出现的顺序增长的，id 列的值越大优先级越高，id 相同则是按照执行计划列从上往下执行，id 为空则是最后执行。

## 2. `select_type`

查询类型，表示对应行是简单查询还是复杂查询。

| `select_type` Value                                          | JSON Name                    | Meaning                                                      |
| :----------------------------------------------------------- | :--------------------------- | :----------------------------------------------------------- |
| `SIMPLE`                                                     | None                         | Simple [`SELECT`](https://dev.mysql.com/doc/refman/8.1/en/select.html) (not using [`UNION`](https://dev.mysql.com/doc/refman/8.1/en/union.html) or subqueries) |
| `PRIMARY`                                                    | None                         | Outermost [`SELECT`](https://dev.mysql.com/doc/refman/8.1/en/select.html) |
| [`UNION`](https://dev.mysql.com/doc/refman/8.1/en/union.html) | None                         | Second or later [`SELECT`](https://dev.mysql.com/doc/refman/8.1/en/select.html) statement in a [`UNION`](https://dev.mysql.com/doc/refman/8.1/en/union.html) |
| `DEPENDENT UNION`                                            | `dependent` (`true`)         | Second or later [`SELECT`](https://dev.mysql.com/doc/refman/8.1/en/select.html) statement in a [`UNION`](https://dev.mysql.com/doc/refman/8.1/en/union.html), dependent on outer query |
| `UNION RESULT`                                               | `union_result`               | Result of a [`UNION`](https://dev.mysql.com/doc/refman/8.1/en/union.html). |
| [`SUBQUERY`](https://dev.mysql.com/doc/refman/8.1/en/optimizer-hints.html#optimizer-hints-subquery) | None                         | First [`SELECT`](https://dev.mysql.com/doc/refman/8.1/en/select.html) in subquery |
| `DEPENDENT SUBQUERY`                                         | `dependent` (`true`)         | First [`SELECT`](https://dev.mysql.com/doc/refman/8.1/en/select.html) in subquery, dependent on outer query |
| `DERIVED`                                                    | None                         | Derived table                                                |
| `DEPENDENT DERIVED`                                          | `dependent` (`true`)         | Derived table dependent on another table                     |
| `MATERIALIZED`                                               | `materialized_from_subquery` | Materialized subquery                                        |
| `UNCACHEABLE SUBQUERY`                                       | `cacheable` (`false`)        | A subquery for which the result cannot be cached and must be re-evaluated for each row of the outer query |
| `UNCACHEABLE UNION`                                          | `cacheable` (`false`)        | The second or later select in a [`UNION`](https://dev.mysql.com/doc/refman/8.1/en/union.html) that belongs to an uncacheable subquery (see `UNCACHEABLE SUBQUERY`) |

- **simple**：不包含子查询和 `union` 的简单查询
  ![](https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/MySQL/images/explain1.png)

- **primary**：复杂查询中最外层的 `select`
- **subquery**：包含在 `select` 中的子查询（不在 `from` 的子句中）
  用如下图展示 primary 和 subquery 类型
  ![](https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/MySQL/images/explain2.png)

- **derived**：包含在 `from` 子句中的子查询。MySQL 会将查询结果放入一个临时表中，此临时表也叫衍生表。
  ![](https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/MySQL/images/explain3.png)

- **union**：在 `union` 中的第二个和随后的 `select`，UNION RESULT 为合并的结果

  ![](https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/MySQL/images/explain4.png)

## 3. `table`

表示当前行访问的是哪张表，这也可以是以下值之一：

- <union*`M`*,*`N`*>: 当有 `union` 查询时，UNION RESULT 的 table 列的值为 `<union1,2>`，1 和 2 表示参与 `union` 的行 id。

- <derived*`N`*>: 当 `from` 中有子查询时，table 列的格式为<derivedN>，表示当前查询依赖 `id=N` 行的查询，所以先执行 `id=N` 行的查询，如上面 `select_type` 列图4所示。
- <subquery*`N`*>: 该行引用 id 值为 N 的行的具体化子查询的结果。

## 4. `partitions`

查询将匹配记录的分区。对于非分区表，该值为 NULL。

## 5. `type`

详细信息见：[EXPLAIN Join Types](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html#explain-join-types)

此列表示关联类型或访问类型。也就是 MySQL 决定如何查找表中的行。

依次从最优到最差分别为：`system` > `const` > `eq_ref` > `ref` > `range` > `index` > `ALL`。

- **NULL**：MySQL 能在优化阶段分解查询语句，在执行阶段不用再去访问表或者索引。
  ![](https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/MySQL/images/explain5.png)

- **system**、**const**：MySQL对查询的某部分进行优化并把其转化成一个常量（可以通过 `show warnings` 命令查看结果）。system 是 const 的一个特例，表示表里只有一条元组匹配时为 system。
  ![](https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/MySQL/images/explain6.png)
  ![](https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/MySQL/images/explain7.png)
- **eq_ref**：主键或唯一键索引被连接使用，最多只会返回一条符合条件的记录。简单的 `select` 查询不会出现这种 type。
  ![](https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/MySQL/images/explain8.png)
- **ref**：相比 eq_ref，不使用唯一索引，而是使用普通索引或者唯一索引的部分前缀，索引和某个值比较，会找到多个符合条件的行。
  ![](https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/MySQL/images/explain9.png)
- **range**：通常出现在范围查询中，比如 `in`、`between`、大于、小于等。使用索引来检索给定范围的行。
  ![](https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/MySQL/images/explain10.png)
- **index**：扫描全索引拿到结果，一般是扫描某个二级索引，二级索引一般比较少，所以通常比 ALL 快一点。
  ![](https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/MySQL/images/explain11.png)
- **ALL**：全表扫描，扫描聚簇索引的所有叶子节点。
  ![](https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/MySQL/images/explain12.png)

## 6. `possible_keys`

此列显示在查询中可能用到的索引。如果该列为 NULL，则表示没有相关索引，可以通过检查 `where` 子句看是否可以添加一个适当的索引来提高性能。

## 7. `key`

此列显示 MySQL 在查询时实际用到的索引。在执行计划中可能出现 possible_keys 列有值，而 key 列为 Null，这种情况可能是表中数据不多，MySQL 认为索引对当前查询帮助不大而选择了全表查询。如果想强制 MySQL 使用或忽视 possible_keys 列中的索引，在查询时可使用 force index、ignore index。

## 8. `key_len`

此列显示 MySQL 在索引里使用的字节数，通过此列可以算出具体使用了索引中的那些列。索引最大长度为 768 字节，当长度过大时，MySQL 会做一个类似最左前缀处理，将前半部分字符提取出做索引。当字段可以为 Null 时，还需要 1 个字节去记录。

key_len计算规则：

- 字符串：
  - char(n)：n 个数字或者字母占 n 个字节，汉字占 3n 个字节。
  - varchar(n)： n 个数字或者字母占 n 个字节，汉字占 3n+2 个字节。+2 字节用来存储字符串长度。
- 数字类型：
  - tinyint：1字节
  - smallint：2字节
  - int：4字节
  - bigint：8字节
- 时间类型：
  - date：3字节
  - timestamp：4字节
  - datetime：8字节

## 9. `ref`

此列显示 key 列记录的索引中，表查找值时使用到的列或常量。常见的有 const、字段名。

## 10. `rows`

此列是MySQL在查询中估计要读取的行数。注意这里不是结果集的行数。、

## 11. `filtered`

5.7 之后的版本默认就有这个字段，不需要使用 explain extended 了。这个字段表示存储引擎返回的数据在 server 层过滤后，剩下多少满足查询的记录数量的比例，注意是百分比，不是具体记录数。

## 12. `Extra`

详细信息见：[EXPLAIN Extra Information](https://dev.mysql.com/doc/refman/8.1/en/explain-output.html#explain-extra-information)

此列是一些额外信息。常见的重要值如下：

1. **Using index**：使用覆盖索引（如果select后面查询的字段都可以从这个索引的树中获取，不需要通过辅助索引树找到主键，再通过主键去主键索引树里获取其它字段值，这种情况一般可以说是用到了覆盖索引）。
2. **Using where**：使用 where 语句来处理结果，并且查询的列未被索引覆盖。
3. **Using index condition**：查询的列不完全被索引覆盖，where条件中是一个查询的范围。
4. **Using temporary**：MySQL需要创建一张临时表来处理查询。出现这种情况一般是要进行优化的。
5. **Using filesort**：将使用外部排序而不是索引排序，数据较小时从内存排序，否则需要在磁盘完成排序。
6. **Select tables optimized away**：使用某些聚合函数（比如 max、min）来访问存在索引的某个字段时