
# 使用Explain优化SQL语句


> 原文出处：<https://my.oschina.net/liughDevelop/blog/1788148>

索引类似大学图书馆建书目索引，可以提高数据检索的效率，降低数据库的IO成本。MySQL在300万条记录左右性能开始逐渐下降，虽然官方文档说500~800w记录，所以大数据量建立索引是非常有必要的。MySQL提供了Explain，用于显示SQL执行的详细信息，可以进行索引的优化。

## 一、导致SQL执行慢的原因：

1. 硬件问题。如网络速度慢，内存不足，I/O吞吐量小，磁盘空间满了等。
2. 没有索引或者索引失效。（一般在互联网公司，DBA会在半夜把表锁了，重新建立一遍索引，因为当你删除某个数据的时候，索引的树结构就不完整了。所以互联网公司的数据做的是假删除.一是为了做数据分析,二是为了不破坏索引 ）
3. 数据过多（分库分表）
4. 服务器调优及各个参数设置（调整my.cnf）


## 二、分析原因时，一定要找切入点：

1. 先观察，开启慢查询日志，设置相应的阈值（比如超过3秒就是慢SQL），在生产环境跑上个一天过后，看看哪些SQL比较慢。
2. Explain和慢SQL分析。比如SQL语句写的烂，索引没有或失效，关联查询太多（有时候是设计缺陷或者不得以的需求）等等。
3. Show Profile是比Explain更近一步的执行细节，可以查询到执行每一个SQL都干了什么事，这些事分别花了多少秒。
4. 找DBA或者运维对MySQL进行服务器的参数调优。

## 三、什么是索引？

MySQL官方对索引的定义为：索引(Index)是帮助MySQL高效获取数据的数据结构。我们可以简单理解为：**快速查找排好序的一种数据结构**。Mysql索引主要有两种结构：B+Tree索引和Hash索引。我们平常所说的索引，如果没有特别指明，一般都是指B树结构组织的索引(B+Tree索引)。索引如图所示：

![](https://raw.githubusercontent.com/nezha/picdb/master/20201011234722.jpg)

最外层浅蓝色磁盘块1里有数据17、35（深蓝色）和指针P1、P2、P3（黄色）。P1指针表示小于17的磁盘块，P2是在17-35之间，P3指向大于35的磁盘块。真实数据存在于子叶节点也就是最底下的一层3、5、9、10、13......非叶子节点不存储真实的数据，只存储指引搜索方向的数据项，如17、35。

查找过程：例如搜索28数据项，首先加载磁盘块1到内存中，发生一次I/O，用二分查找确定在P2指针。接着发现28在26和30之间，通过P2指针的地址加载磁盘块3到内存，发生第二次I/O。用同样的方式找到磁盘块8，发生第三次I/O。

真实的情况是，上面3层的B+Tree可以表示上百万的数据，上百万的数据只发生了三次I/O而不是上百万次I/O，时间提升是巨大的。

## 四、Explain分析

前文铺垫完成，进入实操部分，先来插入测试需要的数据：

```sql
CREATE TABLE `user_info` (
	`id` BIGINT (20) NOT NULL AUTO_INCREMENT,
	`name` VARCHAR (50) NOT NULL DEFAULT '',
	`age` INT (11) DEFAULT NULL,
	PRIMARY KEY (`id`),
	KEY `name_index` (`name`)
) ENGINE = INNODB DEFAULT CHARSET = utf8;

INSERT INTO user_info (NAME, age) VALUES ('xys', 20);

INSERT INTO user_info (NAME, age) VALUES ('a', 21);

INSERT INTO user_info (NAME, age) VALUES ('b', 23);

INSERT INTO user_info (NAME, age) VALUES ('c', 50);

INSERT INTO user_info (NAME, age) VALUES ('d', 15);

INSERT INTO user_info (NAME, age) VALUES ('e', 20);

INSERT INTO user_info (NAME, age) VALUES ('f', 21);

INSERT INTO user_info (NAME, age) VALUES ('g', 23);

INSERT INTO user_info (NAME, age) VALUES ('h', 50);

INSERT INTO user_info (NAME, age) VALUES ('i', 15);

CREATE TABLE `order_info` (
	`id` BIGINT (20) NOT NULL AUTO_INCREMENT,
	`user_id` BIGINT (20) DEFAULT NULL,
	`product_name` VARCHAR (50) NOT NULL DEFAULT '',
	`productor` VARCHAR (30) DEFAULT NULL,
	PRIMARY KEY (`id`),
	KEY `user_product_detail_index` (
		`user_id`,
		`product_name`,
		`productor`
	)
) ENGINE = INNODB DEFAULT CHARSET = utf8;

INSERT INTO order_info (user_id, product_name, productor) VALUES (1, 'p1', 'WHH');

INSERT INTO order_info (user_id, product_name, productor) VALUES (1, 'p2', 'WL');

INSERT INTO order_info (user_id, product_name, productor) VALUES (1, 'p1', 'DX');

INSERT INTO order_info (user_id, product_name, productor) VALUES (2, 'p1', 'WHH');

INSERT INTO order_info (user_id, product_name, productor) VALUES (2, 'p5', 'WL');

INSERT INTO order_info (user_id, product_name, productor) VALUES (3, 'p3', 'MA');

INSERT INTO order_info (user_id, product_name, productor) VALUES (4, 'p1', 'WHH');

INSERT INTO order_info (user_id, product_name, productor) VALUES (6, 'p1', 'WHH');

INSERT INTO order_info (user_id, product_name, productor) VALUES (9, 'p8', 'TE');
```

初体验，执行Explain的效果：

![](http://www.wailian.work/images/2018/11/11/640.png)

索引使用情况在possiblekeys、key和keylen三列，接下来我们先从左到右依次讲解。

#### **1.id**

```sql
--id 相同,执行顺序由上而下 
EXPLAIN SELECT u.*, o.* FROM user_info u, order_info o WHERE u.id = o.user_id;
```

![](http://www.wailian.work/images/2018/11/11/640-2.png)

```sql
--id 不同, 值越大越先被执行 
EXPLAIN SELECT * FROM user_info WHERE id = ( SELECT user_id FROM order_info WHERE product_name = 'p8' );
```
![](http://www.wailian.work/images/2018/11/11/640-3.png)
#### **2.select_type**

可以看id的执行实例，总共有以下几种类型：

- SIMPLE： 表示此查询不包含 UNION 查询或子查询
- PRIMARY： 表示此查询是最外层的查询
- SUBQUERY： 子查询中的第一个 SELECT
- UNION： 表示此查询是 UNION 的第二或随后的查询
- DEPENDENT UNION： UNION 中的第二个或后面的查询语句, 取决于外面的查询 
- UNION RESULT, UNION 的结果
- DEPENDENT SUBQUERY: 子查询中的第一个 SELECT, 取决于外面的查询. 即子查询依赖于外层查询的结果.
- DERIVED：衍生，表示导出表的SELECT（FROM子句的子查询）

#### **3.table**

table表示查询涉及的表或衍生的表：

```sql
EXPLAIN SELECT tt.* FROM ( SELECT u.* FROM user_info u, order_info o WHERE u.id = o.user_id AND u.id = 1 ) tt
```

![](http://www.wailian.work/images/2018/11/11/640.jpg)

id为1的表是id为2的u和o表衍生出来的。

#### **4.type**

type 字段比较重要，它提供了判断查询是否高效的重要依据依据。 通过 type 字段，我们判断此次查询是 全表扫描 还是 索引扫描等。

![](http://www.wailian.work/images/2018/11/11/640-2.jpg)

type 常用的取值有:

- `system`: 表中只有一条数据， 这个类型是特殊的 const 类型。 const: 针对主键或唯一索引的等值查询扫描，最多只返回一行数据。 const 查询速度非常快， 因为它仅仅读取一次即可。例如下面的这个查询，它使用了主键索引，因此 type 就是 const 类型的：explain select * from user_info where id = 2；
- `eqref`: 此类型通常出现在多表的 join 查询，表示对于前表的每一个结果，都只能匹配到后表的一行结果。并且查询的比较操作通常是 =，查询效率较高。例如：explain select * from userinfo, orderinfo where userinfo.id = orderinfo.userid;
- `ref`: 此类型通常出现在多表的 join 查询，针对于非唯一或非主键索引，或者是使用了 最左前缀 规则索引的查询。例如下面这个例子中， 就使用到了 ref 类型的查询：explain select * from userinfo, orderinfo where userinfo.id = orderinfo.userid AND orderinfo.user_id = 5
- `range`: 表示使用索引范围查询，通过索引字段范围获取表中部分数据记录。这个类型通常出现在 =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, IN() 操作中。例如下面的例子就是一个范围查询：explain select * from user_info  where id between 2 and 8；
- `index`: 表示全索引扫描(full index scan)，和 ALL 类型类似，只不过 ALL 类型是全表扫描，而 index 类型则仅仅扫描所有的索引， 而不扫描数据。index 类型通常出现在：所要查询的数据直接在索引树中就可以获取到, 而不需要扫描数据。当是这种情况时，Extra 字段 会显示 Using index。
- `ALL`: 表示全表扫描，这个类型的查询是性能最差的查询之一。通常来说， 我们的查询不应该出现 ALL 类型的查询，因为这样的查询在数据量大的情况下，对数据库的性能是巨大的灾难。 如一个查询是 ALL 类型查询， 那么一般来说可以对相应的字段添加索引来避免。

通常来说, 不同的 type 类型的性能关系如下：`ALL < index < range ~ indexmerge < ref < eqref < const < system` ALL 类型因为是全表扫描， 因此在相同的查询条件下，它是速度最慢的。而 index 类型的查询虽然不是全表扫描，但是它扫描了所有的索引，因此比 ALL 类型的稍快.后面的几种类型都是利用了索引来查询数据，因此可以过滤部分或大部分数据，因此查询效率就比较高了。

#### **5.possible_keys**

它表示 mysql 在查询时，可能使用到的索引。 注意，即使有些索引在 possible_keys 中出现，但是并不表示此索引会真正地被 mysql 使用到。 mysql 在查询时具体使用了哪些索引，由 key 字段决定。

#### **6.key**

此字段是 mysql 在当前查询时所真正使用到的索引。比如请客吃饭,possible_keys是应到多少人，key是实到多少人。当我们没有建立索引时：

```sql
EXPLAIN SELECT o.* FROM order_info o WHERE o.product_name = 'p1' AND o.productor = 'whh'; 
CREATE INDEX idx_name_productor ON order_info (productor);
DROP INDEX idx_name_productor ON order_info;
```

![](http://www.wailian.work/images/2018/11/11/640-3.jpg)

建立复合索引后再查询：

![](http://www.wailian.work/images/2018/11/11/640-4.jpg)

#### **7.key_len**

表示查询优化器使用了索引的字节数，这个字段可以评估组合索引是否完全被使用。

#### **8.ref**

这个表示显示索引的哪一列被使用了，如果可能的话,是一个常量。前文的type属性里也有ref，注意区别。

![](https://raw.githubusercontent.com/nezha/picdb/master/20201011234644.jpg)

#### **9.rows**

rows 也是一个重要的字段，mysql 查询优化器根据统计信息，估算 sql 要查找到结果集需要扫描读取的数据行数，这个值非常直观的显示 sql 效率好坏， 原则上 rows 越少越好。可以对比key中的例子，一个没建立索引钱，rows是9，建立索引后，rows是4。

#### **10.extra**

![](https://raw.githubusercontent.com/nezha/picdb/master/20201011234625.png)

explain 中的很多额外的信息会在 extra 字段显示, 常见的有以下几种内容:

- `using filesort` ：表示 mysql 需额外的排序操作，不能通过索引顺序达到排序效果。一般有 using filesort都建议优化去掉，因为这样的查询 cpu 资源消耗大。
- `using index`：覆盖索引扫描，表示查询在索引树中就可查找所需数据，不用扫描表数据文件，往往说明性能不错。
- `using temporary`：查询有使用临时表, 一般出现于排序， 分组和多表 join 的情况， 查询效率不高，建议优化。
- `using where` ：表名使用了where过滤。

## 五、优化案例

```sql
EXPLAIN SELECT u.*, o.* FROM user_info u LEFT JOIN order_info o ON u.id = o.user_id;
```

执行结果，type有ALL，并且没有索引：

![](https://raw.githubusercontent.com/nezha/picdb/master/20201011234604.png)

开始优化，在关联列上创建索引，明显看到type列的ALL变成ref，并且用到了索引，rows也从扫描9行变成了1行：

![](https://raw.githubusercontent.com/nezha/picdb/master/20201011234549.png)

这里面一般有个规律是：左连接索引加在右表上面，右连接索引加在左表上面。

## 六、是否需要创建索引？   

索引虽然能非常高效的提高查询速度，同时却会降低更新表的速度。实际上索引也是一张表，该表保存了主键与索引字段，并指向实体表的记录，所以索引列也是要占用空间的。

![](https://raw.githubusercontent.com/nezha/picdb/master/20201011234504.jpg)

## 参考文献

[【mysql优化专题】本专题终极总结（共13篇）](https://mp.weixin.qq.com/s/lRdwG8vlJeXrtTn5gULyeg)
