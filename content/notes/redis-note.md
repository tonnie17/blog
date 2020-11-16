+++
title = "Redis笔记"
date = 2019-12-19T00:00:00+08:00
tags = ["go"]
categories = [""]
draft = false
+++

Redis是一种基于键值对（key-value）的 NoSQL 数据库，它支持存储多种类型数据结构，因为Redis会把所有的数据都存放在内存中，所以它的读写性能非常的高。

## Redis特性

- IO多路复用下的单线程模型（Redis 6.0后支持多线程）
- 所有数据都存放于内存
- 支持Lua脚本功能
- 支持键过期，订阅发布，简单事务，流水线功能
- 支持持久化，主从复制功能

## 应用场景

- 缓存
- 实时排行榜
- 计数器应用
- 社交关系应用
- 消息队列

## 读流程

- 主线程负责接收建连请求，读事件到来(收到请求)则放到一个全局等待读处理队列
- 主线程处理完读事件之后，通过 RR将这些连接分配给这些 IO 线程，然后主线程忙等待(spinlock 的效果)状态
- IO 线程将请求数据读取并解析完成(这里只是读数据和解析并不执行)
- 主线程执行所有命令并清空整个请求等待读处理队列(执行部分串行)

## 数据结构实现

### 字符串

#### SDS

Redis使用SDS（simple dynamic string，简单动态字符串）作为默认的字符串表示。

SDS 具有以下优点：

1. 常数复杂度获取字符串长度。
2. 杜绝缓冲区溢出。
3. 减少修改字符串长度时所需的内存重分配次数。

#### SDS定义

```c
struct sdshdr {

    // 记录 buf 数组中已使用字节的数量
    // 等于 SDS 所保存字符串的长度
    int len;

    // 记录 buf 数组中未使用字节的数量
    int free;

    // 字节数组，用于保存字符串
    char buf[];

};
```

#### SDS示例

初始化一个字符串”Redis“，SDS的结构中记录了几个信息：

- free：代表SDS中空闲的字节数
- len：代表字符串的长度，进行strlen获取字符串长度时可以直接读取这里的值
- buf：存放字符串的字节数组

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5aec62f0-04ee-41e7-8607-a765d6e58ec7/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/sds-1.png)

**空间预分配**

当对SDS进行扩展（比如拼接一个字符串）时，SDS会检查自身空间是否满足需要，否则会扩充自身空间，再进行拼接的操作，同时SDS在扩充空间时还会预分配多余的空间（free）来满足未来需要。

![http://redisbook.com/_images/graphviz-a52da469a2a921623086793193a2d35eb1fed716.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/sds-2.png)

预分配规则：

- 如果对 SDS 进行修改之后， SDS 的长度（也即是 `len` 属性的值）将小于 `1 MB` ， 那么程序分配和 `len` 属性同样大小的未使用空间， 这时 SDS `len` 属性的值将和 `free` 属性的值相同。 举个例子， 如果进行修改之后， SDS 的 `len` 将变成 `13` 字节， 那么程序也会分配 `13` 字节的未使用空间， SDS 的 `buf` 数组的实际长度将变成 `13 + 13 + 1 = 27` 字节（额外的一字节用于保存空字符）。
- 如果对 SDS 进行修改之后， SDS 的长度将大于等于 `1 MB` ， 那么程序会分配 `1 MB` 的未使用空间。 举个例子， 如果进行修改之后， SDS 的 `len` 将变成 `30 MB` ， 那么程序会分配 `1 MB` 的未使用空间， SDS 的 `buf` 数组的实际长度将为 `30 MB + 1 MB + 1 byte` 。

**惰性空间释放**

在SDS进行缩短字符串操作时，Redis不会马上把空闲的内存回收，而是将这些字节用free记录起来，等待未来使用。

![http://redisbook.com/_images/graphviz-e0b39c48a2c522f5f7802f1e325b5cb25ac92579.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/sds-3.png)

![http://redisbook.com/_images/graphviz-c58adbc4441b5622084daeee71c0cb306db28741.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/sds-4.png)

当字符串未来需要进行拼接操作时，不需要去申请新的空间，而是复用记录在free中的空间：

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/06e832ec-cc70-44ec-816a-b99192b388a4/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/sds-5.png)

### 链表

链表被广泛用于实现 Redis 的各种功能， 比如列表键， 发布与订阅， 慢查询， 监视器， 等等。

Redis链表特点：

- 双端链表，链表持有头指针和尾指针
- 链表记录自身长度
- 无环链表，首节点的prev和尾节点的next都指向NULL

#### 链表实现

listNode：

```c
typedef struct listNode {

    // 前置节点
    struct listNode *prev;

    // 后置节点
    struct listNode *next;

    // 节点的值
    void *value;

} listNode;
```

list：

```c
typedef struct list {

    // 表头节点
    listNode *head;

    // 表尾节点
    listNode *tail;

    // 链表所包含的节点数量
    unsigned long len;

    // 节点值复制函数
    void *(*dup)(void *ptr);

    // 节点值释放函数
    void (*free)(void *ptr);

    // 节点值对比函数
    int (*match)(void *ptr, void *key);

} list;
```

多个listNode和指针组成了双向链表：

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/02812458-5998-4c9b-9826-113097132d5d/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/linklist.png)

List使用场景：

- lpush+lpop=Stack（栈）
- lpush+rpop=Queue（队列）
- lpush+ltrim=Capped Collection（有限集合）
- lpush+brpop=Message Queue（消息队列）

### 字典

```c
typedef struct dictht {

    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;

    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;

} dictht;
```

table属性是一个数组，数组中每个元素都是指向dictEntry的一个指针：

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1358d0e4-4486-4348-acbc-716812a0dbd5/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/dict-1.png)

dictEntry实现：

```c
typedef struct dictEntry {

    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;

} dictEntry;
```

字典实现：

```c
typedef struct dict {

    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表
    dictht ht[2];

    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */

} dict;
```

- `type` 属性是一个指向 `dictType` 结构的指针， 每个 `dictType` 结构保存了一簇用于操作特定类型键值对的函数， Redis 会为用途不同的字典设置不同的类型特定函数。
- 而 `privdata` 属性则保存了需要传给那些类型特定函数的可选参数。

```c
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

`ht` 属性是一个包含两个项的数组， 数组中的每个项都是一个 `dictht` 哈希表， 一般情况下， 字典只使用 `ht[0]` 哈希表， `ht[1]` 哈希表只会在对 `ht[0]` 哈希表进行 rehash 时使用。

除了 ht[1] 之外， 另一个和 rehash 有关的属性就是 rehashidx ： 它记录了 rehash 目前的进度， 如果目前没有在进行 rehash ， 那么它的值为 -1 。

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b47c183b-226b-4673-8d88-1165fa5b86ec/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/dict-2.png)

Redis计算哈希值和索引值的方法如下：

```c
// 使用字典设置的哈希函数，计算键 key 的哈希值
hash = dict->type->hashFunction(key);

// 使用哈希表的 sizemask 属性和哈希值，计算出索引值
// 根据情况不同， ht[x] 可以是 ht[0] 或者 ht[1]
index = hash & dict->ht[x].sizemask;
```

Redis 的哈希表使用链地址法（separate chaining）来解决键冲突： 每个哈希表节点都有一个 next 指针， 多个哈希表节点可以用 next 指针构成一个单向链表：

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/610f3a39-74a6-4478-9f29-26fe36721224/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/dict-3.png)

Redis为了将哈希表的负载因子维持在一个合理的范围内，会对哈希表进行扩展和收缩操作，这些操作通过rehash来完成：

1. 为字典的`ht[1]` 哈希表分配空间：
   - 如果执行的是扩展操作， 那么 `ht[1]` 的大小为第一个大于等于 `ht[0].used * 2` 的 2^n （`2` 的 `n` 次方幂）；
   - 如果执行的是收缩操作， 那么 `ht[1]` 的大小为第一个大于等于 `ht[0].used` 的 2^n 。
2. 将保存在 `ht[0]` 中的所有键值对 rehash 到 `ht[1]` 上面。
3. 当 `ht[0]` 包含的所有键值对都迁移到了 `ht[1]` 之后 （`ht[0]` 变为空表）， 释放 `ht[0]` ， 将 `ht[1]` 设置为 `ht[0]` ， 并在 `ht[1]` 新创建一个空白哈希表， 为下一次 rehash 做准备。

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2e78a98e-5826-439f-9560-54bf1a1f6463/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/dict-4.png)

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f61954b0-52b5-4255-a195-c1a4d0f344c1/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/dict-5.png)

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9c2ce778-df28-4d5c-93a1-a591ccc40175/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/dict-6.png)

#### 渐进式Rehash

以下是哈希表渐进式 rehash 的详细步骤：

1. 为 `ht[1]` 分配空间， 让字典同时持有 `ht[0]` 和 `ht[1]` 两个哈希表。
2. 在字典中维持一个索引计数器变量 `rehashidx` ， 并将它的值设置为 `0` 。
3. 在 rehash 进行期间， 每次对字典执行添加、删除、查找或者更新操作时， 程序除了执行指定的操作以外， 还会顺带将 `ht[0]` 哈希表在 `rehashidx` 索引上的所有键值对 rehash 到 `ht[1]` ， 当 rehash 工作完成之后， 程序将 `rehashidx` 属性的值增一。
4. 最终在某个时间点上， `ht[0]` 的所有键值对都会被 rehash 至 `ht[1]` ， 这时程序将 `rehashidx` 属性的值设为 `-1` 。

因为在进行渐进式 rehash 的过程中， 字典会同时使用 `ht[0]` 和 `ht[1]` 两个哈希表， 所以在渐进式 rehash 进行期间， 字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行。

另外， 在渐进式 rehash 执行期间， 新添加到字典的键值对一律会被保存到 `ht[1]` 里面， 而 `ht[0]` 则不再进行任何添加操作： 这一措施保证了 `ht[0]` 包含的键值对数量会只减不增， 并随着 rehash 操作的执行而最终变成空表。

### 整数集合

整数集合（intset）是 Redis 用于保存整数值的集合抽象数据结构， 它可以保存类型为 int16_t 、 int32_t 或者 int64_t 的整数值， 并且保证集合中不会出现重复元素。

#### 整数集合定义

```c
typedef struct intset {

    // 编码方式
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组
    int8_t contents[];

} intset;
```

整数集合的结构如下：

- encoding：代表contents数组的编码方式，可以是INTSET_ENC_INT8，INTSET_ENC_INT16，INTSET_ENC_INT32，INTSET_ENC_INT16，由数组中最大元素的类型来决定
- length：contents数组的长度
- contents：按由小到大的顺序保存集合元素

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ccf2325d-330f-438a-a50e-ea6f51629998/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/set-1.png)

#### 升级

每当添加到整数集合的一个新元素的类型比整数集合现有所有元素的类型都要长时， 整数集合需要先进行升级， 然后才能将新元素添加到整数集合（集合升级后无法降级）：

1. 根据新元素的类型， 扩展整数集合底层数组的空间大小， 并为新元素分配空间。
2. 将底层数组现有的所有元素都转换成与新元素相同的类型， 并将类型转换后的元素放置到正确的位上， 而且在放置元素的过程中， 需要继续维持底层数组的有序性质不变。
3. 将新元素添加到底层数组里面。

下面要将65535（int32）添加到以下集合：

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9a94fefb-f442-4c18-858e-99c5a4ee84dc/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/set-2.png)

因为每个 int32_t 整数值需要占用 32 位空间， 所以在空间重分配之后， 底层数组的大小将是 32 * 4 = 128 位：

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8450c302-5dec-4019-a9b5-4df1b0f3b0db/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/set-3.png)

将前面三个元素按顺序移动到新的位置上，过程中需要维持底层数组的有序性质不变：

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4ea45887-a8d1-44f9-9053-9d333e6090af/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/set-4.png)

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/45fb6d22-5a42-4b7c-ba20-e3922bb2ed37/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/set-5.png)

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c44972ba-ed37-4400-891c-7a854f66e0b3/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/set-6.png)

最后将新元素移动到新数组上：

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0b6aea1c-7f33-46d3-9d1c-9be33ba812ea/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/set-7.png)

完成操作后的整数集合如下：

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/57fbf04f-744e-46ea-8f77-40f408ac0ca9/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/set-8.png)

特点：

- 按需升级，节约内存
- 支持多种类型，提高灵活性

### 跳跃表

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4ee84c25-5a6d-4520-952d-7f487c7a783a/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/skiplist-1.png)

跳跃表属性：

- header：指向跳跃表的头部
- tail：指向跳跃表的尾部
- level：记录跳跃表内所有节点中最大的层数
- length：跳跃表中节点的数量

节点属性：

- level：层，每个层具有两个属性：前进指针和跨度
- backward：指向节点的前一个节点
- score：记录节点保存的分值
- obj：记录节点保存的对象

#### 节点定义

```c
typedef struct zskiplistNode {

    // 后退指针
    struct zskiplistNode *backward;

    // 分值
    double score;

    // 成员对象
    robj *obj;

    // 层
    struct zskiplistLevel {

        // 前进指针
        struct zskiplistNode *forward;

        // 跨度
        unsigned int span;

    } level[];

} zskiplistNode;
```

#### 层

每次创建一个新跳跃表节点的时候， 程序都根据幂次定律 （power law，越大的数出现的概率越小） 随机生成一个介于 1 和 32 之间的值作为 level 数组的大小， 这个大小就是层的“高度”。

因为节点的层数是随机生成的，所以插入操作只需要修改插入节点前后的指针，而不需要对很多节点都进行调整。

#### 跨度

**查询节点的排名**

在跳跃表中查找分值为 3.0 、 成员对象为 o3 的节点时， 沿途经历的层： 查找的过程只经过了一个层， 并且层的跨度为 3 ， 所以目标节点在跳跃表中的排位为 3 。

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/49eaeb16-eb2c-411f-b0c4-89e4671e9c4c/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/skiplist-2.png)

#### 分值

在跳跃表中节点按分数数值从小到大排序，如果分值相同则用字典序顺序从小到大排序：

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b4ee3de5-24bb-4548-9fba-702a6f82fe2c/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/skiplist-3.png)

#### 跳跃表特点

- 跳跃表更容易做范围查询（找到最小值后在节点第一层开始遍历）
- 插入和删除操作只需要调整前后指针
- 第一层支持双向链表，方便往回遍历

#### 查询时间复杂度

- zscore只用查询一个dict，所以时间复杂度为O(1)
- zrevrank, zrevrange, zrevrangebyscore由于要查询skiplist，所以zrevrank的时间复杂度为O(log n)，而zrevrange, zrevrangebyscore的时间复杂度为O(log(n)+M)，其中M是当前查询返回的元素个数。

### 压缩列表

- zlbytes：代表列表的总长度
- zltail：代表尾结点的偏移量
- zllen：压缩列表中的节点数量

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/956df8b6-7138-4dfd-a598-ee46e7319e54/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/ziplist-1.png)

#### 压缩列表节点结构

- previous_entry_length：记录前一个节点的长度，大小为1字节，如果前一节点的长度大于等于 254 字节，大小为5字节，后面4字节保存节点长度，因为previous_entry_length记录了前面的节点长度，因此可通过这个值用指针进行节点回溯
- encoding：记录字节数组编码或整数编码
- content：保存节点的值

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0f4ece31-e84e-42b2-a45c-804f9c7a3a64/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/ziplist-2.png)

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/b73c8472-a4f1-48b5-b65c-1e96e053a3ce/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/ziplist-3.png)

![http://redisbook.com/_images/graphviz-cfb376f8015f3af9f59acd30dc71a35e90c6d763.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/ziplist-4.png)

**连锁更新：**将一个长度大于等于 254 字节的新节点 new 设置为压缩列表的表头节点，因为 e1 的 previous_entry_length 属性仅长 1 字节， 它没办法保存新节点 new 的长度， 所以程序将对压缩列表执行空间重分配操作， 并将 e1 节点的 previous_entry_length 属性从原来的 1 字节长扩展为 5 字节长。

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/aa9c6dfe-d3b7-4672-95de-53e9d8472c2e/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/ziplist-5.png)

#### 特点

- 压缩列表是一种为节约内存而开发的顺序型数据结构。
- 压缩列表被用作列表键和哈希键的底层实现之一。
- 压缩列表可以包含多个节点，每个节点可以保存一个字节数组或者整数值。
- 添加新节点到压缩列表， 或者从压缩列表中删除节点， 可能会引发连锁更新操作， 但这种操作出现的几率并不高。

#### 满足ziplist条件

- list：所有元素长度小于64字节，元素数量小于512
- hash：所有元素长度小于64字节，元素数量小于512
- set：所有元素都是整数，元素数量小于512
- zset：所有元素长度小于64字节，元素数量小于128

## 持久化

### RDB

将Redis进程的数据快照存储到硬盘

- save：阻塞命令，直至RDB过程完成
- bgsave：redis 进程执行 fork 操作创建子进程，RDB 持久化过程由子进程负责

### AOF

以独立日志的方式记录每次写命令，重启时再重新执行 AOF 文件中的命令

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2ca2bf24-7661-42f0-9872-e01cc3110f5a/Untitled.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/aof.png)

其流程如下：

- 所有的写命令会追加到 AOF 缓冲中
- AOF 缓冲区根据对应的策略向硬盘进行同步操作
- 随着 AOF 文件越来越大，需要定期对 AOF 文件进行重写，达到压缩的目的
- 当 Redis 重启时，可以加载 AOF 文件进行数据恢复

写入模式：

1. `no` ：不保存，效率最高，同步交给操作系统，间隔较长
2. `everysec` ：原则上每一秒钟保存一次，效率较高，只会丢1秒数据
3. `always` ：每执行一个命令保存一次，效率最差，最安全
