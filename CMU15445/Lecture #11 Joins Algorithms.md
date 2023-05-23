# Lecture #11: Joins Algorithms

## **1 Joins**

在关系型数据库中，我们常常通过规范化 (Normalization) 设计避免信息冗余；因此查询时，就需要通过 Join 将不同 table 中的数据合并来重建数据。

本课关注双表的**内等值**连接。原则上我们希望，连接时将小表放到左侧 (作为外表)。

首先要讨论的是：Join 的输出和成本分析。

### **Operator Output**

逻辑上 Join 的操作的结果是：对任意一个 tuple r∈R和任意一个在 Join Attributes 上对应的 tuple s∈S，将 r 和 s 串联成一个新的 tuple。

<img src="D:\Notes\images\1684840823584-1.png" alt="img" style="zoom:67%;" />

- 现实中，Join 操作输出元组内容还取决于 `processing model`, `storage model` 和 `query` 本身。下面列举了几种输出元组的内容：

- - **Data**： Join 时，将内表和外表的所有非 Join Attributes 的属性都复制到输出元组中。又称 **提前物化**。优点是：Join 之后的操作都无需从原表中重新获取数据；缺点是：需要更多地内存。

- <img src="D:\Notes\images\1684840838649-4.png" alt="img" style="zoom: 67%;" />

- - **Record Ids**：Join 时，只将内外表的 Join Attributes 及 record id 复制到输出元组，后续操作自行根据 record id 去原表中获取相关数据。又称 **推迟物化(Late Materialization)**。适用于列存储数据库，因为这样 DBMS 不需要复制对于查询没用的属性。

- <img src="D:\Notes\images\1684840855282-7.png" alt="img" style="zoom:67%;" />

### **I/O Cost Analysis**

由于数据库中的数据量通常较大，无法一次性载入内存，因此 Join Algorithm 的设计目的，在于减少磁盘 I/O，因此我们衡量 Join Algorithm 好坏的标准，就是 I/O 的数量。此外我们不需要考虑 Join 结果的大小，因为不论使用怎样的 Join Algorithm，结果集的大小都一样。

- 之后的讨论都建立在这样的情景：

- - 对R和S两个 tables 做 Join
  - R中有 M个 pages，m个tuples
  - S中有 N个 pages，n个tuples

下面介绍的连接算法都有各自的适用场景，没有一个算法可以在所有场景下表现良好，需要具体问题具体分析。

## **2 Nested Loop Join**

也就是暴力循环算法。

总体上来看，这个嵌套循环算法主要由 2 个 for 嵌套，每个 for 分别迭代一个表，如果两个元组符合连接谓词，就输出它们。

在外循环的表叫做**外表**，在内循环的表叫做**内表**。以下的讨论中 R表是外表，S表是内表。

DBMS总是希望小表当做外表，这里的“小”是指占用的页数或者元组个数。DBMS 希望在内存中缓存尽量多的外表，在遍历内表时也可以使用索引加速。

### **Simple Nested Loop Join**

傻瓜式暴力两层大循环嵌套。

对于外表R的每一个元组，都遍历一遍S表，没有任何缓存机制，没有利用任何局部性。

<img src="D:\Notes\images\1684840940545-10.png" alt="img" style="zoom:67%;" />

磁盘 I/O Cost：$$M+(m×N)$$

M是读入外表 (R) 的 I/O 次数，然后对于R表的 m个元组，每个都扫描一遍内表 (S)。

举例：用时一个多小时

<img src="D:\Notes\images\1684840948341-13.png" alt="img" style="zoom:67%;" />

### **Block Nested Loop Join**

将外表和内表分块，能更好得利用缓存和局部性原理。对于外表的每一个块，获取内表的每一个块，然后将两个块内部进行连接操作。

这样每一个外表的块扫描一次内表，而不是每一个外表元组扫描一次内表，减少了磁盘I/O。

这里的外表要选择页数少的，而不是元组个数少的。

<img src="D:\Notes\images\1684840955294-16.png" alt="img" style="zoom:67%;" />

假设对于外表 (R) 的划分是一个页为一个块，磁盘 I/O Cost：$$M+(M×N)$$。M是读入外表的 I/O 次数，然后对于每一个块 (这里一个页就是一个块，共 M个块) 遍历一遍内表。

举例：用时50s：

![img](D:\Notes\images\1684840963034-19.png)

假如可以使用 B 个 buffer，我们使用一个 buffer 存储输出结果，一个 buffer 存储内表，剩下 B - 2 个 buffer 存储外表。这样就能将外表的一个 block 大小设置为 B - 2 个页。则磁盘 I/O Cost：$$M+(\lceil\frac{M}{B-2}\rceil\times N)$$。M是读入外表的 I/O 次数，然后对于每一个块 (这里B - 2个页是一个块，共 $$\lceil\frac{M}{B-2}\rceil$$个块) 遍历一遍内表。

假如外表能完全放入内存中，则磁盘 I/O Cost：$$M+N$$

举例：用时 0.15 s：

![img](D:\Notes\images\1684840969638-22.png)

### **Index Nested Loop Join**

之前的两种 Nested Loop Join 速度慢的原因在于，需要线性扫描一遍内表，如果内表在 Join Attributes 上有索引的话，就不用每次都线性扫描了。DBMS 可以在 Join Attributes 上临时构建一个索引，或者利用已有的索引。

<img src="D:\Notes\images\1684840980424-25.png" alt="img" style="zoom:67%;" />

假设每次索引查询都需要C的花费，那么磁盘 I/O Cost：$$M+(m\times C)$$。

> Summary：
>
> 1. 选择小表作为外表
> 2. 缓存尽可能多的外表，以减小对内表的循环扫描次数
> 3. 扫描内表时，尽可能利用索引

## **3 Sort-Merge Join**

**Phase #1 - Sort**：根据 Join Keys 对两个表进行排序（可使用外部排序）

**Phase #2 - Merge**：用双指针遍历两个表，输出匹配的元组（如果 Join Keys 并不唯一，则可能需要指针的回退）

![img](D:\Notes\images\1684841004576-28.png)

举例：

<img src="D:\Notes\images\1684841008054-31.png" alt="img"  />

Lecture# 0中我们了解到外部归并算法的需遍历数据 $$1+\lceil\log_{B-1}{\frac{N}{B}}\rceil$$次（B表示缓冲页数量，N表示数据页总数）

因此 Sort-Merge Join 的磁盘 I/O Cost：

Sort Cost(R)：$$2M\times(1+\lceil\log_{B-1}{\frac{M}{B}}\rceil)$$

Sort Cost(S)：$$2N\times(1+\lceil\log_{B-1}{\frac{N}{B}}\rceil)$$

Merge Cost： $$M+N$$

Total Cost： Sort + Merge

> 这种算法最坏的情况是，两个表的所有 tuple 在 Join Keys 上的值都相同，则 Merge Cost 会退化到$$M\times N$$。
>
> 不过这种情况在真实数据库中不太可能发生。

举例： 用时 0.75 s：

<img src="D:\Notes\images\1684841063109-34.png" alt="img" style="zoom:67%;" />

> Sort-Merge Join 适用于：
>
> - 当 tables 中的一个或者两个都已按 Join Key 排好序时 (如聚簇索引)
> - SQL 的输出必须按 Join Key 排序时

## **4 Hash Join**

核心思想：如果分别来自 R 和 S 中的两个 tuples 满足 Join 的条件，它们的 Join Attributes 必然相等，那么它们的 Join Attributes 经过某个 hash function 得到的值也必然相等，因此 Join 时，我们只需对两个 tables 中 hash 到同样值的 tuples 分别执行 Join 操作即可。

### **Basic Hash Join**

- **Phase #1: Build**：扫描外表，使用哈希函数 h1 对 Join Attributes 建立哈希表 T

- - 若提前已知外表的大小，则可用静态哈希表 (现行探查哈希表现最好)；若不知道，应用动态哈希表，且支持溢出页。

**Phase #2: Probe**：扫描内表，对每个 tuple 使用哈希函数 h1，计算在T中对应的 bucket，在 bucket 中找到匹配的 tuples

<img src="D:\Notes\images\1684841069130-37.png" alt="img" style="zoom:67%;" />

这里明确哈希表T的定义：

- Key： Join Attributes
- Value：根据查询要求不同及实现不同，Value 不同（与上面讨论的 Join 的输出类似，可有提前物化和延迟物化两种）
  - Full Tuple：可以避免在后续操作中再次获取数据，但需要占用更多的空间
  - Tuple Identifier：是列存储数据库的理想选择，占用最少的空间，但之后需要重新获取数据

一个大小为$$N$$页的表，需要$$\sqrt{N}$$个buffer。使用一个附加系数$$f$$，I/O Cost：$$B\times\sqrt{f\times N}$$

但 Basic Hash Join Algorithm 有一个弱点，就是有可能T无法被放进内存中，由于 hash table 的查询一般都是随机查询，因此在 Probe 阶段，T可能在 memory 与 disk 中来回移动。

**Optimization： Probe Filter**

可以使用 Bloom Filter 来优化查询，因为当键值不在哈希表中的时候，可以不用去查。

于是在哈希表中找之前，先使用 Bloom Filter (一个概率数据结构，可放入 CPU cache，通过它可得知一个元素在不在，不会出现假阴性，但会出现假阳性)，若 Bloom Filter 结果是不存在，那么就不用去哈希表中找。

Bloom Filter 插入阶段，使用 k 个哈希函数，将过滤器对应位设为1；查询阶段，查看过滤器 k 个哈希函数值对应的位是否都是1。

<img src="D:\Notes\images\1684841087363-40.png" alt="img" style="zoom:67%;" /><img src="D:\Notes\images\1684841093986-43.png" alt="img" style="zoom:67%;" />

出现假阳性'ODB'：

<img src="D:\Notes\images\1684841105299-46.png" alt="img" style="zoom:67%;" />

### **Partitioned Hash Join(GRACE hash join)**

当内存不足以放下表时，你不希望 buffer pool manager 不断将表换入换出内存，导致低性能。

Grace hash join是基础哈希连接的一个扩展，将内表也制作成可以写入磁盘的哈希表。

**Phase #1: Build**：对两个表都使用哈希函数 h1，分别对 Join Attributes 建立哈希表

**Phase #2: Probe**：比较两个表对应 bucket 的 tuples，输出匹配结果

<img src="D:\Notes\images\1684841110126-49.png" alt="img" style="zoom:67%;" />

假设我们有足够的 buffers 能够存下中间结果，则磁盘 I/O Cost：$$3(M+N)$$。其中 partition 阶段，需要读、写两个表各一次，Cost: $$2(M+N)$$，probing 阶段要读两个表各一次，Cost: $$M+N$$。

举例：用时0.45s：

![img](D:\Notes\images\1684841114441-52.png)

如果一个 bucket 不能放到内存中，那么就使用 `recursive partition`，将这个大 bucket 使用另一个哈希函数再进行哈希，使其能放入内存中。

通过哈希函数 h1 partition 后的 bucket1 太大没办法放入内存，则对 bucket1 使用哈虚函数 h2 再进行一次 partition，使它能放入内存。另一个表匹配时也要进行相同的哈希。

<img src="D:\Notes\images\1684841117738-55.png" alt="img" style="zoom:67%;" />

**Optimization：Hybrid Hash Join**

如果 key 是有偏斜的，那么可以使用 Hybrid Hash Join：让 DBMS 将 hot partition 放在内存立即进行匹配操作，而不是将它们溢出到磁盘中。（难以实现，现实中少见）

<img src="D:\Notes\images\1684841122500-58.png" alt="img" style="zoom:67%;" />

## **Conclusion**

<img src="D:\Notes\images\1684841127613-61.png" alt="img" style="zoom:67%;" />

Hash Join 在绝大多数场景下是最优选择。但当查询包含 `ORDER BY` 或者数据极其不均匀的情况下，Sort-Merge Join 会是更好的选择。

DBMSs 在执行查询时，可能使用其中的一种到两种方法。

## **Supplement**

`表连接`：基于表格之间的相同字段，使表之间发生关联，让两个或多个表连接在一起。使用 `JOIN` 关键字，且条件语句使用 ON 而不是 WHERE。

连接可以替换子查询，并且比子查询的效率一般会更快。

可以用 AS 给列名、计算字段和表名取别名，给表名取别名是为了简化 SQL 语句以及连接相同表。

> JOIN 等价于 INNER JOIN 内连接。

`内连接`：又称 **等值连接**，使用 `INNER JOIN` 关键字。返回两个表中都有的符合条件的行。

```SQL
SELECT A.value, B.value
FROM tablea AS A INNER JOIN tableb AS B
ON A.key = B.key;
```

可以不明确使用 INNER JOIN，而使用普通查询并在 WHERE 中将两个表中要连接的列用等值方法连接起来。

```SQL
SELECT A.value, B.value
FROM tablea AS A, tableb AS B
WHERE A.key = B.key;
```

`自连接`：可以看成内连接的一种，只是连接的表是自身而已。

```SQL
-- 子查询版本 --
SELECT name
FROM employee
WHERE department = (
      SELECT department
      FROM employee
      WHERE name = "Jim");
-- 自连接版本 --
SELECT e1.name
FROM employee AS e1 INNER JOIN employee AS e2
ON e1.department = e2.department
      AND e2.name = "Jim";
```

`自然连接`：把同名列通过等值测试连接起来的，同名列可以有多个。关键字 `NATUAL JOIN`。

内连接和自然连接的区别：内连接提供连接的列，而自然连接自动连接所有同名列。

```SQL
SELECT A.value, B.value
FROM tablea AS A NATURAL JOIN tableb AS B;
```

`外连接`：保留了没有关联的那些行。包括左外连接、右外连接、全外连接。

- 左外连接：保留左表没有关联的行。没有关联的行对应右表的所有选择列为空值。关键字 `LEFT JOIN/LEFT OUTER JOIN`。
- 右外连接：保留右表没有关联的行。关键字 `RIGHT JOIN/RIGHT OUTER JOIN`。
- 全外连接：保留左右表所有没有关联的行。当某行在另一个表中没有匹配行时，则另一个表的选择列表列包含空值。关键字 `FULL JOIN/FULL PUTER JOIN`。