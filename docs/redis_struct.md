[SDS-Redis字符串](#SDS-Redis字符串)

[list&quicklist-Redis链表](#list&quicklist-Redis链表)

[ziplist&listpack-Redis数组](#ziplist&listpack-Redis数组)

[dict-Redis哈希表](#dict-Redis哈希表)



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





