---
layout: post
title: 优化 B-tree 的 range scan 性能
tags: [database]
---

欢迎关注我的微信公众号：黄金架构师。

---

最近，我在工作中遇到了一个性能有问题的场景：客户的某种查询负载的 TPS 不满足他们的预期。经排查发现，瓶颈在二级索引 B-tree 的 range scan 上。老板知晓此事后，郑重其事的告诉我，赶紧把 B-tree 的 range scan 性能优化一下，这个客户是我们最头部的客户，要是优化不好，客户不满意，你就先回家休息几天吧！（开玩笑 :)

我很害怕，我不想回家休息。于是我打算先写下此文，厘清一下自己的思路，同时和各位读者探讨一下，然后再拿着方案去和老板 solo，最后再搞定客户。

先还原一下场景（其实是简化版）便于大家置身事内。

客户的场景：
1. scan-heavy，并发高，极少点查/点更新
2. 数据量非常大，查询分布均匀，内存不大，buffer pool 经常不命中，不得不磁盘读取
3. 只涉及一个非唯一的二级索引。索引中会有大量重复的 key（key 相同，value 不同），大多数 key 重复上千次，有的甚至上万次
4. 所有查询都是针对这个二级索引的 index only scan（有的数据库叫 covering index）。（换句话说，要查的数据都在索引上，不用回表查数据。）
5. 所有查询都是针对这个二级索引的等值查询（只查一个 key）。尽管如此，由于 key 是可重复的，等值查询到 B-tree 中就成了 range scan
经过一顿毛利毛糙的分析，确定下来这个场景的瓶颈确实是在二级索引 B-tree 的 range scan 上。range scan 扫描大量数据，要读取大量叶子结点，叶子结点的 buffer pool 命中率低，每读一个叶子结点会有一次磁盘读 IO，客户的环境用的是恶劣的磁盘而不是 SSD，性能很差，这里就成了明显的瓶颈了。

那么，如何优化呢？我想到了三种方案：

1. B-tree 结点的 page size 调大
2. B-tree 结点压缩
3. B-tee 结点 prefetch
方案 1 和方案 2 其实是一个思路，即通过减少需要读取的叶子结点的数量，来减少磁盘的随机读次数，以提升性能。

**方案 1 B-tree 结点的 page size 调大**，让每个叶子结点有更大的容量，需要读的叶子结点数量减少，磁盘随机读数量减少，性能得到提升。这个方案优点是简单实用，不用改一行代码，改一下配置就行。缺点是增加了点查/点更新的 IO 放大。幸好客户环境点查/点更新极少，这个缺点我们就可以忽略了。

**方案 2 B-tree 结点压缩**。我们的二级索引不是唯一的，索引中有大量重复的 key。基于这个特点，我们可以对 B-tree 结点做 prefix compression，进而节省掉重复 key 占据的空间，使得一个叶子结点能放下更多的 KV 记录，叶子结点数量减少，磁盘随机读数量减少，性能得到提升。

更进一步，其实我们还可以在 prefix compression 的基础上对 B-tree 结点做 dictionary compression，让叶子结点可以容纳更多的 KV，进一步减少要读取的叶子结点的数量，以提升性能。

**方案 3 B-tree 结点 prefetch**。prefetch 是常见的用来隐藏磁盘 latency 的一个手段。一般来说，有两种实现方式：
1. 若要读的数据在物理上连续，一次读取连续多个的 page
2. 若要读的数据在物理上不连续，并发读取多个 page（依赖于底层硬盘的并发 IO（要么是多块磁盘 + RAID，要么是 SSD））
方式 1 显然不适合我们这个二级索引的场景。二级索引不像有些单调递增的聚簇索引，数据顺序插入，逻辑上相邻的叶子结点在物理上一般也是连续的（除非非叶子结点 page split），二级索引的数据都是无序随机插入的，各个结点频繁 split，相邻的叶子结点物理上很可能不连续。

方式 2 适合二级索引的场景。不过它实现起来更复杂一些，因为叶子结点是以链表的形式组织的，我们读取一个叶子结点的时候，无法预知后续 N 个叶子结点的信息，就不能并发读取。我能想到的实现方案是，把叶子结点的上面一层父亲结点也连接成链表，在 scan 叶子结点前，先 scan 父亲结点，收集到要扫描的所有叶子结点，然后一个线程去执行 scan 叶子结点，同时其他线程并发 prefetch 后续要 scan 的其他结点。由于要 scan 父亲结点的数量比叶子结点的数量少两个数量级（fan-out 为数百），所以 scan 父亲结点的开销相对很小，可以接受。

此外，关于方案 3，我后来在网上找了一些资料，还看到了另外一种实现：[Fractal Prefetching B+-Trees](https://www.pdl.cmu.edu/PDL-FTP/Database/fpbtree.pdf)。不过我还没有仔细看，改天有时间了再研究一下。

回归主题，现在这 3 个方案已经列出来了。嗯，我感觉都还不错，接下来我就拿着这 3 个方案去找老板聊一聊吧，让老板做一个选择题就成。

---

更新：靠，老板说了，这三个方案，我全都要！。。。