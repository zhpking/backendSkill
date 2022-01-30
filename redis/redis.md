# redis知识点汇总
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

###hash冲突
hash冲突指的是根据key值所计算出来的hash值相同，导致两个元素落到同一个hash桶里。

redis解决哈希冲突的方式，就是用链式法。链式法也很容易理解，就是指同一个哈希桶中的多个元素用一个链表来保存，它们之间依次用指针连接。如下图所示：
![](https://img2020.cnblogs.com/blog/901559/202201/901559-20220101160047355-1184225591.jpg)

entry1、entry2和entry3都需要保存在哈希桶3中，导致了哈希冲突。此时，entry1元素会通过一个*next指针指向entry2，同样，entry2也会通过*next指针指向entry3。这样一来，即使哈希桶3中的元素有100个，我们也可以通过entry元素中的指针，把它们连起来。这就形成了一个链表，也叫作哈希冲突链。

###rehash
同一个哈希桶中，元素越多，查询效率越慢（链表查询时间复杂度为O(n)）,为了解决这个问题，采用的方法就是增加哈希桶的数量，而这个就是rehash操作。

redis默认使用两个全局hash表（下文称作hash表1和hash表2），一个作为插入和查询数据使用，另一个就是用来在rehash过程中迁移数据使用，具体步骤如下：
1、hash表2扩充哈希桶（比如hash表2扩充成hash表2的两倍）
2、重新新建hash表1元素在hash表2的hash值，然后迁移到hash表2
3、切换hash表，hash表2作为数据的查询和插入，释放hash表1的空间并作为下次rehash操作的数据迁移表

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
- type:值的类型，表示redis五大基本类型(string, hash, list, set, sorted set)
- encoding:值的编码方式,表示值对应使用的底层结构（sds，压缩列表，跳表等）
- lru：lru缓存淘汰机制信息（记录了这个对象最后一次被访问的时间，用于淘汰过期的键值对）
- refcount:引用计数器,记录了对象的引用计数
- *ptr:是指向数据的指针（指向实际存储的数据指针）

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

- buf：字节数组，保存实际数据。为了表示字节数组的结束，Redis会自动在数组最后加一个“\0”，这就会额外占用1个字节的开销。
- len：占4个字节，表示buf的已用长度。
- alloc：也占个4字节，表示buf的实际分配长度，一般大于len。

#### 字符串编码方式
一共3中编码，如下图所示：
![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220124164741213-916938951.jpg)

##### int编码
当value是一个整数，并且可以使用long类型（8字节）来表示时，那么会属于int编码，redisObj的*ptr直接存储数值。（并且Redis会进行优化，启动时创建0~9999的字符串对象作为共享变量。）

##### embstr编码
字符串小于等于44字节时，RedisObject中的元数据、指针和SDS是一块连续的内存区域，这样就可以避免内存碎片。

##### raw编码
字符串大于44字节时，SDS的数据量就开始变多了，Redis就不再把SDS和RedisObject布局在一起了，而是会给SDS分配独立的空间，并用指针指向SDS结构。

### 整数数组
在一片连续的内存中，存储内存空间相同大小的元素，查询时间复杂度为O(n)
### 双向链表
用前后指针，分别指向当前元素的上一个元素和下一个元素内存地址，查询时间复杂度为O(n)
### 哈希表
参照全局hash表描述，hashtable结构体如下：

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
相比于数组每个元素占用的内存空间大小必须一致，压缩列表就是可以存放不同内存空间大小的元素（相比于数组节省了空间），如下图所示

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

## redis数据类型
### 字符串
### hash
hash数据类型使用的底层数据结构是ziplist和hashtable

当hash表存储的元素个数小于512，元素小于64字节，hash表会使用ziplist作为的底层实现，否则则会使用hashtable。

#### 相关参数配置
- hash-max-ziplist-entries（表示用压缩列表保存时哈希集合中的最大元素个数）
- hash-max-ziplist-value（表示用压缩列表保存时哈希集合中单个元素的最大长度）

### list
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

```GEOADD user:locations 116.034579 39.030452 33 ```

那么用户想要获取当前位置(116.034579,39.030452)附近的人的信息的话，只要执行下面的命令即可

```GEORADIUS user:locations 116.054579 39.030452 5 km ASC COUNT 10```

其中：

- 5 距离
- km 距离单位
- ASC 距离当前位置的中心位置从近到远的方式来排序
- COUNT 10 返回10个数据




