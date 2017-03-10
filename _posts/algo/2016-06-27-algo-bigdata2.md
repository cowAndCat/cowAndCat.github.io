---
layout: post
title: 海量数据分析处理的十个方法
category: algorithm
comments: false
---

## 引子
本文参考多个网文，总结一些处理海量数据问题的常见方法，这些方法对面试、开发很有帮助。

总结方法如下： 

Bit-map、Bloom filter、Hashing、Heap、双层桶划分

数据库索引、倒排索引、外排序、trie树、分布式处理（map/reduce)

## 方法一：Bit-map
【适用范围】可进行数据的快速查找，判重，删除，一般来说数据范围是int的10倍以下

【基本原理及要点】使用bit数组来表示某些元素是否存在，比如8位电话号码

【扩展】bloom filter可以看做是对bit-map的扩展

【问题实例】

1)已知某个文件内包含一些电话号码，每个号码为8位数字，统计不同号码的个数。

	8位最多99 999 999，大概需要99m个bit，大概十多个m字节的内存即可。

2)2.5亿个整数中找出不重复的整数的个数，内存空间不足以容纳这2.5亿个整数。

    将bit-map扩展一下，用2bit表示一个数即可，0表示未出现，1表示出现一次，2表示出现2次及以上。
    或者我们不用2bit来进行表示，我们用两个bit-map即可模拟实现这个2bit-map。

-----

## 方法二：Bloom filter （布隆过滤器）

【介绍】Bloom Filter是由Bloom在1970年提出的一种**多哈希函数映射的快速查找算法**。通常应用在一些需要快速判断某个元素是否属于集合，但是并不严格要求100%正确的场合。

【适用范围】数据字典；判断数据重复；求集合的交集

【实例场景】
一、假设要你写一个网络蜘蛛（web crawler）。由于网络间的链接错综复杂，蜘蛛在网络间爬行很可能会形成“环”。为了避免形成“环”，就需要知道蜘蛛已经访问过那些URL。给一个URL，怎样知道蜘蛛是否已经访问过呢？


稍微想想，就会有如下几种方案：

1. 将访问过的URL保存到数据库，通过数据库内置方法判断unique。
2. 用HashSet将访问过的URL保存起来。那只需接近O(1)的代价就可以查到一个URL是否被访问过了。
3. URL经过MD5或SHA-1等单向哈希后再保存到HashSet或数据库。
4. Bit-Map方法。建立一个BitSet，将每个URL经过一个哈希函数映射到某一位。

　　方法1~3都是将访问过的URL完整保存，方法4则只标记URL的一个映射位。

以上方法在数据量较小的情况下都能完美解决问题，但是当数据量变得非常庞大时问题就来了。

1. 方法1的缺点：数据量变得非常庞大后关系型数据库查询的效率会变得很低。而且每来一个URL就启动一次数据库查询是不是太小题大做了？
2. 方法2的缺点：太消耗内存。随着URL的增多，占用的内存会越来越多。就算只有1亿个URL，每个URL只算50个字符，就需要5GB内存。
3. 方法3：由于字符串经过MD5处理后的信息摘要长度只有128Bit，SHA-1处理后也只有160Bit，因此方法3比方法2节省了好几倍的内存。
4. 方法4：消耗内存是相对较少的，但缺点是单一哈希函数发生冲突的概率太高。还记得数据结构课上学过的Hash表冲突的各种解决方法么？若要降低冲突发生的概率到1%，就要将BitSet的长度设置为URL个数的100倍。

　　实质上上面的算法都忽略了一个重要的隐含条件：允许小概率的出错，不一定要100%准确！也就是说少量url实际上没有没网络蜘蛛访问，而将它们错判为已访问的代价是很小的——大不了少抓几个网页呗。 


【基本原理及要点】

**Algorithm: 创建一个m位BitSet，先将所有位初始化为0，然后选择k个不同的哈希函数。第i个哈希函数对字符串str哈希的结果记为h（i，str），且h（i，str）的范围是0到m-1 。**

例如：对于字符串str，分别计算h（1，str），h（2，str）…… h（k，str）。然后将BitSet的第h（1，str）、h（2，str）…… h（k，str）位设为1。这样就将字符串str映射到BitSet中的k个二进制位了。

对于原理来说很简单，"位数集合"（只有1个）+"k个独立hash函数"。将hash函数对应的值的位数组置1，查找时如果发现所有hash函数对应位都是1说明存在，很明显这个过程并不保证查找的结果是100%正确的。

同时也不支持删除一个已经插入的关键字，因为该关键字对应的位会牵动到其他的关键字。所以一个简单的改进就是 counting Bloom filter，这是一种基本Bloom Filter的变体，CBF将基本Bloom Filter每一个Bit改为一个计数器，这样就可以实现删除字符串的功能了。

【重要的问题】如何根据输入元素个数n，确定位数组m的大小及hash函数个数。

当hash函数个数为

``` k = (ln2)*(m/n) ```

时错误率最小。 证明文献参考：
[```http://pages.cs.wisc.edu/~cao/papers/summary-cache/node8.html```](http://pages.cs.wisc.edu/~cao/papers/summary-cache/node8.html)

根据参考文献，哈希函数个数k取10，位数组大小m设为字符串个数n的20倍时，false positive发生的概率是0.0000889 ，这个概率基本能满足网络爬虫的需求了。  

在错误率不大于E的情况下，m至少要等于n\*lg(1/E)才能表示任意n个元素的集合。但m还应该更大些，因为还要保证bit数组里至少一半为0，则m应该>=nlg(1/E)\*lge 大概就是nlg(1/E)1.44倍(lg表示以2为底的对数)。

举个例子：**假设错误率为0.01，则此时m应大概是n的13倍。这样k大概是8个。**

注意这里m与n的单位不同，m是bit为单位，而n则是以元素个数为单位(准确的说是不同元素的个数)。通常单个元素的长度都是有很多bit的。所以使用bloom filter内存上通常都是节省的。

【实例场景】二、给你A,B两个文件，各存放50亿条URL，每条URL占用64字节，内存限制是4G，让你找出A,B文件共同的URL。如果是三个乃至n个文件呢？

计算一下内存，4G=2^32大概是40亿，按bit算大概是340亿，n=50亿，如果按出错率0.01算需要的大概是650亿个bit。现在可用的是340亿，相差并不多，这样可能会使出错率上升些。总的来说还是可以使用该算法的。

【技巧】

- 选择k个不同的哈希函数比较麻烦，一种简单的方法是选择一个哈希函数，然后送入k个不同的参数。

- java 里面的BitSet数据结构可以很好地帮助算法的实现。 （该算法的一种实现可以参考最底部的3rd资料）

-----

## 方法三：Hashing
【适用范围】快速查找（基本上，涉及到查找，都要想到hash）

【基本原理及要点】

hash函数选择，针对字符串，整数，排列，具体相应的hash方法。

碰撞处理，一种是open hashing，也称为拉链法；另一种就是closed hashing，也称开地址法，opened addressing。

【扩展】
d-left hashing

d-left hashing中的d是多个的意思，我们先简化这个问题，看一看2-left hashing。

2-left hashing指的是将一个哈希表分成长度相等的两半，分别叫做T1和T2，给T1和T2分别配备一个哈希函数，h1和h2。在存储一个新的key时，同时用两个哈希函数进行计算，得出两个地址h1[key]和h2[key]。这时需要检查T1中的h1[key]位置和T2中的h2[key]位置，哪一个位置已经存储的（有碰撞的）key比较多，然后将新key存储在负载少的位置。如果两边一样多，比如两个位置都为空或者都存储了一个key，就把新key存储在左边的T1子表中，2-left也由此而来。在查找一个key时，必须进行两次hash，同时查找两个位置。

【问题实例】

1. 海量日志数据，提取出某日访问百度次数最多的那个IP。

IP的数目还是有限的，最多2^32个，所以可以考虑使用hash将ip直接存入内存，然后进行统计。

-----

## 方法四：堆（Heap)
【适用范围】求海量数据前n大，并且n比较小，堆可以放入内存

【基本原理及要点】

**最大堆求前n小，最小堆求前n大。**

方法，比如求前n小，我们比较当前元素与最大堆里的最大元素，如果它小于最大元素，则应该替换那个最大元素。这样最后得到的n个元素就是最小的n个。适合大数据量，求前n小，n的大小比较小的情况，这样可以扫描一遍即可得到所有的前n元素，效率很高。

【扩展】双堆，一个最大堆与一个最小堆结合，可以用来维护中位数。

【问题实例】

1)100w个数中找最大的前100个数。

	用一个100个元素大小的最小堆即可。

----

## 方法五：双层桶划分

其实本质上就是【分而治之】的思想，重在分的技巧上！

【适用范围】第k大，中位数，不重复或重复的数字

【基本原理及要点】因为元素范围很大，不能利用直接寻址表，所以通过多次划分，逐步确定范围，然后最后在一个可以接受的范围内进行。可以通过多次缩小，双层只是一个例子。

【问题实例】

1).2.5亿个整数中找出不重复的整数的个数，内存空间不足以容纳这2.5亿个整数。

有点像鸽巢原理，整数个数为2^32,也就是，我们可以将这2^32个数，划分为2^8个区域(比如用单个文件代表一个区域)，然后将数据分离到不同的区域，然后不同的区域在利用bitmap就可以直接解决了。也就是说只要有足够的磁盘空间，就可以很方便的解决。

2).5亿个int找它们的中位数。

这个例子比上面那个更明显。首先我们将int划分为2^16个区域，然后读取数据统计落到各个区域里的数的个数，之后我们根据统计结果就可以判断中位数落到那个区域，同时知道这个区域中的第几大数刚好是中位数。然后第二次扫描我们只统计落在这个区域中的那些数就可以了。

实际上，如果不是int是int64，我们可以经过3次这样的划分即可降低到可以接受的程度。即可以先将int64分成2^24个区域，然后确定区域的第几大数，在将该区域分成2^20个子区域，然后确定是子区域的第几大数，然后子区域里的数的个数只有2^20，就可以直接利用direct addr table进行统计了。

------

## 方法六：数据库索引

【适用范围】大数据量的增删改查

【基本原理及要点】利用数据的设计实现方法，对海量数据的增删改查进行处理。

------

## 方法七：倒排索引(Inverted index)

【适用范围】搜索引擎，关键字查询

【基本原理及要点】为何叫倒排索引？一种索引方法，被用来存储在全文搜索下某个单词在一个文档或者一组文档中的存储位置的映射。

以英文为例，下面是要被索引的文本： T0 = “it is what it is” T1 = “what is it” T2 = “it is a banana”

我们就能得到下面的反向文件索引：

“a”: {2} “banana”: {2} “is”: {0, 1, 2} “it”: {0, 1, 2} “what”: {0, 1}

检索的条件”what”,”is”和”it”将对应集合的交集。

正向索引开发出来用来存储每个文档的单词的列表。正向索引的查询往往满足每个文档有序频繁的全文查询和每个单词在校验文档中的验证这样的查询。在正向索引中，文档占据了中心的位置，每个文档指向了一个它所包含的索引项的序列。也就是说文档指向了它包含的那些单词，而反向索引则是单词指向了包含它的文档，很容易看到这个反向的关系。

【问题实例】文档检索系统，查询那些文件包含了某单词，比如常见的学术论文的关键字搜索。

-----

## 方法八：外排序

【适用范围】大数据的排序，去重

【基本原理及要点】外排序的归并方法，置换选择败者树原理，最优归并树

【问题实例】

1).有一个1G大小的一个文件，里面每一行是一个词，词的大小不超过16个字节，内存限制大小是1M。返回频数最高的100个词。

这个数据具有很明显的特点，词的大小为16个字节，但是内存只有1m做hash有些不够，所以可以用来排序。内存可以当输入缓冲区使用。

-----

## 方法九：trie树

【适用范围】数据量大，重复多，但是数据种类小可以放入内存

【基本原理及要点】实现方式，节点孩子的表示方式

【扩展】压缩实现。

【问题实例】

1).有10个文件，每个文件1G，每个文件的每一行都存放的是用户的query，每个文件的query都可能重复。要你按照query的频度排序。

2).1000万字符串，其中有些是相同的(重复),需要把重复的全部去掉，保留没有重复的字符串。请问怎么设计和实现？

3).寻找热门查询：查询串的重复度比较高，虽然总数是1千万，但如果除去重复后，不超过3百万个，每个不超过255字节。

-----

## 方法十：分布式处理 map/reduce

【适用范围】数据量大，但是数据种类小可以放入内存

【基本原理及要点】将数据交给不同的机器去处理，数据划分，结果归约。

【问题实例】

1).The canonical example application of MapReduce is a process to count the appearances of each different word in a set of documents。（计算多个文件中单词出现频度）

2).海量数据分布在100台电脑中，想个办法高效统计出这批数据的TOP10。

3).一共有N个机器，每个机器上有N个数。每个机器最多存O(N)个数并对它们操作。如何找到N^2个数的中数(median)？

----

## 经典问题分析

》上千万or亿数据（有重复），统计其中出现次数最多的前N个数据,分两种情况：可一次读入内存，不可一次读入。《

可用思路：trie树+堆，数据库索引，划分子集分别统计，hash，分布式计算，近似统计，外排序

所谓的是否能一次读入内存，实际上应该指去除重复后的数据量。如果去重后数据可以放入内存，我们可以为数据建立字典，比如通过 map，hashmap，trie，然后直接进行统计即可。当然在更新每条数据的出现次数的时候，我们可以利用一个堆来维护出现次数最多的前N个数据，当然这样导致维护次数增加，这种方式比完全统计后再求前N大的效率要高。

如果数据无法放入内存。一方面我们可以考虑上面的字典方法能否被改进以适应这种情形，可以做的改变就是将字典存放到硬盘上，而不是内存，这可以参考数据库的存储方法。

当然还有更好的方法，就是可以采用分布式计算，基本上就是map-reduce过程，首先可以根据数据值或者把数据hash(md5)后的值，将数据按照范围划分到不同的机子，最好可以让数据划分后可以一次读入内存，这样不同的机子负责处理各种的数值范围，实际上就是map。得到结果后，各个机子只需拿出各自的出现次数最多的前N个数据，然后汇总，选出所有的数据中出现次数最多的前N个数据，这实际上就是reduce过程。

实际上直接将数据均分到不同的机子上进行处理，这样可能无法得到正确的解。因为一个数据可能被均分到不同的机子上，而另一个则可能完全聚集到一个机子上，同时还可能存在具有相同数目的数据。

比如我们要找出现次数最多的前100个，我们将1000万的数据分布到10台机器上，找到每台出现次数最多的前 100个，归并之后这样不能保证找到真正的第100个，因为比如出现次数最多的第100个可能有1万个，但是它被分到了10台机子，这样在每台上只有1千个，假设这些机子排名在1000个之前的那些都是单独分布在一台机子上的，比如有1001个，这样本来具有1万个的这个就会被淘汰，即使我们让每台机子选出出现次数最多的1000个再归并，仍然会出错，因为可能存在大量个数为1001个的发生聚集。因此不能将数据随便均分到不同机子上，而是**要根据hash 后的值将它们映射到不同的机子上处理，让不同的机器处理一个数值范围。**

而外排序的方法会消耗大量的IO，效率不会很高。而上面的分布式方法，也可以用于单机版本，也就是将总的数据根据值的范围，划分成多个不同的子文件，然后逐个处理。处理完毕之后再对这些单词的及其出现频率进行一个归并。实际上就可以利用一个外排序的归并过程。

另外还可以考虑近似计算，也就是我们可以通过结合自然语言属性，只将那些真正实际中出现最多的那些词作为一个字典，使得这个规模可以放入内存。

以上

-----

### 参考
> 1. (站内阅读)[处理大数据的算法](/algorithm/2015/10/10/algo-bigdata.html)

> 2. [海量数据分析处理的十个方法](http://www.36dsj.com/archives/2556)

> 3. [BloomFilter——大规模数据处理利器](http://www.cnblogs.com/heaad/archive/2011/01/02/1924195.html)