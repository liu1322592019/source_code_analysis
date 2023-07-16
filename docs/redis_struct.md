[SDS-Redis字符串](#SDS-Redis字符串)

[list-Redis链表](#list-Redis链表)



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

## list-Redis链表




