# Lecture #10: Sorting & Aggregation Algorithms

接下来将学习使用我们现在学习的 DBMS 组件来执行查询。

今天要讨论的算法都是基于 Disk 的，即查询的中间结果也需要存储到磁盘中。我们需要使用 Buffer Pool 去实现这些算法，要最大化磁盘连续 I/O。

<img src="D:\Notes\images\image-20230523190232703.png" alt="image-20230523190232703" style="zoom:67%;" />



`Query Plan`：算子组织成树形结构，数据从叶子节点流向根节点，根节点的输出就是查询的结果，我们将会在下节课讨论数据移动的粒度。

<img src="D:\Notes\images\image-20230523190447723.png" alt="image-20230523190447723" style="zoom:67%;" />

## 1.Sorting

DBMS需要对数据进行排序，因为在关系模型下，表中的tuple没有特定的顺序，排序在`ORDER BY`、`GROUP BY`、`JOIN`和`DISTINCT`操作符中可能使用。如果需要排序的数据在内存中可以放下，那么DBMS可以使用标准的排序算法（eg.快排）。如果放不下，则需要使用`external sorting`，能够根据需要溢出到磁盘，并且倾向于顺序而不是随机I/O。



如果一个查询包含一个带有`LIMIT`的`ORDER BY`，则DBMS只需要扫描一次数据就可以找到前N个元素。这就是所谓的 **`Top-N Heap Sort`**。堆排序的理想情况是前N个元素可以放在内存中，这样DBMS只需要维护一个内存中的堆排序优先队列即可。



对太大而无法装入内存的数据进行排序的标准算法是外部合并排序（`external merge sort`）。是一个分治排序算法，将数据集分成独立的运行，然后对它们进行单独排序。它们可以根据需要将runs溢出到磁盘中，然后一次读回它们。该算法分两个阶段：

- **Phase #1 - Sorting:**  首先该算法对适合主内存的小块数据进行排序，然后将排序后的页面写入磁盘。

- **Phase #2 - Merge:**  然后，将排序后的子文件合并为一个更大的单一文件。

### Two-way Merge Sort

最基本的版本是二路归并排序。该算法在排序阶段读取每个页面，对其排序，并将排序后的版本写回磁盘。然后在合并阶段，它使用3个缓冲页。它从磁盘中读取两个排序页，并将它们合并到第三个缓冲页中。 每当第三页被填满时，就会将第三页写回磁盘，并替换为一个空页。每一组排序的页面称为一个 run。该算法递归的将这些runs合并。（每个pass可以理解为每一个遍历）



如果N是数据页的总数，该算法在数据中一共进行了$1+\lceil\log_2N\rceil$次pass（1次用于第一个排序步骤，剩下的用于归并）。总是 IO成本是2N × (# of passes)。因为每一个pass对每一个页面都有一个IO读和写。



如下图，一开始一共有 8 个页，每个页是一个独立的 run，然后第一次遍历，也就是 pass0，先将每一个 run 都排好序；第二次遍历中，每次读取进两个相邻的长度为 2 的 run，然后进行合并，输出长度为 4 的排好序的 run（被放置在 2 个页中）；第三次遍历中，每次读取相邻两个长度为 4 的 run，然后进行合并，输出长度为 8 的排好序的 run（放置在 4 个页中）；第四次遍历中，将两个长度为 8 的run合并，最终生成长度为 16 的run（放置在 8 个页中），算法结束。

<img src="D:\Notes\images\1684840058783-5.png" alt="img" style="zoom:67%;" />

### General (K-way) Merge Sort

该算法的一般版本允许DBMS使用3个以上的缓冲页。 B表示缓冲页的总数，在排序阶段，该算法可以一次读取B页，并将$$\lceil\frac{N}{B}\rceil$$个排序好了的runs写回磁盘。合并阶段，可以在每个通道中合并最多 B-1 个runs，同样为合并后的数据使用一个缓冲页，并根据需要写回磁盘。

在一般的版本中，该算法要经历$$1+\lceil\log_{B-1}{\frac{N}{B}}\rceil$$个passes（1个是排序阶段，剩下是归并阶段）。然后，总共的IO花费是2N×(# of passes)。因为这个算法每一个pass对每一个page都需要一次读和写。

### **Double Buffering Optimization**

外部合并排序的一个优化是：在后台预取下一个run。并在系统处理当前运行时将其存储在第二个缓冲区。这通过不断利用磁盘来减少每一步的IO请求的等待时间。如在处理 page1 中的 run 时，同时把 page2 中的 run 放进内存。这种优化需要多线程。因为预取应该在当前运行的计算过程中进行。

![img](D:\Notes\images\1684840067139-8.png)

### **Using B+Tree**

如果我们要进行排序的属性，正好有一个构建好的B+树索引，那么可以直接使用B+树排序，而不是用外排序。

如果 B+ 树是 **聚簇B+树**，那么可以直接找到最左的叶子节点，然后遍历叶子节点，这样总比外排序好，因为没有计算消耗，所有的磁盘访问都是连续的，而且时间复杂度更低。

![img](D:\Notes\images\1684840204482-11.png)



如果 B+ 树是 **非聚簇B+树**，那么遍历树总是更坏的，因为每个 record 可能在不同的页中，所有几乎每一个 record 访问都需要磁盘读取。



![img](D:\Notes\images\1684840208990-14.png)

## 2. Aggregations

查询计划 聚合运算符将一个或多个tuple的值折叠成一个标量值。有两种实现聚合的方法：（1）排序（2）哈希

### **Sorting**

DBMS首先根据`GROUP BY`对tuples进行排序。如果所有数据都在缓冲池中，它可以使用内存中的排序算法（eg.快排）。如果数据大小比内存大，则可以使用外部合并排序算法。然后DBMS对排序后的数据进行顺序扫描来计算聚合。操作符的输出将在key上进行排序。

在执行排序聚合，重要是要对查询操作进行排序，以效率最大化。例如：如果查询需要一个过滤器（filter），最好先执行过滤器，然后对过滤后的数据进行排序，以减少数据量。



如下图，首先先进行 filter 操作，将 grade 是 B/C 的 tuple 筛选出来；然后进行投影操作，将无用的列去掉；然后进行排序，对排好序的结果进行聚集。



<img src="D:\Notes\images\1684840214697-17.png" alt="img" style="zoom:67%;" />

### **Hashing**

如果我们不需要数据最终是排好序的，如 `GROUP BY` 和 `DISTINCT` 算子，输出结果都不需要进行排序，那么 Hashing 是一个更好的选择，因为不需要排序，而且哈希计算更快。



在计算聚合时，散列的计算成本比排序低。DBMS在扫描表的时候会填充一个暂时的哈希表（`ephemeral hash table`）。对每一条记录，检查哈希表中是否已经有一个条目，并进行适当的修改。如果哈希表过大，无法容纳在内存，则DBMS可以将其溢出到磁盘上。有两个阶段来完成这个任务：

- **Phase #1 – Partition:** 使用哈希函数 h1，将元组划分成磁盘上的 partition，一个 partition 是指由同样 hash value 的 key 组成的一个或多个页。如果我们有 B 个 buffer 可以用，那么我们使用 B - 1 个 buffer 用来做 partition，剩下 1 个buffer 用来输入数据。

- **Phase #2 – ReHash:** 对于每一个在磁盘上的 partition，将它的页面 (可能是多个) 读进内存，并且用另一个哈希函数 h2(h1≠h2) 构建一个 in-memory 哈希表。之后遍历哈希表每个 bucket，并将相匹配的 tuples 计算 aggregation。(这里假设了一个 partition 可以被放进内存)

  

在`ReHash`阶段，DBMS可以存储形式为（`GroupByKey`→`RunningValue`）的配对，以计算聚合。`RunningValue`的内容取决于聚合函数。要在哈希表中插入一个新tuple。

- 如果它找到一个匹配的`GroupByKey`，那么就适当地更新`RunningValue`。

- 否则就插入一个新的（`GroupByKey`→`RunningValue`）对。

  

首先是 Partition，因为我们要做`DISTINCT`，所以将这些元组按照 cid 作为 key 进行 partition，我们将这些 key 分为了B - 1 个 partition，然后将这些 partition 放入磁盘。



<img src="D:\Notes\images\1684840305863-20.png" alt="img" style="zoom:67%;" />



然后进行ReHash，因为在上一个阶段，一个 partition 内可能有碰撞，所以读入 partition 进行二次哈希，注意图中显示的，第一次哈希的一个 partition 可能有好几页。最终形成结果。



<img src="D:\Notes\images\1684840336580-23.png" alt="img" style="zoom:67%;" />



根据聚集任务不同，RunningValue 可能不一样，如果我们要进行 `AVG` 聚集，那么 RunningValue 就是 (COUNT, SUM)。在 ReHash 阶段，每次对一个 tuple 进行哈希，都更改哈希表中的对应 key 的 RunningValue 值。



<img src="D:\Notes\images\1684840352694-26.png" alt="img" style="zoom:67%;" />



> 总结就是，如果有 filter 就先过滤。然后根据 `GROUP BY` 或 `DISTINCT` 的元素作为 key 使用哈希函数 h1 对 tuples 进行 partition 存入磁盘。然后依次读入 partition，通过哈希函数 h2 构建 in-memory 哈希表。然后遍历所有 buckets，计算 aggregation。



