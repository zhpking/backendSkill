# redis知识点汇总

前排提示，此redis知识点汇总是基于5.0.14版本的，包括文中的源码分析，源码分析中会对一些c语言的特有语法或者结构进行简要讲解，尽可能让没有c语言基础的同学也能看得明白

## 键和值的数据组织

### 全局哈希表
为了实现从键到值的快速访问，Redis使用了一个哈希表来保存所有键值对。

一个哈希表，其实就是一个数组，数组的每个元素称为一个哈希桶。所以，我们常说，一个哈希表是由多个哈希桶组成的，每个哈希桶中保存了键值对数据。

哈希桶中的元素保存的并不是值本身，而是指向具体值的指针。这也就是说，不管值是String，还是集合类型，哈希桶中的元素都是指向它们的指针。如下图所示：

![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220101142554407-1946202291.jpg)

哈希桶中的entry元素中保存了*key和*value指针，分别指向了实际的键和值，这样一来，即使值是一个集合，也可以通过*value指针被查找到，entry的具体数据结构如下：

	typedef struct dictEntry {
	    void *key;
	    void *val;
	    struct dictEntry *next;
	} dictEntry;

因为hash查找的时间复杂度是O(1)，所以无论全局哈希表中有多少个键都不会影响到查找效率，我们只需要计算键的哈希值，就可以知道它所对应的哈希桶位置，然后就可以访问相应的entry元素。

### hash冲突

hash冲突指的是根据key值所计算出来的hash值相同，导致两个元素落到同一个hash桶里。

redis解决哈希冲突的方式，就是用链式法。链式法也很容易理解，就是指同一个哈希桶中的多个元素用一个链表来保存，它们之间依次用指针连接。如下图所示：

![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220101160047355-1184225591.jpg)

entry1、entry2和entry3都需要保存在哈希桶3中，导致了哈希冲突。此时，entry1元素会通过一个*next指针指向entry2，同样，entry2也会通过*next指针指向entry3。这样一来，即使哈希桶3中的元素有100个，我们也可以通过entry元素中的指针，把它们连起来。这就形成了一个链表，也叫作哈希冲突链。

###rehash

同一个哈希桶中，元素越多，查询效率越慢（链表查询时间复杂度为O(n)）,为了解决这个问题，采用的方法就是增加哈希桶的数量，而这个就是rehash操作。

redis默认使用两个全局hash表（下文称作hash表1和hash表2），一个作为插入和查询数据使用，另一个就是用来在rehash过程中迁移数据使用，具体步骤如下：

1. hash表2扩充哈希桶（比如hash表2扩充成hash表2的两倍）

2. 重新新建hash表1元素在hash表2的hash值，然后迁移到hash表2

3. 切换hash表，hash表2作为数据的查询和插入，释放hash表1的空间并作为下次rehash操作的数据迁移表

###渐进式rehash

rehash第2步的过程中，会涉及到大量的数据拷贝，而这个过程中会造成redis的线程阻塞，无法处理其他请求，而渐进式rehash就是优化第2步中数据拷贝的过程。如下图所示：
![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220101165225448-77629838.jpg)

在第二步拷贝数据时，Redis仍然正常处理客户端请求，每处理一个请求时，从hash表1中的第一个索引位置开始，顺带着将这个索引位置上的所有元素拷贝到hash表2中；等处理下一个请求时，再顺带拷贝hash表1中的下一个索引位置的元素。

## redis内存分配

Redis使用的内存分配库是jemalloc，jemalloc在分配内存时，会根据我们申请的字节数N，找一个比N大，但是最接近N的2的幂次数作为分配的空间，这样可以减少频繁分配的次数。如申请6字节空间，jemalloc实际会分配8字节空间；如果你申请24字节空间，jemalloc则会分配32字节

## key过期策略

key的过期策略能分为3种类型

### 惰性删除

- 访问key的时候，如果过期才删除

### 定时删除

- Redis配置项hz定义了serverCron任务的执行周期，默认每次清理时间为25ms，每次清理会依次遍历所有DB，从db随机取出20个key，如果过期就删除，如果其中有5个key过期，那么就继续对这个db进行清理，否则开始清理下一个db

### 内存不够时删除（3种情况）

#### 不删除，等报错

- noeviction（默认配置）

#### 从所有key中删除

- allkeys-random : 从所有的key中随机删除

- allkeys-lru : 从所有的key中挑选最近使用时间距离现在最远的key删除

- allkeys-lfu : 从所有的key中挑选使用频率最低的key删除

#### 从设置了过期时间的key中删除

- volatile-random : 从设置了过期时间的key中随机删除

- volatile-lru : 从设置了过期时间且最近用到的次数最少的key删除

- volatile-ttl : 距离过期时间最近的key删除

- volatile-lfu : 从设置了过期时间的key中挑选使用频率最低的key删除

##redis底层数据结构

redis数据对象结构体定义如下：

	typedef struct redisObject {
	    unsigned type:4;
	    unsigned encoding:4;
	    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
	                            * LFU data (least significant 8 bits frequency
	                            * and most significant 16 bits access time). */
	    int refcount;
	    void *ptr;
	} robj

其结构体对应的属性如下所示：

- type

	值的类型，表示redis五大基本类型(string, hash, list, set, sorted set)

- encoding

	值的编码方式,表示值对应使用的底层结构（sds，压缩列表，跳表等）

- lru

	lru缓存淘汰机制信息（记录了这个对象最后一次被访问的时间，用于淘汰过期的键值对）

- refcount

	引用计数器,记录了对象的引用计数

- *ptr

	是指向数据的指针（指向实际存储的数据指针）

具体如下入所示：

![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220130152853619-138683751.jpg)


### 字符串

#### C语言字符串

- 二进制不安全，用\0作为一个字符串的结束，那么字符串中就不能包含\0了

- 字符串修改的时候，会频繁的重新分配内存

redis的动态字符串，就是为了优化这些情况

#### 动态字符串

redis的动态字符串，就是对C语言中的字符串的封装和优化，其数据结构如下：

	typedef char *sds;
	
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

以sdshdr32为例（不同的sdshdr结构体，其属性占用大小看具体定义）

- buf

	字节数组，保存实际数据。为了表示字节数组的结束，Redis会自动在数组最后加一个“\0”，这就会额外占用1个字节的开销。

- len

	占4个字节，表示buf的已用长度。

- alloc

	占4个字节，表示buf的实际分配长度，一般大于len。

- flags

	占1个字节，只用3bit，5bit没用到但保留，用于计算sds属于什么类型（与SDS_TYPE_MASK进行&运算）

注意，__packed__ 声明了编译的时候不内存对齐，结构体占多少字节就是多少字节，比如sdshdr16一共占用5字节，但因为内存对齐的缘故，编译器会为其填充3个字节，共8个字节，节省了3字节内存

#### 字符串编码方式

一共3中编码，如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220124164741213-916938951.jpg)

##### int编码

当value是一个整数，并且可以使用long类型（8字节）来表示时，那么会属于int编码，redisObj的*ptr直接存储数值。（并且Redis会进行优化，启动时创建0~9999的字符串对象作为共享变量。）

##### embstr编码

字符串小于等于44字节时，RedisObject中的元数据、指针和SDS是一块连续的内存区域，这样就可以避免内存碎片。

##### raw编码

字符串大于44字节时，SDS的数据量就开始变多了，Redis就不再把SDS和RedisObject布局在一起了，而是会给SDS分配独立的空间，并用指针指向SDS结构。

#### 内存扩容

	源码文件：sds.c:sdsMakeRoomFor

	/* Enlarge the free space at the end of the sds string so that the caller
	 * is sure that after calling this function can overwrite up to addlen
	 * bytes after the end of the string, plus one more byte for nul term.
	 *
	 * Note: this does not change the *length* of the sds string as returned
	 * by sdslen(), but only the free buffer space we have. */
	sds sdsMakeRoomFor(sds s, size_t addlen) {
	    void *sh, *newsh;
		// 计算buf剩余可用长度，实际分配长度 - 已用长度（alloc - len）
	    size_t avail = sdsavail(s);
	    size_t len, newlen, reqlen;
		// s[-1] 指针向前偏移1个字节，因为s就是buf[]，也就是指向了flags
	    char type, oldtype = s[-1] & SDS_TYPE_MASK;
	    int hdrlen;
	
	    /* Return ASAP if there is enough space left. */
		// 如果可用内存足够，就不需要扩充内存
	    if (avail >= addlen) return s;
	
	    len = sdslen(s);
	    sh = (char*)s-sdsHdrSize(oldtype);
	    reqlen = newlen = (len+addlen);
	    assert(newlen > len);   /* Catch size_t overflow */
		// 如果新的长度 < 1024*1024，那么扩容自身两倍，否则就扩容1024*1024
	    if (newlen < SDS_MAX_PREALLOC)
	        newlen *= 2;
	    else
	        newlen += SDS_MAX_PREALLOC;
	
		// 根据新的长度，获取对应的类型（也就是上面的sdshdr8，sdshdr16这些）
	    type = sdsReqType(newlen);
	
	    /* Don't use type 5: the user is appending to the string and type 5 is
	     * not able to remember empty space, so sdsMakeRoomFor() must be called
	     * at every appending operation. */
	    if (type == SDS_TYPE_5) type = SDS_TYPE_8;
	
	    hdrlen = sdsHdrSize(type);
	    assert(hdrlen + newlen + 1 > reqlen);  /* Catch size_t overflow */
	    if (oldtype==type) {
			// 如果扩容后的类型和旧的类型一样，那么就在原内存地址上直接扩充内存(realloc)
	        newsh = s_realloc(sh, hdrlen+newlen+1);
	        if (newsh == NULL) return NULL;
	        s = (char*)newsh+hdrlen;
	    } else {
			// 如果不一致，那么就重新分配一块内存地址，将数据复制过去，最后回收旧内存(molloc,memcpy,free)
	        /* Since the header size changes, need to move the string forward,
	         * and can't use realloc */
	        newsh = s_malloc(hdrlen+newlen+1);
	        if (newsh == NULL) return NULL;
	        memcpy((char*)newsh+hdrlen, s, len+1);
	        s_free(sh);
	        s = (char*)newsh+hdrlen;
	        s[-1] = type;
	        sdssetlen(s, len);
	    }
	    sdssetalloc(s, newlen);
	    return s;
	}

参考上面的源码，重点部分已经给出了注释，动态字符串的扩容方案一共有两种

- 扩容后的长度 < SDS_MAX_PREALLOC,也就是1024*1024，那么双倍扩容

- 扩容后的长度 >= SDS_MAX_PREALLOC，那么扩容SDS_MAX_PREALLOC长度

#### 内存缩容

	/* Reallocate the sds string so that it has no free space at the end. The
	 * contained string remains not altered, but next concatenation operations
	 * will require a reallocation.
	 *
	 * After the call, the passed sds string is no longer valid and all the
	 * references must be substituted with the new pointer returned by the call. */
	sds sdsRemoveFreeSpace(sds s) {
	    void *sh, *newsh;
	    char type, oldtype = s[-1] & SDS_TYPE_MASK;
	    int hdrlen, oldhdrlen = sdsHdrSize(oldtype);
	    size_t len = sdslen(s);
	    size_t avail = sdsavail(s);
	    sh = (char*)s-oldhdrlen;
	
	    /* Return ASAP if there is no space left. */
	    if (avail == 0) return s;
	
	    /* Check what would be the minimum SDS header that is just good enough to
	     * fit this string. */
	    type = sdsReqType(len);
	    hdrlen = sdsHdrSize(type);
	
	    /* If the type is the same, or at least a large enough type is still
	     * required, we just realloc(), letting the allocator to do the copy
	     * only if really needed. Otherwise if the change is huge, we manually
	     * reallocate the string to use the different header type. */
	    if (oldtype==type || type > SDS_TYPE_8) {
			// 如果扩容后的类型和旧的类型一样，那么就在原内存地址上直接扩充内存(realloc)
	        newsh = s_realloc(sh, oldhdrlen+len+1);
	        if (newsh == NULL) return NULL;
	        s = (char*)newsh+oldhdrlen;
	    } else {
			// 如果不一致，那么就重新分配一块内存地址，将数据复制过去，最后回收旧内存
	        newsh = s_malloc(hdrlen+len+1);
	        if (newsh == NULL) return NULL;
	        memcpy((char*)newsh+hdrlen, s, len+1);
	        s_free(sh);
	        s = (char*)newsh+hdrlen;
	        s[-1] = type;
	        sdssetlen(s, len);
	    }
	    sdssetalloc(s, len);
	    return s;
	}

缩容的处理，其实和扩容是一致的，只不过缩容是并不需要计算需要扩容的内存长度而已

### 整数数组

在一片连续的内存中，存储内存空间相同大小的元素，查询时间复杂度为O(n)

	源代码 intset.h
	typedef struct intset {
	    uint32_t encoding;
	    uint32_t length;
	    int8_t contents[];
	} intset;

- encoding

	编码类型，决定contents[]每个元素占多少字节，一共有3种类型，值分别为2,4,8

		/* Return the required encoding for the provided value. */
		static uint8_t _intsetValueEncoding(int64_t v) {
		    if (v < INT32_MIN || v > INT32_MAX)
		        return INTSET_ENC_INT64;
		    else if (v < INT16_MIN || v > INT16_MAX)
		        return INTSET_ENC_INT32;
		    else
		        return INTSET_ENC_INT16;
		}

	根据传入的元素值，返回对应的encoding

- length

	总元素格式，也就是contents[]数组的元素个数

- contents[]

	存储元素的数组（有序）

#### 整数数组的元素查找

	源代码 intset.c
	/* Determine whether a value belongs to this set */
	uint8_t intsetFind(intset *is, int64_t value) {
		// 获取查找值的encoding编码
	    uint8_t valenc = _intsetValueEncoding(value);
		// 如果查找值的编码大于整数数组的编码，返回false
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
		// contents数组没有一个元素，就返回0
	    if (intrev32ifbe(is->length) == 0) {
	        if (pos) *pos = 0;
	        return 0;
	    } else {
	        /* Check for the case where we know we cannot find the value,
	         * but do know the insert position. */
	        // 先检查首个元素和最后一个元素，如果就是需要查找的值，那么就直接返回，如果需要获取查找值所在contents数组的位置（传了*pos参数），那么就把位置赋值给*pos
	        if (value > _intsetGet(is,max)) {
	            if (pos) *pos = intrev32ifbe(is->length);
	            return 0;
	        } else if (value < _intsetGet(is,0)) {
	            if (pos) *pos = 0;
	            return 0;
	        }
	    }
	
		// 二分查找
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
	
		// 判断二分查找的结果是否等于查找的值，是的话就返回1，并把位置赋值给*pos（如有传参），否则的话就返回0，并把小于查找值的最接近位置赋值给*pos（如有传参）
	    if (value == cur) {
	        if (pos) *pos = mid;
	        return 1;
	    } else {
	        if (pos) *pos = min;
	        return 0;
	    }
	}

#### 整数数组的元素添加

	源代码 intset.c
	/* Insert an integer in the intset */
	intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
	    uint8_t valenc = _intsetValueEncoding(value);
	    uint32_t pos;
	    if (success) *success = 1;
	
	    /* Upgrade encoding if necessary. If we need to upgrade, we know that
	     * this value should be either appended (if > 0) or prepended (if < 0),
	     * because it lies outside the range of existing values. */
	    // 如果插入值的encoding编码 > 整数数组的encoding，那么就扩容后，在添加
	    if (valenc > intrev32ifbe(is->encoding)) {
	        /* This always succeeds, so we don't need to curry *success. */
	        return intsetUpgradeAndAdd(is,value);
	    } else {
	        /* Abort if the value is already present in the set.
	         * This call will populate "pos" with the right position to insert
	         * the value when it cannot be found. */
	        // 查找添加元素，如果已经存在了，那么就不需要添加了
	        if (intsetSearch(is,value,&pos)) {
	            if (success) *success = 0;
	            return is;
	        }
	
			// intset扩容
	        is = intsetResize(is,intrev32ifbe(is->length)+1);
			// 如果返回contents数组位置，是小于当前整数集合的长度的话，说明元素要插入中间或者首位，那么就需要在插入位置后的元素往后移动，
	        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
	    }
		// 修改contents数组下标为pos的值为新插入的值
	    _intsetSet(is,pos,value);
		// 元素个数 + 1
	    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
	    return is;
	}

	/* Upgrades the intset to a larger encoding and inserts the given integer. */
	static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
	    uint8_t curenc = intrev32ifbe(is->encoding);
	    uint8_t newenc = _intsetValueEncoding(value);
	    int length = intrev32ifbe(is->length);
		// 如果添加的值是小于0，则插入到数组首位置，如果value大于0，则插入到数组最后
		// 因为既然要升级encoding编码，那么说明插入的数不是最小值就是最大值（插入的值超出了原encoding编码范围能表示的数值，才需要扩容）
		// 所以也就不需要像intsetAdd一样，需要进行元素是否存在的判断
	    int prepend = value < 0 ? 1 : 0;
	
	    /* First set new encoding and resize */
	    is->encoding = intrev32ifbe(newenc);
		// 扩展内存空间
	    is = intsetResize(is,intrev32ifbe(is->length)+1);
	
	    /* Upgrade back-to-front so we don't overwrite values.
	     * Note that the "prepend" variable is used to make sure we have an empty
	     * space at either the beginning or the end of the intset. */
	    // 因为intset扩容了，所以需要把contents数据的值重新分配内存大小后，再重新赋值给contents数组
    	// 需要从后往前赋值，因为如果从前往后的话，就会导致元素被覆盖，举个例子，有3个元素的contents数组，每个元素原占用内存空间为2，扩展后变成4，
    	// 又问题contents内存地址是不变的（仅仅只是在原内存地址上进行内存扩充），那么如果从前往后复制的话，新的contents[0]占用4字节
    	// contents[0]重新赋值后，原contents[1]的的值就会覆盖了（原contents[0]+contents[1]占用4字节），所以只能从后往前赋值
	    while(length--)
			// prepend决定在迁移数组的过程中，contents数组是保留首位的内存空间插入新元素，还是最后一位插入新元素
	        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));
	
	    /* Set the value at the beginning or the end. */
	    if (prepend)
			// contents[0]赋值给新元素
	        _intsetSet(is,0,value);
	    else
			// contents最后一位赋值给新元素
	        _intsetSet(is,intrev32ifbe(is->length),value);
		// 元素长度+1
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

	// 这个函数简单来说就是contents[0],contents[1],contents[2],现在要往下标1插入元素，那么下标1,2就要往后移动
	// 下面的代码，就是contents通过内存地址偏移实现的，从from的位置，移动到to的位置，空出的form - to的内存空间，就用来保存新添加的值
	static void intsetMoveTail(intset *is, uint32_t from, uint32_t to) {
	    void *src, *dst;
	    uint32_t bytes = intrev32ifbe(is->length)-from;
	    uint32_t encoding = intrev32ifbe(is->encoding);
	
	    if (encoding == INTSET_ENC_INT64) {
	        src = (int64_t*)is->contents+from;
	        dst = (int64_t*)is->contents+to;
	        bytes *= sizeof(int64_t);
	    } else if (encoding == INTSET_ENC_INT32) {
	        src = (int32_t*)is->contents+from;
	        dst = (int32_t*)is->contents+to;
	        bytes *= sizeof(int32_t);
	    } else {
	        src = (int16_t*)is->contents+from;
	        dst = (int16_t*)is->contents+to;
	        bytes *= sizeof(int16_t);
	    }
	    memmove(dst,src,bytes);
	}

	/* Set the value at pos, using the configured encoding. */
	static void _intsetSet(intset *is, int pos, int64_t value) {
	    uint32_t encoding = intrev32ifbe(is->encoding);
	
	    if (encoding == INTSET_ENC_INT64) {
	        ((int64_t*)is->contents)[pos] = value;
	        memrev64ifbe(((int64_t*)is->contents)+pos);
	    } else if (encoding == INTSET_ENC_INT32) {
	        ((int32_t*)is->contents)[pos] = value;
	        memrev32ifbe(((int32_t*)is->contents)+pos);
	    } else {
	        ((int16_t*)is->contents)[pos] = value;
	        memrev16ifbe(((int16_t*)is->contents)+pos);
	    }
	}

#### 整数数组的元素删除

	源代码 intset.c
	/* Delete integer from intset */
	intset *intsetRemove(intset *is, int64_t value, int *success) {
		// 获取需要删除的值的encoding编码
	    uint8_t valenc = _intsetValueEncoding(value);
	    uint32_t pos;
	    if (success) *success = 0;
	
		// 如果删除值编码大于整数数组的编码，或者在整数数组中找不到删除值，就不需要进行删除操作了
	    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
	        uint32_t len = intrev32ifbe(is->length);
	
	        /* We know we can delete */
	        if (success) *success = 1;
	
	        /* Overwrite value with tail and update length */
			// 在intsetSearch中获取到了需要删除的元素的pos位置，那么对contents数组进行一次内存偏移
	        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos);
			// 因为删除了元素，那么contents[len-1]已经移动到了contents[len-2]，所以contents[len-1]的内存就可以回收了，那么就进行一次内存缩容
	        is = intsetResize(is,len-1);
			// 元素长度 - 1
	        is->length = intrev32ifbe(len-1);
	    }
	    return is;
	}


### 双向链表

用前后指针，分别指向当前元素的上一个元素和下一个元素内存地址，查询时间复杂度为O(n)

### 哈希表

参照全局hash表描述，hashtable结构体如下：

	typedef struct dict {
	    dictType *type;
	    void *privdata;
	    dictht ht[2];
	    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
	    unsigned long iterators; /* number of iterators currently running */
	} dict;

- type

	类型的处理函数，redis会为用途不同的字典设置不同类型特定函数,其定义如下：

	源文件：dict.h

	typedef struct dictType {
		// hash函数
	    uint64_t (*hashFunction)(const void *key);
		// key复制函数
	    void *(*keyDup)(void *privdata, const void *key);
		// 值复制函数
	    void *(*valDup)(void *privdata, const void *obj);
		// key比较
	    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
		// key删除
	    void (*keyDestructor)(void *privdata, void *key);
		// 值删除
	    void (*valDestructor)(void *privdata, void *obj);
	} dictType;

- privdata

	dictCompareKeys方法中使用，用来对比查找的key和dictEntry中的key值进行对比

- ht[2]

	ht数组保存了ht[0]和ht[1]两个元素，通常使用ht[0]保存键值对，ht[1]只在渐进式rehash时使用。

- rehashidx

	当前rehash状态

- iterators

	有多少个迭代器（安全迭代器）正在访问这个hash表

再来看下dictht的结构体定义

	/* This is our hash table structure. Every dictionary has two of this as we
	 * implement incremental rehashing, for the old to the new table. */
	typedef struct dictht {
	    dictEntry **table;
	    unsigned long size;
	    unsigned long sizemask;
	    unsigned long used;
	} dictht;

- table

	hash桶，每个桶都是一个链表来存储数据，用来解决hash冲突的

- size

	hash表当前大小

- sizemask

	总是等于size - 1 ，这个属性和哈希值一起决定（通过&运算）一个键应该被放到table数组（hash桶）的哪个索引上面

- used

	哈希表目前已有节点（键值对）的数量

再来看下dictEntry的结构体定义：
	
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

- key

	hash表里，键值对中的key值

- v

	hash表里，键值对中的value值

	union：联合体，与结构体的各个成员会占用不同的内存，互相之间没有影响；而共用体的所有成员占用同一段内存，修改一个成员会影响其余所有成员

	结构体占用的内存大于等于所有成员占用的内存的总和（成员之间可能会存在缝隙【内存对齐】），共用体占用的内存等于最长的成员占用的内存。共用体使用了内存覆盖技术，同一时刻只能保存一个成员的值，如果对新的成员赋值，就会把原来成员的值覆盖掉。

- next

	redis是使用链地址法解决hash冲突的，这里其实就是一个链表，next指向下一个dictEntry

#### hash桶的内存分配（dictht.size）

	源文件 dict.c:_dictNextPower
	/* Our hash table capability is a power of two */
	static unsigned long _dictNextPower(unsigned long size)
	{
		// 初始化大小为4
	    unsigned long i = DICT_HT_INITIAL_SIZE;
	
	    if (size >= LONG_MAX) return LONG_MAX + 1LU;

		// 每次扩大两倍，直到大于size为止才返回
	    while(1) {
	        if (i >= size)
	            return i;
	        i *= 2;
	    }
	}

按照最接近size的2的幂次数作为分配的空间

#### rehash

	源文件 dict.h：dictIsRehashing
	#define dictIsRehashing(d) ((d)->rehashidx != -1)

	源文件 dict.c：dictRehash
	/* Performs N steps of incremental rehashing. Returns 1 if there are still
	 * keys to move from the old to the new hash table, otherwise 0 is returned.
	 *
	 * Note that a rehashing step consists in moving a bucket (that may have more
	 * than one key as we use chaining) from the old to the new hash table, however
	 * since part of the hash table may be composed of empty spaces, it is not
	 * guaranteed that this function will rehash even a single bucket, since it
	 * will visit at max N*10 empty buckets in total, otherwise the amount of
	 * work it does would be unbound and the function may block for a long time. */
	int dictRehash(dict *d, int n) {
		// n代表的是迁移hash桶数量，由于rehash过程会阻塞主线程，所以通过n来控制每次迁移hash桶的数量，n为0后，rehash操作还没操作完，就等待下一次继续执行rehash操作（渐进式rehash）
		// 最大空hash桶数
	    int empty_visits = n*10; /* Max number of empty buckets to visit. */
		
		// 如果rehashidx == -1，不需要进行rehash操作
	    if (!dictIsRehashing(d)) return 0;
	
		// ht[0]不为空的时候，才进行rehash
	    while(n-- && d->ht[0].used != 0) {
	        dictEntry *de, *nextde;
	
	        /* Note that rehashidx can't overflow as we are sure there are more
	         * elements because ht[0].used != 0 */
	        // 断言，保证ht的大小，要大于正在进行rehash处理的hash桶下标，否则会直接结束
	        assert(d->ht[0].size > (unsigned long)d->rehashidx);
			// hash桶如果为null，这个桶就不需要执行rehash操作（不用迁移数据到ht[1]）
	        while(d->ht[0].table[d->rehashidx] == NULL) {
	            d->rehashidx++;
				// 如果空桶数总计达到empty_visits，那么就直接直接返回，防止占用主线程做大量的无用功
	            if (--empty_visits == 0) return 1;
	        }
	        de = d->ht[0].table[d->rehashidx];
	        /* Move all the keys in this bucket from the old to the new hash HT */
			// 迁移数据到ht[1]
	        while(de) {
	            uint64_t h;
	
	            nextde = de->next;
	            /* Get the index in the new hash table */
	            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
	            de->next = d->ht[1].table[h];
	            d->ht[1].table[h] = de;
	            d->ht[0].used--;
	            d->ht[1].used++;
	            de = nextde;
	        }
			// 每一个hash桶数据迁移到ht[1]之后，ht[0]对应的hash桶置为null
	        d->ht[0].table[d->rehashidx] = NULL;
			// 下一个hash桶进行rehash操作
	        d->rehashidx++;
	    }
	
	    /* Check if we already rehashed the whole table... */
		// 使用ht[1]覆盖ht[0]，rehashidx重新置为-1，代表完成rehash操作
	    if (d->ht[0].used == 0) {
	        zfree(d->ht[0].table);
	        d->ht[0] = d->ht[1];
			// 重置ht[1]数据
	        _dictReset(&d->ht[1]);
	        d->rehashidx = -1;
	        return 0;
	    }
	
	    /* More to rehash... */
	    return 1;
	}

	// 重置dictht数据
	/* Reset a hash table already initialized with ht_init().
	 * NOTE: This function should only be called by ht_destroy(). */
	static void _dictReset(dictht *ht)
	{
	    ht->table = NULL;
	    ht->size = 0;
	    ht->sizemask = 0;
	    ht->used = 0;
	}

从上述代码可以看出，当dict.rehashidx == -1，表示不需要进行rehash操作，当rehashidx >=0 代表第几个hash桶正在进行rehash操作

而ht[1]的作用，其实就是在rehash的过程中，作为临时存储迁移数据的一个容器，迁移完毕后ht[1]会进行重置

##### rehash的条件

	源文件 dich.c:_dictExpandIfNeeded
	/* Expand the hash table if needed */
	static int _dictExpandIfNeeded(dict *d)
	{
	    /* Incremental rehashing already in progress. Return. */
		// rehash正在进行中，就不需要进行扩展
	    if (dictIsRehashing(d)) return DICT_OK;
	
	    /* If the hash table is empty expand it to the initial size. */
		// 如果ht[0]的大小为0，就扩展成4
	    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);
	
	    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
	     * table (global setting) or we should avoid it but the ratio between
	     * elements/buckets is over the "safe" threshold, we resize doubling
	     * the number of buckets. */
	    // dict_can_resize是 redis 的一个常量，当有BGSAVE或者BGREWRITEAOF命令在执行时，会将该常量置为0
		// 负载因子 = used / size
		// 当redis在做持久化的时候，负载因子要>5
		// 当redis没在做持久化的时候，负载因子要>=1
	    if (d->ht[0].used >= d->ht[0].size &&
	        (dict_can_resize ||
	         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
	    {
	        return dictExpand(d, d->ht[0].used*2);
	    }
	    return DICT_OK;
	}

如上述代码所示，在不同的情况下，rehash的条件不同

- 持久化

	负载因子 > 5

- 非持久化

	负载因子 >= 1

至于为啥在持久化和非持久化的情况下，负载因子不同，我猜想是因为redis在进行持久化的时候，因为是开启子线程去处理的，因为linux写时复制的机制，此时主线程和子线程共享同一份数据

但如果在进行持久化的过程中，突然间进行了rehash操作，这时由于主线程开始写操作（rehash过程中，每一个hash桶迁移完毕后，以前的数据就会被清空），所以子线程不得不拷贝多一份内存数据继续进行持久化操作，这时，redis的内存使用率就会大幅度上升

所以为了避免这种情况，在持久化的时候，加大负载因子，能尽可能的避免出现这种情况发生

##### rehash中增

	源文件 dict.c:dictAdd
	/* Add an element to the target hash table */
	int dictAdd(dict *d, void *key, void *val)
	{
	    dictEntry *entry = dictAddRaw(d,key,NULL);
	
	    if (!entry) return DICT_ERR;
	    dictSetVal(d, entry, val);
	    return DICT_OK;
	}

	/* Low level add or find:
	 * This function adds the entry but instead of setting a value returns the
	 * dictEntry structure to the user, that will make sure to fill the value
	 * field as he wishes.
	 *
	 * This function is also directly exposed to the user API to be called
	 * mainly in order to store non-pointers inside the hash value, example:
	 *
	 * entry = dictAddRaw(dict,mykey,NULL);
	 * if (entry != NULL) dictSetSignedIntegerVal(entry,1000);
	 *
	 * Return values:
	 *
	 * If key already exists NULL is returned, and "*existing" is populated
	 * with the existing entry if existing is not NULL.
	 *
	 * If key was added, the hash entry is returned to be manipulated by the caller.
	 */
	dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
	{
	    long index;
	    dictEntry *entry;
	    dictht *ht;
	
	    if (dictIsRehashing(d)) _dictRehashStep(d);
	
	    /* Get the index of the new element, or -1 if
	     * the element already exists. */
	    // 返回key所在hash桶下标,如果key已经存在的话就直接返回
	    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
	        return NULL;
	
	    /* Allocate the memory and store the new entry.
	     * Insert the element in top, with the assumption that in a database
	     * system it is more likely that recently added entries are accessed
	     * more frequently. */
	    // 如果在rehash中的话，那么就往ht[1]里插入数据，否则就往ht[0]插入数据
	    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
	    entry = zmalloc(sizeof(*entry));
	    entry->next = ht->table[index];
	    ht->table[index] = entry;
	    ht->used++;
	
	    /* Set the hash entry fields. */
	    dictSetKey(d, entry, key);
	    return entry;
	}

##### rehash中删

	/* Remove an element, returning DICT_OK on success or DICT_ERR if the
	 * element was not found. */
	int dictDelete(dict *ht, const void *key) {
	    return dictGenericDelete(ht,key,0) ? DICT_OK : DICT_ERR;
	}

	/* Search and remove an element. This is an helper function for
	 * dictDelete() and dictUnlink(), please check the top comment
	 * of those functions. */
	static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
	    uint64_t h, idx;
	    dictEntry *he, *prevHe;
	    int table;
	
	    if (d->ht[0].used == 0 && d->ht[1].used == 0) return NULL;
	
	    if (dictIsRehashing(d)) _dictRehashStep(d);
	    h = dictHashKey(d, key);
	
	    for (table = 0; table <= 1; table++) {
	        idx = h & d->ht[table].sizemask;
	        he = d->ht[table].table[idx];
	        prevHe = NULL;
	        while(he) {
				// 在hash桶中，寻找对应的key
	            if (key==he->key || dictCompareKeys(d, key, he->key)) {
	                /* Unlink the element from the list */
	                if (prevHe)
	                    prevHe->next = he->next;
	                else
	                    d->ht[table].table[idx] = he->next;
	                if (!nofree) {
	                    dictFreeKey(d, he);
	                    dictFreeVal(d, he);
	                    zfree(he);
	                }
	                d->ht[table].used--;
	                return he;
	            }
	            prevHe = he;
	            he = he->next;
	        }

			// 如果不在rehash，那么就不需要删除ht[1]的数据
	        if (!dictIsRehashing(d)) break;
	    }
	    return NULL; /* not found */
	}

##### rehash中的改

	/* Add or Overwrite:
	 * Add an element, discarding the old value if the key already exists.
	 * Return 1 if the key was added from scratch, 0 if there was already an
	 * element with such key and dictReplace() just performed a value update
	 * operation. */
	int dictReplace(dict *d, void *key, void *val)
	{
	    dictEntry *entry, *existing, auxentry;
	
	    /* Try to add the element. If the key
	     * does not exists dictAdd will succeed. */
	    // key不存在，直接新增
	    entry = dictAddRaw(d,key,&existing);
	    if (entry) {
	        dictSetVal(d, entry, val);
	        return 1;
	    }
	
	    /* Set the new value and free the old one. Note that it is important
	     * to do that in this order, as the value may just be exactly the same
	     * as the previous one. In this context, think to reference counting,
	     * you want to increment (set), and then decrement (free), and not the
	     * reverse. */
	    // 修改值，先新增值后释放旧值
	    auxentry = *existing;
	    dictSetVal(d, existing, val);
	    dictFreeVal(d, &auxentry);
	    return 0;
	}

##### rehash中查

	dictEntry *dictFind(dict *d, const void *key)
	{
	    dictEntry *he;
	    uint64_t h, idx, table;
	
	    if (d->ht[0].used + d->ht[1].used == 0) return NULL; /* dict is empty */
	    if (dictIsRehashing(d)) _dictRehashStep(d);
	    h = dictHashKey(d, key);
	    for (table = 0; table <= 1; table++) {
	        idx = h & d->ht[table].sizemask;
	        he = d->ht[table].table[idx];
	        while(he) {
	            if (key==he->key || dictCompareKeys(d, key, he->key))
	                return he;
	            he = he->next;
	        }
			// 如果在rehash，那么还需要查找ht[1]
	        if (!dictIsRehashing(d)) return NULL;
	    }
	    return NULL;
	}

##### hash缩容条件

	源代码文件 dict.h
	#define dictSlots(d) ((d)->ht[0].size+(d)->ht[1].size)
	#define dictSize(d) ((d)->ht[0].used+(d)->ht[1].used)

	源代码文件 server.c:htNeedsResize
	// 判断是否需要缩容
	int htNeedsResize(dict *dict) {
	    long long size, used;
	
	    size = dictSlots(dict);
	    used = dictSize(dict);
		// size > 4 和 used / size < 10%
		// 当使用内存/分配内存 < 10%的时候，就需要进行缩容
	    return (size > DICT_HT_INITIAL_SIZE &&
	            (used*100/size < HASHTABLE_MIN_FILL));
	}

	源代码文件 dict.c:dictResize
	/* Resize the table to the minimal size that contains all the elements,
	 * but with the invariant of a USED/BUCKETS ratio near to <= 1 */
	// 缩容操作
	int dictResize(dict *d)
	{
	    int minimal;
	
		// 如果正在持久化或者rehash操作，就不缩容
	    if (!dict_can_resize || dictIsRehashing(d)) return DICT_ERR;
		// 尽可能让used:size = 1:1
	    minimal = d->ht[0].used;
		// used小于默认值的话，把size设为默认值
	    if (minimal < DICT_HT_INITIAL_SIZE)
	        minimal = DICT_HT_INITIAL_SIZE;
	    return dictExpand(d, minimal);
	}

每当有redis数据进行删除的时候(hash:hashTypeDelete,set:setTypeRemove,zset:zsetDel)，都会调用上面的方法检测是否需要缩容，除此之外，还有redis的定时任务去检测(databaseCron，心跳检测函数，调用tryResizeHashTables)

##### hash表的迭代器

	源代码 dict.h
	/* If safe is set to 1 this is a safe iterator, that means, you can call
	 * dictAdd, dictFind, and other functions against the dictionary even while
	 * iterating. Otherwise it is a non safe iterator, and only dictNext()
	 * should be called while iterating. */
	typedef struct dictIterator {
	    dict *d; // 遍历的字典对象
	    long index; // 当前遍历的槽位置（hash桶下标），初始为-1
	    int table, safe; // table 就是ht[0]或ht[1] safe 迭代器是否安全（1安全0不安全）
	    dictEntry *entry, *nextEntry; // entry hash桶的链表中，当前遍历到的元素，nextEntry 指向下一个元素
	    /* unsafe iterator fingerprint for misuse detection. */
	    long long fingerprint; // 用于不安全的迭代器，验证迭代过程中字典是否有被修改
	} dictIterator;

###### hash表的安全迭代器和非安全迭代器

安全和非安全，其实指的就是是否会遍历重复元素。

以执行keys命令为例，执行 ```keys *``` 是遍历出redis所有的key，但是在遍历的时候，执行rehash操作，结果有可能会出现重复的key

比如说，在ht[0]中，index=3的哈希桶中的key1，在遍历返回输出之后，由于rehash操作，key1值的index变为了6，那么如果index为6的hash桶已经遍历过了，结果集不会出现重复，但如果还没遍历过，在遍历index=6的hash桶的时候，key1就会再次出现在结果集中

可以看下keys命令的源码

	源代码 db.c:keysCommand
	void keysCommand(client *c) {
	    dictIterator *di;
	    dictEntry *de;
		// 获取匹配字符串
	    sds pattern = c->argv[1]->ptr;
	    int plen = sdslen(pattern), allkeys;
	    unsigned long numkeys = 0;
	    void *replylen = addDeferredMultiBulkLength(c);
	
		// 初始化一个安全迭代器，所以keys 是不会返回重复的数据的
	    di = dictGetSafeIterator(c->db->dict);
		// 判断是否全遍历
	    allkeys = (pattern[0] == '*' && plen == 1);
	    while((de = dictNext(di)) != NULL) {
			// 获取key值
	        sds key = dictGetKey(de);
	        robj *keyobj;
			// allkeys 为true，就是全返回
			// allkeys 为false，就是符合pattern表达式的才返回(stringmatchlen)
	        if (allkeys || stringmatchlen(pattern,plen,key,sdslen(key),0)) {
	            keyobj = createStringObject(key,sdslen(key));
				// 判断key是否过期
				// key过期了，就不返回，由于有这个判断，所以redis的删除策略就算是延迟删除，也不会返回过期的数据
	            if (!keyIsExpired(c->db,keyobj)) {
	                addReplyBulk(c,keyobj);
	                numkeys++;
	            }
	            decrRefCount(keyobj);
	        }
	    }

		// 释放安全迭代器
	    dictReleaseIterator(di);
	    setDeferredMultiBulkLength(c,replylen,numkeys);
	}

	/* This function performs just a step of rehashing, and only if there are
	 * no safe iterators bound to our hash table. When we have iterators in the
	 * middle of a rehashing we can't mess with the two hash tables otherwise
	 * some element can be missed or duplicated.
	 *
	 * This function is called by common lookup or update operations in the
	 * dictionary so that the hash table automatically migrates from H1 to H2
	 * while it is actively used. */
	static void _dictRehashStep(dict *d) {
		// 安全迭代器的数量为0的时候，才能进行rehash操作
		// dict.iterators只有在真实查找hash表的时候，才会+1（dictNext方法）
		// 在释放安全迭代器的时候，dict.iterators会-1（dictReleaseIterator方法）
	    if (d->iterators == 0) dictRehash(d,1);
	}

###### fingerprint的作用

还有以```keys``` 命令举例，因为redis的网络IO和键值对读写是由一个线程来完成的，因此在执行keys命令的时候，redis的其他增删改查命令都会被阻塞，再加上因为安全迭代器数为0的时候，才进行rehash，那么在执行keys命令的时候，对于redis的整个hash表来说，应该不会被修改数据了

然而还有一种情况，就是redis的定时任务，redis的过期数据删除策略中，其中有一项是redis会定期删除过期数据的，所以就算是安全迭代器，也无法防止在遍历期间hash表的修改

在非安全迭代器下，因为允许rehash，因为没有安全迭代器那种强制性，所以我们只能通过fingerprint得知hash表在遍历期间，数据是否发生过修改，然后再作处理

先来看下下面的代码

	源代码 dict.c:dictNext
	dictEntry *dictNext(dictIterator *iter)
	{
	    while (1) {
			// iter->entry 是hash桶的链表中，当前遍历到的元素
			// 当迭代器首次运行遍历的时候，该值为null
			// 遍历到当前hash桶的链表中最后一个元素后，下一次循环，该值为null（因为nextEntry指向下一个指针的元素为null，而且下面也有一个赋值操作iter->entry = iter->nextEntry）
	        if (iter->entry == NULL) {
				// 指向被遍历的哈希表
	            dictht *ht = &iter->d->ht[iter->table];
				// 如果是首次遍历的话（index是hash桶的下标），而且遍历的是ht[0]，所以下面的if只会执行一次
	            if (iter->index == -1 && iter->table == 0) {
	                if (iter->safe)
						// 如果是安全迭代器（safe为1），那么更新iterators计数器
	                    iter->d->iterators++;
	                else
						// 如果是不安全迭代器，那么在开始遍历之前，计算指纹并记录
	                    iter->fingerprint = dictFingerprint(iter->d);
	            }
				// 更新迭代器的当前需要遍历的hash桶下标
	            iter->index++;
				// 要是下标大于hash桶数量，如果正在执行rehash操作，那么还需要遍历ht[1]
				// 如果没有执行rehash操作，那么遍历完成，退出循环
	            if (iter->index >= (long) ht->size) {
	                if (dictIsRehashing(iter->d) && iter->table == 0) {
	                    iter->table++;
	                    iter->index = 0;
	                    ht = &iter->d->ht[1];
	                } else {
	                    break;
	                }
	            }
				// 指向ht[0]或ht[1]的第一个hash桶的头结点
	            iter->entry = ht->table[iter->index];
	        } else {
				// 指向当前遍历的hash桶的下一个元素
	            iter->entry = iter->nextEntry;
	        }
			// 如果当前hash桶中的元素不为空，那么更新迭代器指向下一个元素，然后返回迭代器，这样就能通过了迭代器，外层用一个循环就能获取到每个元素的key值和value值（dictNext方法通常在循环里面调用）
	        if (iter->entry) {
	            /* We need to save the 'next' here, the iterator user
	             * may delete the entry we are returning. */
	            iter->nextEntry = iter->entry->next;
	            return iter->entry;
	        }
	    }
		// 遍历结束，返回null
	    return NULL;
	}

	/* A fingerprint is a 64 bit number that represents the state of the dictionary
	 * at a given time, it's just a few dict properties xored together.
	 * When an unsafe iterator is initialized, we get the dict fingerprint, and check
	 * the fingerprint again when the iterator is released.
	 * If the two fingerprints are different it means that the user of the iterator
	 * performed forbidden operations against the dictionary while iterating. */
	// 通过ht中的size,used和table来计算指纹
	long long dictFingerprint(dict *d) {
	    long long integers[6], hash = 0;
	    int j;
	
	    integers[0] = (long) d->ht[0].table;
	    integers[1] = d->ht[0].size;
	    integers[2] = d->ht[0].used;
	    integers[3] = (long) d->ht[1].table;
	    integers[4] = d->ht[1].size;
	    integers[5] = d->ht[1].used;
	
	    /* We hash N integers by summing every successive integer with the integer
	     * hashing of the previous sum. Basically:
	     *
	     * Result = hash(hash(hash(int1)+int2)+int3) ...
	     *
	     * This way the same set of integers in a different order will (likely) hash
	     * to a different number. */
	    for (j = 0; j < 6; j++) {
	        hash += integers[j];
	        /* For the hashing step we use Tomas Wang's 64 bit integer hash. */
	        hash = (~hash) + (hash << 21); // hash = (hash << 21) - hash - 1;
	        hash = hash ^ (hash >> 24);
	        hash = (hash + (hash << 3)) + (hash << 8); // hash * 265
	        hash = hash ^ (hash >> 14);
	        hash = (hash + (hash << 2)) + (hash << 4); // hash * 21
	        hash = hash ^ (hash >> 28);
	        hash = hash + (hash << 31);
	    }
	    return hash;
	}

这样的话，遍历完后通过 ```iter->fingerprint == dictFingerprint(iter->d)``` 就可以得知hash表是否被修改过了，只要是不等于，就说明被改过








	struct dict {
	    int rehashindex;
	    dictht ht[2]; 
	}
	struct dictht {
	    dictEntry** table; // 二维
	    long size; // 第一维数组的长度 
	    long used; // hash 表中的元素个数 ...
	}
	typedef struct dictEntry {
	  //键  
	  void *key;
	  //值，可以是一个指针，也可以是一个uint64_t整数，也可以是int64_t的整数
	  union {
	    void *val;
	    uint64_tu64;
	    int64_ts64;
	  } v；
	  //指向下一个节点的指针
	  struct dictEntry *next;
	} dictEntry；

dict结构中的ht数组保存了ht[0]和ht[1]两个元素，通常使用ht[0]保存键值对，ht[1]只在渐进式rehash时使用。

hashtable是通过链地址法来解决冲突的，table数组存储的是链表的头结点（添加新元素，首先根据键计算出hash值，然后与数组长度取模之后得到数组下标，将元素添加到数组下标对应的链表中去）。

### 压缩列表

相比于数组每个元素占用的内存空间大小必须一致，压缩列表就是可以存放不同内存空间大小的元素（相比于数组节省了空间），如下图所示：

![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220101173258715-1041078072.png)

![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220101173805426-962968121.png)

因为数组是通过元素大小作为偏移量来找到下一个元素数据的，但因为压缩数组元素大小不一致，所以其元素的数据结构如下图所示：

![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220101174240540-2068634598.png)

data-length记录元素大小，而data-value记录的则是元素值，通过data-length来计算下一个元素的偏移量

除此之外，redis的压缩列表在表头有三个字段zlbytes、zltail和zllen，分别表示列表长度、列表尾的偏移量和列表中的entry个数；压缩列表在表尾还有一个zlend，表示列表结束。如下入所示：

![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220101175044538-519851684.png)

在压缩列表中，如果我们要查找定位第一个元素和最后一个元素，可以通过表头三个字段的长度直接定位，复杂度是O(1)。而查找其他元素时，就没有这么高效了，只能逐个查找，此时的复杂度就是O(N)了。如下如所示：

![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220101175058286-1934080440.png)

图中展示了一个总长为80字节，包含3个节点的压缩列表。如果我们有一个指向压缩列表起始地址的指针p，那么表为节点的地址就是P+60。

而压缩列表的entry数据结构如下：

	/* We use this function to receive information about a ziplist entry.
	 * Note that this is not how the data is actually encoded, is just what we
	 * get filled by a function in order to operate more easily. */
	typedef struct zlentry {
		// 编码 prevrawlen 所需的字节大小
	    unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
		// 前置节点的长度
	    unsigned int prevrawlen;     /* Previous entry len. */
		// 编码 len 所需的字节大小
	    unsigned int lensize;        /* Bytes used to encode this entry type/len.
	                                    For example strings have a 1, 2 or 5 bytes
	                                    header. Integers always use a single byte.*/
		// 当前节点值的长度
	    unsigned int len;            /* Bytes used to represent the actual entry.
	                                    For strings this is just the string length
	                                    while for integers it is 1, 2, 3, 4, 8 or
	                                    0 (for 4 bit immediate) depending on the
	                                    number range. */
		// 当前节点 header 的大小
		// 等于 prevrawlensize + lensize
	    unsigned int headersize;     /* prevrawlensize + lensize. */
		// 当前节点值所使用的编码类型
	    unsigned char encoding;      /* Set to ZIP_STR_* or ZIP_INT_* depending on
	                                    the entry encoding. However for 4 bits
	                                    immediate integers this can assume a range
	                                    of values and must be range-checked. */
		// 指向当前节点的指针
	    unsigned char *p;            /* Pointer to the very start of the entry, that
	                                    is, this points to prev-entry-len field. */
	} zlentry;


### 跳表

因为双向链表在查找数据的时候，只能一个一个的去查，效率不高，所以跳表就是在链表的基础上，增加了一层索引。如下图所示：

![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220101175955710-518775319.jpg)

可以看到，查询33这个值，需要在链表上查询6次，而在跳表中，随着索引层级的越多，查询的次数越少，而这个查找过程就是在多级索引上跳来跳去，最后定位到元素，其时间复杂度为O(logN)。

	源代码 server.h
	/* ZSETs use a specialized version of Skiplists */
	typedef struct zskiplistNode {
	    sds ele; // zset元素值
	    double score; // zset权重值
	    struct zskiplistNode *backward; // 前一个节点
	    struct zskiplistLevel {
	        struct zskiplistNode *forward; // 后一个节点
	        unsigned long span; // 本层跳表索引的下一个节点，相对于下一层索引横跨的节点数
	    } level[]; // 跳表索引数组，上图每一级索引，就代表一个level数组
	} zskiplistNode;
	
	typedef struct zskiplist {
	    struct zskiplistNode *header, *tail; // 跳表头尾节点
	    unsigned long length; // 跳表长度（跳表元素个数）
	    int level; // 跳表层级（level数组的长度）
	} zskiplist;

其结构如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202204/901559-20220416113304093-1585767828.jpg)

该跳表有10个节点（length=10），一共有3层索引（level=3），tail指向最后一个节点，而header指向的是头节点，头结点是一个特殊的节点，该节点并不会保存节点元素，在创建跳表的时候，头结点会和跳表一起同时创建

#### 跳表的创建

	源代码 server.h
	#define ZSKIPLIST_MAXLEVEL 64 /* Should be enough for 2^64 elements */

	源代码 t_zset.c
	/* Create a skiplist node with the specified number of levels.
	 * The SDS string 'ele' is referenced by the node after the call. */
	zskiplistNode *zslCreateNode(int level, double score, sds ele) {
	    zskiplistNode *zn =
	        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
	    zn->score = score;
	    zn->ele = ele;
	    return zn;
	}
	
	/* Create a new skiplist. */
	zskiplist *zslCreate(void) {
	    int j;
	    zskiplist *zsl;
	
	    zsl = zmalloc(sizeof(*zsl));
	    zsl->level = 1;
	    zsl->length = 0;
		// 初始化头结点，并把zsl的header指向头结点
	    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
		// 头结点预先初始化好64个数组
	    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
	        zsl->header->level[j].forward = NULL;
	        zsl->header->level[j].span = 0;
	    }
	    zsl->header->backward = NULL;
	    zsl->tail = NULL;
	    return zsl;
	}

### 跳表新增节点

	源代码 t_zset.c
	/* Returns a random level for the new skiplist node we are going to create.
	 * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL
	 * (both inclusive), with a powerlaw-alike distribution where higher
	 * levels are less likely to be returned. */
	int zslRandomLevel(void) {
	    int level = 1;
	    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
	        level += 1;
	    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
	}
	
	/* Insert a new node in the skiplist. Assumes the element does not already
	 * exist (up to the caller to enforce that). The skiplist takes ownership
	 * of the passed SDS string 'ele'. */
	zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
	    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
	    unsigned int rank[ZSKIPLIST_MAXLEVEL];
	    int i, level;
	
	    serverAssert(!isnan(score));
	    x = zsl->header;
		// 从最大层数开始遍历查找
	    for (i = zsl->level-1; i >= 0; i--) {
	        /* store rank that is crossed to reach the insert position */
	        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
			// 如果level[i]的下一个节点不为空 &&
			// 下一个level[i]的权重小于插入节点的权重 ||
			// 下一个level[i]的权重等于插入节点的权重 && 下一个level[i]的元素值你插入的元素值要小
			// 其实就是要找到比插入节点当前权重要小的最后一个节点
	        while (x->level[i].forward &&
	                (x->level[i].forward->score < score ||
	                    (x->level[i].forward->score == score &&
	                    sdscmp(x->level[i].forward->ele,ele) < 0)))
	        {
				// 记录每一层的level[i]距离header节点的总步长
	            rank[i] += x->level[i].span;
	            x = x->level[i].forward;
	        }
			// 记录每一层比插入的权重小的最后一个节点，如果改层没有比插入的权重要小的节点，值为头结点
	        update[i] = x;
	    }
	    /* we assume the element is not already inside, since we allow duplicated
	     * scores, reinserting the same element should never happen since the
	     * caller of zslInsert() should test in the hash table if the element is
	     * already inside or not. */
	    // 生成插入的节点需要插入的高度，如level=2，就要插入level[0],level[1],level[2]中
	    level = zslRandomLevel();
		// 如果需要插入的高度，是大于当前跳表层级高度的话，则调整跳表的高度
	    if (level > zsl->level) {
	        for (i = zsl->level; i < level; i++) {
	            rank[i] = 0;
	            update[i] = zsl->header;
				// 步长预先赋值为跳表的总长度，之后会重新计算的
	            update[i]->level[i].span = zsl->length;
	        }
			// 更新跳表层级数
	        zsl->level = level;
	    }

		// 创建需要插入的节点
	    x = zslCreateNode(level,score,ele);
	    for (i = 0; i < level; i++) {
			// 插入的节点，指向下一个节点
	        x->level[i].forward = update[i]->level[i].forward;
			// 比插入的权重小的最后一个节点，指向插入的节点
	        update[i]->level[i].forward = x;
	
	        /* update span covered by update[i] as x is inserted here */
			// 计算新插入的节点，每层新插入的节点，到下一个节点的步长
	        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
			// 因为插入了新节点，所以插入位置的上一个节点，需要步长+1
	        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
	    }
	
	    /* increment span for untouched levels */
		// 如果level的值，没超过zsl->level，那么在level深度以上的节点步长+1
	    for (i = level; i < zsl->level; i++) {
	        update[i]->level[i].span++;
	    }
	
		// 如果在头节点插入之后插入，那么插入节点的后一个节点指针赋值为null，否则指向上一个节点
	    x->backward = (update[0] == zsl->header) ? NULL : update[0];
	    if (x->level[0].forward)
			// 下一个节点的后一个节点指针赋值为新插入节点
	        x->level[0].forward->backward = x;
	    else
			// 如果插入的地方是最后一个节点，跳表的tail执行更新为新插入的节点
	        zsl->tail = x;
		// 跳表长度+1
	    zsl->length++;
	    return x;
	}

	/* Create a skiplist node with the specified number of levels.
	 * The SDS string 'ele' is referenced by the node after the call. */
	zskiplistNode *zslCreateNode(int level, double score, sds ele) {
	    zskiplistNode *zn =
	        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
	    zn->score = score;
	    zn->ele = ele;
	    return zn;
	}

上面的代码看着比较复杂，但是梳理一下流程的话，其实也比较好理解

- 初始化rank数组和update数组

	rank数组，用来保存每一层新插入节点插入位置的上一个节点，到header节点的总步长，而update数组则保存每一层新插入节点插入位置的上一个节点

- 生成新节点level数组，如果比跳表的level大，那么就初始化多出来的level数组

- 创建插入节点x更新x的步长（span），如果新节点的生成的level比跳表的level小的话，那么下标level以上的level数组，步长+1

- 维护新插入节点的forward和backward节点，新插入节点的上一个节点的forward和新插入节点下一个节点的backward，如果是最后一个元素位置插入的话，还要维护跳表的tail

- 维护跳表长度

### 跳表删除节点

	源代码 t_zset.c
	/* Free the specified skiplist node. The referenced SDS string representation
	 * of the element is freed too, unless node->ele is set to NULL before calling
	 * this function. */
	void zslFreeNode(zskiplistNode *node) {
	    sdsfree(node->ele);
	    zfree(node);
	}

	/* Internal function used by zslDelete, zslDeleteByScore and zslDeleteByRank */
	void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
	    int i;
	    for (i = 0; i < zsl->level; i++) {
			// 寻找每一层节点，是否包含了需要删除的节点x
	        if (update[i]->level[i].forward == x) {
				// 更新上一个节点的步长，步长为上一个节点到删除节点步长 + 删除节点到下一个节点的步长 - 1
	            update[i]->level[i].span += x->level[i].span - 1;
				// 更新上一个节点指向下一个节点的指针指向
	            update[i]->level[i].forward = x->level[i].forward;
	        } else {
				// level[i]不存在删除的节点，就只需要上一个节点的步长-1
	            update[i]->level[i].span -= 1;
	        }
	    }
	    if (x->level[0].forward) {
			// 如果被删除的节点不是最后一个节点，那么更新下一个节点指向上一个节点的指针指向
	        x->level[0].forward->backward = x->backward;
	    } else {
			// 如果被删除的节点是最后一个节点，那么就更新跳表的tail指针，指向被删除节点的上一个节点
	        zsl->tail = x->backward;
	    }

		// 检查header节点每一层level[i]的forward指向，如果指向null，说明level[i]已经没有任何节点了，跳表的节点高度-1
	    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
	        zsl->level--;

		// 节点被删除了，那么跳表的长度 - 1
	    zsl->length--;
	}
	
	/* Delete an element with matching score/element from the skiplist.
	 * The function returns 1 if the node was found and deleted, otherwise
	 * 0 is returned.
	 *
	 * If 'node' is NULL the deleted node is freed by zslFreeNode(), otherwise
	 * it is not freed (but just unlinked) and *node is set to the node pointer,
	 * so that it is possible for the caller to reuse the node (including the
	 * referenced SDS string at node->ele). */
	int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
	    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
	    int i;
	
	    x = zsl->header;
		// 这里的逻辑和添加节点的一样，就是找到删除节点的前一个节点
	    for (i = zsl->level-1; i >= 0; i--) {
	        while (x->level[i].forward &&
	                (x->level[i].forward->score < score ||
	                    (x->level[i].forward->score == score &&
	                     sdscmp(x->level[i].forward->ele,ele) < 0)))
	        {
	            x = x->level[i].forward;
	        }
	        update[i] = x;
	    }
	    /* We may have multiple elements with the same score, what we need
	     * is to find the element with both the right score and object. */
	    // x为待删除的节点
	    x = x->level[0].forward;
	    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
			// 删除节点
	        zslDeleteNode(zsl, x, update);
	        if (!node)
				// 释放节点
	            zslFreeNode(x);
	        else
				// 如果node参数不为null的话，那么node就赋值为需要删除的节点，然调用该方法的地方能获取到需要删除节点的信息
	            *node = x;
			// 删除成功返回1
	        return 1;
	    }

		// 如果x的权重和需要删除的权重和元素不一致的话，那么就说明节点不存在，返回0
	    return 0; /* not found */
	}

删除节点的逻辑，其实有点像新增节点，先找到需要被删除节点的上一个节点，删除之后，需要维护原跳表tail，length和level，上一个节点的forward和下一个节点的backward

## redis数据类型

### 字符串

见sds分析

### hash

hash数据类型使用的底层数据结构是ziplist和hashtable

当hash表存储的元素个数小于512，元素小于64字节，hash表会使用ziplist作为的底层实现，否则则会使用hashtable。

#### 相关参数配置

- hash-max-ziplist-entries（表示用压缩列表保存时哈希集合中的最大元素个数）

- hash-max-ziplist-value（表示用压缩列表保存时哈希集合中单个元素的最大长度）

### list

先看下list的结构体定义

	源代码文件adlist.h

	typedef struct list {
	    listNode *head;
	    listNode *tail;
	    void *(*dup)(void *ptr);
	    void (*free)(void *ptr);
	    int (*match)(void *ptr, void *key);
	    unsigned long len;
	} list;

- head

	指向链表的头结点

- tail

	指向链表的尾结点

- dup

	复制节点的时候定义的处理函数

- free

	释放节点的时候定义的处理函数

- match

	链表节点比较的时候定义的处理函数

- len

	链表节点数

再来看下链表节点的结构体定义

	源代码文件adlist.h
	
	typedef struct listNode {
	    struct listNode *prev;
	    struct listNode *next;
	    void *value;
	} listNode;

- prev

	指向链表的上一个节点

- next

	指向链表的下一个节点

- value

	节点保存的数据

除了这两个数据结构之外，list还有个重要的数据接口，就是listIter，这是双向链表的迭代器，方便从两个方向对双端链表进行迭代，其结构体定义及其迭代器方法如下：

	源代码文件adlist.h

	typedef struct listIter {
	    listNode *next;
	    int direction;
	} listIter;

- listNode 

	listNode的prev或者tail指针

- direction

	listNode遍历的方向（从前往后/从后往前）


	源代码文件adlist.c:listGetIterator

	/* Returns a list iterator 'iter'. After the initialization every
	 * call to listNext() will return the next element of the list.
	 *
	 * This function can't fail. */
	listIter *listGetIterator(list *list, int direction)
	{
	    listIter *iter;
	
	    if ((iter = zmalloc(sizeof(*iter))) == NULL) return NULL;
	    if (direction == AL_START_HEAD)
	        iter->next = list->head;
	    else
	        iter->next = list->tail;
	    iter->direction = direction;
	    return iter;
	}

创建一个list的迭代器，迭代的方向由direction决定，每次对迭代器调用 listNext() ，迭代器就返回列表的下一个节点

### set
### sorted set
### bit map
Bitmap是用String类型作为底层数据结构实现的一种统计二值状态的数据类型。String类型是会保存为二进制的字节数组，所以，Redis就把字节数组的每个bit位利用起来，用来表示一个元素的二值状态。你可以把Bitmap看作是一个bit数组
### GEO
底层是sorted set，举个例子，如果以Sorted Set来保存车辆的经纬度信息，member保存车辆id，score保存经纬度，保存结构如下图所示：
![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220130103025650-577432187.jpg)

然而，sorted set的权重只能保存一个浮点数，那么要怎么保存经纬度这两个值呢，这就要用到GeoHash编码

#### GeoHash编码
简单来说，就是先把经纬度分别编码，然后把编码后的经纬度再组合成一个最终编码

##### 经纬度编码过程
经度的区间范围是[-180, 180]，那么首先将这个区间二分，分成[-90,0]和[0,90]，最后GeoHash编码会把这个经度值编码成一个N位的二进制值（N可以自定义），为了方便描述，下文统称左右区间。

如果目标经度落在做区间，那么编码为0，落在右区间，则编码为1，直到二分了N次，最终得到一个N位的二进制值

纬度的编码方式也是一样，只不过范围是[-90,90]

举个例子，如果要把[116.37,39.86]进行GeoHash编码，将会得到[11010, 10111],其过程如下图所示：
![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220130105645709-469776864.jpg)

![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220130105706007-36281533.jpg)

##### 最终编码组合过程
上一步我们得到了经纬度的编码值，那么只要按照偶数位是经度的编码值，奇数位是纬度的编码值，偶数从0开始，奇数从1开始，最终便能组合出一个最终编码，按照上面的例子，最终编码便是1110011101，其组合过程如下图所示：
![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220130110552246-621769222.jpg)

#### 查找最近的人
在得到最终编码后，我们就可以通过查找最终编码为1110011101，得到一个carId的集合，然而只获取这个区间的话，会存在误差，如下图所示：
![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220130121953377-2119066486.jpg)

carId1的最终编码为1110011101，就只会返回carId2，然而carId3其实比carId2距离carId1更近，但是因为carId3最终编码并不是1110011101，所以没有返回。为了避免查询不准确问题，我们可以同时查询给定经纬度所在的方格周围的4个或8个方格。

## 单线程Redis为什么那么快
1、存内存操作

2、单线程

我们通常说，Redis是单线程，主要是指Redis的网络IO和键值对读写是由一个线程来完成的，这也是Redis对外提供键值存储服务的主要流程。但Redis的其他功能，比如持久化、异步删除、集群数据同步等，其实是由额外的线程执行的。正因为键值对读写是由一个线程来完成的，并不会发生多线程对资源共享的访问的并发问题（锁）

3、多路复用机制

Linux中的IO多路复用机制是指一个线程处理多个IO流，就是我们经常听到的select/epoll机制。

例如一个IO流称为一个fd，目前一共有1000个fd
### select
每次有一个fd收到一个事件回调的时候（被激活），内核就会遍历这个1000个fd，遍历到有被激活的fd后就进行处理
### poll
相比于select，优化了select限制的fd个数，poll的fd个数无限制
### epoll
优化了select，只遍历被激活的fd，不需要遍历所有的fd




## 基础数据类型
### string
### hash
源代码在/src/hict.h
    typedef struct dict {
		dictType *type;
		void *privdata;
		dictht ht[2];
		long rehashidx;
		usigned long iterators;
	}
### list
### set
### sorted set

## 持久化
redis实现持久化，主要是有两种方式，分别是AOF和RDB
### AOF
redis在执行了相关命令后，先将命令添加到缓冲区（aof_buf），然后再根据appendfsync配置的写回策略持久化到磁盘。
#### AOF3种写回策略
##### Always
同步写回，每个写命令执行完，立马同步地将日志写回磁盘。
##### Everysec
每秒写回，每个写命令执行完，只是先把日志写到AOF文件的内存缓冲区，每隔一秒把缓冲区中的内容写入磁盘。
##### No
No，操作系统控制的写回，每个写命令执行完，只是先把日志写到AOF文件的内存缓冲区，由操作系统决定何时将缓冲区内容写回磁盘。
#### AOF3种写回策略优劣比较
Always：不丢数据，但是每次操作都需要同步等待数据落盘，性能较差

Everysec：丢1s数据，命令写入缓冲区后就可以执行后续命令，每秒子线程异步落盘

No：数据落盘时机redis无法控制，只由操作系统决定，所以性能较好，但是00一丢数据的话，没落盘的数据都会全都丢失

总结如下如所示：
![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220105002838714-1579634408.jpg)

#### AOF文件重写机制
AOF文件越大，恢复的时候需要执行的命令就越多，所以redis提供了重写机制，把AOF文件变小。

AOF文件重写，简单来说就是在AOF文件日志中如果存在对同一个键值使用多条命令反复修改，就会合并成一条命令，比如说set a 1和set a 2这两条操作命令，AOF重写后就只会剩下set a 2这条命令。

再举个复杂点的例子，如下图所示：
![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220105004202233-1899823215.jpg)

当我们对一个列表先后做了6次修改操作后，列表的最后状态是[“D”, “C”, “N”]，此时，只用LPUSH u:list “N”, “C”, "D"这一条命令就能实现该数据的恢复，这就节省了五条命令的空间。对于被修改过成百上千次的键值对来说，重写能节省的空间当然就更大了。

#### AOF重写作用
- 降低磁盘占用量，提高磁盘利用率
- 提高持久化效率，降低持久化写时间，提高IO性能
- 降低数据恢复时间，提高数据恢复效率

#### AOF重写流程
和AOF日志由主线程写回不同，重写过程是由后台线程bgrewriteaof来完成的，这也是为了避免阻塞主线程，导致数据库性能下降。如下如所示：

![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220105233533510-25050876.jpg)

1、主进程fork子进程，执行bgrewriteaof，fork操作会把主线程的内存拷贝一份给bgrewriteaof子进程，这里面就包含了数据库的最新数据。然后，bgrewriteaof子进程就可以在不影响主线程的情况下，逐一把拷贝的数据写成操作，写入新的aof文件。

2、在重写的过程中，有新的命令需要执行，那么主进程会把命令分别写入aof缓存区和aof重写缓存区，在重写没结束时，旧的aof会继续记录数据，以防数据丢失。

3、aof重写完毕后，会从aof重写缓冲区获取重写过程中执行的命令，写入新的aof文件。

4、新的aof文件替换掉旧的aof文件，完成aof重写。

#### aof重写触发机制
##### 手动触发
	执行 bgrewriteaof 命令
##### 自动触发
自动重写触发条件配置

	auto-aof-rewrite-min-size size(如64MB) // aof文件达到该大小触发自动重写
	auto-aof-rewrite-percentage percent(如100) // aof文件大小达到该百分比触发自动重写

自动重写触发比对参数（运行指令info Persistence获取具体信息）

	aof_current_size // aof文件当前的大小
	aof_base_size // aof文件基础的大小

自动重写触发条件

	aof_current_size>auto-aof-rewrite-min-size

	(aof_current_size-aof_base_size)/aof_base_size>=auto-aof-rewrite-percentage

#### AOF文件追加阻塞
修改命令添加到aof_buf之后，如果配置是everysec，那么会有一个线程每秒执行fsync操作，调用write写入磁盘一次，但是如果此时来了很多Redis请求，Redis主线程持续高速向aof_buf写入命令，硬盘的负载可能会越来越大，IO资源消耗更快，fsync操作可能会超过1s，aof_buf缓冲区堆积的命令会越来越多，所以Redis的处理逻辑是会对比上次fsync成功的时间，如果超过2s，则主线程阻塞直到fsync同步完成，所以最多可能丢失2s的数据，而不是1s。（每当 AOF 追加阻塞事件发生时，在 info Persistence 统计中，aof_delayed_fsync 指标会累加，查看这个指标方便定位 AOF 阻塞问题。）

### RDB
RDB持久化指的是在满足一定的触发条件时（在一个的时间间隔内执行修改命令达到一定的数量，或者手动执行SAVE和BGSAVE命令），对这个时间点的数据库所有键值对信息生成一个压缩文件dump.rdb，然后将旧的删除，进行替换。（在Redis默认的配置下，RDB是开启的，AOF持久化是关闭的）

#### RDB持久化过程
主进程会fork一个子进程，然后对键值对进行遍历，生成rdb文件，在生成过程中，父进程会继续处理客户端发送的请求，当父进程要对数据进行修改时，会对相关的内存页拷贝出一份副本后再进行修改（写时复制技术），而rdb子进程则使用拷贝出来的副本继续进行持久化过程。如下如所示：
![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220107000811936-705862793.jpg)

如果主线程对这些数据也都是读操作（例如图中的键值对A），那么，主线程和bgsave子进程相互不影响。但是，如果主线程要修改一块数据（例如图中的键值对C），那么，这块数据就会被复制一份，生成该数据的副本。然后，bgsave子进程会把这个副本数据写入RDB文件，而在这个过程中，主线程仍然可以直接修改原来的数据。

#### 查看最近一次fork的耗时
使用INFO命令查看Redis的latest_fork_usec指标值（单位微妙）

### RDB-AOF混合持久化
使用RDB持久化的话，数据丢失程度取决于每次生成快照的间隔时间，如10s，如果服务器在下一次持久化成功之前宕机，那么最多可能会丢失10s数据，而使用AOF持久化的话，当服务器宕机后恢复数据时，恢复的时间又会过长。所以就出现了RDB-AOF混合持久化（4.0版本后才有）。

RDB-AOF混合持久化简单来说就是内存快照以一定的频率执行，在两次快照之间，使用AOF日志记录这期间的所有命令操作。如下图所示：
![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220107002743808-759180547.jpg)

T1和T2时刻的修改，用AOF日志记录，等到第二次做全量快照时，就可以清空AOF日志，因为此时的修改都已经记录到快照中了，恢复时就不再用日志了。

这个方法既能享受到RDB文件快速恢复的好处，又能享受到AOF只记录操作命令的简单优势，唯一的缺点就是不能兼容4.0之前的版本，aof里面的rdb部分在4.0版本之前是无法识别的。



## 事务

## 主从
### 全量复制
![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220112001047891-2093210932.jpg)
#### 第一阶段 建立连接
从库使用slave of 命令，向主库申请建立连接，主库响应后主从建立一个socket通道。
#### 第二阶段 数据同步
从库发送psync ? -1信号向主库请求数据同步（psync runid offset），其中

- runid：每个Redis实例启动时都会自动生成的一个随机ID，用来唯一标记这个实例。当从库和主库第一次复制时，因为不知道主库的runID，所以将runID设为“?”。
- offset 复制偏移量，因为是第一次复制，所以值为-1

主库接收到? -1，知道从库是新连接的从库，开始全量同步

主库收到psync命令后，会用FULLRESYNC响应命令带上两个参数：主库runID和主库目前的复制进度offset，返回给从库。从库收到响应后，会记录下这两个参数。之后，主库会把当前所有的数据都复制给从库。（FULLRESYNC响应表示第一次复制采用的全量复制），流程如下：

1、使用bgsave导出rdb文件，而之后客户端请求的修改命令(set和select)，记录到复制缓冲区（replication_buffer）

2、通过socket把rdb文件传递给从库，同时把主库的runid和数据的偏移量（offset）发送给从库（从库会保存runid和offset，数据传播阶段需要用到）

3、当主库完成RDB文件发送后，就会把此时replication buffer中的修改操作发给从库，从库再重新执行这些操作

#### 第三阶段 数据传播
从库从全量复制后获取到了主库的runID和offset，之后就可以开始发送psync {runid} {offset}指令进行后续的增量复制了。
### 增量复制
增量复制和全量复制不同，就是根基偏移量去获取主库的数据，其流程如下：

从复制积压缓存区（replication_backlog_buffer）获取全量同步的时候写入的数据（aof格式的），并通过socket传输进行部分同步（从库进行bgwriteaof）

- 如果主库自身记录的runid和传递过来的runid不匹配，传递过来的offset没有记录在复制积压缓存区，则进行全量同步
- 如果符合runid匹配，offset有记录在复制积压缓存区，则判断从库传递过来的offset和主库记录的offset对比
- 相等，不需要同步，不相等，则根据偏移量从复制积压缓存区获取数据进行部分同步，数据区间为[从库传递的offset,主库记录的offset]（主库发送 +CONTINUE {当前主库记录的offset} 信号）
- 从库在收到+CONTINUE信号后，用主库传过来的offset更新自身记录的offset，并进行bgwriteaof

### 在增量复制过程中，掉线了怎么办？
从库每秒发送replconf ACK offset到主库请求数据同步（主库也会每10s去ping一下从库，确认从库是否掉线）

- 如果从库记录的offset不存在于复制积压缓存区，那么则重新进行全量同步
- 如果从库记录的offset存在于复制积压缓存区，那么则开始部分同步（从库进行bgwriteaof）

### replication_buffer和replication_backlog_buffer的区别
#### replication_buffer
replication_buffer用于全量复制，在生成和传输rdb文件的过程中，用于记录新生成的数据，但如果主库传输 RDB 文件以及从库加载 RDB 文件耗时长，同时主库接收的写命令操作较多，就会导致复制缓冲区被写满而溢出。一旦溢出，主库就会关闭和从库的网络连接，重新开始全量同步。
#### replication_backlog_buffer
repl_backlog_buffer用于增量复制，是一个环形缓冲区，主库会记录自己写到的位置，从库则会记录自己已经读到的位置，如下图所示：
![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220115160421181-1786076460.jpg)

因为repl_backlog_buffer是一个环形缓冲区，所以如果主库写压力过大，从库读取速度过慢，会导致从库未读的数据被新写入的操作覆盖，导致重新触发全量复制。为了避免这种情况，可以调大repl_backlog_size的值。
https://blog.csdn.net/succing/article/details/121230604


## 哨兵
Redis中的哨兵服务器是一个运行在哨兵模式下的Redis服务器，核心功能是监测主节点和从节点的运行情况，在主节点出现故障后，完成自动故障转移，让某个从节点升级为主节点。简单来说，哨兵主要负责的就是三个任务：监控、选主（选择主库）和通知。如下图所示：
![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220120001343939-1125403029.jpg)

### 监控
哨兵会周期性的给redis（主库和从库）发送ping命令，如果从库没有在规定时间内响应，会被标记为下线状态(主观下线和客观下线)，主库也一样，不过如果主库被标记了的话会多一步操作，就是选主操作。

#### 主观下线
哨兵发送ping命令，redis服务器没有在规定时间内响应，会被标记为主观下线，但是这样会存在误判，比如说哨兵和redis间的网络发生了问题导致响应超时，而不是redis真的挂掉了，于是为了解决这个问题（如果是主库发生这种问题，会触发主从切换导致额外的开销），就出现了客观下线。

#### 客观下线
客观下线就是引入了一个哨兵集群，在某个哨兵标记了某台redis服务器主观下线后，通知别的哨兵服务器也去ping一下，如果哨兵判断为该台redis主观下线的总数超过一半(N/2+1,N为哨兵数量,该值可由quorum配置)，就会标记为客观下线，说明这台redis服务器确实挂了，反之就是哨兵误判了，redis还在正常运行，如下图所示：
![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220120003036896-799640079.jpg)

### 选主
主库down掉后，会从从库中按照一定的规则选出新的主库，这样这个集群中，就会出现新的主库，出现新的主库后，哨兵会执行通知操作。

#### 选主流程
简单总结的话，两次筛选，三个优先规则，如下图所示：
![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220123113839050-653921243.jpg)
##### 两次筛选
- 从库当前在线状态

	如果当前从库已经下线了，那么当然是不能选择的

- 从库网络连接状态

	如果哨兵在ping完后，在down-after-milliseconds时间内（down-after-milliseconds哨兵配置）没有收到redis回复，就认为从库网络连接状态不良
##### 三个优先规则
- 从库优先级

	通过slave-priority配置项，可以给从库设置不同的优先级

- 从库复制进度

	根据从库的复制进度，进度越快，越优先选择（从库的slave_repl_offset，用复制偏移量判断，偏移量越大，就说明复制进度越快）

- 从库ID号

	从库ID越小，优先级越高（每次redis实例启动时，都会生成一个id号，id号越小说明redis持续运行时间越长）

### 通知
哨兵会把新的主库信息发送给从库，让它们执行replicaof命令，和新主库建立连接，并进行数据复制。同时，哨兵会把新主库的连接信息通知给客户端，让它们把请求操作发到新主库上。

## 哨兵集群
在部署哨兵的过程中，就只需要配置主库的ip和端口，并不需要配合其他哨兵的连接信息，因为这些哨兵实例是通过redis的pub/sub机制组成集群的

### 基于pub/sub机制的哨兵集群组成
新哨兵实例和主库建立连接之后，会把自己的ip和端口发布到redis主库的```__sentinel__:hello```频道上,其他哨兵实例订阅了该频道的话，就能够获取到新哨兵实例的ip和端口，如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220124103248434-2097783300.jpg)

### 哨兵获取从库的配置信息
哨兵在与主库建立连接之后，会想主库发送INFO命令，从而获取从库的配置信息，获取到从库的ip和port后，就能和从库建立连接，进行监控。如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220124104504577-334656037.jpg)

### 选举哨兵集群领导者
在redis发生主从切换的时候，哨兵集群需要推选出一个领导者，进行redis主从切换的操作，选举使用的是raft算法。

简单来说，就是某个哨兵实例在想在哨兵集群中成为领导者，就需要在选举过程中，拿到的票数大于哨兵实例的一半（如果4个哨兵实例，那么就是要>=3），以3个哨兵为例，具体的选举过程如下图所示：
![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220124110043258-488399873.jpg)

注意，如果在选举过程中，没有一个哨兵实例获取到的票数大于quorum，哨兵集群会等待一段时间（也就是哨兵故障转移超时时间的2倍），再重新选举，知道选出领导者。

## redis数据扩展
为了保存大量数据，我们使用了大内存云主机和切片集群两种方法。实际上，这两种方法分别对应着Redis应对数据量增多的两种方案：纵向扩展（scale up）和横向扩展（scale out）。
### 纵向扩展
升级单个Redis实例的资源配置，包括增加内存容量、增加磁盘容量、使用更高配置的CPU。

####优点
实施起来简单直接

####缺点
- 数据量增加，需要的内存也会增加，当使用RDB对数据进行持久化时，主线程fork子进程时就可能会阻塞
- 受到硬件和成本的限制，比如要扩充到1TB，就会面临硬件容量和成本上的限制

### 横向扩展
横向增加当前Redis实例的个数（分片集群）

####优点
不用担心单个实例的硬件和成本限制，理论上只要redis实例足够多，能做到无限扩展

####缺点
比较麻烦，存在多个实例的分布式管理问题

## 分片集群
分片集群就是指启动多个Redis实例组成一个集群，然后按照一定的规则，把收到的数据划分成多份，每一份用一个实例来保存。（一种保存大量数据的通用机制）

### Redis Cluster
redis官方提供的，一种实现分片集群的方案

#### Redis Cluster数据切片和实例的对应分布关系
Redis Cluster方案采用哈希槽（Hash Slot，下文简称为Slot）来处理数据和实例之间的映射关系。

在Redis Cluster方案中，一个切片集群共有16384个哈希槽，这些哈希槽类似于数据分区，每个键值对都会根据它的key，被映射到一个哈希槽中，其流程如下：

- 根据键值对的key，按照CRC16算法计算一个16 bit的值
- 用这个16bit值对16384取模，得到0~16383范围内的模数，每个模数代表一个相应编号的哈希槽

而在部署Redis Cluster方案时，可以使用cluster create命令创建集群。此时，Redis会自动把这些槽平均分布在集群实例上。例如，如果集群中有N个实例，那么，每个实例上的槽个数为16384/N个。

#### 客户端定位数据
当redis实例与其他redis实例相互连接形成集群的时候，它们会把自己的Solt信息发送给与它相连接的其他实例，来完成哈希槽分配信息的扩散。当实例之间相互连接后，每个实例就有所有哈希槽的映射关系了。

客户端收到哈希槽信息后，会把哈希槽信息缓存在本地。当客户端请求键值对时，会先计算键所对应的哈希槽，然后就可以给相应的实例发送请求了。

但是，redis实例和Slot的映射关系并不是固定的，而是有可能会发生变化的（比如集群新加了新的redis实例，就需要重新分配Slot），所以当客户端请求某个key的时候，一共有3种情况：

- 这个Solt存在在redis实例

	那么就直接返回这个键值对给客户端

- 这个Solt已经迁移到其他redis实例（Moved指令）

	说明客户端的本地存储的映射关系已经过期了，那么该redis实例会给客户端返回Moved指令及正确的节点信息，纠正客户端缓存的错误槽位信息。

		GET hello:key
		(error) MOVED 13320 172.16.19.5:6379
	

	客户端收到后会更新本地的槽位关系表，然后向正确的节点发送查询指令,其流程如下图所示：
	
	![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220124123917186-533387738.jpg)

- 这个槽位正在迁移中(ASK指令)

	客户端访问的Solt正在迁移中(槽位状态为IMPORTING)，如果这个key还没迁移到新实例，那么就直接返回，如果已经迁移到新的实例的话，就会发送ASK指令及正确的节点信息。

		GET hello:key
		(error) ASK 13320 172.16.19.5:6379

	客户端收到之后不会更新本地的槽位关系表，只会向新实例发送ASKING命令，然后再发送操作命令（因为还有部分key还在迁移中，如果客户端更新了映射关系的话，还没迁移的key在新的实例中并不存在，所以不能更新），其流程如下图所示：
	
	![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220124123934216-1942106552.jpg)

#### Redis Cluster的故障转移和发现
与redis的哨兵的流程有点像，但是有区别

##### 主观下线
当节点1向节点2例行发送Ping消息的时候，如果节点2正常工作就会返回Pong消息，同时会记录节点1的相关信息，更新与节点2的最近通讯时间。如果节点1的定时任务检测到与节点2上次通讯的时间超过了cluster-node-timeout 的时候就会更新本地节点状态，把节点2更新为主观下线。

##### 客观下线
由于 Redis Cluster 的节点不断地与集群内的节点进行通讯，下线信息也会通过 Gossip 消息传遍所有节点。

因此集群内的节点会不断收到下线报告，当半数以上持有槽的主节点标记了某个节点是主观下线时，便会认为节点2客观下线，执行后面的流程。

##### 从节点资格检查（相当于上文中哨兵的两次筛选）
每个从节点都会检查与主节点断开的时间。如果这个时间超过了 cluster-node-timeout*cluster-slave-validity-factor（从节点有效因子，默认为 10），那么就没有故障转移的资格。也就是说这个从节点和主节点断开的太久了，很久没有同步主节点的数据了，不适合成为新的主节点，因为成为新的主节点以后，其他的从节点回同步它的数据。

##### 从节点触发选举（相当于上文中哨兵的三个优先规则）
通过资格的从节点都可以触发选举。但是触发选举是有先后顺序的，这里按照复制偏移量的大小来判断。

复制偏移量越大说明从节点延迟越低，也就是该从节点和主节点沟通更加频繁，该从节点上面的数据也会更新一些，因此复制偏移量大的从节点会率先发起选举。

##### 从节点发起选举
首先每个主节点会去更新配置纪元（clusterNode.configEpoch），这个值是不断增加的整数（相当于raft协议的选举版本号）

每个主节点自身维护一个配置纪元 (clusterNode.configEpoch)标示当前主节点的版本，所有主节点的配置纪元 都不相等，从节点会复制主节点的配置纪元。

整个集群又维护一个全局的配置纪元(clusterState.current Epoch)，用于记录集群内所有主节点配置纪元的最大版本。

在节点进行 Ping/Pong 消息交互时也会更新这个值，它们都会将最大的值更新到自己的配置纪元中。

这个值记录了每个节点的版本和整个集群的版本。每当发生重要事情的时候，例如：出现新节点，从节点精选。都会增加全局的配置纪元并且赋给相关的主节点，用来记录这个事件。

更新完配置纪元以后，每个从节点会向集群内发起广播选举的消息。

##### 主节点为选举投票
参与投票的只有主节点，从节点没有投票权。每个主节点在收到从节点请求投票的信息后，如果它还没有为其他从节点投票，那么就会把票投给从节点。(也就是主节点的票只会投给第一个请求它选票的从节点。)

超过半数的主节点通过某一个节点成为新的主节点时投票完成。如下图所示：
![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220124133421801-2081309256.jpg)

如果在 cluster-node-timeout*2 的时间内从节点没有获得足够数量的票数，本次选举作废，进行第二轮选举。

##### 选举完成
当从节点收集到足够的选票之后，触发替换主节点操作:
- 当前从节点取消复制变为主节点
- 执行 clusterDelSlot 操作撤销故障主节点负责的槽，并执行```clusterAddSlot```把这些槽添加到自己的节点上。
- 向集群广播自己的 pong 消息，通知集群内所有的节点当前从节点变为主节点并接管了故障主节点的槽信息。

#### Redis Cluster一致性
CAP理论认为C一致性，A可用性，P分区容错性，一般最多只能满足两个，也就是只能满足CA和CP，而Redis Cluster的主从复制的模式是异步复制的模式，也就是主节点执行修改命令后，返回结果给客户端后，有一个异步线程会一直从aof_buf缓冲区里面取命令发送给从节点，所以不是一种强一致性，只满足CAP理论中的CA。









## redis使用技巧
### 统计每天的新增用户数和第二天的留存用户数
使用set

user:login 记录所有登录过的用户id

user:login:20220126 记录2022-01-26登录过的用户id

每天结束后，user:login和user:login:20220126取并集，保存到user:login中

user:login和user:login:20220126的差集，就是新增用户

user:login和user:login:20220126的交集，就是留存用户

注意：Set的差集、并集和交集的计算复杂度较高，在数据量较大的情况下，如果直接执行这些计算，会导致Redis实例阻塞。所以，这里有个建议：你可以从主从集群中选择一个从库，让它专门负责聚合计算，或者是把数据读取到客户端，在客户端来完成聚合统计，这样就可以规避阻塞主库实例和其他从库实例的风险了。

### 统计评论列表中的最新评论
使用sorted set

如果数据需要有序，那么可以考虑list和sorted set

- list
	按照位置排序，获取数据的时候用下标0~N来获取

- sorted set
	按照权重排序，获取数据的时候用权重值的范围[minWeight, maxWeight]来获取

如果只是获取最新评论，用list或者sorted set都可以，但是评论一般会涉及到分页，假设现有评论{A,B,C,D,E,F}，每页3条记录，在获取第二页的时候，新加入了一条评论，到时评论数据变成了{G,A,B,C,D,E,F}，那么list获取到第二页的数据是{C,D,E}而不是{D,E,F}

### 用户签到
bitmap

- 统计用户2022-01的签到次数

	user:sign:用户id:202201(1签到0未签到)
	
	```BITCOUNT user:sign:用户id:202201```

- 检查该用户2022-01-26是否签到

	user:sign:用户id:202201(1签到0未签到)

	```GETBIT user:sign:用户id:202201 25```

- 检查用户2022-01连续10天签到情况（假如有一亿个用户）

	user:sign:20220101(1签到0未签到，有1亿位，每一位代表一个用户【位数:用户id%1亿】)，2022-01-01这1亿用户的签到情况

	```BITOP AND retBitMap user:sign:20220101 user:sign:20220102 ... user:sign:20220110 //retBitMap 就是这10个bitmap 与运算得出的结果生成的新bitmap```

	最后算一下这个retBitMap有多少个1就说明有多少个用户连续签到10天了

#### 补充一些位运算的知识点
- 与&

	运算规则：0&0=0;   0&1=0;    1&0=0;     1&1=1;
	两位同时为“1”，结果才为“1”，否则为0
- 或|

	运算规则：0|0=0；   0|1=1；   1|0=1；    1|1=1；
	参加运算的两个对象只要有一个为1，其值为1
- 异或^

	运算规则：0^0=0；   0^1=1；   1^0=1；   1^1=0；
	参加运算的两个对象，如果两个相应位为“异”（值不同），则该位结果为1，否则为0

### 统计独立访客（Unique Visitor，UV）量
可以使用set或者hash，但是如果用户量比较多，用set和hash来统计的话占用内存比较多，如果不需要很精准的数据的话，可以使用HyperLogLog（HyperLogLog的统计规则是基于概率完成的，所以它给出的统计结果是有一定误差的，标准误算率是0.81%）

```PFADD page:uv userId1 userId2 userId3 userId4 userId5 // 加入```

```PFCOUNT page:uv // 统计计算```

### 附近的人
geo

在使用GEO类型时，经常会用到两个命令，分别是GEOADD和GEORADIUS。使用这两个命令就可以获取附近的人。

假如用户id是101，用户所在经纬度为116.034579,39.030452，那么使用以下命令就可以把用户的经纬度信息保存在geo集合中。

```GEOADD user:locations 116.034579 39.030452 101 ```

那么用户想要获取当前位置(116.034579,39.030452)附近的人的信息的话，只要执行下面的命令即可

```GEORADIUS user:locations 116.054579 39.030452 5 km ASC COUNT 10```

其中：

- 5 距离
- km 距离单位
- ASC 距离当前位置的中心位置从近到远的方式来排序
- COUNT 10 返回10个数据




