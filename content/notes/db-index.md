+++
title = "数据库索引笔记"
date = 2018-08-04T10:18:55+08:00
tags = ["数据库"]
categories = [""]
draft = false
+++


### 表和索引的物理结构

表和索引行都存储在页中。页的大小一般为4KB/8KB，页的大小决定了一个页可以存放多少个索引行、表行。

对于一个唯一索引（如主键索引），一个索引行等同叶子页中的一个索引条目，字段的值从表中复制到索引上，并加上一个指向表中记录的指针。

### 聚集索引和辅助索引

- 聚集索引：按照每张表的主键构造一颗B+树，同时叶子节点中存放的即为整张表的记录数据。
- 辅助索引：叶子结点存放的是索引数据以及主键索引值。由于检索数据时，总是先获取到书签值(主键值)，再返回查询，因此辅助索引也被称之为二级索引。
- 覆盖索引：一个查询语句的执行只需要从辅助索引中就可以得到查询记录，而不需要查询聚集索引中的记录。

### 索引与匹配列/过滤列

存在索引

```shell
(A, B, C, D)
```

查询

```sql
WHERE A = :A
	AND B > :B
	AND C = :C
```

从开头到结尾依次检查索引列：

1. 在`WHERE`子句中是否至少拥有一个足够简单的谓词与之对应，如果有，那么这个列就是匹配列，如果没有，那么这个列和后面的索引都是非匹配列。
2. 如果该谓词是一个范围谓词，那么剩余的索引列都是非匹配列。
3. 对于最后一个匹配列之后的索引列，如果拥有一个足够简单的谓词与之对应，那么该列为过滤列。

举例：`A`为等值谓词且为一个匹配列，因为`B`是范围谓词，所以`C`为非匹配列，但作为过滤列。



### 过滤因子

过滤因子=结果集的数量/表行的数量

最大过滤因子=最大结果集的数量/表行的数量

最大过滤因子表示了最差情况下的过滤因子，因此更具有代表性。



组合谓词的过滤因子

举例：

```sql
CITY = :CITY AND LNAME = :LNAME
```

如果`CITY`有500个不同的值，`LNAME`有10000个不同的纸，组合谓词的过滤因子就是`1/500 x 1/10000`。

组合谓词的过滤因子：谓词的最大过滤因子 x 谓词的最大过滤因子。

过滤因子决定了索引片的厚度。



### 选择率

选择率公式

```shell
选择率 = 100 x (某个键值对应的行数)/表的总记录数
```

举例：

如果CUST表有1,000,000行记录，列`CITY`为HELSINKI的行数为200,000行，那么HELSINKI的选择率为20%。



### 不合适的索引

QUERY

```sql
SELECT	CNO, FNAME
FROM	CUST
WHERE	LNAME = :LNAME
		AND
		CITY = :CITY
ORDER BY FNAME
```

1. 索引扫描（LNAME，FNAME）
2. 全表扫描

对于第一个选择，对于索引片中每一个索引行，都需要回表检查`CITY`的值，由于**表中的行识根据CNO而不是LNAME字段来聚簇的**，所以每个校验操作需要做一次硬盘的随机读。

假如顺序读的速度为40MB/s，一次随机读需要花费10ms，索引`（LNAME，FNAME）`大小为100MB，10,000次随机读将花费10,000 x 10ms = 100s，读取一个宽度为1%的索引片（1MB）需要花费在IO的时间是10ms + 1MB/40MB/s = 35ms。

对于全表扫描，表的大小为600MB，只有第一个页需要随机读，花费在IO的时间则是10ms + 600MB/40MB/s = 15s。

因此对于不适合的索引，查询效率可能比全表扫描更低。



### 三星索引

- 第一颗星：取出所有等值谓词的列（WHERE COL=...）作为索引的最开头，任意顺序都可以。在这种情况下，索引片的宽度被缩减到最窄。
- 第二颗星：将ORDER BY列加入到索引中，不要改变这些列的顺序，但是忽略那些在第一步中已经加入索引的列。
- 第三颗星：将查询语句剩余的列加入到索引中去，列在索引中添加的顺序对查询语句的性能没有影响。

举例：

```sql
SELECT	CNO, FNAME
FROM	CUST
WHERE	LNAME BETWEEN :LNAME1 AND :LNAME2
		AND
		CITY = :CITY
ORDER BY FNAME
```

满足第三颗星：确保查询语句中的所有列都在索引中。

满足第二颗星：添加ORDER BY列，但只有`FNAME`（ORDER BY列）放在`BETWEEN`谓词列`LNAME`之前才成立，如索引（CITY，FNAME，LNAME），因为`CITY`是等值谓词，所以结果以`FNAME`的顺序排列，不需要额外排序，但如果`FNAME`在`LNAME`后面，如（CITY，LNAME，FNAME），则需要进行排序操作。

满足第一颗星：等值谓词`CITY`作为索引的最开头，列的过滤因子越低，索引片越窄。如果使用索引（CITY，LNAME）的话，索引片会更窄，但为了这样FNAME就不能放在这两列之间。

因此这里无法同时满足第一颗星与第二颗星，只能二选一：

- 避免排序----拥有第二颗星。（CITY，FNAME，LNAME，CNO）
- 拥有更窄的索引片----拥有第一颗星。（CITY，LNAME，FNAME，CNO）



### 评估索引

基本问题法（BQ）：

> 是否有一个已存在的或者计划中的索引包含了WHERE子句所应用的所有列（一个半宽索引）？

- 如果答案为否，首先应考虑将**缺少的谓词**加到一个现有的索引上去，尽管索引等值匹配过程不令人满意（一星），但可索引过滤后有效降低需要回表访问的数量。
- 将所有涉及的列都加到索引上，这产生一个宽索引，避免回表访问。



快速上限估算法（QUBE）：

>  LRT = 服务时间 + 排队时间 = 本地响应时间
>
> TR = 随机访问的数量
>
> TS = 顺序访问的数量
>
> F = 有效FETCH的数量

举例：

假如一次随机读花费10ms，一次顺序读花费0.01ms。

聚簇索引访问：

```sql
SELECT	CNO, LNAME, FNAME
FROM	CUST
WHERE	ZIP = :ZIP
		AND
		LNAME = :LNAME
ORDER BY FNAME
```

索引列为（ZIP，LNAME，FNAME）

假如ZIP=30103的地区有1000位名为Joneses的客户：

```shell
索引	ZIP, LNAME, FNAME	TR = 1		TS = 1000
表	    CUST				TR = 1	    TS = 999
提取 1000 x 0.1 ms

LRT			TR = 2	  	TS = 1999
			2 x 10 ms   1999 x 0.01 ms
			20 ms + 20 ms + 100 ms = 140 ms
```

非聚簇索引访问：

```shell
索引	ZIP, LNAME, FNAME	TR = 1		TS = 1000
表	    CUST				TR = 1000   TS = 0
提取 1000 x 0.1 ms

LRT				TR = 1001	  	 TS = 1000
				1001 x 10 ms   1000 x 0.01 ms
				10 s + 10 ms + 100 ms = 10s
```

添加聚簇列`CNO`使索引变成三星索引：

```shell
索引	ZIP, LNAME, FNAME, CNO	TR = 1		TS = 1000
表	CUST					 TR = 0   	 TS = 0
提取 1000 x 0.1 ms

LRT				TR = 1	  	 TS = 1000
				1 x 10 ms    1000 x 0.01 ms
				10 ms + 10 ms + 100 ms = 120ms
```

对于查询：

```sql
SELECT	CNO, FNAME
FROM	CUST
WHERE	LNAME = :LNAME
		AND
		CITY = :CITY
ORDER BY FNAME
```

QUBE分析如下：

三星索引：

```shell
索引 CITY, LNAME, FNAME, CNO	TR = 1		TS = 1% x 10% x 1000000
提取 1000 x 0.1 ms

LRT				TR = 1	  	 TS = 1000
				1 x 10 ms    1000 x 0.01 ms
				10 ms + 10 ms + 100 ms = 120ms
```

半宽索引

```shell
索引 CITY, LNAME, FNAME, CNO	TR = 1		      TS = 1% x 1000000
表   CUST					 	TR = 10% x 10000  TS = 0
提取 1000 x 0.1 ms

LRT				TR = 1001	  	 TS = 10000
				1001 x 10 ms     10000 x 0.01 ms
				10 s + 0.1 s + 0.1 s = 10s
```

宽索引

```shell
索引 LNAME, FNAME, CITY, CNO	TR = 1		      TS = 1% x 1000000
表   CUST					    TR = 0			  TS = 0
提取 1000 x 0.1 ms

LRT				TR = 1	  	 TS = 10000
				1 x 10 ms    10000 x 0.01 ms
				10 ms + 0.1 s + 0.1 s = 0.2s
```



### 分页查询的优化

- 覆盖索引
- 通过索引消除排序

```sql
SELECT * FROM table inner join (select pk from table order by created desc limit N) as t using(pk);
```

