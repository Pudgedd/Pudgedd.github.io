---
title: Redis数据结构与对象(3)——字典
date: 2019-01-31 23:23:17
categories: Redis
tags: 
    - Redis
    - 数据结构
    - 字典
---

## 字典
字典，又称为符号表(symbol table)、关联数组(associative array)或映射(map)，是一种用于保存键值对的抽象数据结构。字典中的每个键都是独一无二的，程序可以在字典中通过键查找与之关联的值，或进行更新删除等操作。
<!-- more -->
Redis的数据库就是使用字典来作为底层实现的。例如：

    redis>SET msg "hello world"
    OK
 
在数据库中创建了一个键为"msg",值为"hello world"的键值对，保存在字典中。
除了用来表示数据库外，字典还是**哈希键**的底层实现之一。

    redis>  HMSET user name "kendall" sex "man"
    OK
    redis>  HGETALL user
    1) "name"
    2) "kendall"
    3) "sex"
    4) "man"
    
`user`就是一个一个包含2个键值对的哈希键。

Redis的字典使用哈希表作为底层实现，一个哈希表可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。

哈希表
: 
``` c
typedef struct dictht{
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;//哈希表大小掩码，总是等于size-1，用于计算索引值
    unsigned long used;
}dictht;
```
`table`是一个数组，数组中的每一个元素都是一个指向dictEntry结构的指针，每个dictEntry结构都保存着一个键值对。
``` c
typedef struct dictEntry{
    void key;
    union{
        void *val;
        uint64_t u64;
        int64_t s64;
    }v;
    struct dictEntry *next;//哈希冲突时，采用链表法解决
}dictEntry;
```
字典
: 
``` c
typedef struct dict{
    dictType *type;//类型特定函数
    void *privdata;//私有数据
    dictht ht[2];//哈希表
    int trehashidx;//rehash索引，当rehash不进行时，为-1；
}dict;
```
了解：`type`属性是指向dictType结构的指针，每个dictType结构保存了一簇用于操作特定类型键值对的函数，`privdate`保存了传给那些函数的可选参数。

`ht`是一个长度为2的数组，包含了两个dictht哈希表。一般情况下字典只使用`ht[0]`哈希表，`ht[1]`只有在进行rehash的时候使用。`rehashidx`记录了rehash的进度。

哈希算法
: 当要将一个新的键值对添加到字典时，程序需要先根据键值对的键计算出哈希值和索引值，然后根据索引值将节点放到韩系标书组的指定索引上面。Redis使用MurmurHash2算法来计算哈希值(Redis v3.0)。
``` c
    hash = dict->type->hashFunction(key);
    index = hash & dict->ht[x].sizemask;
```

### 解决键冲突
当有两个或以上数量的键被分配到了哈希表数组的同一个索引上时，我们称这些键发生了冲突。Redis的哈希表使用了**链地址法**解决键冲突，每个哈希表节点都由一个next指针，被分配到同一个索引上的节点可以用next连接成单向链表，新加入的节点总是排在所有节点的前面。

### 扩容（收缩）和rehash
哈希表的键值对会逐渐的增加或减少，为了让哈希表的负载因子(load factor)，维持在一个合理的范围，程序需要对哈希表进行相应的扩容或收缩。当满足
以下两点任意一点时，扩容开始。当满足负**载因子小于0.1**时，进行收缩。

 1. 服务器目前没有在执行*BGSAVE*命令或者*BGREWRITEAOF*命令，并且哈希表的负载因子大于等于1
 2. 服务器目前正在执行*BGSAVE*命令或者*BGREWRITEAOF*命令，并且哈希表的负载因子大于等于5

其中负载因子计算方法为：

        load_factor = ht[0].used/dictht.size

步骤如下：

 1. 为字典的`ht[1]`分配空间，如果执行的是扩展操作，那么`ht[1]`的大小为大于等于`ht[0].used*2`的最小2^n；若是收缩操作，那么`ht[1]`的大小为大于等于`ht[0].used`的最小2^n。
 2. 将保存在`ht[0]`中的所有键值对rehash（重新计算键的哈希和索引）到`ht[1]`上，并将键值对放置到`ht[1]`。
 3. 当`ht[0]`的所有键值对都迁移到了`ht[1]`上后，释放`ht[0]`，将`ht[1]`设置为`ht[0]`，并在`ht[1]`新建一个空白的哈希表，为下一次rehash做准备。
 
Redis的rehash时**渐进式**的，即不是一次性、集中式完成的，而是分多次完成。这样做是避免键值对过多时，庞大的计算量导致服务器停止服务。渐进式的步骤如下：
 1. 为`ht[1]`分配空间
 2. 在字典中维持`rehashidx`,将其值设置为0，表示rehash正式开始
 3. 在rehash期间，每次对字典的增删查改，程序都会顺带将`ht[0]`哈希表在`rehashidx`索引上的所有键值对rehash到`ht[1]`，当rehash工作完成后，`rehashidx`增加1
 4. 随着字典操作不断执行，最终某个时间上，ht[0]的所有键值对都会被rehash至`ht[1]`,`rehashidx`设为-1，代表rehash完成。
 5. rehash过程中，字典会同时使用两个哈希表，如`ht[0]`没找到，会继续去`ht[1]`找。而添加一律会被保存在`ht[1]`。

举例: `set msg HelloWorld`这个命令执行以后，redis会在`dict->ht->dictht->table`中加入一个`dictEntry`，entry的位置由`dict`内置的hash算法(比如MurmurHash2)与大小掩码作与运算得出，这个`dictEntry`包含了`key-value`和`next`属性，key和value是redis的`SDS`简单动态字符串的抽象类。

### 总结
- 字典被广泛用于实现Redis（远程字典服务）的各种功能，其中包括数据库和哈希键。
- Redis中的字典使用哈希表作为底层实现，每个字典待优两个哈希表，一个平时使用，另一个仅在进行rehash时使用
- Redis目前使用MurmurHash2算法计算键的哈希值
- 哈希表使用链地址法来解决冲突，被分配到同一个索引上的多个键值对会连接成一个单向链表
- 在对哈希表进行扩展或收缩操作时，程序需要将现有哈希表包含的所有键值对rehash到新哈希表里面，并且这个rehash过程不是一次性完完成的，而是伴随着每一次增删改查渐进式完成的