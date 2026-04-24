Z-Order最早是1966提出的一项将多维数据映射到一维的方法.随着数据库技术的发展,这种映射方法由于其特性,被应用到了数据库技术中,特别是在大数据时代再次被提及,在[hudi](https://zhida.zhihu.com/search?content_id=197299611&content_type=Article&match_order=1&q=hudi&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NTg2ODEyMjMsInEiOiJodWRpIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MTk3Mjk5NjExLCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.PNsbxE3x-cxS8f1YD00l4330iLBtHvl1Sp8LQLIj8ak&zhida_source=entity)、[iceberg](https://zhida.zhihu.com/search?content_id=197299611&content_type=Article&match_order=1&q=iceberg&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NTg2ODEyMjMsInEiOiJpY2ViZXJnIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MTk3Mjk5NjExLCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.6HbJi9qHhUMrnLW2AsG8ZbgleI-dW4WEog1P6sqt8wI&zhida_source=entity)

中都有应用。本文将对数据库领域使用Z-Order的情形进行介绍，分析其使用场景，最后对比多个数据库领域的相关技术，得出Z-Order的特点。

## 第一部分 数据库领域的Z-Order

在数据库领域，Z-Order的应用可以分为两个阶段：[OLTP](https://zhida.zhihu.com/search?content_id=197299611&content_type=Article&match_order=1&q=OLTP&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NTg2ODEyMjMsInEiOiJPTFRQIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MTk3Mjk5NjExLCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.BA9iejtT07H9b_O19Xz83_s_Wb69xxpPWEeU7i2ANOk&zhida_source=entity)

和[OLAP](https://zhida.zhihu.com/search?content_id=197299611&content_type=Article&match_order=1&q=OLAP&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NTg2ODEyMjMsInEiOiJPTEFQIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MTk3Mjk5NjExLCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.Cf-2PTwH2p6UaoN3vsaiYx9nfmsa_ZjimnRb0AKynXo&zhida_source=entity)

阶段。这两个阶段由于面向的场景不同，导致其存储引擎在数据堆叠机制上无法统一，进而影响了Z-Order的使用方式。但无论是OLTP还是OLAP，使用Z-Order的动机都是相同的。本节分析Z-Order的使用动机，接着对Z-Order不同阶段的使用进行分开描述。

### 1.1 最左匹配原则

```text
CREATE TABLE members(
id int not null,
age int not null,
viplevel int not null;
……
constraint PK_members primary key(id,viplevel)     // *
);
SELECT avg(age) FROM members WHERE id >= 1 and id <= 3;   // 1
SELECT avg(age) FROM members WHERE viplevel >= 1 and viplevel <= 3;  // 2
```

代码清单1 表结构和查询语句

考虑代码清单1中的查询语句与对应的表结构，members表中的主键是一个复合主键，数据库会对主键做[B+树索引](https://zhida.zhihu.com/search?content_id=197299611&content_type=Article&match_order=1&q=B%2B%E6%A0%91%E7%B4%A2%E5%BC%95&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NTg2ODEyMjMsInEiOiJCK-agkee0ouW8lSIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjE5NzI5OTYxMSwiY29udGVudF90eXBlIjoiQXJ0aWNsZSIsIm1hdGNoX29yZGVyIjoxLCJ6ZF90b2tlbiI6bnVsbH0.2LXrtQGPyyRYu5w_dqmyLQ4ruenihMhKwOTFq-tT5ps&zhida_source=entity)

。**B+树索引的本质是将数据按照自然顺序进行排列，以提升范围查询的性能**。因此，在执行代码清单中//1号语句时，由于主键已经做了B+树索引，id在[1,3]之间的数据已经被连续地放在了一起，因此可以相对高效的完成查询。

  
![](https://pica.zhimg.com/v2-9b041901d5e711751cb1e271c618d68e_1440w.jpg)

图1 复合索引的影响

图1 复合索引的影响

但是，当执行//2号语句时，对viplevel进行范围查询。由于viplevel在索引中在右侧，排序时必须首先按照id进行排序，id相同的数据才按照viplevel排序。因此，viplevel在[1,3]之间的数据并没有聚集到一起。

图1描述了一个可能的排序，可以说明这情况。在图1中，ID∈[1,3]的数据连续堆放在一起，因此通过一次匹配即可快速完成查询。viplevel作为索引的第二个排序键，需要优先保证第一排序键的有序，因此在整体数据中被分割成了4个区域，需要对索引项进行遍历才能取得所有数据。

该现象体现了数据库查询的**最左匹配原则**。数据库对B+树类型的索引进行匹配时，查询条件精确匹配最左边的一列或连续的多列时，才会命中索引，否则无法命中索引。当不满足最左原则时，索引对于查询性能的提升起到的作用已经不大了。

### 1.2 Z-Order动机

Z-Order索引通过其特殊的排序机制，可以有效避免B+树带来的最左匹配原则。其被应有于数据索引的目的就是为了解决B+树类型的索引在不满足最左匹配原则的情况下失效的问题。

传统的B+树对数据进行排序时使用自然顺序，因此在遇到复合索引时，按照从左至右的顺序对排序键进行处理。首先完成最左侧排序键的排序，在此基础上完成后续排序键的排序。由此带来了最左匹配原则。

  

![](https://pic1.zhimg.com/v2-84f983af8cab67b110ededdb2277a6a6_1440w.jpg)

图2 Z-Order示意图

图2 Z-Order示意图

而Z-Order这是将两个或多个排序键同时进行排序，按照间隔排列的顺序对数据进行排列。可以简单理解为当AB两个属性进行Z-Order索引时，假设AB两个属性从小达到排列后为{a1,a2,a3,……,an}和{b1,b2,b3,……,bn}第一个数据是AB同时最小的即（a1,b1）、第二个数据是A次小的和B最小的即(a2,b1)、第三个数据是剩余的AB中A最小的B次小的即(a1,b2)、第四个数据是剩余AB中同时最小的即(a2,b2)……图2展示了这种情况的示例，图中2（一）中表格中的数字是排序的序号，由于其顺序类似于因为字母Z，因此被称为Z-ORDER。

图2（二）中展示了三维维度下的Z-Order的顺序排列示意。Z-Order可以推广到N维的场景下，为更多维度的复合主键提供索引。

### 1.3 OLTP

Z-Order由于其特殊的数据聚集特性，解决了B+树类型的联合索引在不满足最左原则时的失效问题，因此在传统数据库中可以用于联合索引中。但是实际上，并没有多少数据库真正使用了该索引技术，大部分事务数据库的索引依旧以B+树索引解决范围查询问题。

OLTP时代，Z-Order并没有被大规模应用。一个很重要的原因在于OLTP时代的存储引擎，数据一般按照事务顺序写入磁盘。在这种情况下，即使使用了Z-Order对数据进行了索引，保证了数据在索引上的聚集效应。但是实际上，数据存储在磁盘上依然是不连续的。因此，Z-Order的应用仅仅只是减少了遍历索引的时间，获得索引信息后，依然需要从不连续的磁盘上获取真实数据。即**Z-Order带来的加速效应无法直接影响磁盘的IO时间。**

同时，由于数据在物理磁盘上不连续，因此对于事务性数据库完全可以对复合索引建立多个B+树索引来解决该问题，而不一定需要使用复杂的Z-Order索引。

除了上述提到的两个原因之外，还有一个更核心原因：Z-Order客观上的确提升了B+树的索引效果，但同时也付出了一些代价。该内容会在本文第三部分进行分析。正是由于Z-Order付的的代价和带来的收益不成正比，因此只是在事务数据库的阶段昙花一现。

### 1.4 OLAP

OLAP时代，Z-Order重新进入人们的视野。在OLAP阶段，由于不需要提供复杂的事务支持，因此存储引擎的设计和事务数据库产生了很大的区别。其中一个重要的特性是：OLAP的存储引擎不需要按照数据的插入顺序来堆叠数据。而可以依据不同的排序策略，对数据进行堆叠。这个特性使的Z-Order在OLTP阶段的两个小问题获得了根本性的解决。

第一个问题是Z-Order带来的加速效应无法直接影响磁盘的IO时间。这个问题本质上是由于索引顺序并不等于存储引擎中数据的物理顺序。而在OLAP数据库中，完全可以按照索引顺序来安排OLAP数据库的物理存储顺序，因此，Z-Order获得的加速效应可以直接影响磁盘的IO时间，从而获得性能的极大提升。

第二个问题是事务数据库可以对多个排序键建立多个有序索引，来获得和Z-Order同样的加速效果。这个优化在OLAP中是做不到的。因为数据在磁盘上的物理顺序只能有一种。按照A有序排序了就不能再按照B有序排序，按照B有序排序了就不能再按照A排序。事务数据库由于数据的物理顺序和索引顺序本身不一致，所以可以做到多个顺序索引并存，也正是由于两种顺序不一致，使得多个顺序索引并存达到的加速效应一致。而对于OLAP数据库，将物理顺序和索引顺序保持一致可以提高查询性能，在这个前提下，物理顺序只能有一种，因此对于复合索引，Z-Order的用武之地大增。

综上，在OLAP时代，由于底层存储引擎不需要像事务数据库一样，按照事务顺序堆叠数据，可以将物理顺序和索引顺序保持一致，因此，Z-Order在OLTP数据库时代的几大问题都不再存在。Z-Order开始在OLAP阶段发力。

## 第二部分 Z-Order效率分析

第一部分向读者介绍了Z-Order解决问题以及在不同阶段的应用，本章将基于图2的例子，对Z-Order的加速效率进行分析。

### 2.1 按照A进行查询

图2（三）展示了基于A字段进行查询的例子，即执行了where a >= a1 and a <= a2语句。在这种查询条件下，只需要最多读取阴影部分的一般数据即可，阴影部分之外的数据不可能存在满足条件的数据，因此可以对阴影部分之外的数据进行跳过。换句话说，使用了Z-Order后，对A进行范围查询，至少能跳过50%的数据。

### 2.2 按照B进行查询

图2（四）展示了基于B字段进行查询的例子，即执行了where b >= b1 and b <= b2语句。同2.1的情况，只需要读取阴影部分的数据即可，可以跳过非阴影部分的数据。同样跳过了50%的数据。但由于

### 2.3 总结

通过图2的例子，不难发现，对于一个二维的查询条件来说，无论对A还是对B进行范围查询，都能至少过滤掉50%的数据量。如果放到OLAP的大数据场景下，是一个非常可观的收益。推广到一般情况，通过Z-Order排序的数据，至少可以保证一部分数据，过滤效果与查询条件的覆盖率成反比，查询的范围越小，过滤效果越明显。而不像单一顺序索引一样，过滤效果无法预估，最差的情况可能需要遍历数据。

## 第三部分 Z-Order缺陷

架构就是有得必有失，不分析代价的架构分析都是不全面的。本章将对Z-Order的缺陷进行分析。

**表格1 两种不同顺序数据堆叠方式**

|   |   |   |
|---|---|---|
|序号|(A,B)自然顺序|Z-Order顺序|
|0|a1,b1|a1,b1|
|1|a1,b2|a1,b2|
|2|a1,b3|a2,b1|
|3|a1,b4|a2,b2|
|4|a2,b1|a1,b3|
|5|a2,b2|a1,b4|
|6|a3,b3|a2,b3|
|7|a4,b4|a2,b4|

表格1中展示了图2例子中两种不同顺序的数据堆叠方式，表格中加深的单元格表示执行where A=a1时返回的数据。很明显可以看出，Z-Order顺序相比于(A,B)自然顺序排序，在满足最左原则的情况下，由于A的不连续，需要多进行一次磁盘IO，拖慢了满足最左原则场景下的查询速度。

需要注意的是，表格1中的数据只是为了说明原理，实际上由于操作系统的IO机制，对于例子中这么少的数据量来说，差距几乎为0。但在真实的OLAP所面向的大数据场景时，这个现象会非常明显。

的确，使用Z-Order顺序后，无论以A作为查询条件还是以B作为查询条件，都只能跳过50%的数据。但是在满足查询满足最左原则的情况下，由于Z-Order将原本自然顺序下有序的数据变得无序了，因此反而降低了满足最左原则下的查询效率。这种现象在小数据的情况下不明显，但是当数据量增加时，这种现象带来的影响会越来越大。

## 第四部分 总结及建议

Z-Order本质上是**牺牲了主排序键的查询性能换取次排序键查询性能的提升**。客观上降低了数据库工程师建模能力的要求（使用Z-Order可以在任何场景下获得不算差的查询性能，不需要数据库工程师按照业务去优化数据表）。同时也更加适用于一些数据需求多变而无法事先确定哪一列比较常用的场景。

目前hudi、iceberg都有了一些应用Z-Order的方案，读者在应用Z-Order方案时，应当充分理解您的数据需求。Z-Order并不一定能在所有情况下提升查询速度。因此，如果读者的数据需求比较确定，还是建议读者使用最原始的自然顺序。**客观来讲，Z-Order是一种牺牲了极致性能的中庸的方案，没有极致的优点，也没有明显的缺点。**这也是为什么hudi、iceberge只是提供了方案，但并没有将Z-Order作为默认索引的原因。读者应当充分考虑，您是否真的需要这么一种中庸的方案。

当然，如果读者遇到的不满足最左原则的数据需求比较多，且无法固定，此时使用Z-Order会给您带来惊喜，他不能给您带来极致的查询速度，但他能给您带来平均不慢的速度。不会像[ClickHouse](https://zhida.zhihu.com/search?content_id=197299611&content_type=Article&match_order=1&q=ClickHouse&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NTg2ODEyMjMsInEiOiJDbGlja0hvdXNlIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MTk3Mjk5NjExLCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.PDavFtti_2_dQ0v5eLk5qJMcBxvLcI6rALGlylhm8Iw&zhida_source=entity)

一样，一旦不满足最左原则，性能急剧下降，甚至是之前的数十、数百倍的性能差距（也有解决办法，但不是依靠zorder）。