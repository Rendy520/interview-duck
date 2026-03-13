---
source: https://www.mianshiya.com/question/1780933295517757442
---

# MySQL 中 varchar 和 char 有什么区别？ - 面试鸭 | 2026最新面试题+详细答案解析

> ## Excerpt
> 2026最新后端、MySQL、数据库面试题，包含详细答案解析和练习题，13716+程序员已学习，已帮助10万+程序员通过校招/社招/实习面试，拿到offer！

---
## 回答重点

最核心的区别就是 **CHAR 定长，VARCHAR 变长**。

CHAR(10) 不管你存几个字符，都固定占 10 个字符的空间，不够的用空格填充。VARCHAR(10) 存多少占多少，另外再花 1~2 个字节记录实际长度。

```sql
▼sql复制代码CREATE TABLE demo (
    c1 CHAR(10),
    c2 VARCHAR(10)
);

INSERT INTO demo VALUES ('abc', 'abc');

-- CHAR: 存储时填充空格变成 'abc       '，占 10 个字符空间
-- VARCHAR: 存储 'abc' + 1 字节长度信息，实际占 4 字节
```

性能上 CHAR 理论上快一点点，因为定长不需要额外计算长度，但这点差距在实际业务中可以忽略。选型原则很简单：固定长度的短字符串用 CHAR，比如 MD5 摘要、国家代码；其他场景一律用 VARCHAR。

|  特点  |     CHAR      |  VARCHAR  |
|------|---------------|-----------|
| 存储方式 |  定长，不够用空格填充   | 变长，存多少占多少 |
| 额外开销 |       无       | 1~2 字节存长度 |
| 存储效率 |  短字符串可能浪费空间   | 按需占用，更省空间 |
| 读取效率 |    定长读取，略快    | 需解析长度，略慢  |
| 典型场景 | MD5、UUID、国家代码 | 用户名、地址、描述 |

## 扩展知识

### VARCHAR(N) 的 N 会影响排序性能

MySQL 执行 ORDER BY 时会用到 sort\_buffer，如果要排序的数据放不下，就得走 **双路排序**：先把排序字段和主键放进 sort\_buffer 排好序，再回表取完整数据。多了一次回表，性能差不少。

关键点在于 MySQL 计算 sort\_buffer 能放多少行时，用的是 VARCHAR(N) 定义的最大长度 N，不是实际数据长度。

```sql
▼sql复制代码-- a、b、c 都是 VARCHAR(1000)，即使实际数据很短
SELECT a, b, c FROM t1 WHERE a = '面试鸭' ORDER BY b;

-- MySQL 按每行 3000 字符算空间，sort_buffer 放不下就走双路排序
```

如果 SELECT 的字段总长度短，就能用 **单路排序**：把所有需要的字段一次性放进 sort\_buffer，排完直接返回，不用回表。

![](https://pic.code-nav.cn/mianshiya/question_picture/1843904816956411905/Q4ovNfGg_1Jen3S7v9X_mianshiya.webp)

所以 VARCHAR(N) 的 N 别随手写个 255 或 1000，要根据实际业务需求设一个合理的值，否则会隐性影响排序性能。

更多排序细节可查看面试鸭《[MySQL 是如何实现数据的排序的？](https://www.mianshiya.com/question/1780933295526146049)》这个面试题。

### VARCHAR 支持的最大长度

MySQL 单行最大 65535 字节，所以 VARCHAR 的理论上限就是这个数减去必要开销：

1）2 字节存长度信息（超过 255 字节时需要 2 字节）

2）如果允许 NULL，还要 1 字节存 NULL 标记

所以 NOT NULL 的 VARCHAR 最大 65533 字节，允许 NULL 则是 65532 字节。

实际能存多少字符取决于字符集：

|   字符集   | 单字符最大字节 | VARCHAR 最大字符数 |
|---------|---------|---------------|
| latin1  |    1    |     65533     |
| utf8mb3 |    3    |     21844     |
| utf8mb4 |    4    |     16383     |

`VARCHAR(N)` 里的 N 是字符数，不是字节数。用 utf8mb4 时，`VARCHAR(16383)` 就是上限了。

![image.png](https://pic.code-nav.cn/mianshiya/question_picture/1783388929455529986/rx4bXAdY_image_mianshiya.webp)

![](https://pic.code-nav.cn/mianshiya/question_picture/1843904816956411905/clHXgzDZ_wL4eakS97s_mianshiya.webp)

### CHAR 读取时会自动去掉尾部空格

CHAR 存储时会用空格填充到定长，读取时 MySQL 会自动去掉尾部空格。这可能导致意想不到的问题：

```sql
▼sql复制代码CREATE TABLE t (c CHAR(5));
INSERT INTO t VALUES ('abc  ');  -- 故意存两个空格
SELECT CONCAT('[', c, ']') FROM t;
-- 结果是 [abc]，尾部空格被去掉了
```

如果业务真的需要保留尾部空格，只能用 VARCHAR。

### 什么时候用 CHAR

别看 CHAR 好像很鸡肋，在特定场景下还是有优势的：

1）存储固定长度的编码，比如 ISO 国家代码 `CHAR(2)`、MD5 摘要 `CHAR(32)`、UUID 去掉横杠后 `CHAR(32)`

2）存储单字符的状态标记，比如性别 `CHAR(1)` 存 M/F

3）频繁更新的短字段，CHAR 不会产生碎片，VARCHAR 长度变化可能导致页分裂

## 面试官追问

#### 提问：VARCHAR(50) 和 VARCHAR(200) 存同样的字符串，占用空间一样吗？

> 详细解答可以看：[MySQL 中 VARCHAR(100) 和 VARCHAR(10) 的区别是什么？](https://www.mianshiya.com/question/1805142216804081666)

回答：实际存储空间是一样的，都是按实际字符数加 1~2 字节长度信息。但定义的长度会影响内存分配，MySQL 在做排序、JOIN 等操作时会按最大长度预分配内存，所以 VARCHAR(200) 会比 VARCHAR(50) 多占内存，影响 sort\_buffer 能放的行数，间接影响排序性能。

#### 提问：InnoDB 存 CHAR 一定是固定大小吗？

> 详细解答可以看：[MySQL 中 varchar 和 char 有什么区别？](https://www.mianshiya.com/question/1780933295517757442)

回答：当使用 utf8 这类变长字符集时，InnoDB 会把 CHAR 当变长存储。比如 `CHAR(10)` 用 utf8，每个字符 1~3 字节，如果按最大 30 字节固定存储太浪费，所以 InnoDB 优化成按实际字节数存。只有 latin1 这种单字节字符集，CHAR 才真正定长存储。

#### 提问：VARCHAR 长度超过多少会存到行外？

回答：这取决于行格式。在旧版 Compact/Redundant 格式下，若单字段长度超过 768 字节，InnoDB 会把超出部分存到溢出页，行内只保留 768 字节前缀加一个指向溢出页的指针。而在现代默认的 Dynamic/Compressed 格式下，触发机制不再针对单一字段的固定长度，而是遵循 “最长字段优先” 原则：为了保证一个 InnoDB 页至少能容纳两行数据（即整行总长度不超过约 8000 字节），最长的字段会被优先选中并整块移至行外，行内仅保留 20 字节指针。也就是即便字段不到 768 字节，若整行太宽也可能溢出。
