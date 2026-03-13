---
source: https://www.mianshiya.com/question/1780933295513563137
---

# MySQL 中 count(*)、count(1) 和 count(字段名) 有什么区别？ - 面试鸭 | 2026最新面试题+详细答案解析

> ## Excerpt
> 2026最新后端、MySQL、数据库面试题，包含详细答案解析和练习题，20253+程序员已学习，已帮助10万+程序员通过校招/社招/实习面试，拿到offer！

---
## 回答重点

三者都是统计行数的聚合函数，核心区别在于 **对 NULL 值的处理方式不同**。

`count(*)` 和 `count(1)` 会统计所有行，包含 NULL 值的行；`count(字段名)` 只统计该字段不为 NULL 的行数。

```sql
▼sql复制代码-- 假设 user 表有 100 行，其中 email 字段有 20 行是 NULL
SELECT count(*)    FROM user;  -- 返回 100
SELECT count(1)    FROM user;  -- 返回 100
SELECT count(email) FROM user; -- 返回 80
```

**性能上**，`count(*)` 和 `count(1)` 效率完全一样，官方文档写得很明确：

![image.png](https://pic.code-nav.cn/mianshiya/question_picture/1772087337535152129/dKNh3DE8_image_mianshiya.webp)

`There is no performance difference.` 没有任何性能差异。

`count(字段名)` 如果字段没索引就是全表扫描，还要额外判断是否为 NULL，所以一般比前两者慢一点。但如果字段是主键或有索引，差距就不大了。

![](https://pic.code-nav.cn/mianshiya/question_picture/1843904816956411905/E2MrobzV_CaFpBaT4Tw_mianshiya.webp)

## 扩展知识

### MyISAM 为什么 count(\*) 特别快

MyISAM 引擎只支持表锁，对表的修改操作是串行的，所以它能维护一个精确的 **行数计数器**。执行不带 WHERE 条件的 `count(*)` 时，直接返回这个计数器的值，O(1) 复杂度，几乎是瞬间返回。

```sql
▼sql复制代码-- MyISAM 表，不带 WHERE 条件
SELECT count(*) FROM myisam_table;  -- 直接返回维护好的行数，极快
SELECT count(*) FROM myisam_table WHERE status = 1;  -- 带条件就要扫描了
```

### InnoDB 的 count 优化策略

InnoDB 支持行锁和 MVCC，同一时刻不同事务看到的数据可能不同，没办法像 MyISAM 那样维护一个全局计数器。但 InnoDB 也做了优化：

1）优先选择最小的索引扫描。主键索引是聚簇索引，叶子节点存的是完整行数据，体积大；二级索引叶子节点只存主键值，体积小得多。执行 `count(*)` 时，优化器会选择占用空间最小的二级索引来扫描，减少 I/O 开销。

2）MySQL 8.0.17 之后，对 InnoDB 的 `count(*)` 做了并行扫描优化，多线程同时扫不同数据页，速度提升明显。

```sql
▼sql复制代码-- 查看 MySQL 是否用了并行扫描
EXPLAIN SELECT count(*) FROM large_table;
-- Extra 列可能出现 Parallel scan
```

![](https://pic.code-nav.cn/mianshiya/question_picture/1843904816956411905/ndxvNaJz_aOOgtQerTF_mianshiya.webp)

### count(字段) 的适用场景

别以为 `count(字段)` 没用，在特定场景下只能用它：

```sql
▼sql复制代码-- 统计有多少用户填写了手机号
SELECT count(phone) FROM user;

-- 统计不同状态的订单数量
SELECT status, count(status) FROM orders GROUP BY status;

-- 配合 DISTINCT 统计去重后的数量
SELECT count(DISTINCT city) FROM user;  -- 用户分布在多少个城市
```

### 大表 count 的优化方案

线上一张千万级的表，直接 `count(*)` 可能要几十秒，几个优化思路：

1）用缓存维护计数。每次增删数据时同步更新 Redis 里的计数器，查询时直接读缓存。缺点是要保证缓存和数据库的一致性。

2）用单独的计数表。创建一张表专门存各个业务表的行数，通过触发器或应用层维护。

3）用近似值。`SHOW TABLE STATUS` 返回的 Rows 字段是估算值，不精确但很快，适合不需要精确数字的场景。

```sql
▼sql复制代码SHOW TABLE STATUS LIKE 'user';  -- 看 Rows 列，是估算值
```

4）分区表并行统计。把大表按时间或 ID 分区，然后并行统计各分区再汇总。

## 面试官追问

#### 提问：如果要统计一张 5000 万行的订单表有多少条记录，直接 count(\*) 太慢，你怎么优化？

回答：首先看业务对精确度的要求。如果允许一定误差，直接用 `SHOW TABLE STATUS` 拿估算值，毫秒级返回。如果要精确值又要快，就得用缓存方案，在 Redis 里维护一个计数器，每次订单增删时同步更新，读的时候直接查 Redis。要注意处理好缓存和数据库的一致性问题，比如用事务消息或者定期校对。

#### 提问：count(1) 里面的 1 可以换成其他数字或字符串吗？比如 count(2) 或 count('abc')？

回答：可以换，效果完全一样。`count(1)`、`count(2)`、`count('abc')` 都等价于 `count(*)`，因为这里的常量只是占位符，告诉 MySQL 对每一行都计数，实际不会真的去读这个常量值。优化器会把它们统一处理成一样的执行计划。

#### 提问：count(\*) 走的是主键索引还是二级索引？

回答：InnoDB 会选占用空间最小的索引。如果表上有二级索引，优化器通常会选二级索引而不是主键索引，因为二级索引的叶子节点只存主键值，数据量比聚簇索引小很多，扫描的数据页更少，I/O 开销更低。可以用 `EXPLAIN` 看 key 列用的是哪个索引。
