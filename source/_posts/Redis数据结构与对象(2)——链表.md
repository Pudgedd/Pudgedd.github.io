---
title: Redis数据结构与对象(2)——链表
date: 2019-01-31 23:16:58
categories: Redis
tags: 
    - Redis
    - 数据结构
    - 链表
---

## 链表
链表提供了高效的节点重排能力，以及顺序性节点的访问方式，并且可以通过增删节点来灵活地调整长度。Redis中的**列表键**(list)的底层实现之一就是链表。当一个链表键包含了较多元素，又或者列表中包含的元素都是较长的字符串时，Redis就会使用链表作为列表键的底层实现。（实际上链表还被广泛用于实现Redis的各种功能，比如发布与订阅、慢查询、监视器等）
<!-- more -->

    redis>LLEN integers
    (integer) 1024
    redis>LRANGE integers 0 3
    1)"1"
    2)"2"
    3)"3"
    
integers列表键底层实现就是一个链表，每个节点都保存了一个整数值。

Redis链表的实现
: 每个链表节点都包含了指向prev和next的两个指针，同时还有保存节点值的value指针。
``` c
    typedef struct listNode{
        struct listNode *prev;
        struct listNode *next;
        void *value;
    }listNode;
```
Redis list结构
: list结构为链表提供了表头指针，表尾指针，以及链表长度计数器len，这么做的好处是获取prev和next节点、获取head和tail节点、获取链表长度的时间复杂度均为O(1)。此外还提供了三个特定函数。
``` c
    typedef struct list{
        listNode *head;
        listNode *tail;
        unsigned long len;
        void *(*dup)(void *ptr);
        void *(*free)(void *ptr);
        int *(*match)(void *ptr,void *key);
    }list;
```