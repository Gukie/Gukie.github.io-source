---
title: 使用松散索引优化SQL
categories: 
    - [MySQL, SQL优化]

tags: 
    - SQL优化

excerpt: SQL优化有很多种，比如尽量按需获取字段以节省网络带宽; 再比如尽量在where中排列的字段顺序命中索引，最好可以走覆盖索引。本文将介绍下在SQL过程中松散索引的使用

---

<a name="rEULd"></a>
### refer
- [https://dev.mysql.com/doc/refman/5.7/en/group-by-optimization.html](https://dev.mysql.com/doc/refman/5.7/en/group-by-optimization.html) (松散索引的官方解释)



<a name="yjrHT"></a>
### 背景

<br />对于物联网行业来说，当业务增长之后，其对应的数据增长的速度是非常快的，因为设备每时每刻都在产生数据，这样某些表上的数据已经悄悄的上了千万级别，这时候就急需对其做一定的优化。其中最常见的就是水平拆分了。拆分之前第一步要做的就是数据统计分析，而由于数据量级摆在这儿，所以在统计的时候SQL执行超时就很常见了，这时候就需要做优化<br />
<br />
<br />假设某个表记录的设备的上报记录，而且该表由于历史原因存储的不止一种业务数据，其的结构如下:
```sql
CREATE TABLE `dev_history` (
`id` bigint(20) NOT NULL AUTO_INCREMENT,
`dev_id` varchar(64) NOT NULL COMMENT '设备id',
`value` varchar(64) NOT NULL DEFAULT '' COMMENT 'value',
`status` smallint(1) NOT NULL COMMENT '状态',
`gmt_create` bigint(20) NOT NULL,
`gmt_modified` bigint(20) NOT NULL,
PRIMARY KEY (`id`),
UNIQUE KEY `idx_devid` (`dev_id`,)
) 
```
现在希望将某个业务的数据统计出来，然后做拆分。统计分析的工作主要分两块:

- 将每条设备有多少条记录找出来
- 对每个设备进行分析，分析其是否是你期望的业务

有了上面两个数据，就可以知道知道数据增长的趋势，从而对如何水平拆分提供参考.<br />
<br />

<a name="FSoQK"></a>
### 经过
对于统计每个设备的有多少数量，一开始SQL写成如下这样
```sql
select dev_id, count(id) as count from dev_history group by dev_id;
```

<br />从执行计划看，走的是索引。嗯，感觉还不错，应该会挺快的。然而由于设备的数量有几十万条，所以即使走索引也会很慢，而且在统计的时候不可能一次性将几十万的数据都捞出来，统计程序会直接进ICU的<br />![image.png](1.png)<br />
<br />那怎么办，只能limit了，改造之后的SQL如下
```sql
select dev_id as devId, count(id) as count from dev_history group by dev_id order by dev_id limit 0, 5000
```
以上SQL跑起来之后还是很容易超时:

- limit的offset到后面的时候会很大，虽然走的是索引但是还是会遍历很多无用的数据. 对于这种问题，一般会想到的做范围限制，但是由于每个设备无时无刻都在生产数据，所以他们之间不存在先后顺序(即id和gmtCreate都不适用)，所以无法做范围限制. 
- 有个小问题是: 由于group by默认会对作用的字段做排序，所以这里就不需要做 order by了


<br />

<a name="IgePG"></a>
### 最终解决方案
最后的解决方案是: 通过松散索引来解决。松散索引是跟紧凑索引相对的。

- 紧凑索引，是走了索引，一般是查找范围查找或者是 全索引查找
> A Tight Index Scan may be either a full index scan or a range index scan, depending on the query conditions. When the conditions for a Loose Index Scan are not met, it still may be possible to avoid creation of temporary tables for `GROUP BY` queries



- 所谓松散索引，是在做数据统计的时候不是遍历索引，而是会跳过相同的数据。而且一般在group by的时候会使用到。比如MySQL在group by作用的所有的字段都在索引上，那么它就会走松散索引。我们这里的例子就是 dev_id是有索引的
> The most efficient way to process `GROUP BY` is when an index is used to directly retrieve the grouping columns. With this access method, MySQL uses the property of some index types that the keys are ordered (for example, `BTREE`). This property enables use of lookup groups in an index without having to consider all keys in the index that satisfy all `WHERE` conditions. This access method considers only a fraction of the keys in an index, so it is called a Loose Index Scan. When there is no `WHERE` clause, a Loose Index Scan reads as many keys as the number of groups, which may be a much smaller number than that of all keys. If the `WHERE` clause contains range predicates (see the discussion of the [`range`](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#jointype_range) join type in [Section 8.8.1, “Optimizing Queries with EXPLAIN”](https://dev.mysql.com/doc/refman/5.7/en/using-explain.html)), a Loose Index Scan looks up the first key of each group that satisfies the range conditions, and again reads the smallest possible number of keys


<br />
<br />
<br />所以可以对我们的SQL做一定的改造

- 第一步中只获取不重复的dev_id
- 第二步，再通过dev_id去获取对应的count数据。因为每个dev_id的记录数据不会很大，最多几十万，并且它有索引所以查询起来还是比较快的

这样以来，就可以减轻每一步的压力，还能获取到对应的数据。对于第一步，获取不重复的dev_id的SQL如下
```sql
select dev_id from dev_history  where 1=1 group by dev_id;
```

<br />对应的执行计划如下, 其中 "**Using index for group-by**" 表示的就是当前SQL执行时使用的是松散索引<br />![image.png](2.png)<br />
<br />
<br />


