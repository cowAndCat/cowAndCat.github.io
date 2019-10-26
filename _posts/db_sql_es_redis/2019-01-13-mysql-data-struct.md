---
layout: post
title: mysql的索引实现
category: mysql
comments: false
---

## 一、关于MySQL索引
强烈建议看一遍文章：[MySQL索引背后的数据结构及算法原理](http://blog.codinglabs.org/articles/theory-of-mysql-index.html)

特别需要说明的是，MySQL支持诸多存储引擎，而各种存储引擎对索引的支持也各不相同，因此MySQL数据库支持多种索引类型，如BTree索引，哈希索引，全文索引等等。为了避免混乱，本文将只关注于BTree索引。

### 1.1 一些基础

目前大部分数据库系统及文件系统都采用B-Tree或其变种B+Tree作为索引结构。

在B-Tree中按key检索数据的算法非常直观：首先从根节点进行二分查找，如果找到则返回对应节点的data，否则对相应区间的指针指向的节点递归进行查找，直到找到节点或找到null指针，前者查找成功，后者查找失败。

B-Tree有许多变种，其中最常见的是B+Tree，例如MySQL就普遍使用B+Tree实现其索引结构。

与B-Tree相比，B+Tree有以下不同点：每个节点的指针上限为2d而不是2d+1；内节点不存储data，只存储key；叶子节点不存储指针。

一般来说，B+Tree比B-Tree更适合实现外存储索引结构。

## 二、覆盖索引
覆盖索引是非常有用的工具，能够极大的提高性能（减少 IO 操作）。因为，只需要读取索引，而无需读表，极大减少数据访问量

可以通过explain关键字查看sql语句用到的索引

    explain sql 语句：
    extra（use index）有这个说明是覆盖索引

如果索引覆盖了 WHERE 条件中的字段，但不是整个查询涉及的字段
explain sql 语句会得到 extra（use where）
如果没有任何索引能覆盖查询的所有列，那么这个查询就需要回表（就是根据 where 条件重新查询一次数据，然后返回结果）。但是索引还是用到了

**覆盖索引必须要存储索引列的值**，而哈希索引、空间索引和全文索引等都不存储索引列的值，所以 MySQL 只能使用 B-Tree 索引做覆盖索引。并且不同的存储引擎实现覆盖索引都是不同的。并不是所有的存储引擎都支持它们。如果要使用覆盖索引。一定要注意 SELECT 列表值取出需要的列。不可以是 SELECT * ，因为如果将所有字段一起做索引会导致索引文件过大。

## 三、MySQL索引实现

### 3.1 MyISAM索引实现
MyISAM引擎使用B+Tree作为索引结构，叶节点的data域存放的是数据记录的地址。下图是MyISAM索引的原理图：

![MyISAM](/images/201901/myisam.png)

MyISAM的索引文件仅仅保存数据记录的地址。在MyISAM中，主索引和辅助索引（Secondary key）在结构上没有任何区别，只是主索引要求key是唯一的，而辅助索引的key可以重复。

MyISAM的索引方式也叫做“非聚集”的，之所以这么称呼是为了与InnoDB的聚集索引区分。

### 3.2 InnoDB索引实现
虽然InnoDB也使用B+Tree作为索引结构，但具体实现方式却与MyISAM截然不同。

第一个重大区别是InnoDB的数据文件本身就是索引文件。MyISAM索引文件和数据文件是分离的，索引文件仅保存数据记录的地址。而在InnoDB中，表数据文件本身就是按B+Tree组织的一个索引结构，这棵树的叶节点data域保存了完整的数据记录。这个索引的key是数据表的主键，因此InnoDB表数据文件本身就是主索引。

![InnoDB](/images/201901/innodb.png)

第二个与MyISAM索引的不同是InnoDB的辅助索引data域存储相应记录主键的值而不是地址。换句话说，InnoDB的所有辅助索引都引用主键作为data域。

例如，下图为定义在Col3上的一个辅助索引：

![InnoDB2](/images/201901/innodb2.png)

这里以英文字符的ASCII码作为比较准则。聚集索引这种实现方式使得按主键的搜索十分高效，但是辅助索引搜索需要检索两遍索引：首先检索辅助索引获得主键，然后用主键到主索引中检索获得记录。

### 3.3 常见问题解答
1.为什么不建议使用过长的字段作为主键？

如果了解InnoDB的索引构造就会知道答案，因为所有辅助索引都引用主索引，过长的主索引会令辅助索引变得过大。

2.为什么非单调的字段作为主键在InnoDB中不是个好主意？

因为InnoDB数据文件本身是一颗B+Tree，非单调的主键会造成在插入新记录时数据文件为了维持B+Tree的特性而频繁的分裂调整，十分低效，而使用自增字段作为主键则是一个很好的选择。

3.为什么联合索引要遵循最左前缀原理？

因为B+树结构，先按第一列排序，再按第二列排序，以此类推，中间不能跳。

4.在用InnoDB时，为什么建议永远使用一个与业务无关的自增字段作为主键？

如果表使用自增主键，那么每次插入新的记录，记录就会顺序添加到当前索引节点的后续位置，当一页写满，就会自动开辟一个新的页。由于每次插入时也不需要移动已有数据，因此效率很高，也不会增加很多开销在维护索引上。

如果使用非自增主键（如果身份证号或学号等），由于每次插入主键的值近似于随机，因此每次新纪录都要被插到现有索引页得中间某个位置，此时MySQL不得不为了将新记录插到合适位置而移动数据，甚至目标页面可能已经被回写到磁盘上而从缓存中清掉，此时又要从磁盘上读回来，这增加了很多开销，同时频繁的移动、分页操作造成了大量的碎片，得到了不够紧凑的索引结构，后续不得不通过OPTIMIZE TABLE来重建表并优化填充页面。

因此，只要可以，请尽量在InnoDB上采用自增字段做主键。

## 四、索引选择性与前缀索引

既然索引可以加快查询速度，那么是不是只要是查询语句需要，就建上索引？答案是否定的。因为索引虽然加快了查询速度，但索引也是有代价的：索引文件本身要消耗存储空间，同时索引会加重插入、删除和修改记录时的负担，另外，MySQL在运行时也要消耗资源维护索引，因此索引并不是越多越好。


### 4.1 索引选择性
一般两种情况下不建议建索引。

第一种情况是表记录比较少，例如一两千条甚至只有几百条记录的表，没必要建索引，让查询做全表扫描就好了。至于多少条记录才算多，这个个人有个人的看法，我个人的经验是以2000作为分界线，记录数不超过 2000可以考虑不建索引，超过2000条可以酌情考虑索引。

另一种不建议建索引的情况是索引的选择性较低。所谓索引的选择性（Selectivity），是指不重复的索引值（也叫基数，Cardinality）与表记录数（#T）的比值：

    Index Selectivity = Cardinality / #T

显然选择性的取值范围为(0, 1]，**选择性越高的索引价值越大，这是由B+Tree的性质决定的**。

用SQL语句来计算：

    > SELECT count(DISTINCT(title))/count(*) AS Selectivity FROM employees.titles;
    +-------------+
    | Selectivity |
    +-------------+
    |      0.0000 |
    +-------------+

### 4.2 前缀索引

有一种与索引选择性有关的索引优化策略叫做前缀索引，就是用列的前缀代替整个列作为索引key，当前缀长度合适时，可以做到既使得前缀索引的选择性接近全列索引，同时因为索引key变短而减少了索引文件的大小和维护开销。

看个例子：

    >SELECT count(DISTINCT(first_name))/count(*) AS Selectivity FROM employees.employees;
    +-------------+
    | Selectivity |
    +-------------+
    |      0.0042 |
    +-------------+
    >SELECT count(DISTINCT(concat(first_name, last_name)))/count(*) AS Selectivity FROM employees.employees;
    +-------------+
    | Selectivity |
    +-------------+
    |      0.9313 |
    +-------------+

`<first_name>`显然选择性太低，`<first_name, last_name>`选择性很好，但是first_name和last_name加起来长度为30，有没有兼顾长度和选择性的办法？**可以考虑用first_name和last_name的前几个字符建立索引**，例如`<first_name, left(last_name, 4)>`，看看其选择性：

    >SELECT count(DISTINCT(concat(first_name, left(last_name, 4))))/count(*) AS Selectivity FROM employees.employees;
    +-------------+
    | Selectivity |
    +-------------+
    |      0.9007 |
    +-------------+

这时选择性已经很理想了，而这个索引的长度只有18，比`<first_name, last_name>`短了接近一半，我们把这个前缀索引 建上：

    ALTER TABLE employees.employees
    ADD INDEX `first_name_last_name4` (first_name, last_name(4));

此时再执行一遍按名字查询，比较分析一下与建索引前的结果：

    SHOW PROFILES;
    +----------+------------+---------------------------------------------------------------------------------+
    | Query_ID | Duration   | Query                                                                           |
    +----------+------------+---------------------------------------------------------------------------------+
    |       87 | 0.11941700 | SELECT * FROM employees.employees WHERE first_name='Eric' AND last_name='Anido' |
    |       90 | 0.00092400 | SELECT * FROM employees.employees WHERE first_name='Eric' AND last_name='Anido' |
    +----------+------------+---------------------------------------------------------------------------------+

性能的提升是显著的，查询速度提高了120多倍。

**前缀索引兼顾索引大小和查询速度，但是其缺点是不能用于ORDER BY和GROUP BY操作，也不能用于Covering index（即当索引本身包含查询所需全部数据时，不再访问数据文件本身）。**

## REF
> [MySQL索引背后的数据结构及算法原理](http://blog.codinglabs.org/articles/theory-of-mysql-index.html)