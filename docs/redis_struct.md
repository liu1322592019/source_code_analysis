[SDS-Redis字符串](#SDS-Redis字符串)

[list&quicklist-Redis链表](#list&quicklist-Redis链表)

[ziplist&listpack-Redis数组](#ziplist&listpack-Redis数组)


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




