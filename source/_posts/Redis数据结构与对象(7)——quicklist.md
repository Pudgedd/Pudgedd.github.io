---
title: Redis数据结构与对象(7)——quicklist
date: 2019-03-18 17:08:02
categories: Redis
tags: 
    - Redis
    - 数据结构
    - quicklist
---

## quicklist
Redis3.2分支以后引入了一种新的内部数据结构——`quicklist`，`quicklist`经常被用作**列表键**的实现之一，4.0版本以后取消了**列表键对象**的`ziplist`和`linkedlist`编码，统一使用`quicklist`编码。
<!-- more -->

    redis> RPUSH lst 1 3 5 10086 "hello" "world"
    "6"
    reids> OBJECT ENCODING lst
    "quicklist"

### quicklist的构成
Redis4.0版本以后用自定义的一种复杂数据结构`quicklist`取代了`ziplsit`和`linkedlist`，它本身是一个**双向无环链表**，它的每一个节点都是一个`ziplist`，可以看出`quicklist`其实是将`ziplist`和`linkedlist`结合到了一个数据结构中。为什么这么设计呢？
- 链表在插入，删除节点的时间复杂度很低；但是内存利用率低，且由于内存不连续容易产生内存碎片
- 压缩表内存连续，存储效率高；但是插入和删除的成本太高，需要频繁的进行数据搬移、释放或申请内存（每次数据变动都会引发一次内存的realloc。特别是当`ziplist`长度很长的时候，一次realloc可能会导致大批量的数据拷贝，进一步降低性能）。
  
而`quicklist`通过将每个压缩表用双向链表的方式连接起来，来寻求一种收益最大化。

#### 定义
首先是`quicklist`的节点`quicklistNode`：

```c
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;  // ziplist
    unsigned int sz;             // ziplist的内存大小
    unsigned int count : 16;     // zpilist中数据项的个数
    unsigned int encoding : 2;   // 1为ziplist 2是LZF压缩存储方式
    unsigned int container : 2;  
    unsigned int recompress : 1;   // 压缩标志, 为1 是压缩
    unsigned int attempted_compress : 1; // 节点是否能够被压缩,只用在测试
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```

`quicklistNode`实际上就是对`ziplist`的进一步封装，其中包括：
- `prev`: 指向链表前一个节点的指针。
- `next`: 指向链表后一个节点的指针。
zl: 数据指针。如果当前节点的数据没有压缩，那么它指向一个`ziplist`结构；否则，它指向一个`quicklistLZF`结构。
- `sz`: 表示`zl`指向的`ziplist`的总大小（包括`zlbytes`, `zltail`, `zllen`, `zlend`和各个数据项）。需要注意的是：如果`ziplist`被压缩了，那么这个`sz`的值仍然是压缩前的`ziplist`大小。
- `count`: 表示`ziplist`里面包含的数据项个数。这个字段只有16bit。稍后我们会一起计算一下这16bit是否够用。
- `encoding`: 表示`ziplist`是否压缩了（以及用了哪个压缩算法）。目前只有两种取值：2表示被压缩了（而且用的是LZF压缩算法），1表示没有压缩。
- `container`: 是一个预留字段。本来设计是用来表明一个`quicklist`节点下面是直接存数据，还是使用`ziplist`存数据，或者用其它的结构来存数据（用作一个数据容器，所以叫`container`）。但是，在目前的实现中，这个值是一个固定的值2，表示使用`ziplist`作为数据容器。
- `recompress`: 当我们使用类似`lindex`这样的命令查看了某一项本来压缩的数据时，需要把数据暂时解压，这时就设置`recompress=1`做一个标记，等有机会再把数据重新压缩。
- `attempted_compress`: 这个值只对Redis的自动化测试程序有用。我们不用管它。
- `extra`: 其它扩展字段。目前Redis的实现里也没用上。

`quicklistLZF`结构表示一个被压缩过的`ziplist`。其中：

- `sz`: 表示压缩后的`ziplist`大小。
- `compressed`: 是个柔性数组（flexible array member），存放压缩后的`ziplist`字节数组。

`quicklist`: 

```c
typedef struct quicklist {
    quicklistNode *head;    // 头结点
    quicklistNode *tail;    // 尾节点
    unsigned long count;    // 所有数据的数量
    unsigned int len;       // quicklist节点数量
    int fill : 16;          // 单个ziplist的大小限制
    unsigned int compress : 16;   // 压缩深度
} quicklist;
```

真正表示`quicklist`的数据结构是同名的`quicklist`这个struct：

- `head`: 指向头节点（左侧第一个节点）的指针。
- `tail`: 指向尾节点（右侧第一个节点）的指针。
- `count`: 所有`ziplist`数据项的个数总和。
- `len`: `quicklist`节点的个数。
- `fill`: 16bit，`ziplist`大小设置，存放`list-max-ziplist-size`参数的值。
- `compress`: 16bit，节点压缩深度设置，存放`list-compress-depth`参数的值。
  
### quicklistNode的大小
到底一个`quicklistNode`节点包含多长的`ziplist`合适呢？比如，同样是存储12个数据项，既可以是一个`quicklist`包含3个节点，而每个节点的`ziplist`又包含4个数据项，也可以是一个`quicklist`包含6个节点，而每个节点的`ziplist`又包含2个数据项。

这又是一个需要找平衡点的难题。我们只从存储效率上分析一下：

- 每个`quicklist`节点上的`ziplist`越短，则内存碎片越多。内存碎片多了，有可能在内存中产生很多无法被利用的小碎片，从而降低存储效率。这种情况的极端是每个`quicklist`节点上的`ziplist`只包含一个数据项，这就蜕化成一个普通的双向链表了。
- 每个`quicklist`节点上的`ziplist`越长，则为`ziplist`分配大块连续内存空间的难度就越大。有可能出现内存里有很多小块的空闲空间（它们加起来很多），但却找不到一块足够大的空闲空间分配给`ziplist`的情况。这同样会降低存储效率。这种情况的极端是整个`quicklist`只有一个节点，所有的数据项都分配在这仅有的一个节点的`ziplist`里面。这其实蜕化成一个`ziplist`了。
  
可见，一个`quicklist`节点上的`ziplist`要保持一个合理的长度。那到底多长合理呢？这可能取决于具体应用场景。实际上，Redis提供了一个配置参数`list-max-ziplist-size`，就是为了让使用者可以来根据自己的情况进行调整。
    
    list-max-ziplist-size -2
    
我们来详细解释一下这个参数的含义。它可以取正值，也可以取负值。

当取**正值**的时候，表示**按照数据项个数**来限定每个`quicklist`节点上的`ziplist`长度。比如，当这个参数配置成5的时候，表示每个`quicklist`节点的`ziplist`最多包含5个数据项。当取**负值**的时候，表示**按照占用字节数**来限定每个`quicklist`节点上的`ziplist`长度。这时，它只能取-1到-5这五个值，每个值含义如下：

* -5: 每个`quicklist`节点上的`ziplist`大小不能超过64 Kb。（注：1kb => 1024 bytes）
* -4: 每个`quicklist`节点上的`ziplist`大小不能超过32 Kb。
* -3: 每个`quicklist`节点上的`ziplist`大小不能超过16 Kb。
* -2: 每个`quicklist`节点上的`ziplist`大小不能超过8 Kb。（**-2是Redis给出的默认值**）
* -1: 每个`quicklist`节点上的`ziplist`大小不能超过4 Kb。
* 为正数时，表示单个节点最大允许的元素个数，最大为32768个

当列表很长的时候，最容易被访问的很可能是两端的数据，中间的数据被访问的频率比较低（访问起来性能也很低）。如果应用场景符合这个特点，那么list还提供了一个选项，能够把中间的数据节点进行压缩，从而进一步节省内存空间。Redis的配置参数list-compress-depth就是用来完成这个设置的。

    list-compress-depth 0

这个参数表示一个`quicklist`两端**不被压缩**的节点个数。注：这里的节点个数是指`quicklist`双向链表的节点个数，而不是指`ziplist`里面的数据项个数。实际上，一个`quicklist`节点上的`ziplist`，如果被压缩，就是整体被压缩的。

参数`list-compress-depth`的取值含义如下：
- 0: 是个特殊值，表示**都不压缩**。这是Redis的**默认值**。
- 1: 表示`quicklist`两端各有1个节点不压缩，中间的节点压缩。
- 2: 表示`quicklist`两端各有2个节点不压缩，中间的节点压缩。
- 3: 表示`quicklist`两端各有3个节点不压缩，中间的节点压缩。
依此类推…
- 由于0是个特殊值，很容易看出`quicklist`的头节点和尾节点总是不被压缩的，以便于在表的两端进行快速存取。

Redis对于`quicklist`内部节点的压缩算法，采用的`LZF`——**一种无损压缩算法**。

### 总结
- `quicklist`除了常用的增删改查外还提供了merge、将`ziplist`转换为`quicklist`等API
- `quicklist`是Redis3.2分支后在`ziplist`和`linkedlist`两种数据结构的基础上融合而成的一个实用的复杂数据结构
- `quicklist`在4.0之后取代了`linkedlist`和`ziplist`作为list的基础数据类型
- `quicklist`的大部分API都是直接复用`ziplist`
- `quicklist`的单个节点最大存储默认为8kb
- `quicklist`提供了基于LZF算法的压缩API，通过将不常用的中间节点数据压缩达到节省内存的目的