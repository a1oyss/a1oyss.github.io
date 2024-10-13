---
title: Redis源码学习-Ziplist
date: 2024-10-13 20:21
updated: 2024-10-13 20:21
tags:
  - Redis
  - 源码分析
---
## 描述：
redis官方描述([ziplist.c](https://github.com/redis/redis/blob/3fcddfb61f903d7112da186cba8b1c93a99dc87f/src/ziplist.c#L1)):

ziplist 是一种特殊编码的双链表，设计上非常节省内存。它既能存储字符串，也能存储整数值，其中整数被编码为实际整数，而不是一系列字符。它允许以 O(1) 的时间在列表的两边进行推送和弹出操作。不过，由于每次操作都需要重新分配 ziplist 使用的内存，因此实际复杂度与 ziplist 使用的内存量有关。

## Ziplist的结构

ziplist 的一般布局如下：

```
<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>
```

```
|<---- ziplist header ---->|<----------- entries ------------->|<-end->|
  4 bytes  4 bytes  2 bytes    ?        ?        ?        ?      1 byte
+---------+--------+-------+--------+--------+--------+--------+-------+
| zlbytes | zltail | zllen | entry1 | entry2 |  ...   | entryN | zlend |
+---------+--------+-------+--------+--------+--------+--------+-------+
                           ^                          ^        ^
                           |                          |        |
                 ZIPLIST_ENTRY_HEAD                   |   ZIPLIST_ENTRY_END
                                                      |
                                             ZIPLIST_ENTRY_TAIL
```

NOTE: 没有特别说明的情况，所有的字段存储为小端序

|   |   |   |
|---|---|---|
|字段|类型|描述|
|zlbytes|uint32_t|无符号整数，用于保存 ziplist 占用的字节数，包括 zlbytes 字段本身的 4 个字节。存储该值是为了调整整个结构的大小，而无需先遍历结构。|
|zltail|uint32_t|最后一个entry再ziplist中的偏移量。这样就可以在列表的尾端进行pop操作，而不需要完全遍历列表。|
|zllen|uint16_t|记录entry的数量，当zllen小于2^16-1时，zllen的值就是entry的数量，当zllen等于2^16-1时，需要便利entry才能得到真实的数量。|
|zlend|uint8_t|代表ziplist的结束，固定是255 (1111 1111)|

## Ziplist Entry的结构

ziplist 中的每个entry都以元数据为前缀，其中包含两条信息：

- prevlen：代表前一个entry的长度，用于从后向前遍历entry
- encoding：代表entry的类型，整数或者字符串，如果是字符串，还代表了字符串的长度

所以大致上entry的结构为：

```
<prevlen> <encoding> <entry-data>
```

有时会省略掉后面的entry-data部分，例如存储小整数的时候。

```
<prevlen> <encoding>
```

### prevlen

其中，prevlen的表示方式为：

- 如果前一个entry的长度小于254，则占用一个字节（8位无符号整数）表示长度。
- 如果大于等于254，prevlen会占用5个字节，并将第一个字节设置为254（0xFE）作为标记，剩余的4个字节标识长度。

所以实际上entry的结构为：

- 前一个entry的长度小于254时：

```
<prevlen from 0 to 253> <encoding> <entry-data>
```

- 前一个entry的长度大于等于254时

```
0xFE <4 bytes unsigned little endian prevlen> <encoding> <entry-data>
```

### encoding

encoding决定了保存的数据类型以及数据长度，根据前两位的不同可以表示为：

- 00 、 01 和 10 表示 content 部分保存着字符数组。

- |00pppppp|：占用1个字节，长度小于等于63（2^6-1）字节的字符串
- |01pppppp|qqqqqqqq|：占用2个字节，长度小于等于16383（2^14-1）字节的字符串
- |10000000|qqqqqqqq|rrrrrrrr|ssssssss|tttttttt|：占用5个字节，其中第1个字节只用到了前两位，后6位没有用到，剩余4个字节代表长度，最大可表示4,294,967,296（2^32-1）字节的字符串

- 11 表示 content 部分保存着整数。

- |11000000|：占用1个字节，代表int16_t类型的整数，entry-data存储2个字节的数据
- |11010000|：占用1个字节，代表int32_t类型的整数，entry-data存储4个字节的数据
- |11100000|：占用1个字节，代表int64_t类型的整数，entry-data存储8个字节的数据
- |11110000|：占用1个字节，代表24位有符号类型的整数，entry-data存储3个字节的数据
- |11111110|：占用1个字节，代表8位有符号类型的整数，entry-data存储3个字节的数据
- |1111xxxx|：xxxx范围再0001到1101之间，存储4位无符号整数0-12，此时存储的是真实的数据而不是长度，后面也不再需要entry-data。实际上0001和1101分别是1和13，但是由于前面已经使用了0000，所以xxxx的值减一才是实际表示的值。
- |11111111|：ziplist的结束标志

## Ziplist实际范例

下面是一个包含两个元素："2"，"5"的ziplist：

```
[0f 00 00 00] [0c 00 00 00] [02 00] [00 f3] [02 f6] [ff]
      |             |          |       |       |     |
   zlbytes        zltail     zllen    "2"     "5"   end
```

- [0f 00 00 00] 头四个字节为zlbytes，说明该ziplist总共15个字节长度
- [0c 00 00 00] 接下来四个字节zltail，说明12个字节处为ziplist entry的结尾
- [02 00] 说明ziplist有2个entry
- [00 f3] 为entry，第一个字节00表示前一个entry的长度，这里没有前一个entry，所以是00。f3转为2进制为1111 0011，说明这是一个整数，满足|1111xxxx|的情况，实际保存的值为0011 - 1 = 0010（2）。
- [02 f6] 也是entry，第一个字节代表前一个entry的长度，这里前一个entry的长度为2个字节，所以是02，f6就不过多赘述，类似上面的分析。
- [ff] 代表ziplist的结束

接下来看一个字符串的：

```
[02] [0b] [48 65 6c 6c 6f 20 57 6f 72 6c 64]
```

- 第一个字节[02]，表示前一个entry的长度
- [0b] 转为2进制：0000 1011，符合|00pppppp|的情况，1011为字符串的长度，后面的[48 65 6c 6c 6f 20 57 6f 72 6c 64]就是字符串数据

## 参考

1. [Redis内部数据结构详解(4)——ziplist - 铁蕾的个人博客 (zhangtielei.com)](http://zhangtielei.com/posts/blog-redis-ziplist.html)
2. [redis/src/ziplist.h at 5.0 · redis/redis (github.com)](https://github.com/redis/redis/blob/5.0/src/ziplist.h)
