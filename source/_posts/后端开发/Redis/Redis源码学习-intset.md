---
title: Redis源码学习-intset
date: 2024-10-13 20:27
updated: 2024-10-13 20:27
tags:
  - Redis
  - 源码分析
---
顾名思义，整数集合，Set的底层数据结构实现之一，另外一种实现是hashtable。

实际上intset是一个有序数组，通过二分查找快速判断一个元素是否属于这个集合，但是为了保证有序性，插入的时候也会先二分查找找到对应的插入位置，将插入位置之后的元素统一向后移动一个位置，redis使用了该结构来降低内存使用率，时间换空间。

## intset的结构

```
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

- encoding：数据编码，表示intset中的每个数据元素用几个字节存储，有三种可能的取值:

- INTSET_ENC_INT16：16位整数，表示每个元素用2个字节存储
- INTSET_ENC_INT32：32位整数，表示每个元素用4个字节存储
- INTSET_ENC_INT64：64位整数，表示每个元素用8个字节存储

- length：表示intset中的元素数量
- contents：柔性数组，存储具体的数据，长度=encoding * length

随着数据的增加，encoding可能会改变：

![](https://cdn.nlark.com/yuque/0/2024/png/2658344/1726023017324-4d05f718-1d43-4b52-8b92-217b9b550661.png)

如上图，新创建的intset只有encoding和length，encoding默认值是INTSET_ENC_INT16，添加13和5之后，length变成02。但是继续添加32768，10，100000时，encoding就从INTSET_ENC_INT16变为了INTSET_ENC_INT32，这是因为INTSET_ENC_INT16的最大长度为-215~215-1，32768超出了返回，所以升级到INTSET_ENC_INT32。

## intset的查找与新增

### 查找元素

```
/* Determine whether a value belongs to this set */
uint8_t intsetFind(intset *is, int64_t value) {
    uint8_t valenc = _intsetValueEncoding(value);
    return valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,NULL);
}

/* Search for the position of "value". Return 1 when the value was found and
 * sets "pos" to the position of the value within the intset. Return 0 when
 * the value is not present in the intset and sets "pos" to the position
 * where "value" can be inserted. */
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    /* The value can never be found when the set is empty */
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
        /* Check for the case where we know we cannot find the value,
         * but do know the insert position. */
        if (value > _intsetGet(is,max)) {
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }

    while(max >= min) {
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else {
        if (pos) *pos = min;
        return 0;
    }
}
```

intsetFind在指定的intset中查找value，找到返回1，否则返回0。

首先使用_intsetValueEncoding判断value的encoding，如果要搜索的value的encoding小于当前的encoding，那元素肯定不存在。

encoding符合当前intset，则调用intsetSearch，该方法是二分查找算法的实现，查到元素时，返回元素位置，**没查到元素时，会将pos设置为min，这是为了之后的插入做准备。**

### 插入元素

```
/* Insert an integer in the intset */
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;

    /* Upgrade encoding if necessary. If we need to upgrade, we know that
     * this value should be either appended (if > 0) or prepended (if < 0),
     * because it lies outside the range of existing values. */
    if (valenc > intrev32ifbe(is->encoding)) {
        /* This always succeeds, so we don't need to curry *success. */
        return intsetUpgradeAndAdd(is,value);
    } else {
        /* Abort if the value is already present in the set.
         * This call will populate "pos" with the right position to insert
         * the value when it cannot be found. */
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }

        is = intsetResize(is,intrev32ifbe(is->length)+1);
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    _intsetSet(is,pos,value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}


/* Upgrades the intset to a larger encoding and inserts the given integer. */
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    uint8_t curenc = intrev32ifbe(is->encoding);
    uint8_t newenc = _intsetValueEncoding(value);
    int length = intrev32ifbe(is->length);
    int prepend = value < 0 ? 1 : 0;

    /* First set new encoding and resize */
    is->encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    /* Upgrade back-to-front so we don't overwrite values.
     * Note that the "prepend" variable is used to make sure we have an empty
     * space at either the beginning or the end of the intset. */
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* Set the value at the beginning or the end. */
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}

/* Resize the intset */
static intset *intsetResize(intset *is, uint32_t len) {
    uint64_t size = (uint64_t)len*intrev32ifbe(is->encoding);
    assert(size <= SIZE_MAX - sizeof(intset));
    is = zrealloc(is,sizeof(intset)+size);
    return is;
}
```

依旧是先判断插入的value的encoding，如果大于当前intset的encoding，则调用intsetUpgradeAndAdd对encoding进行升级。

如果不需要扩容，就调用intsetSearch查找是否已经存在，如果能查到，就直接返回，不添加value。

如果没查到，调用intsetResize扩容，同时调用intsetMoveTail将待插入位置后面的元素统一向后移动1个位置。

显然插入元素的时间复杂度为O(n)。