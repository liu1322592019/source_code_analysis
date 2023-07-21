[SDS-Redis字符串](#SDS-Redis字符串)

[list&quicklist-Redis链表](#list&quicklist-Redis链表)

[ziplist&listpack-Redis数组](#ziplist&listpack-Redis数组)

[dict-Redis哈希表](#dict-Redis哈希表)

[intset-Redis整数集合](#intset-Redis整数集合)

## SDS-Redis字符串

首先看一下，SDS这一组数据结构, 5因为没有长度字段，所以实际上只用来表达空字符串。从8到64是实际存储的数据的结构，适用于不同长度的字符串存储。
```C
typedef char *sds; // 操作sds数据结构的句柄, 世纪指向char buf[]这段内存, 我们向前移动一位就可以
                   // 获取到flags，进而判断是哪种具体的数据结构, 然后cast一下，进行实际操作。

/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```
接下来我们通过sds的操作函数来实际感受一下它是怎么被使用的
```C
// init 原来的sds指针, initlen 原始的长度
sds _sdsnewlen(const void *init, size_t initlen, int trymalloc) {
    void *sh; // 申请到的内存存在这里
    sds s;
    char type = sdsReqType(initlen); // 根据长度选择type, 节省内存
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */
    size_t usable;

    assert(initlen + hdrlen + 1 > initlen); /* Catch size_t overflow */

    sh = trymalloc?
        s_trymalloc_usable(hdrlen+initlen+1, &usable) :
        s_malloc_usable(hdrlen+initlen+1, &usable);
    if (sh == NULL) return NULL;

    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        // init是nullptr, 清空一下
        memset(sh, 0, hdrlen+initlen+1);

    // s指向的时sds结构体的末尾
    // |--------------|-------------------------------|
    //  \____________/|\_______________/\____________/
    //        |       |         |              |
    //     Header    sds*      data          unused
    s = (char*)sh+hdrlen;
    fp = ((unsigned char*)s)-1;
    usable = usable-hdrlen-1;
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);
    
    // 初始化数据
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        ...
    }
    if (initlen && init)
        memcpy(s, init, initlen); // 拷贝原来的数据到新的结构体
    s[initlen] = '\0';
    return s;
}
```
sds扩容函数
```C
sds _sdsMakeRoomFor(sds s, size_t addlen, int greedy) {
    void *sh, *newsh;
    size_t avail = sdsavail(s);
    size_t len, newlen, reqlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;
    size_t usable;

    /* Return ASAP if there is enough space left. */
    if (avail >= addlen) return s;

    // 长度不够, 重新申请并拷贝
    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);
    reqlen = newlen = (len+addlen);
    assert(newlen > len);   /* Catch size_t overflow */
    if (greedy == 1) {
        // SDS_MAX_PREALLOC = 1MB
        if (newlen < SDS_MAX_PREALLOC)
            newlen *= 2;
        else
            newlen += SDS_MAX_PREALLOC;
    }

    ...
}
```

## list&quicklist-Redis链表

在redis的早些版本, 是直接使用链表的, 链表的实现相对简单, 相信大家看完下面的定义自然就知道了
```C
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```
链表每个节点之间的内存都是不连续的，意味着无法很好利用 CPU 缓存。能很好利用 CPU 缓存的数据结构就是数组，因为数组的内存是连续的，这样就可以充分利用 CPU 缓存来加速访问。

为了解决这个问题, redis使用了全新版本的quicklist。

首先我们看一下quicklist的定义
```C
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *entry; // 指向listpack
    size_t sz;             /* entry size in bytes */

    // 下面这四个byte描述的是listpack的特点，是否压缩和大小等
    unsigned int count : 16;     /* count of items in listpack */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* PLAIN==1 or PACKED==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int dont_compress : 1; /* prevent compression of entry that will be used later */
    unsigned int extra : 9; /* more bits to steal for future usage */
} quicklistNode;

typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all listpacks */
    unsigned long len;          /* number of quicklistNodes */

    // 相对list特殊的
    signed int fill : QL_FILL_BITS;       /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[]; // 书签, 存储指向节点的指针, 几乎不使用或者说只对大节点使用
} quicklist;
```
接下来我们通过quicklist的操作函数来实际感受一下它是怎么被使用的
```C
/* Add new entry to head node of quicklist.
 *
 * Returns 0 if used existing head.
 * Returns 1 if new head created. */
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_head = quicklist->head;

    // 大buffer, 拷贝会比较耗时, 直接新建一个独享的PLAIN的node
    if (unlikely(isLargeElement(sz))) {
        __quicklistInsertPlainNode(quicklist, quicklist->head, value, sz, 0);
        return 1;
    }

    // 还不是允许插在head entry里
    // head entry 不存在 -> 不允许
    // head entry 是PLAIN或者是large element -> 不允许
    // head entry 还有足够空间 -> 不允许
    // 其他 -> 允许
    if (likely(_quicklistNodeAllowInsert(quicklist->head, quicklist->fill, sz))) {
        // 插入到head entry里
        quicklist->head->entry = lpPrepend(quicklist->head->entry, value, sz);
        // 更新entry实际使用的byte
        quicklistNodeUpdateSz(quicklist->head);
    } else {
        // 创建并插入到新entry里
        quicklistNode *node = quicklistCreateNode();
        node->entry = lpPrepend(lpNew(0), value, sz);
        // 更新entry实际使用的byte
        quicklistNodeUpdateSz(node);

        // 把新entry插入到头部
        _quicklistInsertNodeBefore(quicklist, quicklist->head, node);
    }

    // 更新count
    quicklist->count++;
    quicklist->head->count++;
    return (orig_head != quicklist->head);
}
```
一言以蔽之，quicklist就是节点是listpack的双向链表。

## ziplist&listpack-Redis数组

首先我们来看一下ziplist
```C
/*
 * ZIPLIST的结构是:
 *     <zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>
 *
 * <uint32_t zlbytes> ziplist占用的总buffer长度
 * <uint32_t zltail>  尾部的偏移长度
 * <uint16_t zllen>   entry的个数
 * <uint8_t zlend>    标志ziplist的结束，值为0xFF
 */

/*
 * ZIPLIST ENTRIES的结构是
 *   存储字符串时
 *     <prevlen> <encoding> <entry-data>
 *   存数数字时
 *     <prevlen> <encoding>
 *
 * <prevlen>有如下两种编码方式
 *   长度在0到253:
 *     <prevlen from 0 to 253> <encoding> <entry>
 *   长度>253: (0xFE的值是254)
 *     0xFE <4 bytes unsigned little endian prevlen> <encoding> <entry>
 *
 * 因为prevlen表示的时上一个entry的长度, 且prevlen的长度本身可变, 所以可能出现
 * 一个entry变更后, 下一个entry的prevlen变长，甚至导致连锁变更的情况, 这也是为什么
 * ziplist会被废弃的原因。
 * 
 * <encoding>
 *   字符串: 前两个bit表示长度需要几个byte, 后面是实际长度(前两位是00,01,10, 分别用1,2,5个bytes表示长度)
 *   数字:   前两个bit都是1，接下来两个bit表示是哪种数字 前两位是11
 */
```
接下来以插入节点为例，探究一下ziplist的数据组织方式
```C
unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
    size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen, newlen;
    unsigned int prevlensize, prevlen = 0;
    size_t offset;
    int nextdiff = 0;
    unsigned char encoding = 0;
    long long value = 123456789; /* initialized to avoid warning. Using a value
                                    that is easy to see if for some reason
                                    we use it uninitialized. */
    zlentry tail;

    // 找到当前插入点的prevlen
    /* Find out prevlen for the entry that is inserted. */
    if (p[0] != ZIP_END) {
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);
    } else {
        unsigned char *ptail = ZIPLIST_ENTRY_TAIL(zl);
        if (ptail[0] != ZIP_END) {
            // 感觉是怕参数非法, 插入到了ziplist的尾部之后的位置
            prevlen = zipRawEntryLengthSafe(zl, curlen, ptail);
        }
    }

    // 判断是数字还是字符串, 以及该怎么存储(encoding)
    /* See if the entry can be encoded */
    if (zipTryEncoding(s,slen,&value,&encoding)) {
        /* 'encoding' is set to the appropriate integer encoding */
        reqlen = zipIntSize(encoding);
    } else {
        /* 'encoding' is untouched, however zipStoreEntryEncoding will use the
         * string length to figure out how to encode it. */
        reqlen = slen;
    }
    /* We need space for both the length of the previous entry and
     * the length of the payload. */
    reqlen += zipStorePrevEntryLength(NULL,prevlen);
    reqlen += zipStoreEntryEncoding(NULL,encoding,slen);

    /* When the insert position is not equal to the tail, we need to
     * make sure that the next entry can hold this entry's length in
     * its prevlen field. */
    // 看一下 插入的总长度 是否可以用原来的prevlen表示
    // 我们从A处插入一个entry, A后面那个entry B原来的prevlen是否可以不扩容的表示插入的这段长度
    int forcelarge = 0;
    nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
    if (nextdiff == -4 && reqlen < 4) {
        nextdiff = 0;
        forcelarge = 1;
    }

    /* Store offset because a realloc may change the address of zl. */
    offset = p-zl;
    newlen = curlen+reqlen+nextdiff;
    zl = ziplistResize(zl,newlen);
    p = zl+offset;

    /* Apply memory move when necessary and update tail offset. */
    if (p[0] != ZIP_END) {
        /* Subtract one because of the ZIP_END bytes */
        memmove(p+reqlen,p-nextdiff,curlen-offset-1+nextdiff);

        // 更新B的prevlen
        /* Encode this entry's raw length in the next entry. */
        if (forcelarge)
            zipStorePrevEntryLengthLarge(p+reqlen,reqlen);
        else
            zipStorePrevEntryLength(p+reqlen,reqlen);

        /* Update offset for tail */
        ZIPLIST_TAIL_OFFSET(zl) =
            intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+reqlen);

        /* When the tail contains more than one entry, we need to take
         * "nextdiff" in account as well. Otherwise, a change in the
         * size of prevlen doesn't have an effect on the *tail* offset. */
        assert(zipEntrySafe(zl, newlen, p+reqlen, &tail, 1));
        if (p[reqlen+tail.headersize+tail.len] != ZIP_END) {
            ZIPLIST_TAIL_OFFSET(zl) =
                intrev32ifbe(intrev32ifbe(ZIPLIST_TAIL_OFFSET(zl))+nextdiff);
        }
    } else {
        /* This element will be the new tail. */
        ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(p-zl);
    }

    // 这里进行连锁更新的处理, 从p + reqlen 即新的B的开始处进行处理！！！
    /* When nextdiff != 0, the raw length of the next entry has changed, so
     * we need to cascade the update throughout the ziplist */
    if (nextdiff != 0) {
        offset = p-zl;
        zl = __ziplistCascadeUpdate(zl,p+reqlen);
        p = zl+offset;
    }

    /* Write the entry */
    p += zipStorePrevEntryLength(p,prevlen);
    p += zipStoreEntryEncoding(p,encoding,slen);
    if (ZIP_IS_STR(encoding)) {
        memcpy(p,s,slen);
    } else {
        zipSaveInteger(p,value,encoding);
    }
    ZIPLIST_INCR_LENGTH(zl,1);
    return zl;
}
```

现在我们已经大概了解了ziplist和它的小缺点，接下来我们看看它的继任者listpack，以及看看listpack是怎么解决ziplist的问题的。

```C
/*
 * LISTPACK的结构是:
 *
 *     <lzbytes> <lzlen> <entry> <entry> ... <entry> <lzend>
 *
 *   <uint32_t lzbytes> listpack占用的总buffer长度
 *   <uint16_t lzlen>   entry的个数
 *   <uint8_t lzend>    标志ziplist的结束，值为0xFF
 *
 *   比起ziplist, 这里少了tail的偏移量，使用的时候是用lzbytes来便宜到末尾的，然后反推最后一项
 */

/*
 * LISTPACK ENTRIES的结构是
 *
 *       <encoding> <data> <len>
 *
 *   相对于ziplist, 看起来像是简单的把每个entry的prevlen给了前一个entry
 *   问题来了，len放在末尾的话, 它占用几个byte呢？是固定占用N个byte吗？
 *       这样是不是就不够省内存，不够redis了？
 *   还有一个不同是，ziplist的prevlen是包括上一个entry的所有长度的,
 *       listpack的len只包含本entry的encoding和data的长度, why?
 *   
 *   这两个问题相信你看完下面的长度计算代码就清楚了.
 */
```
接下来我们看看listpack是怎么编码len的
```C
/* Decode the backlen and returns it. If the encoding looks invalid (more than
 * 5 bytes are used), UINT64_MAX is returned to report the problem. */
static inline uint64_t lpDecodeBacklen(unsigned char *p) {
    uint64_t val = 0;
    uint64_t shift = 0;
    do {
        // 127 = Ob0111,1111, 128 = 0b1000,0000
        // 每个byte的后7位存储的实际数字, 第一位是1表示这个没有结束，第一位是0表示结束
        // 以这个数字为例 0b 0111,1111, 1001,1010
        //                  |--------  |--------
        //                  C    D     A    B
        // 一开始指针指向A处, val值是后七个bit(B处), 即0b 001,1010, 第一个bit(A)是1表示长度未结束
        // 第二轮指针指向C处, val值新增七个bit(D处), 将D偏移7个bit后与val取按位或(|)
        //     得到val的新值 0b 111,1111, 001,1010, 第一个bit(C)是0表示长度结束
        //                     --------  --------
        //                         D         B
        val |= (uint64_t)(p[0] & 127) << shift;
        if (!(p[0] & 128)) break;
        shift += 7;
        p--;
        if (shift > 28) return UINT64_MAX;
    } while(1);
    return val;
}
```
看完这里感觉和protobuf的[Varint](protobuf.md#Pb字段的编解码)编解码很像

## dict-Redis哈希表

本章我们来看一下redis下差不多是最复杂的数据结构, 照例, 先看定义

```C
// dict的配置项, 存储一些操作函数
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(dict *d, const void *key);
    void *(*valDup)(dict *d, const void *obj);
    int (*keyCompare)(dict *d, const void *key1, const void *key2);
    void (*keyDestructor)(dict *d, void *key);
    void (*valDestructor)(dict *d, void *obj);
    int (*expandAllowed)(size_t moreMem, double usedRatio);

    unsigned int no_value:1; // 表示是否是set
    
    unsigned int keys_are_odd:1; // 当时set的时候的一种优化使用

    /* Allow each dict and dictEntry to carry extra caller-defined metadata. The
     * extra memory is initialized to 0 when allocated. */
    size_t (*dictEntryMetadataBytes)(dict *d);
    size_t (*dictMetadataBytes)(void);
    /* Optional callback called after an entry has been reallocated (due to
     * active defrag). Only called if the entry has metadata. */
    void (*afterReplaceEntry)(dict *d, dictEntry *entry);
} dictType;

// dictEntry的元素，或者说桶object
struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;     /* Next entry in the same hash bucket. */
    void *metadata[];           /* An arbitrary number of bytes (starting at a
                                 * pointer-aligned address) of size as returned
                                 * by dictType's dictEntryMetadataBytes(). */
};

struct dict {
    dictType *type;

    // 内部有两个实际的哈希表, 用于扩容等操作使用
    dictEntry** ht_table[2];
    unsigned long ht_used[2];

    long rehashidx; /* -1的时候没有rehash，其他时候记录rehash的进度 */

    /* Keep small vars at end for optimal (minimal) struct padding */
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
    signed char ht_size_exp[2]; /* exponent of size. (size = 1<<exp) */

    void *metadata[];           /* An arbitrary number of bytes (starting at a
                                 * pointer-aligned address) of size as defined
                                 * by dictType's dictEntryBytes. */
};

```
接下来我们看看dict的基本操作

- 插入和查找的函数
```C
/* Finds and returns the position within the dict where the provided key should
 * be inserted using dictInsertAtPosition if the key does not already exist in
 * the dict. If the key exists in the dict, NULL is returned and the optional
 * 'existing' entry pointer is populated, if provided. */
void *dictFindPositionForInsert(dict *d, const void *key, dictEntry **existing) {
    unsigned long idx, table;
    dictEntry *he;
    uint64_t hash = dictHashKey(d, key);
    if (existing) *existing = NULL;
    // 如果正在rehash, 执行一步, 这里实现了rehash的分步操作，避免单次操作太耗时
    if (dictIsRehashing(d)) _dictRehashStep(d);

    /* Expand the hash table if needed */
    // 判断是否需要扩容rehash
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        return NULL;

    for (table = 0; table <= 1; table++) {
        idx = hash & DICTHT_SIZE_MASK(d->ht_size_exp[table]);
        /* Search if this slot does not already contain the given key */
        he = d->ht_table[table][idx];
        while(he) {
            // 遍历链表寻找
            void *he_key = dictGetKey(he);
            if (key == he_key || dictCompareKeys(d, key, he_key)) {
                if (existing) *existing = he;
                return NULL;
            }
            he = dictGetNext(he);
        }
        // 如果没在rehash, 就不用看第二个表了, 减少一次操作
        if (!dictIsRehashing(d)) break;
    }

    /* If we are in the process of rehashing the hash table, the bucket is
     * always returned in the context of the second (new) hash table. */
    // 到这里就是没找到, 生成一个新的bucket返回
    dictEntry **bucket = &d->ht_table[dictIsRehashing(d) ? 1 : 0][idx];
    return bucket;
}
```
在这里我们看到了
```C
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    unsigned long s0 = DICTHT_SIZE(d->ht_size_exp[0]);
    unsigned long s1 = DICTHT_SIZE(d->ht_size_exp[1]);
    if (dict_can_resize == DICT_RESIZE_FORBID || !dictIsRehashing(d)) return 0;
    if (dict_can_resize == DICT_RESIZE_AVOID && 
        ((s1 > s0 && s1 / s0 < dict_force_resize_ratio) ||
         (s1 < s0 && s0 / s1 < dict_force_resize_ratio)))
    {
        return 0;
    }

    while(n-- && d->ht_used[0] != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(DICTHT_SIZE(d->ht_size_exp[0]) > (unsigned long)d->rehashidx);
        
        // 遍历, 找到接下来第一个非空的entry => de
        while(d->ht_table[0][d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht_table[0][d->rehashidx];

        /* Move all the keys in this bucket from the old to the new hash HT */
        // 把这个bucket及链表后面的都挪到新table
        while(de) {
            uint64_t h;

            nextde = dictGetNext(de);
            void *key = dictGetKey(de);
            /* Get the index in the new hash table */
            if (d->ht_size_exp[1] > d->ht_size_exp[0]) {
                // 扩容的话需要重新算hash值
                h = dictHashKey(d, key) & DICTHT_SIZE_MASK(d->ht_size_exp[1]);
            } else {
                /* We're shrinking the table. The tables sizes are powers of
                 * two, so we simply mask the bucket index in the larger table
                 * to get the bucket index in the smaller table. */
                // 因为表的大小都是2的指数, 缩容的情况不需要重新计算
                h = d->rehashidx & DICTHT_SIZE_MASK(d->ht_size_exp[1]);
            }

            if (d->type->no_value) {
                // 一些很redis的操作, 针对key指针可以有一些优化
                // keys_are_odd保证 key是奇数 => 最后一个bit一定是1
                // 而指针一定是奇数, 最后一个bit一定是0， 因为字节对齐吗 TODO: ??????
                // 这里针对这个操作做了一个优化
                if (d->type->keys_are_odd && !d->ht_table[1][h]) {
                    /* Destination bucket is empty and we can store the key
                     * directly without an allocated entry. Free the old entry
                     * if it's an allocated entry.
                     *
                     * TODO: Add a flag 'keys_are_even' and if set, we can use
                     * this optimization for these dicts too. We can set the LSB
                     * bit when stored as a dict entry and clear it again when
                     * we need the key back. */
                    assert(entryIsKey(key));
                    if (!entryIsKey(de)) zfree(decodeMaskedPtr(de));
                    de = key;
                } else if (entryIsKey(de)) {
                    /* We don't have an allocated entry but we need one. */
                    de = createEntryNoValue(key, d->ht_table[1][h]);
                } else {
                    // 插到新表的entry的前面
                    /* Just move the existing entry to the destination table and
                     * update the 'next' field. */
                    assert(entryIsNoValue(de));
                    dictSetNext(de, d->ht_table[1][h]);
                }
            } else {
                // 插到新表的entry的前面
                dictSetNext(de, d->ht_table[1][h]);
            }

            // 把这个entry挪到新的table, 并更新相关统计量. 然后移动到链表的下一个entry
            d->ht_table[1][h] = de;
            d->ht_used[0]--;
            d->ht_used[1]++;
            de = nextde;
        }
        d->ht_table[0][d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    // 判断是否rehash完成, 完成的话切换表
    if (d->ht_used[0] == 0) {
        zfree(d->ht_table[0]);
        /* Copy the new ht onto the old one */
        d->ht_table[0] = d->ht_table[1];
        d->ht_used[0] = d->ht_used[1];
        d->ht_size_exp[0] = d->ht_size_exp[1];
        _dictReset(d, 1);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```
接下来我们看看redis的另一个核心逻辑, iter迭代器
```C
dictEntry *dictNext(dictIterator *iter)
{
    while (1) {
        if (iter->entry == NULL) {
            // 首次迭代
            if (iter->index == -1 && iter->table == 0) {
                if (iter->safe)
                    dictPauseRehashing(iter->d);
                else
                    iter->fingerprint = dictFingerprint(iter->d);
            }
            iter->index++;
            if (iter->index >= (long) DICTHT_SIZE(iter->d->ht_size_exp[iter->table])) {
                // index超过表的size
                if (dictIsRehashing(iter->d) && iter->table == 0) {
                    // 如果正在rehash, 且我们在ht_0, 这个时候可能是扩容, 切换到ht_1去试试
                    iter->table++;
                    iter->index = 0;
                } else {
                    // 没有在rehash, 表明已经迭代完了!
                    break;
                }
            }
            // 跳转到当前表的index++ || ht_0到ht_1的新开头
            iter->entry = iter->d->ht_table[iter->table][iter->index];
        } else {
            // 非首次迭代
            iter->entry = iter->nextEntry;
        }

        if (iter->entry) {
            // 找到了非空的entry, 设置下一个
            /* We need to save the 'next' here, the iterator user
             * may delete the entry we are returning. */
            iter->nextEntry = dictGetNext(iter->entry);
            return iter->entry;
        }
        // entry是空的, 继续找
    }

    // 迭代完了!!
    return NULL;
}
```
// TODO: 确认一下之前那个神奇的迭代操作是在哪里的

## intset-Redis整数集合

下一节我们看看简单的





