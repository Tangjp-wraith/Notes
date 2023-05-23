#  Lecture #12: Query Execution I

## **1 Query Plan**

通常一个 SQL 语句会被组织成如图的树状查询计划，数据从叶节点流到根节点，查询结果在根节点中得出。

通常，树上的操作符 operators 是二元的 (1~2个子运算符)。

<img src="D:\Notes\images\1684841385827-64.png" alt="img" style="zoom:67%;" />

## **2 Processing Models**

- DBMS 的 processing model 定义了系统如何执行一个 query plan。目前主要有三种模型：

- - 迭代模型（Iterator Model）
  - 物化模型（Materialization Model）
  - 向量化/批处理模型（Vectorized/Batch Model）

不同模型适用于不同的 workload。

### **Iterator Model**

最常见的 processing model，也称为 Volcano/Pipeline Model。

query plan 中的每步 operator 都实现一个 `next` 函数，**每次调用时，operator 返回一个 tuple 或者 null**，后者表示数据已经遍历完毕。operator 本身实现一个循环，每次调用其 child operators 的 next 函数，从它们那边获取下一条数据供自己操作，这样整个 query plan 就被从上至下地串联起来：

<img src="D:\Notes\images\1684841397671-67.png" alt="img" style="zoom: 67%;" />

此模型几乎被用在每个 (基于行) DBMS 中，包括 sqlite、MySQL、PostgreSQL 等等。需要注意的是：

- 有些 operators 一直阻塞，直到 children 返回所有 tuples，这些 operators 被称为 `pipeline breakers`。如 Joins, Subqueries 和 Order By。
- 在该模型中实现 Output Control 比较容易，如 Limit，只用按需调用 next 即可 (一旦获取了所需的足够 tuples 就停止对 child operator 调用 next)。

### **Materialization Mode**

每个 operator **一次处理其所有输入**，然后**将所有结果一次性输出**。DBMS 会将一些参数传递到 operator 中防止处理过多的数据，这是一种 bottom-to-top 的思路。

每个查询计划操作符都实现一个 `Output` 函数：

- 操作符一次处理其子代的所有元组。
- 此函数的返回结果是运算符将发出的所有元组。当操作符完成执行时，DBMS**再也不需要返回到**它来检索更多数据。

<img src="D:\Notes\images\1684841430332-70.png" alt="img" style="zoom:67%;" />

materialization model：

- 更适合 **OLTP** 场景，因为后者通常指需要处理少量的 tuples，这样能减少不必要的执行、调度成本。
- 不太适合会产生大量中间结果的 OLAP 查询，因为DBMS可能不得不在 operators 之间将这些结果溢出到磁盘。

### **Vectorization Model**

Vectorization Model 是 Iterator 与 Materialization Model 折中的一种模型：

- 每个 operator 实现一个 `next` 函数，但**每次 next 调用返回一批 tuples**，而不是单个 tuple
- operator 内部的循环每次也是一批一批 tuples 地处理
- batch 的大小可以根据需要改变 ( hardware、query properties)

<img src="D:\Notes\images\1684841438552-73.png" alt="img" style="zoom:67%;" />

vectorization model 是 **OLAP** 查询的理想模型：

- 极大地减少每个 operator 的调用次数
- 允许 operators 使用 vectorized instructions (SIMD) 来批量处理 tuples

目前在使用这种模型的 DBMS 有 VectorWise, Peloton, Preston, SQL Server, ORACLE, DB2 等。

### **Processing Direction**

1、Top-to-Bottom：从上往下执行，从孩子结点 pull 数据。

2、Bottom-to-Top：从下往上执行，向父结点 push 数据。

<img src="D:\Notes\images\1684841447857-76.png" alt="img" style="zoom:67%;" />

## **3 Access Methods**

access method 指的是 DBMS 从数据表中获取数据的方式，它并没有在 relational algebra 中定义。主要有三种方法：

- Sequential Scan
- Index Scan
- Multi-Index/"Bitmap" Scan

> 相同的查询计划可以以多种方式执行，多数 DBMS 都希望尽可能多地使用 Index Scan。

### **3.1 Sequential Scan**

sequential scan 就是按顺序从 table 所在的 pages 中取出 tuple，这种方式是 DBMS 能做的最坏的打算。

```C++
for page in table.pages:
    for t in page.tuples:
        if evalPred(t):
            # do something
```

DBMS 内部需要维护一个 cursor 来追踪之前访问到的位置 (page/slot)。

Sequential Scan 是最差的方案，因此也针对地有许多优化方案：

- Prefetching
- Parallelization
- Buffer Pool Bypass
- (本节) Zone Maps
- (本节) Late Materialization
- (本节) Heap Clustering

#### **Zone Maps**

预先为每个 page 计算好 attribute values 的一些统计值 (最大值，最小值，平均值)，并存入zone map中。DBMS 在访问 page 之前先检查 zone map，确认一下是否要继续访问，如下图所示：

<img src="D:\Notes\images\1684841457189-79.png" alt="img" style="zoom:67%;" />

当 DBMS 发现 page 的 Zone Map 中记录 val 的最大值为 400 时，就没有必要访问这个 page。

#### **Late Materialization**

对于列存储的 DBMS，我们可以延迟将数据从一个 operator 传递到另一个 operator 的时间。若某列数据在查询树上方并不需要，那我们只需要向上传递 offset 或者将 clumn id 即可。该方法允许稍后才去获取所需的数据。

#### **Heap Clustering**

使用 clustering index 时，tuples 在 page 中按照相应的顺序排列，如果查询访问的是被索引的 attributes，DBMS 就可以直接跳跃访问目标 tuples。

### **3.2 Index Scan**

DBMS 选择一个 index 来找到查询需要的 tuples。使用哪个 index 取决于以下几个因素：

- index 包含哪些 attributes
- 查询引用了哪些 attributes
- attribute 的定义域
- predicate composition
- index 的 key 是 unique 还是 non-unique

这些问题都将在后面的课程中详细描述，本节只是对 Index Scan 作概括性介绍。

尽管选择哪个 Index 取决于很多因素，但其核心思想就是，越早过滤掉越多的 tuples 越好，如下面这个 query 所示：

```SQL
SELECT * FROM students
 WHERE age < 30 AND dept = 'CS'AND country = 'US';
```

students 在不同 attributes 上的分布可能如下所示：

Scenario #1：30 岁以下的人有 99 人，但 CS 部门只有 2 人。

Scenario #2：CS 部门有 99 人，但 30 岁以下的只有 2 人。

对于 Scenario1，使用 dept 的 index 能过滤掉更多的 tuples；对于 Scenario 2，使用 country 的 index 能过滤掉更多的 tuples。

#### **Multi-index Scan**

如果有多个 indexes 同时可以供 DBMS 使用，就可以做这样的事情：

- 计算出符合每个 index 的 tuple id sets
- 基于 predicates (union vs. intersection) 来确定是对集合取交集还是并集
- 取出相应的 tuples 并完成剩下的处理

仍然以上一个 SQL 为例，如果我们 在age 和 dept 上建立索引，使用 multi-index scan 的过程如下所示：

![img](D:\Notes\images\1684841475843-82.png)

其中取集合交集可以使用 bitmaps, hash tables 或者 bloom filters。

Postgres 称 multi-index scan 为 Bitmap Scan。

#### **Index Scan Page Sorting**

当使用的不是 clustering index 时，按 index 顺序检索的过程是非常低效的，因为 DBMS 很有可能需要不断地在不同的 pages 之间来回切换，这会导致许多没有必要的I/O。为了解决这个问题，DBMS 通常会先找到所有需要的 tuples，根据它们的 page id 来排序，排序完毕后再读取 tuples 数据，这可让整个过程中访问每个需要的 page 只进行一次I/O。如下图所示：

<img src="D:\Notes\images\1684841478664-85.png" alt="img" style="zoom:67%;" />

## **4 Modification Queries**

修改数据库的 operator (INSERT， UPDATE, DELETE) 需要负责检查约束和更新索引。

- update，delete operator 要求 child operator 传递目标元组的 Record id 过来，由自己去修改数据。该过程需要记录修改过哪些记录，因为更新操作更改了元组的物理位置，导致扫描操作符可能会多次访问该元组。**假设我要为所有工资低于 1000 的人涨 100 元工资，如果没有记录修改了哪些数据，则一个目前 300 元工资的人会被修改 7 次，因为每一次修改后他的工资都低于 1000，继续往后遍历时还会遍历到他。**—— Halloween Problem
- Insert operator 的实现有两种：
  - 在本 operator 内物化然后进行插入，例如，直接 insert (xxx, yyy, zzz)
  - 从 child operator 中获取数据，然后在本 operator 中插入，例如，从其他表中获取数据，进行修改或者组装后插入本表

## **5 Expression Evaluation**

DBMS 使用 expression tree 来表示一个 WHERE 语句。

树中的节点代表不同的表达式类型。

- 比较 Comparisons (=, <, >, !=)
- 与 Conjunction (AND), 或 Disjunction (OR)
- 算术运算符 Arithmetic Operators (+, -, *, /, %)
- 常数 Constant Values
- 元组属性引用 Tuple Attribute References

![img](D:\Notes\images\1684841487099-88.png)

然后根据 expression tree 完成数据过滤的判断，但这个过程比较低效，很多 DBMS 采用 JIT Compilation 的方式，直接将比较的过程编译成机器码来执行，提高 expression evaluation 的效率。

> Expression tree 虽然很灵活，但速度很慢。