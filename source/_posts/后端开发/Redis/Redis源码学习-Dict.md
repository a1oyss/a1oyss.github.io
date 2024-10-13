---
title: Redis源码学习-Dict
date: 2024-10-13 20:25
updated: 2024-10-13 20:25
tags:
  - Redis
  - 源码分析
---
> 文章以redis 5.0版本的源码作为基础，高版本可能有所不同

redis中的dict其实与java中的map实现大差不差，都是基于哈希表实现，使用拉链法解决哈希冲突的问题，在达到装载因子预订值时自动扩展内存（重哈希）。

## dict的结构

dict总体结构图：

![](https://cdn.nlark.com/yuque/0/2024/png/2658344/1725503005537-eb2957c3-d09e-4452-b8e6-4b8ee313f742.png)

```
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```

- `dictType *type`：定义了一些内部会使用到的方法，例如hash函数，key比较函数，其实就类似于java的接口。
- `void *privdata`：保存了需要传给特定键值对函数的可选参数
- `dictht ht[2]`：hash表，之所以有两个hash表，主要是用来进行rehash。在没有rehash的时候，ht[1]大小为0；在rehash的时候，为ht[1]分配空间，将ht[0]中的entry迁移至ht[1]。
- `long rehashidx`：记录rehash过程中ht[0]进行到了哪一个bucket，没有rehash的情况下固定为-1
- `unsigned long iterators`：外界当前迭代该Dict迭代器的个数

### dictType

```
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

- `hashFunction`：hash函数
- `keyDup`：key拷贝函数
- `valDup`：val拷贝函数
- `keyCompare`：key比较函数
- `keyDestructor`：销毁key
- `valDestructor`：销毁value

### dictht

```
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
```

- `dictEntry **table`：hash表，存放一个数组的地址，数组存放着哈希表节点dictEntry的地址
- `unsigned long size`：hash表的大小
- `unsigned long sizemask`：用于将hash值映射到table的位置索引，总是等于（size-1）
- `unsigned long used`：记录entry的数量

### dictEntry

```
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```

entry的结构就比较简单了，相当于一个链表。

- `void *key`：key
- `union {} v`：保存value，可以看到不只有一个`void *val`，这是为了在不同情况下用不同的类型来保存数据，更好的利用内存
- `struct dictEntry *next`：next指针

## dict的创建过程

```
/* Create a new hash table */
dict *dictCreate(dictType *type,
        void *privDataPtr)
{
    dict *d = zmalloc(sizeof(*d));

    _dictInit(d,type,privDataPtr);
    return d;
}

/* Initialize the hash table */
int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    d->type = type;
    d->privdata = privDataPtr;
    d->rehashidx = -1;
    d->iterators = 0;
    return DICT_OK;
}

/* Reset a hash table already initialized with ht_init().
 * NOTE: This function should only be called by ht_destroy(). */
static void _dictReset(dictht *ht)
{
    ht->table = NULL;
    ht->size = 0;
    ht->sizemask = 0;
    ht->used = 0;
}
```

主要就是进行分配内存，然后设置初始化值。

## dict的rehash

和Java的HashMap一样，随着数量的增加，也会产生问题：随着数据越来越多，冲突的次数也就越来越多，单链表的长度就会过长，此时的查询效率就变得很差。

Java中的解决方式为：

1. 链表长度过长时，先检查总桶数是否大于64，如果小于，先扩容，如果大于，转成红黑树
2. 当使用的桶数 >= 总桶数 * 负载因子时，进行扩容

其中负载因子在Java中固定设置为了0.75

redis虽然也使用了负载因子，但是稍有不同。redis中的负载因子计算方式为dictht.used/dictht.size，即使用的entry数量/hashTable长度。

判断是否需要扩容的代码：

```
/* Expand the hash table if needed */
static int _dictExpandIfNeeded(dict *d)
{
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK;

    /* If the hash table is empty expand it to the initial size. */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}
```

1. 如果ht[0]所指向的HashTable大小为0则扩容
2. HashTable负载因子大于等于1，即ht.used>=ht.size
3. HashTable负载因子大于5，即ht.used/ht.size>dict_force_resize_ratio，这里的dict_force_resize_ratio是固定值5

## 参考

1. [redis-dict (axlgrep.github.io)](https://axlgrep.github.io/tech/redis-dict.html)
2. [字典 — Redis 设计与实现 (redisbook.readthedocs.io)](https://redisbook.readthedocs.io/en/latest/internal-datastruct/dict.html)
3. [Redis内部数据结构详解(1)——dict - 铁蕾的个人博客 (zhangtielei.com)](http://zhangtielei.com/posts/blog-redis-dict.html)
4. [redis/src/dict.h at 5.0 · redis/redis (github.com)](https://github.com/redis/redis/blob/5.0/src/dict.h)