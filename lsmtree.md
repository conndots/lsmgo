# 如何实现一个KV数据存储

> 本文参考*Design Data-intensive Applications* Chapter 3 Storage and Retrieval Section 1 Data Structures That PowerYour Database。主要介绍如何实现一个简单的基于日志结构的KV数据库，从简单的版本，到基于hash table的方案，最后介绍Log-Structured Merge Tree。

当前，KV数据库是非常流行的数据库类型。常见的KV数据库除了纯内存存储的Redis外，基于磁盘存储的方案有两种：

 	1. 基于日志的实现，为写入优化，查询效率有所妥协。比如使用Log-Structured Merge Tree的LevelDB,RocksDB；
 	2. 基于排序树结构索引的实现，为查询优化，写入效率有所妥协。比如B树，B+树。

今天介绍一个基于日志的KV数据库的核心方法实现，核心是指只介绍Get(key)和Set(key, value)两种方法的实现，只介绍核心算法，不涉及细节。



## 最简单的实现，最快的写入

最简单的实现非常简单：

```bash
#!/bin/bash

db_set() {
    echo "$1, $2" >> database
}
db_get() {
    grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

```databse```是存储全部数据的文件，每一行都是一次写入的key, value pair构成的特定格式的日志行。每一次写入都是append到数据文件的最后一行，所以同一个key可能会被重复写入很多行，最大的一行是最新的写入。所以，在get的时候，取全部最新的一行作为返回。  

当然，如果要实现一个可用的数据库，需要考虑很多其他的问题。比如，数据文件增长问题，crash恢复问题，因为crash导致的部分不完整写入等问题。

即使如此简单的实现，却有着最好的写入性能，因为append文件很快，如果能缓存做batch写入更快。正因为如此，很多现实中的数据库内部实现都会使用append only的日志。  

相反，查询效率就非常不乐观了，每一次get都需要遍历一遍整个数据文件。  我们需要另外一种数据结构来加快查找特定key的数据的速度，我们称它为**索引（index）**。索引由原始数据额外生成。索引或许可以大大加快数据查询的速度，却会增加数据写入需要的工作和消耗。因为每次数据的写入，除了更新原始文件，还需要更新索引。这是数据库中常见的**trade-off**。