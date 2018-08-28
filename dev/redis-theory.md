# 数据类型

## Redis 5种结构类型

| 结构类型 | 结构存储值  | 结构读写能力 |
| -------: | :---- | ------------ |
| STRING | 字符串、整数、浮点数 | 字符串修改；整数/浮点数的自增自减； |
| LIST | 链表，节点包含字符串数据  | 链表两端推入或弹出元素；根据偏移量修剪(trim)；读取单个或多个元素；根据值查找删除 |
| SET | 包含字符串的无序收集器(unordered collection)，并且不允许重复 | 增删改单个元素；检查单个元素是否存在于集合；计算多个集合的交集、并集、差集；随机获取元素 |
| HASH | 包含键值对的无序散列 | 增删改单个键值对；获取所有键值对 |
| ZSET | 字符串成员(member)与浮点数分值(score)之间的有序映射，元素排序由分值大小决定 | 增删改单个键元素；根据范围(range)或成员获取元素  |



## Redis 中的字符串

| 命令 | 行为                |
| ---- | ------------------ |
| GET  | 获取存储在给定键中的值 |
| SET  | 设置存储在给定键中的值 |
| DEL  | 删除存储在给定键中的值 |



# 数据结构与对象

## 简单动态字符串

Redis没有直接使用C语言传统的字符串表示（以空字符结尾的字符数组，以下简称C字符串），而是自己构建了一种名为简单动态字符串（simple dynamic string，SDS）的抽象类型，并将SDS用作Redis的默认字符串表示。

### 实现

- 键值对的键是一个字符串对象，对象的底层实现是一个保存了字符串“fruits”的SDS。   

- 键值对的值是一个列表对象，列表对象包含了三个字符串对象，这三个字符串对象分别由三个SDS实现：
  - 第一个SDS保存着字符串“apple”
  - 第二个SDS保存着字符串“ba-nana”
  - 第三个SDS保存着字符串“cherry”

> 除了用来保存数据库中的字符串值之外，SDS还被用作缓冲区（buffer）：AOF模块中的AOF缓冲区，以及客户端状态中的输入缓冲区。

```c
struct sdshdr {    
    int len;   // 记录buf数组中已使用字节的数量，等于SDS所保存字符串的长度    
    int free;  // 记录buf数组中未使用字节的数量   
    char buf[];// 字节数组，用于保存字符串
};
```

> Redis只会使用C字符串作为字面量，在大多数情况下，Redis使用SDS（Simple Dynamic String，简单动态字符串）作为字符串表示。 

### 优点

- 常数复杂度获取字符串长度。
- 杜绝缓冲区溢出。
- 减少修改字符串长度时所需的内存重分配次数。
- 二进制安全。
- 遵循C字符串以空字符结尾的惯例，兼容部分C字符串函数。
- 惰性空间释放，避免了缩短字符串时所需的内存重分配操作

### 区别

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/3a134d1acaa28116fe9cab6f5f67278e?thumb=1024x&scale=auto)

### API

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/110620bd29f375409057116deeeb490d?thumb=1024x&scale=auto)
![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/1a42d918398e85e689f72876263d86c3)



## 链表

因为Redis使用的C语言并没有内置这种数据结构，所以Redis构建了自己的链表实现。链表提供了高效的节点重排能力，以及顺序性的节点访问方式，并且可以通过增删节点来灵活地调整链表的长度。

*链表、发布与订阅、慢查询、监视器等功底层实现就是链表。*

### 实现

```c
typedef struct listNode {    
    // 前置节点    
    struct listNode *prev;    
    // 后置节点    
    struct listNode *next;    
    // 节点的值    
    void * value;
}listNode;

typedef struct list {
     // 表头节点    
    listNode * head;    
    // 表尾节点    
    listNode * tail;    
    // 链表所包含的节点数量    
    unsigned long len;    
    // 节点值复制函数    
    void *(*dup)(void *ptr);    
    // 节点值释放函数    
    void (*free)(void *ptr);    
    // 节点值对比函数    
    int (*match)(void *ptr,void *key);
} list;
```

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/520ca4ec8ddb87d5c4a0adec2fb33696?thumb=1024x&scale=auto)![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/520ca4ec8ddb87d5c4a0adec2fb33696?thumb=1024x&scale=auto)

list结构为链表提供了表头指针head、表尾指针tail，以及链表长度计数器len，而dup、free和match成员则是用于实现多态链表所需的类型特定函数：
- dup函数用于复制链表节点所保存的值；
- free函数用于释放链表节点所保存的值；
- match函数则用于对比链表节点所保存的值和另一个输入值是否相等。

### 特性

- 双端：链表节点带有prev和next指针，获取某个节点的前置节点和后置节点的复杂度都是O（1）。 
- 无环：表头节点的prev指针和表尾节点的next指针都指向NULL，对链表的访问以NULL为终点。 
- 带表头指针和表尾指针：通过list结构的head指针和tail指针，程序获取链表的表头节点和表尾节点的复杂度为O（1）。 
- 带链表长度计数器：程序使用list结构的len属性来对list持有的链表节点进行计数，程序获取链表中节点数量的复杂度为O（1）。
- 多态：链表节点使用void*指针来保存节点值，并且可以通过list结构的dup、free、match三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值。

### API

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/490d3fc4720b24652d72e7f7a4285124?thumb=1024x&scale=auto)
![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/ea338d2f71187fe332353ce7b41a292a?thumb=1024x&scale=auto)![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/aa78961fcf4e95c17949f4352992a836?thumb=1024x&scale=auto)


### 总结
- 链表被广泛用于实现Redis的各种功能，比如列表键、发布与订阅、慢查询、监视器等。
- 每个链表节点由一个listNode结构来表示，每个节点都有一个指向前置节点和后置节点的指针，所以Redis的链表实现是双端链表。
- 每个链表使用一个list结构来表示，这个结构带有表头节点指针、表尾节点指针，以及链表长度等信息。
- 因为链表表头节点的前置节点和表尾节点的后置节点都指向NULL，所以Redis的链表实现是无环链表。 
- 通过为链表设置不同的类型特定函数，Redis的链表可以用于保存各种不同类型的值。



## 字典

C语言并没有内置这种数据结构，因此Redis构建了自己的字典实现。字典，又称为符号表（symbol table）、关联数组（associa-tive array）或映射（map），是一种用于保存键值对（key-valuepair）的抽象数据结构。在字典中，一个键（key）可以和一个值（value）进行关联（或者说将键映射为值），这些关联的键和值就称为键值对。

### 实现

Redis的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。

*Redis数据库就是使用字典来作为底层实现的，对数据库的增、删、查、改操作也是构建在对字典的操作之上。除了用来表示数据库之外，字典还是哈希键的底层实现之一，当一个哈希键包含的键值对比较多，又或者键值对中的元素都是比较长的字符串时，Redis就会使用字典作为哈希键的底层实现。*

```c
// Redis中的字典由dict.h/dict结构表示：
typedef struct dict {    
    // 类型特定函数，指向dictType结构的指针，Redis会为用途不同的字典设置不同的类型特定函数。
    dictType *type;    
    // 私有数据，则保存了需要传给那些类型特定函数的可选参数。  
    void *privdata;    
    // 哈希表    
    dictht ht[2];    
    // rehash索引 当rehash不在进行时，值为-1    
    int trehashidx; /* rehashing not in progress if rehashidx == -1 */
} dict;

// Redis字典所使用的哈希表由dict.h/dictht结构定义
typedef struct dictht {    
    // 哈希表数组，每个元素都是一个指向dict.h/dictEntry结构的指针
    dictEntry **table;    
    // 哈希表大小    
    unsigned long size;    
    //哈希表大小掩码，用于计算索引值总是等于size-1    
    unsigned long sizemask;    
    // 该哈希表已有节点的数量    
    unsigned long used;
} dictht;

// 哈希表节点使用dictEntry结构表示
typedef struct dictEntry {
    // 键    
    void *key;    
    // 值    
    union{        
        void *val;    // 指针   
        uint64_tu64;  // uint64_t整数      
        int64_ts64;   // int64_t整数
    } v;    
    // 指向下个哈希表节点，形成链表。
    // 将多个哈希值相同的键值对连接在一起，以此来解决键冲突（collision）的问题。
    // dictEntry节点组成的链表没有指向链表表尾的指针，所以将新节点添加到链表的表头位置 O(1)
    struct dictEntry *next;
} dictEntry;

// 每个dictType结构保存了一簇用于操作特定类型键值对的函数
typedef struct dictType {    
    // 计算哈希值的函数    
    unsigned int (*hashFunction)(const void *key);    
    // 复制键的函数    
    void *(*keyDup)(void *privdata, const void *key);    
    // 复制值的函数    
    void *(*valDup)(void *privdata, const void *obj);    
    // 对比键的函数    
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);    
    // 销毁键的函数    
    void (*keyDestructor)(void *privdata, void *key);    
    // 销毁值的函数    
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

### Hash 算法

- 使用字典设置的哈希函数，计算键key的哈希值 ` hash = dict->type->hashFunction(key);`
- 使用哈希表的 `sizemask `属性和哈希值，计算出索引值
- 根据情况不同，`ht[x]` 可以是 `ht[0]` 或者 `ht[1]` `index = hash & dict->ht[x].sizemask;`

*当字典被用作数据库的底层实现，或者哈希键的底层实现时，Redis使用` MurmurHash2 `算法来计算键的哈希值。https://github.com/aappleby/smhasher*

#### Hash 碰撞

Redis的哈希表使用链地址法（separate chaining）来解决键冲突，每个哈希表节点都有一个next指针，多个哈希表节点可以用next指针构成一个单向链表，被分配到同一个索引上的多个节点可以用这个单向链表连接起来，这就解决了键冲突的问题。

因为dictEntry节点组成的链表没有指向链表表尾的指针，所以为了速度考虑，程序总是将新节点添加到链表的表头位置（复杂度为O（1）），排在其他已有节点的前面。

#### Rehash

为了让哈希表的负载因子（load factor）维持在一个合理的范围之内，当哈希表保存的键值对数量太多或者太少时，程序需要对哈希表的大小进行相应的扩展或者收缩。

扩展和收缩哈希表的工作可以通过执行rehash（重新散列）操作来完成，Redis对字典的哈希表执行rehash的步骤如下：

- 为字典的 ht[1] 哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及ht[0]当前包含的键值对数量（也即是 ht[0].used 属性的值）：   
  - 如果执行的是扩展操作，那么 ht[1] 的大小为第一个大于等于 ht[0].used*2 的 2^n（2的n次方幂）；  
  - 如果执行的是收缩操作，那么 ht[1] 的大小为第一个大于等于 ht[0].used 的 2^n。
- 将保存在 ht[0] 中的所有键值对 rehash 到 ht[1] 上面：re-hash 指的是重新计算键的哈希值和索引值，然后将键值对放置到 ht[1] 哈希表的指定位置上。
- 当 ht[0] 包含的所有键值对都迁移到了 ht[1] 之后（ht[0] 变为空表），释放 ht[0]，将 ht[1] 设置为 ht[0]，并在ht[1] 新创建一个空白哈希表，为下一次 rehash 做准备。（这个rehash动作并不是一次性、集中式地完成的，而是分多次、渐进式地完成的。）

#### 渐进式rehash

为了避免rehash对服务器性能造成影响，服务器不是一次性将 ht[0] 里面的所有键值对全部 rehash 到 ht[1]，而是分多次、渐进式地将ht[0]里面的键值对慢慢地 rehash 到 ht[1]。以下是哈希表渐进式rehash的详细步骤：

- 为 ht[1] 分配空间，让字典同时持有 ht[0] 和 ht[1] 两个哈希表。

- 在字典中维持一个索引计数器变量 rehashidx，并将它的值设置为 0，表示 rehash 工作正式开始。

- 在 rehash 进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将 ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1]，当 rehash 工作完成之后，程序将 rehashidx 属性的值增一。4）随着字典操作的不断执行，最终在某个时间点上，ht[0] 的所有键值对都会被 rehash 至 ht[1]，这时程序将 rehashidx 属性的值设为 -1，表示 rehash 操作已完成。

*分而治之，避免了集中式rehash而带来的庞大计算量。*

渐进式rehash进行期间，字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行。

#### Hash 的扩展与收缩

当以下条件中的任意一个被满足时，程序会自动开始对哈希表执行扩展操作：

- 服务器目前没有在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1。

- 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5。

其中哈希表的负载因子可以通过公式：

	`负载因子= 哈希表已保存节点数量/ 哈希表大小load_factor = ht[0].used / ht[0].size`

*当哈希表的负载因子小于0.1时，程序自动开始对哈希表执行收缩操作。*

### 字典API

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/cccd882f627fb236b2ef07fb300fda6b?thumb=1024x&scale=auto)
![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/ce712ce6839f3081c9b6a2e46ee41c61?thumb=1024x&scale=auto)

### 总结

- 字典被广泛用于实现Redis的各种功能，其中包括数据库和哈希键。 
- Redis中的字典使用哈希表作为底层实现，每个字典带有两个哈希表，一个平时使用，另一个仅在进行rehash时使用。
- 当字典被用作数据库的底层实现，或者哈希键的底层实现时，Redis使用MurmurHash2算法来计算键的哈希值。 
- 哈希表使用链地址法来解决键冲突，被分配到同一个索引上的多个键值对会连接成一个单向链表。
- 在对哈希表进行扩展或者收缩操作时，程序需要将现有哈希表包含的所有键值对rehash到新哈希表里面，并且这个re-hash过程并不是一次性地完成的，而是渐进式地完成的。



## 跳跃表

跳跃表（skiplist）是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

*Redis只在两个地方用到了跳跃表，一个是实现有序集合键，另一个是在集群节点中用作内部数据结构*

### 实现

Redis的跳跃表由redis.h/zskiplistNode和redis.h/zskiplist两个结构定义，其中zskiplistNode结构用于表示跳跃表节点，而zskiplist结构则用于保存跳跃表节点的相关信息。

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/2ec2ac41ad803ad2978cab047f74079c?thumb=1024x&scale=auto)

**zskiplist结构**

- header：指向跳跃表的表头节点   
- tail：指向跳跃表的表尾节点
- level：记录目前跳跃表内，层数最大的那个节点的层数（表头节点的层数不计算在内）
- length：记录跳跃表的长度，也即是，跳跃表目前包含节点的数量（表头节点不计算在内）

**zskiplistNode结构**

- 层（level）：节点中用L1、L2、L3等字样标记节点的各个层，L1代表第一层，L2代表第二层，以此类推。每个层都带有两个属性：前进指针和跨度。前进指针用于访问位于表尾方向的其他节点，而跨度则记录了前进指针所指向节点和当前节点的距离。在上面的图片中，连线上带有数字的箭头就代表前进指针，而那个数字就是跨度。当程序从表头向表尾进行遍历时，访问会沿着层的前进指针进行。 
- 后退（backward）指针：节点中用BW字样标记节点的后退指针，它指向位于当前节点的前一个节点。后退指针在程序从表尾向表头遍历时使用。 
- 分值（score）：各个节点中的1.0、2.0和3.0是节点所保存的分值。在跳跃表中，节点按各自所保存的分值从小到大排列。
- 成员对象（obj）：各个节点中的o1、o2和o3是节点所保存的成员对象。

*表头节点也有后退指针、分值和成员对象，不过表头节点的这些属性都不会被用到*

```c
typedef struct zskiplist {    
    // 表头节点和表尾节点    
    structz skiplistNode *header, *tail;    
    // 表中节点的数量    
    unsigned long length;    
    // 表中层数最大的节点的层数    
    int level;
} zskiplist;

typedef struct zskiplistNode {    
    // 层，可以包含多个元素，每个元素都包含一个指向其他节点的指针，程序可以通过这些层来加快访问其他节点的速度
    struct zskiplistLevel {        
        // 前进指针，用于从表头向表尾方向访问节点
        struct zskiplistNode *forward;        
        // 跨度，用于记录两个节点之间的距离
        unsigned int span;    
    } level[];    
    // 后退指针，用于从表尾向表头方向访问节点，每次只能后退至前一个节点
    struct zskiplistNode *backward;    
    // 分值，一个double类型的浮点数，跳跃表中的所有节点都按分值从小到大来排序
    // 分值相同的节点将按照成员对象在字典序中的大小来进行排序，成员对象较小的节点会排在前面
    double score;    
    // 成员对象，指向一个字符串对象，而字符串对象则保存着一个SDS值 
    robj *obj;
} zskiplistNode;
```

### API
![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/1d6a08c66cea75e0f17dbe22d8d86c85?thumb=1024x&scale=auto)
![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/7f1710e3af802d88916e7a78a7b6bab3?thumb=1024x&scale=auto)

### 总结

- 跳跃表是有序集合的底层实现之一。
- Redis的跳跃表实现由zskiplist和zskiplistNode两个结构组成，其中zskiplist用于保存跳跃表信息（比如表头节点、表尾节点、长度），而zskiplistNode则用于表示跳跃表节点。
- 每个跳跃表节点的层高都是1至32之间的随机数。
- 在同一个跳跃表中，多个节点可以包含相同的分值，但每个节点的成员对象必须是唯一的。
- 跳跃表中的节点按照分值大小进行排序，当分值相同时，节点按照成员对象的大小进行排序。

## 整数集合

整数集合（intset）是集合键的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数集合作为集合键的底层实现。

整数集合（intset）是Redis用于保存整数值的集合抽象数据结构，它可以保存类型为int16_t、int32_t或者int64_t的整数值，并且保证集合中不会出现重复元素。

### 实现

```c
typedef struct intset {    
    // 编码方式    
    uint32_t encoding;    
    // 集合包含的元素数量，即是con-tents数组的长度
    uint32_t length;    
    // 保存元素的数组，是整数集合的底层实现。
    // 整数集合的每个元素都是contents数组的一个数组项（item），各个项在数组中按值的大小从小到大有序地排列，并且数组中不包含任何重复项。
    int8_t contents[];
} intset;
```

- 如果encoding属性的值为INTSET_ENC_INT16，那么contents就是一个int16_t类型的数组，数组里的每个项都是一个int16_t类型的整数值（最小值为-32768，最大值为32767）。   

- 如果encoding属性的值为INTSET_ENC_INT32，那么contents就是一个int32_t类型的数组，数组里的每个项都是一个int32_t类型的整数值（最小值为-2147483648，最大值为2147483647）。   

- 如果encoding属性的值为INTSET_ENC_INT64，那么contents就是一个int64_t类型的数组，数组里的每个项都是一个int64_t类型的整数值（最小值为-9223372036854775808，最大值为9223372036854775807）。

### 升级

当一个新元素添加到整数集合里面，并且新元素的类型比整数集合现有所有元素的类型都要长时，整数集合需要先进行升级（upgrade），然后才能将新元素添加到整数集合里面。

**升级整数集合并添加新元素共分为三步进行：**

- 根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间。

- 将底层数组现有的所有元素都转换成与新元素相同的类型，并将类型转换后的元素放置到正确的位上，而且在放置元素的过程中，需要继续维持底层数组的有序性质不变。

- 将新元素添加到底层数组里面。

*整数集合的升级策略有两个好处，一个是提升整数集合的灵活性，另一个是尽可能地节约内存。*

### 降级

整数集合不支持降级操作，一旦对数组进行了升级，编码就会一直保持升级后的状态。

### API

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/7108172239c0fecbf9550d30095c2931?thumb=1024x&scale=auto)

### 总结

- 整数集合是集合键的底层实现之一。
- 整数集合的底层实现为数组，这个数组以有序、无重复的方式保存集合元素，在有需要时，程序会根据新添加元素的类型，改变这个数组的类型。
- 升级操作为整数集合带来了操作上的灵活性，并且尽可能地节约了内存。
- 整数集合只支持升级操作，不支持降级操作。



## 压缩列表

压缩列表（ziplist）是列表键和哈希键的底层实现之一。当一个列表键只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做列表键的底层实现。

当一个哈希键只包含少量键值对，比且每个键值对的键和值要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做哈希键的底层实现。

### 构成

压缩列表是一系列特殊编码的连续内存块组成的顺序型（sequential）数据结构。一个压缩列表可以包含任意多个节点（entry），每个节点可以保存一个字节数组或者一个整数值。

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/4acc6553117117522df129ca673e4ebc?thumb=1024x&scale=auto)

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/b7000201647b1a97d5f8aa08c4a14add?thumb=1024x&scale=auto)

每个压缩列表节点可以保存一个字节数组或者一个整数值，其中，字节数组可以是以下三种长度的其中一种： 
- 长度小于等于63（26–1）字节的字节数组
- 长度小于等于16383（214–1）字节的字节数组
- 长度小于等于4294967295（232–1）字节的字节数组；

而整数值则可以是以下六种长度的其中一种： 
- 4位长，介于0至12之间的无符号整数
- 1字节长的有符号整数 
- 3字节长的有符号整数
- int16_t类型整数
- int32_t类型整数
- int64_t类型整数

每个压缩列表节点都由previous_entry_length、encoding、content三个部分组成
![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/8c63994b132e4d4d438f22e1444802d1?thumb=1024x&scale=auto)

####  previous_entry_length

节点的previous_entry_length属性以字节为单位，记录了压缩列表中前一个节点的长度。
-  如果前一节点的长度小于254字节，那么previous_en-try_length属性的长度为1字节：前一节点的长度就保存在这一个字节里面。 
- 如果前一节点的长度大于等于254字节，那么previ-ous_entry_length属性的长度为5字节：其中属性的第一字节会被设置为0xFE（十进制值254），而之后的四个字节则用于保存前一节点的长度。

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/10060254d5f99b3593245cabc52f800c?thumb=1024x&scale=auto)

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/65552a65c574b724b534f412d61dc296?thumb=1024x&scale=auto)

#### encoding

节点的encoding属性记录了节点的content属性所保存数据的类型以及长度：
- 一字节、两字节或者五字节长，值的最高位为00、01或者10的是字节数组编码：这种编码表示节点的content属性保存着字节数组，数组的长度由编码除去最高两位之后的其他位记录； 
- 一字节长，值的最高位以11开头的是整数编码：这种编码表示节点的content属性保存着整数值，整数值的类型和长度由编码除去最高两位之后的其他位记录；

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/fb92d7c044e2bd9647460dd14450e566?thumb=1024x&scale=auto)
![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/2b4917ee083e06b62aa9831906fc6cd6?thumb=1024x&scale=auto)

#### content

节点的content属性负责保存节点的值，节点值可以是一个字节数组或者整数，值的类型和长度由节点的encoding属性决定。
- 编码的最高两位00表示节点保存的是一个字节数组
- 编码的后六位001011记录了字节数组的长度11
- content属性保存着节点的值"hello world"

### 连锁更新

特殊情况下产生的连续多次空间扩展操作称之为“连锁更新”（cascade update）。除了添加新节点可能会引发连锁更新之外，删除节点也可能会引发连锁更新。

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/a4f62c4084922a1b007cbb861d5e6dfc?thumb=1024x&scale=auto)

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/e2fab0a3d01d653cf4ec37973c5db931?thumb=1024x&scale=auto)

### API

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/141620407036e7e4be52e50cc9d854b6?thumb=1024x&scale=auto)

*因为ziplistPush、ziplistInsert、ziplistDelete和ziplist-DeleteRange四个函数都有可能会引发连锁更新，所以它们的最坏复杂度都是O（N2）。*

### 总结

- 压缩列表是一种为节约内存而开发的顺序型数据结构。
- 压缩列表被用作列表键和哈希键的底层实现之一。
- 压缩列表可以包含多个节点，每个节点可以保存一个字节数组或者整数值。
- 添加新节点到压缩列表，或者从压缩列表中删除节点，可能会引发连锁更新操作，但这种操作出现的几率并不高。



## 对象

Redis使用对象来表示数据库中的键和值，每次当我们在Redis的数据库中新创建一个键值对时，我们至少会创建两个对象，一个对象用作键值对的键（键对象），另一个对象用作键值对的值（值对象）。

### 结构

Redis中的每个对象都由一个redisObject结构表示，该结构中和保存数据有关的三个属性分别是type属性、encoding属性和ptr属性

```c
typedef struct redisObject {    
    // 类型    
    unsigned type:4;    
    // 编码    
    unsigned encoding:4;    
    // 指向底层实现数据结构的指针
    void *ptr;    
    // ...
} robj;
```

### 类型

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/9d21b224c65ee50e8dc691f60092b255?thumb=384x&scale=auto)

对于Redis数据库保存的键值对来说，键总是一个字符串对象，而值则可以是字符串对象、列表对象、哈希对象、集合对象或者有序集合对象的其中一种，因此：
- 当我们称呼一个数据库键为“字符串键”时，我们指的是“这个数据库键所对应的值为字符串对象”；
- 当我们称呼一个键为“列表键”时，我们指的是“这个数据库键所对应的值为列表对象”。

*TYPE 命令返回的结果为数据库键对应的值对象的类型*

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/83b00a897d896e9dece9021885952dd8?thumb=1024x&scale=auto)

### 编码和底层实现

对象的ptr指针指向对象的底层实现数据结构，而这些数据结构由对象的encoding属性决定。

**对象的编码**

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/536004d4c7984c3bb7ff94717335b3dc?thumb=1024x&scale=auto)

**不同类型和编码的对象**

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/ae68d30b8b8322be8e4146392e76b744?thumb=1024x&scale=auto)

*使用 OBJECT ENCODING 命令可以查看一个数据库键的值对象的编码` OBJECT ENCODING msg`*

![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/3515df35436a2c623f1488bfce7856e5?thumb=1024x&scale=auto)![img](https://cdn.cnbj1.fds.api.mi-img.com/book/images/73dcac8a0dc9ff0a1d8420b2271c0b81?thumb=1024x&scale=auto)













# 单机数据库的实现

## 数据库





## RDB 持久化

## AOF 持久化

## 事件

## 客户端

## 服务器

# 多级数据库的实现

## 复制

## Sentinel（哨兵）模式

## 集群

# 独立功能

## 发布与订阅

## 事务

## Lua 脚本

## 排序

## 二进制位数组

## 慢查询日志

## 监视器







