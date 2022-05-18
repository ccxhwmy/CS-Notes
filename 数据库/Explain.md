## Explain 详解

​	在日常工作中，我们会有时会开慢查询去记录一些执行时间比较久的SQL语句，找出这些SQL语句并不意味着完事了，些时我们常常用到explain这个命令来查看一个这些SQL语句的执行计划，查看该SQL语句有没有使用上了索引，有没有做全表扫描，这都可以通过explain命令来查看。所以我们深入了解MySQL的基于开销的优化器，还可以获得很多可能被优化器考虑到的访问策略的细节，以及当运行SQL语句时哪种策略预计会被优化器采用。

```mysql
-- 实际SQL，查找用户名为Jefabc的员工
select * from emp where name = 'Jefabc';
-- 查看SQL是否使用索引，前面加上explain即可
explain select * from emp where name = 'Jefabc';
```

![img](image/512541-20180803142201303-545775900.png)

expain出来的信息有10列，分别是`id、select_type、table、type、possible_keys、key、key_len、ref、rows、Extra`

### 概要描述

| #    | Column        | Meaning                    |
| ---- | ------------- | -------------------------- |
| 1    | id            | 选择标识符                 |
| 2    | select_type   | 表示查询的类型             |
| 3    | table         | 输出结果集的表             |
| 4    | partitions    | 匹配的分区                 |
| 5    | possible_keys | 表示查询时，可能使用的索引 |
| 6    | key           | 表示实际使用的索引         |
| 7    | key_len       | 索引字段的长度             |
| 8    | ref           | 列与索引的比较             |
| 9    | rows          | 扫描出的行数               |
| 10   | filtered      | 按表条件过滤的行百分比     |
| 11   | Extra         | 执行情况的描述和说明       |

### 字段详解

#### 1.id

SELECT 识别符。这是 SELECT 的查询序列号

### 参考资料

- https://www.cnblogs.com/tufujie/p/9413852.html

