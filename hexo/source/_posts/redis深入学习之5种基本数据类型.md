---
title: Redis深入学习之5种基本数据类型
date: 2019-01-19 15:06:38
categories: "Redis 教程"
tags: [微缓存,Redis]
---

<Excerpt in index | 首页摘要> 

# Redis 5种基本数据类型



## 前言

​	redis平时还是经常使用的，但是只是停留在基本使用上面，5种基本的数据类型基本也只是用了字符串，觉得还是有必要深入学习下Redis，从5种基本数据类型开始。

<!-- more -->
<The rest of contents | 余下全文>

## 字符串

​	字符串 string 是 Redis 最简单的数据结构。Redis 所有的数据结构都是以唯一的 key 字符串作为名称，然后通过这个唯一 key 值来获取相应的 value 数据。不同类型的数据结构的差异就在于 value 的结构不一样。

​	Redis的字符串是动态字符串，是可以修改的字符串，内部结构实现上类似于Java的ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配，如图中所示，内部为当前字符串实际分配的空间capacity一般要高于实际字符串长度len。当字符串长度小于1M时，扩容都是加倍现有的空间，如果超过1M，扩容时一次只会多扩1M的空间。需要注意的是字符串最大长度为512M。​		

​	在Redis里面，字符串可以存储以下3种类型的值。

* 字符串（byte string）
* 整数
* 浮点数

Redis字符串可以对存储的整数或者浮点数字符串执行自增或者自减等操作

| 命令        | 用例和描述                                             |
| ----------- | ------------------------------------------------------ |
| INCR        | INCR key----将键存储的值加上1                          |
| DECR        | DECR key----将键存储的值减一                           |
| INCRBY      | INCRBY key amount----将键存储的值加上整数amount        |
| DECRBY      | DECRBY key amount----将键存储的值减去整数amount        |
| INCRBYFLOAT | INCRBYFLOAT key amount----将键存储的值加上浮点数amount |

Redis除了自增操作和自减操作之外，还拥有对字符串的其中一部分内容进行读取或者写入的操作

| 命令     | 用例和描述                                                   |
| -------- | ------------------------------------------------------------ |
| APPEND   | APPEND key value----将将值value追加到给定键key-name当前存储的值末尾 |
| GETRANGE | INCRBYFLOAT key start end----获取一个由偏移量start至偏移量end范围内的所有字符组成的子串，包括start和end在内 |
| SETRANGE | SETRANGEkey start value----将从start偏移量开始的子串设置为给定值 |
| GETBIT   | GETBIT key offset----将字符串看做是二进制位串，并返回位串中偏移量为offect的二进制位的值 |
| SETBIT   | SETBIT key offset ----将字符串看做是二进制位串，并将位串中偏移量为offect的二进制位的值设置为value |
| BITCOUNT | BITCOUNT key [start end]----统计二进制字符串里面值为1的数量，可以给定范围，统计范围内1的个数 |
| BITTOP   | BITTOP operation dest-key key [key]----对一个或者多个二进制位字符串执行并（and）、或（OR）、异或（XOR）、非（NOT）在内的任意一种按位运算操作，并将计算得出的结果保存在dest-key键里面 |



## 列表

​	Redis 的列表相当于 Java 语言里面的 LinkedList，注意它是链表而不是数组。这意味着 list 的插入和删除操作非常快，时间复杂度为 O(1)，但是索引定位很慢，时间复杂度为 O(n)。Redis列表允许用户从序列的两端推入或者弹出元素获取元素，以及各种常用的列表操作。

常用的列表处理命令

| 命令   | 用例和描述                                                   |
| ------ | ------------------------------------------------------------ |
| RPUSH  | RPUSH key value----将一个或多个值推入列表的右端              |
| LRUSH  | LRUSH key value----将一个或者多个值推入列表的左端            |
| RPOP   | RPOP key----移除并返回列表最右端的元素                       |
| LPOP   | LPOP key----移除并返回列表最左端的元素                       |
| LINDEX | LINDEX key offset----返回列表偏移量为offset的元素            |
| LRANGE | LRANGE key start end----返回列表从start偏移量到end偏移量范围内的所有元素。 |
| LTRIM  | LTRIM key start end----对列表进行修剪，只保留偏移量start到end之间的元素 |

Redis列表还可以将元素从一个列表移到另外一个列表，或者阻塞执行命令的客户端直到有其他客户端给列表添加元素为止。

| 命令       | 用例和描述                                                   |
| ---------- | ------------------------------------------------------------ |
| BLPOP      | BLPOP key timeout----从第一个非空列表中弹出位于最左端的元素，或者在timeout秒之内阻塞并等待可弹出的元素出现 |
| BRPOP      | BRPOP key timeout----从第一个非空列表中弹出位于最右端的元素，或者在timeout秒之内阻塞并等待可弹出的元素 |
| RPOPLPUSH  | RPOPLPUSH key1 key2 从key1列表中弹出位于最右端的元素，然后将这个元素推入key2列表的最左端，并向用户返回这个元素 |
| BRPOPLPUSH | BRPOPLPUSH key1 key2 timeout从key1列表中弹出位于最右端的元素，然后将这个元素推入key2列表的最左端，并向用户返回这个元素；如果key1为空，那么在timeout秒之内阻塞并等待可弹出的元素出现 |



## 集合

​	Redis的集合以无序的方式来存储多个各不相同的元素，用户可以快速的的集合执行添加元素，移除元素，以及判断一个元素是否存在于集合中。

​	Java程序员都知道HashSet的内部实现使用的是HashMap，只不过所有的value都指向同一个对象。Redis的set结构也是一样，它的内部也使用hash结构，所有的value都指向同一个内部值。

一些常用集合命令

| 命令        | 用例和描述                                                   |
| ----------- | ------------------------------------------------------------ |
| SADD        | SADD key item [item]----将一个或多个元素添加到集合里面，并返回被添加元素当中原本并不存在于集合里面的元素数量 |
| SREM        | SREM key item [item]----从集合里面移除一个或者多个元素，并返回被移除的元素数量 |
| SISMEMBER   | SISMEMBER key item----检查元素item是否存在于集合key中        |
| SCARD       | SCARD key----返回集合包含元素的数量                          |
| SMEMBERS    | SMEMBERS key [count]----返回集合包含的所有元素               |
| SRANDMEMBER | SRANDMEMBER key [count]从集合里面随机的返回一个或者多个元素。当couut为正数时，命令返回的随机元素不会重复；当count为负数时，命令返回的随机元素可能会出现重复 |
| SPOP        | SPOP key----随机的移除集合中的一个元素，并返回被移除的元素   |
| SMOVE       | SMOVE key1 key2 item----如果key1包含元素item，那么从集合key1里面移除元素，并将元素item添加到集合key2中，如果item被成功的移除，那么命令返回1，否则返回0 |

集合除了基本的添加，移除元素外，还可以组合和关联多个集合

| 命令        | 用例和描述                                                   |
| ----------- | ------------------------------------------------------------ |
| SDIFF       | SDIFF key [key]----返回那些存在于第一个集合，但不存在于其他集合中的元素（差集） |
| SDIFFSTORE  | SDIFFSTORE key1 key2 [key]---将那些存在于第一个集合但不存在于其他集合中的元素（差集），存储到key1中 |
| SINTER      | SINTER key [key]----返回那些同时存在于所有集合中的元素（交集） |
| SINTRTSTORE | SINTRTSTORE key1 key2 [key]---将那些同时存在于所有集合中的元素（交集）存储到key1里面 |
| SUNION      | SUNION key [key]----返回那些至少存在于一个集合中的元素（并集） |
| SUNIONSTORE | SUNIONSTORE key1 key2 [key]----将那些那些至少存在于一个集合中的元素（并集）存储到key1里面 |



## 散列

​	Redis的散列可以让用户将多个键值对存储到一个Redis键里面。从功能是来说。Redis为散列值提供了一些与字符串值相同的特性，使得散列非常适用于将一些相关的数据存储在一起。

​	哈希等价于Java语言的HashMap或者是Python语言的dict，在实现结构上它使用二维结构，第一维是数组，第二维是链表，hash的内容key和value存放在链表中，数组里存放的是链表的头指针。通过key查找元素时，先计算key的hashcode，然后用hashcode对数组的长度进行取模定位到链表的表头，再对链表进行遍历获取到相应的value值，链表的作用就是用来将产生了「hash碰撞」的元素串起来。Java语言开发者会感到非常熟悉，因为这样的结构和HashMap是没有区别的。哈希的第一维数组的长度也是2^n。

散列的基本命令

| 命令  | 用例和描述                                                   |
| ----- | ------------------------------------------------------------ |
| HMGET | HMGET key -name key [key]----从散列里面获取一个或者多个键的值 |
| HMSET | HMSET key-name key value [key value]----为散列里面一个或者多个键设置值 |
| HDEL  | HDEL key-name key [key]----删除散列里面一个或者多个键值对，返回成功找到并删除的键值对数量 |
| HLEN  | HLEN key-name----返回散列包含的键值对数量                    |

其他几个批量操作命令,以及一些和字符串操作类似的散列命令

| 命令         | 用例和描述                                                   |
| ------------ | ------------------------------------------------------------ |
| HEXISTS      | HEXISTS key-name key----检查给定键是否存在于散列中           |
| HKEYS        | HKEYS key-name ----获取散列包含的所有键                      |
| HVALS        | HVALS key-name----获取散列包含的所有值                       |
| HGETALL      | HGETALLkey-name----获取散列包含的所有键值对                  |
| HINCRBY      | HINCRBY key-name key increment----将键key存储的值加上整数increment |
| HINCRBYFLOAT | HINCRBYFLOAT key-name key increment----将键key存储的值加上浮点数increment |



## 有序集合

​	和散列存储着键与值之间的隐射类似，有序集合也存储着成员与分值之间的映射，并且提供了分值处理命令，以及根据分值大小有序的获取或扫描成员和分值的命令。

​	SortedSet(zset)是Redis提供的一个非常特别的数据结构，一方面它等价于Java的数据结构`Map<String, Double>`，可以给每一个元素value赋予一个权重`score`，另一方面它又类似于`TreeSet`，内部的元素会按照权重score进行排序，可以得到每个元素的名次，还可以通过score的范围来获取元素的列表。

一些常用的有序集合命令

| 命令    | 用例和描述                                                   |
| ------- | ------------------------------------------------------------ |
| ZADD    | ZADD key socre member [score member]----将带有给定分支的成员添加到有序集合里面 |
| ZREM    | ZREM key member [member]----从有序集合里面移除给定的成员，并返回被移除成员的数量 |
| ZCRD    | ZCRD key----返回有序集合包含的成员数量                       |
| ZINCRBY | ZINCRBY key increment member----将member成员的分值加上increment |
| ZCOUNT  | ZCOUNT key min max----返回分值介于min和max之间的成员数量     |
| ZRANX   | ZRANX key member----返回成员member在有序集合中的排名         |
| ZSCORE  | ZSCORE key member----返回成员member的分值                    |
| ZRANGE  | ZRANGE key start stop [WITHSCORES]----返回有序集合中排名介于start和stop之间的成员，如果给定了可选的WITHSCORES选项，那么命令会将成员的分值也一并返回 |

有序集合的范围型数据获取命令和范围型数据删除命令，以及并集命令和交集命令

| 命令             | 用例和描述                                                   |
| ---------------- | ------------------------------------------------------------ |
| ZREVRANK         | ZREVRANK key member----返回有序集合里成员member的排名，成员按照分值从大到小排列 |
| ZREVRANGE        | ZREVRANGE key start stop [WITHSCORES]----返回有序集合给定排名范围内的成员，成员按照分值从大到小排列 |
| ZRANGEBYSCORE    | ZRANGEBYSCORE key min max [WITHSCORES] ----返回有序集合中，分值介于min和max之间的成员 |
| ZREVRANGEBYSCORE | ZREVRANGEBYSCORE key max min [WITHSCORES]----获取有序集合中分值排名介于min和max之间的所有成员，并按照分值从大到小的顺序来返回它们 |
| ZREMRANGEBYRANK  | ZREMRANGEBYRANK key start stop--移除有序集合中分值介于min 和 max之间的所有成员 |
| ZREMRANGEBYSCORE | ZREMRANGEBYSCORE key min max----移除有序集合中分值介于min和max之间的所有成员 |
| ZINTERSTORE      | ZINTERSTORE   key1 key1-count key2 [key]----对给定的有序集合执行类似集合的交集运算 |
| ZUNIONSTORE      | ZUNIONSTORE key key-count key [key] 对给定的有序集合执行类似于集合的并集运算 |

