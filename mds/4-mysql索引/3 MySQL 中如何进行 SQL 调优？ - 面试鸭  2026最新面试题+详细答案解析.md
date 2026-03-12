---
source: https://www.mianshiya.com/question/1780933295521951745
---

# MySQL 中如何进行 SQL 调优？ - 面试鸭 | 2026最新面试题+详细答案解析

> ## Excerpt
> 2026最新后端、MySQL、数据库、场景题面试题，包含详细答案解析和练习题，21744+程序员已学习，已帮助10万+程序员通过校招/社招/实习面试，拿到offer！

---
## 回答重点

SQL 调优的核心思路就是**减少磁盘 I/O 和避免无效计算**。实际操作分三步走：先定位慢 SQL、再分析执行计划、最后针对性优化。

定位慢 SQL 靠 MySQL 的慢查询日志，分析执行计划用 EXPLAIN，优化手段主要有这几类：

1）**索引层面优化**

-   合理设计联合索引，利用覆盖索引避免回表。比如查询只需要 name 和 age，那索引建成 `(name, age)` 就不用回表去主键索引拿数据了
-   注意最左匹配原则，`WHERE b = 1` 吃不到 `(a, b, c)` 这个联合索引
-   避免在索引列上做函数运算，`WHERE YEAR(create_time) = 2024` 会让索引失效，改成 `WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01'`

2）**SQL 写法优化**

-   禁止 `SELECT *`，只查必要字段，减少网络传输和内存占用
-   避免 `%LIKE` 前缀模糊查询，`LIKE '%关键词'` 必然全表扫描
-   连表查询时检查字段字符集是否一致，utf8 和 utf8mb4 的字段 JOIN 会导致隐式转换，索引直接废掉

3）**架构层面优化**

-   热点数据上 Redis 缓存，访问频率高但变化少的数据没必要每次都查库
-   大表考虑分库分表，单表超过 2000 万行查询性能会明显下降
-   读写分离，把查询压力分摊到从库

还可以**通过业务来优化**，例如少展示一些不必要的字段，减少多表查询的情况，将列表查询替换成分页分批查询等等。

## 扩展知识

### EXPLAIN 关键指标解读

EXPLAIN 输出一堆字段，重点盯这几个：

**type** 表示访问类型，性能从好到差排序：`system > const > eq_ref > ref > range > index > ALL`。看到 ALL 基本就是全表扫描，得加索引了。ref 以上算比较健康的。

**rows** 是 MySQL 预估要扫描的行数，这个数字越小越好。如果一个查询 rows 显示 100 万，那肯定有问题。

**Extra** 里的信息很关键：

-   `Using index` 表示用到了覆盖索引，不用回表
-   `Using where` 表示在存储引擎返回数据后还要在 Server 层过滤
-   `Using filesort` 说明排序没用上索引，需要额外排序操作
-   `Using temporary` 说明用了临时表，一般出现在 GROUP BY 或 DISTINCT 场景

```sql
▼sql复制代码-- 一个典型的分析案例
EXPLAIN SELECT * FROM orders WHERE user_id = 10086 ORDER BY create_time;

-- 如果 Extra 显示 Using filesort，说明 create_time 排序没走索引
-- 优化方案：建联合索引 (user_id, create_time)
```

### 索引失效的常见场景

很多人以为建了索引就万事大吉，其实有一堆情况会让索引失效：

1）对索引列做运算或函数调用

```sql
▼sql复制代码-- 索引失效
SELECT * FROM users WHERE YEAR(birthday) = 1990;
-- 优化后
SELECT * FROM users WHERE birthday >= '1990-01-01' AND birthday < '1991-01-01';
```

2）隐式类型转换

```sql
▼sql复制代码-- phone 是 varchar 类型，传入数字会触发隐式转换
SELECT * FROM users WHERE phone = 13800138000;  -- 索引失效
SELECT * FROM users WHERE phone = '13800138000';  -- 正常走索引
```

3）OR 条件中有非索引字段

```sql
▼sql复制代码-- 假设 name 有索引，age 没有
SELECT * FROM users WHERE name = '张三' OR age = 25;  -- 全表扫描
```

4）联合索引不满足最左匹配

```sql
▼sql复制代码-- 索引是 (a, b, c)
SELECT * FROM t WHERE b = 1;  -- 索引失效
SELECT * FROM t WHERE a = 1 AND c = 3;  -- 只能用到 a
```

5）like 左侧通配符

比如 `WHERE name LIKE '%XX'`，这种是无法使用上索引的。

6）优化器的选择

不是有索引 MySQL 就一定会选，它是基于成本来选择执行计划，有时候全表扫描可能比用二级索引更快。

![](https://pic.code-nav.cn/mianshiya/question_picture/1843904816956411905/mISll1Dk_Vy4bwob3Uf_mianshiya.webp)

### 慢查询日志配置

生产环境排查问题，慢查询日志是第一手资料：

```sql
▼sql复制代码-- 开启慢查询日志
SET GLOBAL slow_query_log = ON;
-- 设置阈值，执行时间超过 1 秒的 SQL 会被记录
SET GLOBAL long_query_time = 1;
-- 记录没有使用索引的查询
SET GLOBAL log_queries_not_using_indexes = ON;

-- 查看配置
SHOW VARIABLES LIKE '%slow_query%';
SHOW VARIABLES LIKE 'long_query_time';
```

日志文件路径一般在 `/var/lib/mysql/xxx-slow.log`，可以用 `mysqldumpslow` 工具分析：

```csharp
▼bash复制代码# 按照查询时间排序，取前 10 条
mysqldumpslow -s t -t 10 /var/lib/mysql/xxx-slow.log
# 按照扫描行数排序
mysqldumpslow -s r -t 10 /var/lib/mysql/xxx-slow.log
```

> 不过现在一般用云厂商提供的服务器了，面板上能直接查看和搜索慢 SQL

### 大表优化策略

当单表数据量上来之后，光靠索引已经不够用了。MySQL 单表数据太多的话，即使走索引，B+ 树层级也会变深，查询性能下降明显。

**分页优化**是大表场景的高频问题。`LIMIT 1000000, 10` 这种深分页会扫描前 100 万条数据然后丢弃，非常浪费。优化方案是用游标分页：

```sql
▼sql复制代码-- 深分页，性能差
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;

-- 游标分页，性能好
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;
```

**冷热数据分离**也很有效。把 3 个月前的订单挪到历史表，主表只保留热数据，查询压力小很多。

## 面试官追问

#### 提问：一个慢查询，EXPLAIN 看到走了索引但还是慢，可能是什么原因？

> 详细解答可以看：[MySQL 中使用索引一定有效吗？如何排查索引效果？](https://www.mianshiya.com/question/1780933295463231490)

回答：几种可能。第一，回表次数太多，虽然走了二级索引，但查出来几十万行每行都要回表，随机 IO 吃不消，还是慢。第二，索引选择性差，比如 status 字段只有几个值，走索引过滤效果不好。第三，数据量太大，索引树本身就很大，内存装不下频繁读磁盘。第四，EXPLAIN 是估算，实际执行可能和估算不一样，可以用 EXPLAIN ANALYZE 看真实执行情况。第五，返回数据量大，网络传输耗时。重点看 EXPLAIN 的 rows 估算值和 Extra 列有没有 filesort、temporary。

#### 提问：联合索引 (a, b, c)，查询条件是 WHERE a = 1 AND c = 3，能用到索引吗？

> 详细解答可以看：[MySQL 索引的最左前缀匹配原则是什么？](https://www.mianshiya.com/question/1780933295450648578)

回答：能用到，但只能用到 a 这一列。因为联合索引是按 a、b、c 顺序排列的，跳过了 b 就没法利用 c 的有序性。MySQL 会用 a = 1 快速定位，然后在这个范围内逐行判断 c = 3。如果想让三列都生效，查询条件必须包含 a 和 b。

#### 提问：线上突然出现大量慢查询，排查思路是什么？

回答：先看是不是某个时间点突然出现的。如果是，查一下那个时间有没有发布、有没有跑批任务。然后看慢查询日志，找出最频繁的几条 SQL。用 EXPLAIN 分析执行计划，检查索引有没有失效。同时看一下服务器指标，CPU、磁盘 I/O、连接数是不是打满了。如果是锁等待导致的，用 `SHOW PROCESSLIST` 和 `SHOW ENGINE INNODB STATUS` 排查。

#### 提问：什么情况下 MySQL 优化器会放弃使用索引？

回答：优化器是基于成本估算的，它认为全表扫描更快就不走索引。典型场景是查询结果集占总数据量比例太高，比如表里 100 万条数据，查询条件能匹配 60 万条，走索引反而要多一次回表操作，不如直接全表扫。另外统计信息不准也会导致优化器误判，这时候可以用 `ANALYZE TABLE` 更新统计信息。
