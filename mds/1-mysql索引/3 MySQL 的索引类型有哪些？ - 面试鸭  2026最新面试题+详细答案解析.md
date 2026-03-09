---
source: https://www.mianshiya.com/question/1780933295438065665
---

# MySQL 的索引类型有哪些？ - 面试鸭 | 2026最新面试题+详细答案解析

> ## Excerpt
> 2026最新后端、MySQL、数据库面试题，包含详细答案解析和练习题，37474+程序员已学习，已帮助10万+程序员通过校招/社招/实习面试，拿到offer！

---
## 回答重点

MySQL 索引可以从三个维度来分类：数据结构、存储方式、索引性质。面试时一般先从数据结构角度切入，重点讲 **B+ 树索引**，然后再展开聊聚簇索引和非聚簇索引的区别。

**从数据结构角度看**，MySQL 支持这几种索引：

1）B+ 树索引：最常用的，InnoDB 和 MyISAM 默认都用它。多层平衡树结构，叶子节点用链表串起来，既能快速定位单条记录，又能高效做范围扫描。

2）哈希索引：通过哈希函数直接算出数据位置，等值查询 O(1) 级别，但不支持范围查询和排序。Memory 引擎默认用哈希索引，InnoDB 有个自适应哈希索引会自动建，不用手动管。

3）全文索引：把文本分词后建倒排索引，类似搜索引擎的原理。适合对 TEXT 类型字段做关键字搜索，比如文章内容检索。

4）空间索引：基于 R 树实现，专门处理地理坐标这种多维数据，支持区域查询、距离计算。

**从 InnoDB 存储方式看**，分两类：

1）聚簇索引：主键索引就是聚簇索引，叶子节点直接存完整的行数据，数据按主键顺序物理存储。一张表只能有一个聚簇索引。

2）非聚簇索引：也叫二级索引，叶子节点只存索引字段值和主键值。查完二级索引还得拿着主键去聚簇索引里再查一遍，这个过程叫回表。

**从索引性质看**，有这几种：

1）主键索引：唯一且非空，每张表只能有一个。InnoDB 里主键索引就是聚簇索引。

2）唯一索引：保证列值不重复，但允许有 NULL，可以有多个 NULL。

3）普通索引：没有唯一约束，纯粹为了加速查询。

4）联合索引：多列组合成一个索引，遵循最左前缀原则，列顺序很重要。

5）全文索引：文本搜索用。

6）空间索引：GIS 数据用。

![](https://pic.code-nav.cn/mianshiya/question_picture/1843904816956411905/Abfw7VXZ_xeUchjvrIT_mianshiya.webp)

## 扩展知识

### B+ 树索引为什么是主力

数据库查询有两大类操作：等值查询和范围查询。哈希索引等值查询快，但范围查询搞不定。B+ 树两个都能扛，而且叶子节点用双向链表连起来，范围扫描的时候顺着链表走就行，不用回溯。

B+ 树还有个特点是树高很矮。一个 3 层的 B+ 树，能存 2000 万左右的数据。查任何一条数据最多 3 次磁盘 IO，这个性能在机械硬盘时代非常重要。

![bshu.drawio.png](https://pic.code-nav.cn/mianshiya/question_picture/1783388929455529986/eVz4CDRz_bshu.drawio_mianshiya.webp)

### 聚簇索引和非聚簇索引的本质区别

聚簇索引的叶子节点存的是完整的行数据，数据和索引绑定在一起。InnoDB 表的数据文件本身就是按主键组织的 B+ 树，所以主键查询只需要一次索引查找就能拿到数据。

非聚簇索引的叶子节点只存索引列的值和主键值。比如在 name 列上建索引，叶子节点存的是 (name, id) 这样的组合。查的时候先在 name 索引里找到对应的 id，再拿着 id 去聚簇索引里查完整数据，这就是回表。

比如执行 `SELECT * FROM user WHERE name = 'Alice'`，流程是这样的：

![jucusuoy.drawio.png](https://pic.code-nav.cn/mianshiya/question_picture/1783388929455529986/XD7pBrt6_jucusuoy.drawio_mianshiya.webp)

如果查询只需要索引列本身的值，比如 `SELECT id FROM user WHERE name = 'Alice'`，name 索引的叶子节点里已经有 id 了，不用回表，这就是覆盖索引的原理。

### 联合索引和最左前缀原则

联合索引是把多个列按顺序组合成一个索引。比如建 (a, b, c) 联合索引，B+ 树先按 a 排序，a 相同的再按 b 排序，b 相同的再按 c 排序。

这就导致一个规则：查询条件必须从最左边的列开始才能用上索引。

1）`WHERE a = 1` 能用

2）`WHERE a = 1 AND b = 2` 能用

3）`WHERE a = 1 AND b = 2 AND c = 3` 能用

4）`WHERE b = 2` 用不上，因为没有 a

5）`WHERE a = 1 AND c = 3` 只能用到 a，c 用不上

建联合索引时，选择性高的列放前面。选择性就是不重复值的比例，比如性别只有男女两种，选择性低；手机号每个都不一样，选择性高。

![](https://pic.code-nav.cn/mianshiya/question_picture/1843904816956411905/PDugUn9n_UMKu2IEcLU_mianshiya.webp)

### 索引创建示例

主键索引在建表时指定：

```scss
▼sql复制代码CREATE TABLE users (
    id INT NOT NULL AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100),
    PRIMARY KEY(id)
);
```

唯一索引：

```sql
▼sql复制代码CREATE UNIQUE INDEX idx_username ON users(username);
```

普通索引：

```scss
▼sql复制代码CREATE INDEX idx_email ON users(email);
```

联合索引：

```scss
▼sql复制代码CREATE INDEX idx_name_email ON users(username, email);
```

全文索引：

```scss
▼sql复制代码CREATE FULLTEXT INDEX idx_content ON articles(content);
```

## 面试官追问

#### 提问：InnoDB 里如果没有显式定义主键，聚簇索引怎么建？

回答：InnoDB 会找第一个非空的唯一索引当聚簇索引。如果连唯一索引都没有，InnoDB 会自己生成一个 6 字节的隐藏列 row\_id 作为聚簇索引的键。这个 row\_id 是全局递增的，不是表级别的。所以建表时最好显式指定主键，否则这个隐藏主键既浪费空间，排查问题时也看不到。

#### 提问：哈希索引既然等值查询这么快，为什么不是默认索引？

回答：三个原因。第一，哈希索引不支持范围查询，`WHERE id > 100` 这种条件没法用。第二，不支持排序，ORDER BY 用不上。第三，哈希冲突多的时候性能会退化。数据库查询大部分场景是范围查询、排序、分页，哈希索引应付不了，B+ 树全能选手更适合做默认。

#### 提问：联合索引 (a, b) 和分别建 a、b 两个索引，有什么区别？

回答：联合索引 (a, b) 是一棵 B+ 树，先按 a 排序再按 b 排序。`WHERE a = 1 AND b = 2` 这种查询一次索引扫描就搞定。分别建两个单列索引是两棵树，MySQL 可能会用索引合并，但效率不如联合索引。联合索引还能实现覆盖索引，比如 `SELECT b FROM t WHERE a = 1`，不用回表。缺点是联合索引占用空间更大，更新维护成本也高一点。

#### 提问：全文索引和 LIKE '%keyword%' 有什么区别？

回答：`LIKE '%keyword%'` 是前后模糊匹配，没法用 B+ 树索引，只能全表扫描。全文索引是分词后建倒排索引，能快速定位包含某个词的记录。全文索引支持自然语言搜索、布尔模式搜索，还能按相关性排序。但全文索引对中文支持不太好，需要配合 ngram 分词器。如果是正经的全文搜索需求，一般用 Elasticsearch 比 MySQL 全文索引靠谱。
