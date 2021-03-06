# mysql知识点汇总

## mysql基本架构
mysql主要分为server层和存储引擎层。而server层包括了连接器、查询缓存、分析器、优化器、执行器等。

server层包括了mysql大多数的核心功能，以及所有的内置函数(count,sum等)，所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。

存储引擎层则是负责数据的存储以及读取，支持 InnoDB、MyISAM、Memory 等多个存储引擎（架构模式是插件式的，所以mysql也可以自定义存储引擎来使用）。mysql5.5.5之前默认存储引擎为MyISAM，mysql5.5.5之后则是InnoDB。

其基本架构示意图如下所示：
![](https://img2022.cnblogs.com/blog/901559/202201/901559-20220130171301031-2030452943.jpg)

### server层
#### 连接器
连接器负责跟客户端建立连接、获取权限、维持和管理连接，也就是你用mysql客户端连接工具执行```mysql -u xxx -p xxx```命令。

连接器会校验你的用户名密码是否正确，成功建立连接后连接器会在权限表查询该用户所拥有的权限，之后执行的命令都需要根据此时读取到的权限来校验是否有权执行命令。（在修改用户权限之后，用户需要重新登录权限才能生效的原因就是在建立连接之后读取了账号权限，在断开连接之前都会用该权限来校验）

注意，成功建立连接后（长连接），mysql在执行过程中所申请的内存空间都是由连接对象管理的，这些资源只有在连接对象断开之后才会释放，所以如果连接长期不释放的话（排序、变量这些会占用内存），很容易就会造成内存溢出（客户端如果太长时间没动静，连接器就会自动将它断开。这个时间是由参数 wait_timeout 控制的，默认值是 8 小时）。

长连接不提前释放内存的原因如果这个链接是要复用的，提前释放反而影响性能（因为要重新申请内存），MySQL没法自己决定，所以5.7之后提供了方法让客户端有命令```mysql_reset_connection()```来做

#### 查询缓存
建立连接后，你就可以执行mysql语句了，而在执行```select```语句的时候，会先去查询缓存里找，之前执行过的语句会在查询缓存里以key-value的形式保存起来（key是查询语句，value是查询结果），如果能知道对应的key，则直接返回查询结果，就不需要进行后续查询步骤了（如果命中查询缓存，会在查询缓存返回结果的时候做权限验证，没有权限就不会返回结果）。

不过查询缓存的命中率一般很低，特别是对应更新比较频繁的表，毕竟你一条update语句更新了表的数据，那么对应的缓存数据就全清空了，所以一般不建议使用查询缓存。正因如此，mysql8.0后直接就把查询缓存的整块功能给删掉了（8.0之前的版本只要将参数query_cache_type设置成0就可以不使用查询缓存）

##### query_cache_type
既然提到了这个参数，这里也解释一下：

- 0 关闭查询缓存
- 1 打开查询缓存
- 2 只有select 中明确指定SQL_CACHE才缓存，如：

	```select SQL_CACHE * from test where id = 1```

#### 分析器
如果查询缓存没有命中或者被关闭，那么就会执行sql语句，此时分析器会经过词法分析和语法分析这两个步骤进行处理(图中分析器->查询缓存的意思是，更新完数据后需要失效查询缓存【如果有】)。

##### 词法分析
词法分析就是识别你的sql里面的字符串分表代表什么，比如以下这条语句：

```select * from test where id = 1```

mysql会根据```select```关键字分析出这条语句是查询语句，```from```关键字后面跟着的是表名，```where```关键字后面跟着的就是查询条件。

##### 语法分析
做完词法分析后，mysql就会根据语法规则，检查你这条语句是否有满足sql的语法，表是否存在，列是否存在等，不满足则返回错误提示，如：

```MySQL server version for the right syntax to use near 'xxxx'``` 或者 ```Unknown column 'xxx'```

主要关注```use near```和```Unknown column```，一般这个后面就是提示mysql语句发生错误出现的第一个位置

#### 优化器
经过了分析器之后，检查了sql语句没有语法错误了，在执行之前，还需要优化器对sql进行优化。比如说：

```select * from test where aid = 1 and bid > 2```

test表里分别对aid和bid建立了索引，优化器就要对该条语句选择使用哪个索引。

又比如说:

```select * from test1 as t1 left join test2 as t2 on t1.id = t2.id where t1.aid = 10 and t2.bid = 20```

mysql优化器需要选择这个join语句哪个表作为驱动表(explain结果中，第一行出现的表就是驱动表)，根据驱动表的不同，执行效率会不同。

优化器是一个十分复杂的东西，后文会根据不同的例子，详细的说明优化器的内容，以及结合自身的经验如何优化sql语句。

#### 执行器
经过了语法校验和优化之后，就真正的开始执行sql语句了，在开始执行之前，会先校验权限，看登录用户是否有权执行这条sql（连接器已经取出用户的权限了，在执行器的时候才用来校验权限，既然连接器已经取出了权限，为啥不在优化器之前校验，原因是有些时候，SQL语句要操作的表不只是SQL字面上那些。比如如果有个触发器，得在执行器阶段（过程中）才能确定。优化器阶段前是检测不到的）

以```select * from test where id = 10```为例，如果id上没有索引，那么其执行流程如下所示：

1. 调用InnoDB引擎接口取这个表的第一行，判断id值是不是10，如果不是则跳过，如果是则将这行存在结果集中
2. 调用引擎接口取“下一行”，重复相同的判断逻辑，直到取到这个表的最后一行
3. 执行器将上述遍历过程中所有满足条件的行组成的记录集作为结果集返回给客户端

如果有索引，则是：

1. 调用InnoDB引擎接口取这个表满足条件的第一行
2. 调用引擎接口取“下一行”，直到下一行的id不为10则停止继续取出
3. 执行器将上述遍历过程中所有满足条件的行组成的记录集作为结果集返回给客户端

### 存储引擎层
在经过server层连接器，分析器，优化器，执行器等一系列过程之后，最后到达的就是存储引擎层，而存储引擎层主要是负责数据的读写，这里主要介绍的是innodb存储引擎。读就比较简单，找到数据直接返回就可以了，但写话，我们平时执行的update，insert语句，如何保证能成功写入呢？

#### innodb存储引擎的写入
虽说我们更新或者插入数据仅仅只是用一条sql就可以搞定了，看似简单，但是其流程却是经历一系列步骤，其详细流程如下图所示：
![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220201162649518-315247849.png)

##### undo log
innodb每次对数据进行写入或者更新的时候，都会生成一个事务id（递增），根据执行的语句生成一个回滚数据的语句（如：insert语句就会生成delete语句，update语句就会生成重新更新回旧数据的update语句），将这个语句和事务id写入undo log，会得到一个回滚指针，然后更新对应数据行（比如id=2的这行数据）事务id和回滚指针，当事务需要回滚的时候，就可以根据事务回滚指针找到对应的回滚sql，进行旧数据的恢复（回滚操作）。除此之外，undo log 也通过事务id来判断事务的可见性，详情将会在事务隔离详细介绍

##### change buffer
change buffer 是用来优化二级且非唯一索引，且对应的数据页没在内存（buffer pool）中的数据插入和更新上的（简单来说就是优化为批量操作，而且对update操作来说还可以减轻从磁盘加载数据页的性能损耗）。

因为每次插入或者更新，假如对应的数据页不存在内存中，都需要从磁盘中把对应的数据页读取出来，然后再进行数据页的修改操作。

有了change buffer之后，假如遇到数据页不在内存中，且不影响数据的唯一性的情况下，那么可以先把数据写入到change buffer中，等对应的数据页被访问从而加载到内存中的时候，再把change buffer中对应的新数据修改到内存页中（称为merge操作），亦或者每隔一段时间，由后台线程定期的触发merge操作，这样就可以把若干对同一页面的更新缓存起来做，合并为一次性更新操作。（减少IO，转随机IO为顺序IO,这样可以避免随机IO带来性能损耗，提高数据库的写性能）

- 为什么数据页在内存中，不使用change buffer

	change buffer 其中一点优化就是为了不要每次更新都把数据从磁盘加载到内存中（对应的数据页不存在内存的情况下），既然数据页已经在内存中了，那么直接改就行了。

- 为什么数据唯一的时候，不使用change buffer

	因为每次唯一索引数据更新或者插入的时候，都需要判断数据的唯一性，不存在这个数据才能更新或者插入，在校验唯一性的过程中，不管怎样都需要把对应的数据页加载到内存中，既然数据页已经在内存中了，那么直接改就行了。

如果数据库down掉的话，change buffer丢失，会导致数据丢失吗？

不会，因为语句写入change buffer后，接下来的流程（图中第9,10步）会写入bin log和redo log，而且事务提交，bin log和redo log写成功后，事务才会执行成功，所以就算change buffer丢了，在恢复数据的时候，redo log也是已经是更新完的数据。

###### 触发merge操作时机

- 访问对应的数据页

- 后台线程定期merge

- 数据库正常关闭（shutdown）

###### change buffer大小设置

innodb_change_buffer_max_size参数，设置为50就是change buffer的大小最多只能占用buffer pool的50%


##### redo log和binlog

redo log和bin log 是mysql两个十分重要的日志，其中redo log是innodb特有的，用来实现crash-safe，而binlog是server层特有的日志，所以无论哪个存储引擎都能够使用。

###### redo log

如果数据每次在每次更新或者插入的时候，都直接写磁盘持久化的话，那么这样会带来io的性能损耗，为了优化这点，innodb使用了一种WAL技术（Write-Ahead Logging），也就是在数据落盘之前，innodb会先写日志，再写磁盘。

具体来说，就是更新和插入的数据，innodb只要把buffer pool的数据更新了，写完了redo log，就认为数据的更新已经完成了（当然还要写bin log，但是这是server层做的事）。同时，innodb一般会在系统比较空闲的时候，再把redo log的操作记录批量更新到磁盘里，这样做的好处就是减少了io，把随机写改成了顺序写。

除此之外，redo log还能保证数据库发生异常重启，之前提交的记录都不会丢失，因为只要写到redo log后，发生了异常重启只要把重新把磁盘中redo log的数据重新加载回内存即可（crash-safe）。

当然，redo log的内存不是无限大的，而是固定大小（参数innodb_log_file_size【每个redo log文件大小】 innodb_mirrored_log_groups【有多少个redo log文件，但5.7该参数就被废弃了】），而且是个环状结构，其结构如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220202230337968-1594751133.png)

- write pos：当前记录写到的位置
 
- check point：开始回收内存的位置

所以write pos和check point就是redo log内存所剩空间，如果write pos追上了check point，那么innodb就无法继续执行数据更新，只能先把redo log中的数据持久化落盘，回收内存，推进check point位置空出剩余内存后，才能继续执行更新操作。

###### redo log的写入机制

先写redo log buffer，事务提交的时候再持久化到磁盘中，其写入流程如下图所示

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220328232945783-697511674.png)

redo log buffer(mysql程序的内存，所有线程共享一个) -> page cache(操作系统里文件系统的页缓存) -> disk(磁盘)


redo log的写入机制，主要是由 ```innodb_flush_log_at_trx_commit``` 参数控制

- 0
	每次事务提交时都只是把redo log留在redo log buffer中
	
- 1
	每次事务提交时都将redo log直接持久化到磁盘
	
- 2
	每次事务提交时都只是把redo log写到page cache（innodb有一个后台线程，会每秒调用write把redo log buffer写到page cache，再调用fsync持久化到磁盘）

innodb 有一个后台线程，每隔1s就会把redo log buffer中的日志，调用write写入到page cache，然后fsync到磁盘

注意，redo log在事务提交的时候，其实是分为两个步骤的，先prepare再commit，所以 ```innodb_flush_log_at_trx_commit = 1```的时候，其实指的是redo log在prepare阶段的时候，需要持久化一次，而commit阶段，其实就只是写到page cache里

因为服务器崩溃恢复的时候，只要有prepare阶段的redo log和bin log，就足够恢复数据了，因此redo log在commit阶段就不需要fsync，只需要写入page cache，然后等待每秒一次后台轮询刷盘fsync就可以了

除了后台线程每秒1次轮询刷盘外，还有另外两种情况会把redo log持久化到磁盘

1. redo log buffer 占用的空间即将达到 innodb_log_buffer_size 一半的时候，后台线程会主动写盘

	由于这个事务并没有提交，所以这个写盘动作只是 write，而没有调用 fsync，也就是只留在了文件系统的 page cache

2. 并行的事务提交的时候，顺带将这个事务的 redo log buffer 持久化到磁盘

	假设一个事务 A 执行到一半，已经写了一些 redo log 到 buffer 中，这时候有另外一个线程的事务 B 提交，如果 innodb_flush_log_at_trx_commit 设置的是 1，那么按照这个参数的逻辑，事务 B 要把 redo log buffer 里的日志全部持久化到磁盘。这时候，就会带上事务 A 在 redo log buffer 里的日志一起持久化到磁盘


###### binlog

用于记录mysql执行插入和更新操作的相关语句，用于数据备份和主从复制。

###### binlog的写入机制

先写binlog cache，事务提交的时候在写入binlog文件中

binlog cache是每个线程都拥有一个(跟redo log buffer不一样，redo log buffer是全局的)，大小由```binlog_cache_size```决定，如果查过binlog_cache_size，那么就要暂存到磁盘

其写入流程如下图所示

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220323000003791-1967457915.png)

binlog cache(mysql线程的内存) -> page cache(操作系统里文件系统的页缓存) -> disk（磁盘）

- wirte

	指的是从binlog cache写入page cache，存内存操作

- fsync

	指的是从page cache写入磁盘，真正的磁盘操作（产生IOPS）

binlog 的写入机制，如要是由 ```sync_binlog``` 参数（mysql双1参数之一）控制

- 0
	每次提交事务都只write，不fsync，由系统自身决定啥时候fsync
	
- 1
	每次提交事务都会执行fsync
	
- N（N>1）
	每次提交事务都write，但累积N个事务后才fsync

###### redo log和binlog的不同点

- redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。

- redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。

- redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

###### 组提交（group commit）

1. LSN
	
	LSN（log sequence number）指的是日志逻辑序列号，它是单调递增的，用来对应着每次redo log的写入点，每次写入的redo log长度假设为length，那么LSN = LSN + length
	
	LSN也会写入到InnoDB的数据页中，其作用是确保数据页不会被多次执行重复的redo log

2. 组提交流程

	而所谓的组提交，就是多个事务并发执行，某个事务提交的时候，会把其他事务已经写入到redo log buffer的内容一起持久化到磁盘，其流程如下图所示：
	
	![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220329233948330-866544733.png)
	
	有3个事务，分别是trx1，trx2和trx3，他们对应的LSN（假设LSN从0开始）依次是50,120,160
	
	1. trx1先到达，就会成为这组的leader
	
	2. trx1开始写盘的时候（提交的prepare阶段），这组里面已经有3个事务了，这时LSN变为了160
	
	3. trx1写盘，带的就是LSN=160，等trx1写盘成功返回后，所有LSN<=160的redo log，都已经被持久化到磁盘了
	
	4. trx2和trx3需要写盘的时候，因为LSN<=160，已经持久化完毕了，所以不需要任何操作直接返回

	当然，因为开启了bin log的情况下，不止redo log需要写日志，binlog也要写日志，所以binlog也是有组提交的

	在事务的提交流程中，其中有一步是先写redo log，再写binlog，然后提交事务，现在结合组提交再来细化一下这个流程，其流程如下图所示：

	![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220330002348693-325954750.png)

	1. redo log prepare write

		redo log日志(prepare阶段)从redo log buffer -> page cache

	2. binlog write

		binlog日志从binlog cache -> page cache

	3. redo log prepare fsync

		redo log日志(prepare阶段)从page cache -> 磁盘

	4. binlog fsync

		binlog日志从page cache -> 磁盘

	5. redo log commit write

		redo log日志(commit阶段)，从redo log buffer -> page cache，等待后台线程每秒持久化到磁盘

	为了能更好的使redo log和binlog利用组提交的效果，mysql尽可能的拖延redo log和binlog持久化到硬盘的时机，比如在redo log持久化之前，先执行binlog的write操作，在binlog落盘之前，先执行redo log的fsync操作

	mysql更是提供了两个参数，用来优化binlog的组提交

	- binlog_group_commit_sync_delay

		表示延迟多少微秒后才调用 fsync

	- binlog_group_commit_sync_no_delay_count

		表示累积多少次以后才调用 fsync

	以上两个参数只要满足其中之一，就会调用fsync把binlog持久化到磁盘

	最后，补充一点，上图中1-5的步骤，每个步骤其实都有一个队列，每个队列都有一把锁保护，第一个进入队列的事务会成为leader，leader领导所在队列的所有事务，全权负责整队的操作，完成后通知队内其他事务操作结束。


##### flush操作

就是把脏页数据，落盘到磁盘上

###### 脏页

内存数据页跟磁盘数据页内容不一致，这个内存页就叫脏页

###### 触发flush操作的时机

- redo log写满

- 系统内存不足，需要淘汰一些内存页，而这些内存页恰巧包含了脏页

- 在mysql任务系统空闲的时候

- mysql正常关闭的时候

其中，要注意的事第1,2中情况

第一种情况，因为redo log满了的话系统是不再接受更新操作，相当于mysql写入挂掉了

第二种情况，内存淘汰其实是InnoDB的常态操作，InnoDB 用缓冲池（buffer pool）管理内存，缓冲池中的内存页有三种状态：

- 还没有使用的

- 使用了并且是干净页

- 使用了并且是脏页

因为InnoDB的策略是尽量使用内存，因此对于一个长时间运行的库来说，未被使用的页面很少

而1,2种情况，都是有脏页导致的（第一种情况是脏页越多，redo log占用就越大），所以，只要降低了脏页的比例，那么就能尽可能的避免上面这两种情况了

###### 刷脏页的控制策略

innodb_io_capacity变量定义了InnoDB后台任务每秒可用的I/O操作数(IOPS)，这个参数控制了innodb每秒刷脏页的速率，所以一般这个值设置成磁盘的IOPS

磁盘的 IOPS 可以通过 fio 这个工具来测试，下面的语句是我用来测试磁盘随机读写的命令：

```fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest 
```

nnodb_io_capacity 定义了后台任务可用的 IOPS 量

innodb_io_capacity_max 定义了后台任务可用的最大 IOPS 量


###### 二阶段提交

9~13的执行过程，其实就是二阶段提交，其过程细化后，如下图所示：
![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220202233717482-1824718105.png)

1. 执行器先找引擎取 ID=2 这一行。ID 是主键，引擎直接用树搜索找到这一行。如果 ID=2 这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。

2. 执行器拿到引擎给的行数据，把这个值加上 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口写入这行新数据。

3. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态。然后告知执行器执行完成了，随时可以提交事务。

4. 执行器生成这个操作的 binlog，并把 binlog 写入磁盘。

5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态，更新完成。

其中，如果在执行的过程中，服务器down掉了，那个分为3种情况进行分析：

- 如果redo log处在prepare阶段，但还没写完binlog，那么恢复之后就会执行数据的回滚操作

- 如果redo log处在prepare阶段，已经写完了binlog，那么恢复之后只要在binlog找到记录，就会把redo log改为commit状态

- 如果redo log处在commit状态， 那么说明binlog肯定有对应的数据，那么直接提交事务即可

为什么要搞这么复杂，其实就是为了保证redo log和binlog数据的一致性。

试想下，如果binlog和redo log数据不一致的话，那么数据一致性肯定会遭到破坏，比如：

- binlog 多了一条update id = 2的操作，而redo log没有，那么从库的数据肯定和主库不一致，而且用binlog在恢复数据的时候，也会莫名其妙多了一条update id = 2

- 反之，redo log有而binlog没有，那么从库肯定会丢了这条数据，而恢复数据的时候，也会丢掉这条数据

###### mysql如何知道binlog已经写完？

mysql binlog一共有两种格式，分别是statement和row。 

- 对于statement来说，最后有commit就是完整的。

- 对于row来说，最后会有一个XID event，如：

		### INSERT INTO `test`.`test3`
		### SET
		###   @1=3008 /* INT meta=0 nullable=0 is_null=0 */
		###   @2='aaa' /* VARSTRING(120) meta=120 nullable=1 is_null=0 */
		###   @3=3000 /* INT meta=0 nullable=0 is_null=0 */
		# at 496
		#220215  0:30:49 server id 1  end_log_pos 527 CRC32 0xab3331fe 	Xid = 903085

###### redo log和binlog是怎么关联起来的?

两者有个共同的字段Xid，崩溃恢复的时候，会顺序扫描redo log，如果redo log既有prepare又有commit，那么直接提交，如果redo log只有prepare，那么就用Xid去找binlog，如果找到的话就提交，找不到就回滚

###### mysql为什么redo log只有perpare，且binlog完整的话，就可以直接提交？

主要是因为binlog涉及到从库的数据的一致性，如果binlog已经完整了，而redo log却不提交，那么就会导致主库少了这次的记录，从而导致主从数据不一致

###### 只用binlog来实现事务，去掉redo log，可以吗？

不行，因为事务提交后，并不是立即刷盘的，而仅仅只是把内存中对应的数据页修改了而已，如果此时发生了crash的话，那么内存中数据页的数据没有redo log进行恢复从而会导致数据丢失，如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220217004923238-542275088.jpg)

如果commit1的数据提交后，还没落盘的话，那么发生了crash后，commit1的数据因为无法恢复内存中数据页的数据，从而导致了数据的丢失

###### 只用redo log而不用binlog来实现事务，可以吗？

这个其实可以的，因为从崩溃恢复的角度来说，innodb存储引擎恢复数据，用的就是redo log，因为设计到主从复制的关系，所以才需要二阶段提交来保证主从数据一致性，既然关掉了binlog，也就是没有了从库，就只需要关心主库的数据恢复问题，而这主库数据页修复仅依靠redo log就完成可以做到

但是，注意一点就是redo log无法替代binlog的功能，比如binlog能保存每一条sql语句执行的记录，而redo log无法做到，因为redo log是循环写的，在优先的空间内，后写的数据会覆盖前写的数据

###### 正常运行中的实例，数据写入后的最终落盘，是从 redo log 更新过来的还是从 buffer pool 更新过来的呢？

从buffer pool更新的，因为redo log并不是记录数据页的完整数据，而仅仅是记录数据的修改记录，比如说将A的值从1更新成2

在正常情况下，buffer pool的数据页与磁盘中的数据页不一致的话，就会被称为脏页。最终数据落盘，就是把脏页的数据写入磁盘中，这个过程完全和redo log无关

在数据库崩溃恢复的时候，innodb先是把磁盘中的数据加载到内存中，然后通过redo log的记录，把因崩溃而在内存中丢失的更新数据，给恢复过来。这个更新过程完成后，buffer pool中的数据页重新变成脏页，等待数据落盘

###### redo log buffer 是什么？是先修改内存，还是先写 redo log 文件？

redo log buffer 其实就是一块内存，在事务执行的过程中，先把执行的语句更新记录写入到redo log buffer，然后在commit的时候，再从redo log buffer 中，把更新记录写入redo log文件，举个例子：

	begin;
	insert into t1 ...
	insert into t2 ...
	commit;

在执行commit之前，两条insert语句的数据页修改记录，会先写入redo log buffer中，最后执行到commit语句的时候，才把这两条insert语句的数据页修改记录写入的redo log文件（文件名是 ib_logfile+ 数字）中



###### 双1参数介绍

双1参数指的是 ```innodb_flush_log_at_trx_commit``` 和 ```sync_binlog``` ，分别表示每次事务的redo log和binlog持久化到磁盘的策略（详情可以看redo log的写入机制和binlog的写入机制）

##### double write

提供可靠性（就是从page cache中持久化到磁盘过程提供可靠性，因为mysql页是16k，而操作系统磁盘写入最小单位是4k，所以刷盘的时候只能4k4k地写）

innoDB存储引擎正在写入某个页到表中，突然发生了宕机（比如16KB的页，只写了4KB），通过重做日志中的记录进行恢复的时候（重新执行），因为页已经发生了损坏（写了4KB的数据），再重新对该页操作是没意义的

所以引入了双写，在对缓冲池的脏页进行刷新时，先记录原先该页的数据，当写入失效发生的时候，先恢复该页原来的数据，在重新通过重做日志重新执行一次操作


### 关于delete语句无法回收磁盘空间问题
delete语句执行之后，只是在对应的页上标记为已删除状态，实际上空间时没有被回收的，如果有新的数据，就会复用这个空间，覆盖原来的数据，举个例子：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220210005517859-1456726384.png)

如果删除的500这个数据，那么innodb就会在R4上标记已删除，但是并不会回收空间，如果下次再有新的值插入，而且值是在400 ~ 600之间（小于400会插入R3之前，大于600则会插入R5之后），才会复用这个空间，否则会一直占着，如果是PAGE A这个页被标记删除了的话，那么如果innodb申请新的内存页的话，可以直接复用这个页

如果要回收被标记为已删除的空间的话，需要重建表，用以下语句即可：

```alter table xxx engine=innodb```

注意，在重建表的时候，InnoDB 不会把整张表占满，每个页留了 1/16 给后续的更新用。也就是说，其实重建表之后不是“最”紧凑的。有这么一种情况，本来表里面每个页都是满了的，但是因为执行了重建表的语句，却让每个表空出了1/16，所以这种特殊情况下，执行上述语句重建表的话反而会将表变得更大

### count(*)的实现方式
#### myisam
myisam引擎会把一个表的总行数存在了磁盘上，因此执行 count(*) 的时候会直接返回这个数，效率很高（当没有where条件的情况下）

#### innodb
需要把数据一行一行地从引擎里面读出来，然后累积计数

为啥innodb不像myisam那样，把count(*)存起来？

由于mvcc的原因，同一时刻不同查询有可能看到的数据行数是不一样的，所以就算存起来了也没啥意义

其实，执行命令```show table status like "%表名%"```中，有一列为Rows，也是表示表数据总数，但是这个值是根据采样估算得来的，官方文档也说误差可能达到40% ~ 50%

#### count(*)的优化
在mysql5.6的版本，使用count(*)是用聚簇索引的，但在mysql5.7中，MySQL优化器会找到最小的那棵索引树来遍历

#### 不同count()的用法
首先要先明白一个概念，count还一个聚合函数，对于返回的结果集，需要一行一行的进行判断，如果count函数的参数不是NULL，累计值就加 1，否则不加。最后返回累计值。所以，不是同一个表，不同的count用法，其结果是不一样的，如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220210231131573-1392647946.jpg)

在分析性能方面，有这么几个原则：
1. server层要什么就给什么
2. innodb只给必要的值

对于count来说，现在的优化器只优化了count(*)的语义为“取行数”，其他显而易见的优化并没有

下面，就来介绍4种count的方式 

##### count(主键)
innodb会遍历整张表，然后返回主键给server层，因为主键不可能为null，所以server层无需判断抓紧是否为null，直接按行累加

##### count(1)
innodb会遍历整张表，但是不取值。server层对于返回的每一行，放个1进去，判断不可能为null，直接按行累加

##### count(字段)
innodb会遍历该字段所在的索引，此时分两种情况

- 该字段定义为not null

	innodb一行行的读出这个字段，因为定义为not null，所以server层判断不可能为null，直接按行累加

- 该字段定义允许为null

	innodb一行行的读出这个字段，因为定义允许为null，所以server层通过判断改字段是否为null，不为null才+1

##### count(*)
count(*)是 mysql专门做了优化的，虽然会遍历整张表，但是并不取值，count(*)默认不为null，直接按行累加

注意，count(主键)和count(字段)，因为涉及到返回字段值，而从innodb引擎返回字段值会涉及到解析数据行，以及拷贝字段值的操作

虽然count(1) 和 count(主键)看着像一样，但是实际上count(1)效率要比count(主键)高

所以，按照效率排的话，count(*) ≈ count(1) > count(主键) > count(字段)


#### order by的实现原理
如果排序的字段，是在索引上的，那么返回的数据，就直接就是有序的数据了。但是，如果排序的字段，并不存在于索引上，innodb会把所有的数据返回给server层，然后server层排序后再返回结果集。

举个例子：

	CREATE TABLE `t` (
	  `id` int(11) NOT NULL auto_increment,
	  `city` varchar(16) NOT NULL,
	  `name` varchar(16) NOT NULL,
	  `age` int(11) NOT NULL,
	  `addr` varchar(128) DEFAULT NULL,
	  PRIMARY KEY (`id`),
	  KEY `city` (`city`)
	) ENGINE=InnoDB;

	select city,name,age from t where city='杭州' order by name limit 1000

##### 索引上有排序字段
我们先假设，现在表结构有一个 ```KEY city_name(`city`,`name`) ``` 索引，那么排序的执行流程如下：
1. 从二级索引city_name中找到city="杭州"的数据，然后根据id回表查询name，age字段，作为结果集的一部分直接返回
2. 重复第1步，一直遍历到city不满足="杭州"的条件为止或者查到第1000条记录，循环结束

其流程如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220220011048128-617312162.jpg)

当然，如果索引是 ```KEY city_name_age(`city`,`name`,`age`) ``` 的话，那么就是覆盖索引了，就不需要回表了，这是流程如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220220011721556-2117399866.jpg)

##### 索引上无排序字段
我们可以先看下该语句explain解析出来的结果

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220222001741021-2032182235.png)

extra中，有Using filesort，就是无索引的排序方式。

name因为没有索引， 所以会在server层的sort_buffer中进行排序，而实现排序的方式，一共有3种。

###### 全字段排序
1. server层初始化sort_buffer，根据sql语句，确定要把city，name，age这几个字段放进来
2. 从二级索引city中查找city="杭州"的数据，然后根据id回表查询name，age字段数据，返回给server层存入sort_buffer中
3. 重复第2步，一直遍历到city不满足="杭州"的条件为止，然后返回server层对sort_buffer中的数据进行排序
4. 排序后，返回前1000条结果

其流程图如下所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220219000051517-123915685.jpg)

###### 外部排序
当sort_buffer设置的大小（sort_buffer_size参数配置），不满足存放从表里读取的所有数据的时候，这时候就需要用到硬盘来作为临时存储从而进行排序，这就是外部排序

外部排序，一般是用归并排序法，也就是把排序的数据分成多份(根据sort_buffer_size来分，sort_buffer_size越小分的份数越高)，每一份单独排序后存放到临时文件中，最后多个临时文件再合并成一个有序的大文件

###### rowid排序
由于外部排序需要用到临时文件，所以在性能方面这个排序方法不太友好，究其原因会出现用外部排序的方法进行排序，就是因为数据行太大导致了sort_buffer装不下，需要借助临时文件进行排序

所以mysql有一个配置参数max_length_for_sort_data，当需要排序的数据行的大小大于该参数值时，就会只取出主键和排序字段放入sort_buffer，而不是整行字段，最后排序后根据主键再去获取所需字段，返回结果集

比如说，因为现在需要的字段是city,name，age，而这3个字段加起来大小为36（16+16+4）就是需要排序的数据行大小，假设说当max_length_for_sort_data的配置大于36的话，就会使用rowid排序

对于全字段排序，rowid排序流程如下：
1. server层初始化sort_buffer，因为数据行大小大于max_length_for_sort_data，所以sort buffer确定只需要把id和name这两个字段放进来
2. 从二级索引city中查找city="杭州"的数据，然后根据id回表查询name字段数据，返回给server层存入sort_buffer中
3. 重复第2步，一直遍历到city不满足="杭州"的条件为止，然后返回server层对sort_buffer中的数据进行排序
4. 排序后的数据，获取前1000条，然后根据id字段遍历去主键索引获取city，age和name字段，最后返回结果

其流程图如下所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220220005528794-35283403.jpg)

其中，结果集只是一个逻辑概念，实际上mysql服务端sort_buffer 中依次取出 id，然后到原表查到 city、name 和 age 这三个字段的结果，不需要在服务端再耗费内存存储结果，是直接返回给客户端的

##### 索引上无排序字段，且排序字段不存在表中
我们再来看一个例子
	
	CREATE TABLE `words` (
  		`id` int(11) NOT NULL AUTO_INCREMENT,
  		`word` varchar(64) DEFAULT NULL,
  		PRIMARY KEY (`id`)
	) ENGINE=InnoDB;

	select word from words order by rand() limit 3;

这sql语句的含义是随机获取3条数据，其explain解析结果如下所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220222001756986-1835465743.png)

其中，extra中有Using temporary说明使用了临时表， Using filesort表示需要排序操作，所以这个extra的意思就是需要在临时表中进行排序操作

因为sql语句只需要一个随机值和word，为了方便描述，随机值简写为R，word简写为W，其排序流程如下：

1. 初始化内存临时表，内存临时表使用的是memory存储引擎，而内存临时表只有两个字段W和R

2. 从主键中按顺序获取word字段值，每一个对应的word值生成一个0-1的随机数(rand()函数)，然后写入临时表的W和R中

3. 初始化sort buffer，sort buffer中有两个字段，字段类型分别是int和double，其中int存放的是临时表的位置信息（相当于临时表的主键，因为就是用这个字段来回表），double则是放rand()函数生成的随机数，然后根据R的值进行排序

4. 排序完成后，因为只需要前3条数据，所以取出前3个位置信息，回内存临时表获取W值，最后返回给客户端

其流程图如下所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220222010824324-2135326575.png)

注意：

- 对于innodb表来说，因为执行全字段排序会减少磁盘访问，因此会被优先选择

- 但是对于内存临时表，也就是memory表来说，因为整个表就已经存在于内存了，回表也是纯内存操作，并不涉及到磁盘访问，因此mysql会优先考虑排序的行越小越好，优先使用的是rowid排序

###### 优先队列排序算法
在mysql5.6之后，为了优化外部排序，mysql引入了优先队列排序算法，在查询语句带有limit n的时候，只排序前n条数据

还是以 ```select word from words order by rand() limit 3``` 举例

假如总数据量为10000条，但是查询的时候排序后只需要返回3条（order by rand() limit 3），那么其实我们是并不需要对10000条数据进行排序的，只需要保证前3条数据是10000条数据中最小的3条就可以了

而优先队列排序算法，就是先把前3条数据放入sort_buffer，当然，因为是order by rand(),这3条数据在sort_buffer中要先从小到大排序，构成一个最大堆，然后对剩下的9997条数据依次比较

为了方便描述，这里把3条数据中，最大的数据设为R，需要对比的数据设为R'

- 如果R <= R'，继续从临时表中取出下一条数据比较

- 如果R > R'，那么把R剔除出最大堆中，然后把R'放入到最大堆中，重新排序

剩余的9997条数据重复以上流程，对比完后最终在sort_buffer的3条数据，就是最小的3条数据，最后根据rowid从临时表中取出word字段，返回结果

我们以6个数据为例，其流程如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220224070816191-1499474812.png)

##### 如何判断sql查询排序时是否用了临时文件
可以通过查看OPTIMIZER_TRACE的结果来确认是否使用了临时文件

	/*只对本线程有影响*/
	set optimizer_trace='enabled=on';

	select bid from hunter.cdb_hunter_group_member order by rand() limit 3;

	SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`;

其结果如下所示

	{
	  "steps": [
	    {
	      "join_preparation": {
	        "select#": 1,
	        "steps": [
	          {
	            "expanded_query": "/* select#1 */ select `test`.`t`.`bid` AS `bid` from `test`.`t` order by rand() limit 3"
	          }
	        ]
	      }
	    },
	    {
	      "join_optimization": {
	        "select#": 1,
	        "steps": [
	          {
	            "table_dependencies": [
	              {
	                "table": "`test`.`t`",
	                "row_may_be_null": false,
	                "map_bit": 0,
	                "depends_on_map_bits": [
	                ]
	              }
	            ]
	          },
	          {
	            "rows_estimation": [
	              {
	                "table": "`test`.`t`",
	                "table_scan": {
	                  "rows": 140928,
	                  "cost": 1315
	                }
	              }
	            ]
	          },
	          {
	            "considered_execution_plans": [
	              {
	                "plan_prefix": [
	                ],
	                "table": "`test`.`t`",
	                "best_access_path": {
	                  "considered_access_paths": [
	                    {
	                      "access_type": "scan",
	                      "rows": 140928,
	                      "cost": 29501,
	                      "chosen": true
	                    }
	                  ]
	                },
	                "cost_for_plan": 29501,
	                "rows_for_plan": 140928,
	                "chosen": true
	              }
	            ]
	          },
	          {
	            "attaching_conditions_to_tables": {
	              "original_condition": null,
	              "attached_conditions_computation": [
	              ],
	              "attached_conditions_summary": [
	                {
	                  "table": "`test`.`t`",
	                  "attached": null
	                }
	              ]
	            }
	          },
	          {
	            "clause_processing": {
	              "clause": "ORDER BY",
	              "original_clause": "rand()",
	              "items": [
	                {
	                  "item": "rand()"
	                }
	              ],
	              "resulting_clause_is_simple": false,
	              "resulting_clause": "rand()"
	            }
	          },
	          {
	            "refine_plan": [
	              {
	                "table": "`test`.`t`",
	                "access_type": "index_scan"
	              }
	            ]
	          }
	        ]
	      }
	    },
	    {
	      "join_execution": {
	        "select#": 1,
	        "steps": [
	          {
	            "creating_tmp_table": {
	              "tmp_table_info": {
	                "table": "intermediate_tmp_table",
	                "row_length": 17,
	                "key_length": 0,
	                "unique_constraint": false,
	                "location": "memory (heap)",
	                "row_limit_estimate": 123361
	              }
	            }
	          },
	          {
	            "converting_tmp_table_to_myisam": {
	              "cause": "memory_table_size_exceeded",
	              "tmp_table_info": {
	                "table": "intermediate_tmp_table",
	                "row_length": 17,
	                "key_length": 0,
	                "unique_constraint": false,
	                "location": "disk (MyISAM)",
	                "record_format": "fixed"
	              }
	            }
	          },
	          {
	            "filesort_information": [
	              {
	                "direction": "asc",
	                "table": "intermediate_tmp_table",
	                "field": "tmp_table_column"
	              }
	            ],
	            "filesort_priority_queue_optimization": {
	              "limit": 3,
	              "rows_estimate": 203614,
	              "row_size": 24,
	              "memory_available": 868352,
	              "chosen": true
	            },
	            "filesort_execution": [
	            ],
	            "filesort_summary": {
	              "rows": 4,
	              "examined_rows": 203604,
	              "number_of_tmp_files": 0,
	              "sort_buffer_size": 128,
	              "sort_mode": "<sort_key, additional_fields>"
	            }
	          }
	        ]
	      }
	    }
	  ]
	}{
	  "steps": [
	    {
	      "join_preparation": {
	        "select#": 1,
	        "steps": [
	          {
	            "expanded_query": "/* select#1 */ select `test`.`t`.`bid` AS `bid` from `test`.`t` order by rand() limit 3"
	          }
	        ]
	      }
	    },
	    {
	      "join_optimization": {
	        "select#": 1,
	        "steps": [
	          {
	            "table_dependencies": [
	              {
	                "table": "`test`.`t`",
	                "row_may_be_null": false,
	                "map_bit": 0,
	                "depends_on_map_bits": [
	                ]
	              }
	            ]
	          },
	          {
	            "rows_estimation": [
	              {
	                "table": "`test`.`t`",
	                "table_scan": {
	                  "rows": 140928,
	                  "cost": 1315
	                }
	              }
	            ]
	          },
	          {
	            "considered_execution_plans": [
	              {
	                "plan_prefix": [
	                ],
	                "table": "`test`.`t`",
	                "best_access_path": {
	                  "considered_access_paths": [
	                    {
	                      "access_type": "scan",
	                      "rows": 140928,
	                      "cost": 29501,
	                      "chosen": true
	                    }
	                  ]
	                },
	                "cost_for_plan": 29501,
	                "rows_for_plan": 140928,
	                "chosen": true
	              }
	            ]
	          },
	          {
	            "attaching_conditions_to_tables": {
	              "original_condition": null,
	              "attached_conditions_computation": [
	              ],
	              "attached_conditions_summary": [
	                {
	                  "table": "`test`.`t`",
	                  "attached": null
	                }
	              ]
	            }
	          },
	          {
	            "clause_processing": {
	              "clause": "ORDER BY",
	              "original_clause": "rand()",
	              "items": [
	                {
	                  "item": "rand()"
	                }
	              ],
	              "resulting_clause_is_simple": false,
	              "resulting_clause": "rand()"
	            }
	          },
	          {
	            "refine_plan": [
	              {
	                "table": "`test`.`t`",
	                "access_type": "index_scan"
	              }
	            ]
	          }
	        ]
	      }
	    },
	    {
	      "join_execution": {
	        "select#": 1,
	        "steps": [
	          {
	            "creating_tmp_table": {
	              "tmp_table_info": {
	                "table": "intermediate_tmp_table",
	                "row_length": 17,
	                "key_length": 0,
	                "unique_constraint": false,
	                "location": "memory (heap)",
	                "row_limit_estimate": 123361
	              }
	            }
	          },
	          {
	            "converting_tmp_table_to_myisam": {
	              "cause": "memory_table_size_exceeded",
	              "tmp_table_info": {
	                "table": "intermediate_tmp_table",
	                "row_length": 17,
	                "key_length": 0,
	                "unique_constraint": false,
	                "location": "disk (MyISAM)",
	                "record_format": "fixed"
	              }
	            }
	          },
	          {
	            "filesort_information": [
	              {
	                "direction": "asc",
	                "table": "intermediate_tmp_table",
	                "field": "tmp_table_column"
	              }
	            ],
	            "filesort_priority_queue_optimization": {
	              "limit": 3,
	              "rows_estimate": 203614,
	              "row_size": 24,
	              "memory_available": 868352,
	              "chosen": true
	            },
	            "filesort_execution": [
	            ],
	            "filesort_summary": {
	              "rows": 4,
	              "examined_rows": 203604,
	              "number_of_tmp_files": 0,
	              "sort_buffer_size": 128,
	              "sort_mode": "<sort_key, additional_fields>"
	            }
	          }
	        ]
	      }
	    }
	  ]
	}

filesort_summary中的number_of_tmp_files表示的就是排序临时文件数，如果number_of_tmp_files为0的话就表示没有使用临时文件

filesort_summary.examined_rows 表示查询记录数

filesort_summary.sort_mode:

- <sort_key, additional_fields> 使用紧凑排序，如用排序字段的定义是varchar(30)，但是在排序过程中还是要按照实际长度来分配空间的

- <sort_key, additional_fields> 使用rowid排序方式进行排序

filesort_priority_queue_optimization.chosen:true 表示使用了优先队列排序算法

#### 当varchar类型查询字段，条件比设置的字符数大的执行流程

	CREATE TABLE `table_a` (
  		`id` int(11) NOT NULL,
  		`b` varchar(10) DEFAULT NULL,
  		PRIMARY KEY (`id`),
  		KEY `b` (`b`)
	) ENGINE=InnoDB;

	select * from table_a where b='1234567890abcd';

假设现在表里面，有 100 万行数据，其中有 10 万行数据的 b 的值是’1234567890’，那么其执行流程如下：

1. 因为varchar定义的字符长度为10，所以1234567890abcd只截取前10个字符（1234567890），然后进行匹配

2. 因为满足条件的数据有10w条，所以需要回表10w次

3. 每次回表的时候，都会在server层判断，b是否等于1234567890abcd，因为都不等，所以返回的结果为空

因此，mysql不会因为定义的是varchar(10)，就直接返回类似```Impossible WHERE noticed after reading const tables```的结果


## 事务隔离
### 事务的四个特性
分别是原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability），即ACID

### 隔离级别
一共4种，分别是读未提交（read uncommitted）、读提交（read committed）、可重复读（repeatable read）和串行化（serializable ）

- 读未提交
	一个事务还没提交时，它做的变更就能被别的事务看到

- 读提交
	一个事务提交之后，它做的变更才会被其他事务看到

- 可重复读
	一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。当然在可重复读隔离级别下，未提交变更对其他事务也是不可见的。

- 串行化
	对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。

#### 读提交和可重复读区别
假如说有事务A和事务B，事务A还在执行过程中，而事务B则是已经执行完成，那么在读提交的隔离级别下，事务A可以即时获取事务B更新的数据，而在可重复读隔离级别下，事务A只能获取开始事务时刻的数据。其执行流程如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220203164655076-2004289873.png)

- 读未提交

	V1 的值就是 2。这时候事务 B 虽然还没有提交，但是结果已经被 A 看到了。因此，V2、V3 也都是 2

- 读提交

	V1 是 1，V2 的值是 2。事务 B 的更新在提交后才能被 A 看到。所以， V3 的值也是 2

- 可重复读

	V1、V2 是 1，V3 是 2。之所以 V2 还是 1，遵循的就是这个要求：事务在执行期间看到的数据前后必须是一致的

- 串行化

	V1、V2 是 1，V3 是 2。之所以 V2 还是 1，遵循的就是这个要求：事务在执行期间看到的数据前后必须是一致的

### mvcc
#### 一致性读视图（consistent read view）
- 可重复读

	在事务启动时创建的，整个事务存在期间都用这个视图

- 读提交

	在每个SQL语句开始执行的时候创建的

- 读未提交

	不使用一致性读视图，直接返回记录上的最新值

- 串行化

	不使用一致性读视图，用加锁的方式来避免并行访问

#### undo log在mvcc中的作用
一致性视图的生成，离不开undo log，每次启动一次事务，都会向innodb的事务系统一个事务id（下文统称trx_id），而且trx_id是按申请顺序严格递增的。

比如说更新k这行数据，不同的线程更新了多次，那么k这行数据就会产生多个版本，如下入所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220204221610739-2132214240.png)

上图中虚线方框部分就是k这一行在undo log中的数据结构，每个版本依次用指针连接，每次生成一致性视图，都会根据生成一致性视图时的trx_id（t_id1），若当前数据行的trx_id(t_id2)大于t_id1，那么就会去undo log中查找trx_id<=t_id1的数据，就是这行在一致性视图中显示的数据（比如t_id1 = 17，t_id2 = 25，那么一致性视图就会去undo log找trx_id <= 17的数据，也就是k=11的数据）。

#### innodb实现mvcc原理
在innodb中，innodb为每个事务构造了一个数组，用来保存这个事务启动瞬间，当前正在“活跃”的所有事务 ID。“活跃”指的就是，启动了但还没提交。

数组里面事务 ID 的最小值记为低水位，当前系统里面已经创建过的事务 ID 的最大值加 1 记为高水位。

这个视图数组和高水位，就组成了当前事务的一致性视图（read-view），如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220204224108854-718970791.png)

而每次事务启动（事务id为t_id1）时当前行的数据可见性，就是根据当前行的trx_id和这个一致性视图的对比结果得到的，一共有如下几种情况：

- 如果trx_id落在绿色部分，表示这个版本是已提交的事务或者是当前事务自己生成的，这个数据是可见的

- 如果trx_id落在红色部分，表示这个版本是由将来启动的事务生成的，是肯定不可见的

- 如果trx_id落在黄色部分，那就包括两种情况

	- 若trx_id在数组中，表示这个版本是由还没提交的事务生成的，不可见

	- 若trx_id不在数组中，表示这个版本是已经提交了的事务生成的，可见

##### 举例分析

	CREATE TABLE `t` (
  		`id` int(11) NOT NULL,
  		`k` int(11) DEFAULT NULL,
  		PRIMARY KEY (`id`)
	) ENGINE=InnoDB;

	insert into t(id, k) values(1,1),(2,2);

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220204230306368-346097145.png)

这里先说明事务的启动时机下：

```begin/start transaction```命令并不是一个事务的起点，在执行到它们之后的第一个操作InnoDB表的语句，事务才真正启动。而这时候一致性视图是在执行第一个快照读语句时创建的

如果你想要马上启动一个事务，可以使用```start transaction with consistent snapshot```这个命令。而这时候一致性视图是在执行```start transaction with consistent snapshot```时创建的

事务C没有显式地使用```begin/commit```，表示这个update语句本身就是一个事务，语句完成的时候会自动提交

分析完事务的启动时机后，就来分析事务的视图数组。这里，我们不妨做如下假设：

- 事务A开始前，系统里面只有一个活跃事务ID是99

- 事务A、B、C的版本号分别是100、101、102，且当前系统里只有这四个事务

- 三个事务开始前，(1,1）这一行数据的row trx_id是90

那么事务A的视图数组就是[99,100], 事务B的视图数组是[99,100,101], 事务C的视图数组是[99,100,101,102],那么先分析事务A，其流程如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220204232013567-466437407.png)

从图中可以看到，第一个有效更新是事务 C，把数据从 (1,1) 改成了 (1,2)。这时候，这个数据的最新版本的 row trx_id 是 102，而 90 这个版本已经成为了历史版本。

第二个有效更新是事务B，把数据从(1,2)改成了(1,3)。这时候，这个数据的最新版本（即row trx_id）是101，而102又成为了历史版本。

虽然在事务A查询的时候，事务B还没有提交，但是在undo log记录中，已经存在了trx_id=101和对应的事务回滚语句了，说明该行数据的当前trx_id已经变成了101

因为读数据都是从当前版本读起的，所以事务 A 查询语句的读数据流程是这样的：

1. 找到 (1,3) 的时候，判断出 row trx_id=101，比高水位大，处于红色区域，不可见
2. 接着，找到上一个历史版本，一看 row trx_id=102，比高水位大，处于红色区域，不可见
3. 再往前找，终于找到了（1,1)，它的 row trx_id=90，比低水位小，处于绿色区域，可见

所以，最终事务A查询的结果为1。

虽说这行数据在查询期间，被修改过多次，但是事务A不论在什么时候查询，看到这行数据的结果都是一致的，所以我们称之为一致性读。

然后，现在我们来分析事务B的

因为涉及到更新操作，而且更新数据都是先读后写的，但是这个读有点特殊，叫当前读（current read），是只能读取已经提交完成的最新数据。

为什么需要当前读呢？

假如没有当前读，那么事务B在更新语句执行的时候，根据一致性视图，获取到k的值为1，而+1后k的值就为2，然后更新数据页中。这样的话，事务C的更新就丢失了

当然，除了update语句是当前读之外，select语句如果加锁也是当前读，如：
```select k from t where id=1 lock in share mode // 加读锁```

```select k from t where id=1 for update // 加写锁```

这里对于可见性分析简单总结：

1. 自己事务内的更新，总是可见的
2. 版本未提交，不可见
3. 版本已提交，但是是在视图创建后提交的，不可见
4. 版本已提交，而且是在视图创建前提交的，可见

所以对于事务B的流程如下：
1. 虽说事务B的一致性视图，是在事务C执行之前生成的，但是由于事务B执行了update语句，需要获取更新的是最新版本的数据，所以此时k的值为2=1=3
2. 因为自己事务内的更新，总是可见的，所以在执行select语句的时候，k读取到的值为3

再来看一个例子，其执行流程如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220205005318809-2068165476.png)

与事务C不同的是，事务C'更新后并没有马上提交，在它提交前，事务B的更新语句先发起了

因为事务C'还没commit，也就是持有这行数据的写锁，所以当事务B执行到update语句的时候，会被阻塞住，当事务C执行完释放写锁后，事务B才能继续执行下去，所以最终的查询结果也是k=3

上面的分析，是隔离级别在rr的情况下的，那么在rc的情况下呢？

首先先说明下，rc和rr在生成一致性视图的时机中的区别

- 在rr隔离级别下，只需要在事务开始的时候创建一致性视图，之后事务里的其他查询都共用这个一致性视图

- 在rc隔离级别下，每一个语句执行前都会重新算出一个新的视图

因为生成一致性视图的时机两种隔离级别不一致，所以```start transaction with consistent snapshot```在rc下相当于```start transaction```

所以在rc隔离级别下，其执行流程图如下所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220205010646662-1825456774.jpg)

因为rc隔离级别下，每次执行语句都会重新生成一致性视图，因为事务C已经commit，而事务B还没commit，所以事务A查到k的值为2，而事务B则是k=3

因此，简单总结一下rc和rr的数据的可见性：
- 对于rr，查询只承认在事务启动前就已经提交完成的数据

- 对于rc，查询只承认在语句启动前就已经提交完成的数据


### 查询当前长事务语句

```select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60```

### 脏读
在一个执行中的事务，读取到了另一个事务未提交的数据，其流程如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220308010142665-1084887425.jpg)

解决：RC以上的隔离级别即可解决

### 不可重复读
在同一个事务中，读取到的数据前后不一致（针对于update操作），其流程如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220308010203250-224062919.png)

解决：会发生在RC中（因为RC的快照是每次执行语句的时候生成的，而RR则是开始事务的时候生成快照，直到事务提交前都一直会使用这个快照），所以RR以上的隔离级别即可解决

### 幻读
在同一个事务中，读取到的数据总量前后不一致，其流程如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220308010221924-660832810.png)

注意，因为mvcc的原因，如果第7步是快照读的话，读取到的数据量还是100条，但是如果是当前读，那么在没有间隙锁的情况下，才会读取到200条

解决：RR隔离级别 + 间隙锁(Gap Lock)

### 不可重复读和幻读的区别

- 不可重复读是针对同一行数据的数值不一致，针对update操作

- 幻读是读取了其他事务新增的数据，针对add和delete操作

### 不同隔离级别对应出现的问题

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220308012211819-745177254.png)


## 索引
可以让服务器快速定位到表的指定位置

### 索引常见模型
#### 哈希表
哈希表是一种键值对的存储数据结构（key-value），key用哈希函数换算后得到的下标，然后把数据放在对应下标的value数组中。

如果出现哈希冲突，就使用拉链法，用链表存储在哈希出相同下标值的多个数据

因为数据不是有序的，所以插入数据，等值查询很快，但是哈希索引做区间查询只能全扫描，效率很慢，因此哈希索引一般用于比如Memcached及其他一些NoSQL引擎

#### 有序数组
有序数组在等值查询和范围查询场景中的性能就都很好，但是对插入或者更新数据，效率就非常慢，因为每次插入或者更新数据，都需要挪动后面所有的记录。

#### 二叉搜索树
特点：左子树的节点均小于父节点，右子树的节点均大于父节点，但查询时间复杂度最高为O(N)（也就是最极端的情况下只有左节点或右节点所形成的链表）。

#### 平衡二叉树
在二叉搜索树的基础上，加上左右子树高度差不超过1，就是平衡二叉树了，相比二叉搜索树，它的查询时间复杂度为O(log(N))，更新的时间复杂度也是 O(log(N))。

#### B树
因为索引不止存在内存中，还要写到磁盘上，假如有一棵100万节点的平衡二叉树，树高20。一次查询可能需要访问 20个数据块。在机械硬盘时代，从磁盘随机读一个数据块需要10ms左右的寻址时间，如果使用二叉树来存储的话，那么查询一个行的查询时间就是200ms了。

为了让一个查询尽量少地读磁盘，就必须让查询过程访问尽量少的数据块。而B树就是为了优化这点，B树其实就是一棵N叉树，N取决于数据块的大小，这样就可以尽可能的降低树的高度了。

#### B+树
相比于B树，B+树有进行了一次优化，它的所有记录节点都是按照从小到大顺序存放在最后一层的叶子节点上，由各叶子节点的指针相连接。

可以认为一个叶子节点就是一个内存页(默认情况下，一个内存页大小为16K)，每个内存页里面存储多个数据行，内存页直接通过指针连接，形成一个双向链表，所以叶子节点在逻辑上是连续的，在物理上不是连续存储的，就是每个叶子节点可以存储在不同地址上，通过指针相互连接。每个非叶子节点(也就是索引节点)也是一个内存页，里面存储了很多索引节点的值，但是B+树索引节点只存索引值，不存数据行的数据，这样可以让每个索引内存页存储更多的索引值，这样可以使得B+树的层数更少（这也是B+树比B树更优的地方）。B+树在数据库的中实现一般是只有2到4层，机械磁盘一般1秒可以进行100次IO，也意味着每次在B+树中的查询操作可以在20ms到40ms之间完成。

#### B树和B+树的区别
- B树每个节点会保存关键字，索引和数据。而B+树只有叶子节点保存数据，其他节点只保存关键字和索引。所以相同的内存空间可以容纳更多的索引节点

- B+树的所有数据都存在叶子节点上，所以查询会更加稳定，而且相邻的叶子节点都是连接在一起的，更加适合区间查找和搜索。

### InnoDB的索引模型
innodb使用的事B+树索引模型，表都是根据主键顺序以索引的形式存放的。在innodb中，每个索引对应着一棵B+树

### 聚簇索引和二级索引
- 聚簇索引（clustered index）

	指的是主键索引，记录的是整行数据

- 二级索引（secondary index）

	指的是非主键索引，记录的是索引上的列和主键的值

#### 聚簇索引和二级索引查询方式

以如下表结构举例说明：

	create table T(
		id int primary key, 
		k int not null, 
		name varchar(16),
		index (k)
	)engine=InnoDB;

	insert into T values(100,1, 'aa'),(200,2,'bb'),(300,3,'cc'),(500,5,'ee'),(600,6,'ff'),(700,7,'gg');

- 聚簇索引

	```select * from T where id = 1```

	直接搜索主键索引上id = 1的数据，然后返回

- 二级索引

	```select * from T where k = 100```

	先搜索k索引树，得到id为1，然后根据id在从主键索引上找id=1的数据，然后返回，这个过程称为回表

在索引定位到对应的数据页后，因为数据页内部是个有序数组，用二分法就能找到对应的数据

#### 索引维护
为了保证B+树的有序性，再插入和更新数据的时候，需要进行索引的维护，如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220204143611529-897397616.png)

如果插入的是700，那么直接在R5后插入即可,如果400的话，就往R3和R4之间插入，然后挪动后面的数据。但如果R3R4R5沾满了数据页，400没有足够的空间插入进去，那么就要触发页分裂了。

##### 页分裂
也分裂就是innodb申请一块新的内存页，然后迁移一半的数据过去，最后把指针分裂的页指针指向新的页即可，如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220204150428083-1674997933.png)

##### 重建索引
对于由于页分裂，或者删除数据锁造成的数据页空洞，可以使用以下语句重建索引，回收空间
```alter table xxx engine=InnoDB```

#### 覆盖索引

假设我们执行```select * from T where k between 3 and 5```这条语句，那么查询过程如下：

1. 在 k 索引树上找到 k=3 的记录，取得 ID = 300
2. 再到 ID 索引树查到 ID=300 对应的 R3
3. 在 k 索引树取下一个值 k=5，取得 ID=500
4. 再回到 ID 索引树查到 ID=500 对应的 R4
5. 在 k 索引树取下一个值 k=6，不满足条件，循环结束

步骤2,4的过程称为回表，因为k索引上没有name字段，所以需要会主键索引去查找，那么假如二级索引上，包含了sql查询所需的所有字段的话，是不是就不需回表这个过程呢？答案是肯定的，而这种二级索引，就叫做覆盖索引。

```select id from T where k between 3 and 5```

如上面这个查询语句，因为sql返回值需要id这个字段，而k索引上也有这个id的数据，所以这个语句就无需回表，而k索引对于这个查询语句而言就是覆盖索引。

需要注意的是，在引擎内部使用覆盖索引在索引 k 上其实读了三个记录，R3~R5（对应的索引 k 上的记录项），但是对于 MySQL 的 Server 层来说，它就是找引擎拿到了两条记录，因此 MySQL 认为扫描行数是 2。

#### 最左前缀
以如下表结构举例说明：

	create table T(
		id int primary key, 
		a int not null, 
		b int not null,
		c int not null,
		d int not null,
		index (a,b,c)
	)engine=InnoDB;

T表有个索引abc，而最左前缀的意思就是先比较a的大小，再比较b的大小然后再比较c的大小，而遇到范围比较时就会停止匹配。

比如：

- a = 1 and b > 2 and c = 1,c是无法使用索引的

- =和in可以乱序，比如 a=1 and c=2 and b=3 也可以使用index(a,b,c)索引，优化器会优化成索引可以识别的形式

- 并不是只要有or就无法走索引，假如or连接的俩个查询条件字段中有一个没有索引的话,innodb才会放弃索引而产生全表扫描

#### 索引下推（index condition pushdown, icp）
还是上面那个例子，如果查询语句为```select * from T where a = 1 and c = 10```，那么根据最左前缀原则，c是无法使用索引的，那么找到a记录的话，就直接回表，但是在5.6版本之后，可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220204165114024-1357325003.png)

虽然结果都只有一条记录符合，不过5.6之前，要回表3次，而5.6之后，只需要回表一次。

#### 强制走索引
```force index(索引名) // 强制走索引```
```select * from t force index(a) where a between 10000 and 20000  //强制走a索引```

#### 前缀索引
某些字段因为是varchar类型，而且需要在为其建立索引，为了节省索引的空间，一般会使用前缀索引。

##### 如何创建前缀索引
格式 ： ```alter table xxx add index 索引名(字段名(6))```，如：

```alter table SUser add index index2(email(6))```

##### 前缀索引的优缺点
- 优点

	节省索引空间

- 缺点

	1. 可能会增加额外的记录扫描次数
	2. 无法使用覆盖索引

比如email字段，现有值zhangsan@qq.com和zhangsan@163.com，如果使用前缀索引index(email(5))只取前5个字符串，那么查询语句```select * from t where email = "zhangsan@qq.com"```就不得不把zhangsan@163.com也遍历出来，因为前缀索引是zhang，两个值都包含，而zhangsan@163.com将会在server层被过滤掉

无法使用覆盖索引也是同样道理，因为从索引中得到结果后，需要回表去进行过滤而不能直接返回数据，比如上述例子中的zhangsan@163.com。同样，就算前缀索引是index(email(100))，已经包含了整个字符串长度，也还是需要回表，因为系统并不确定前缀索引的定义是否截断了完整信息

但是，假如说前缀索引是index(email(10))的话，那么zhangsan@163.com就不匹配了，所以合理的长度的前缀索引就可以做到既节省空间，又不用额外增加太多的查询成本。

##### 建立合理的前缀索引
主要看数据的区分度，区分度越高越合理，用如下语句就能确定区分度：

	select 
	  count(distinct left(email,4)）as L4,
	  count(distinct left(email,5)）as L5,
	  count(distinct left(email,6)）as L6,
	  count(distinct left(email,7)）as L7,
	from xxxx

值越接近1，区分度越高

##### 一些使用前缀索引的技巧
以身份证号来举例子

1. 倒序存储
	
	身份证号前6位是地址码，同一个地区的身份证号一般都相同，这是我们可以多建一列（或者在查询的使用，用reverse函数把身份证重新反转过来，这样就不需要新增一列了），倒过来存储身份证号并在这一列建前缀索引，这样区分度就高了

2. 使用hash字段

	把身份证号进行hash，新增一列用来存储这个hash结果（比如用crc32，hash出来的结果就一个int）并建立索引，这样就算不用前缀索引，也能节省空间，但是会存在hash冲突，所以每次查询的时候，都需要把身份证号带上作为查询条件，如：
	```select * from t where id_card_hash = xxx and id_card = xxx```

如果使用了以上两种方法的话，就不会支持范围查找了，只能只能支持等值查询。因为索引记录的，是反转或者hash结果的顺序，而不再是身份证号顺序了

### explain

### 索引失效的情况
以下面的表结构举例

	CREATE TABLE `tradelog` (
  		`id` int(11) NOT NULL,
		`bid` int(10) NOT NULL,
  		`tradeid` varchar(32) DEFAULT NULL,
  		`operator` int(11) DEFAULT NULL,
  		`t_modified` datetime DEFAULT NULL,
  		PRIMARY KEY (`id`),
		KEY `bid` (`bid`)
  		KEY `tradeid` (`tradeid`),
  		KEY `t_modified` (`t_modified`)
	) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;


1. 条件字段使用函数或直接运算（注意是条件字段，也就是where子句 “=”号左边才是条件字段）

	如：

	```select * from tradelog where month(t_modified) = 10 ```

	```select * from tradelog where bid +1 = 1645803143```

2. 隐式类型转换

	如：

	```select * from tradelog where tradeid = 10```

	因为tradeid是varchar类型，而10是int类型，在字符串和整型比较的过程中，字符串会先转化为整型，然后再进行比较，所以上述语句其实相当于

	```mysql> select * from tradelog where  CAST(tradid AS signed int) = 110717```

	如何判断数据类型转换的规则？
	
	遇到这种情况的时候，可以自己找个例子，去数据库中做实验就好了，比如 select "10" > 9

	- 如果返回1，则说明是10 > 9，那么就是字符串转整型
	
	- 如果返回0，则说明是"10" > "9"，那么就是整型转字符串

3. 隐式字符编码转换

	如：

		CREATE TABLE `trade_detail` (
	  		`id` int(11) NOT NULL,
	 		`tradeid` varchar(32) DEFAULT NULL,
	  		`trade_step` int(11) DEFAULT NULL, /* 操作步骤 */
	  		`step_info` varchar(32) DEFAULT NULL, /* 步骤信息 */
	  		PRIMARY KEY (`id`),
	  		KEY `tradeid` (`tradeid`)
		) ENGINE=InnoDB DEFAULT CHARSET=utf8

		insert into tradelog values(1, 'aaaaaaaa', 1000, now());
		insert into tradelog values(2, 'aaaaaaab', 1000, now());
		insert into tradelog values(3, 'aaaaaaac', 1000, now());
		 
		insert into trade_detail values(1, 'aaaaaaaa', 1, 'add');
		insert into trade_detail values(2, 'aaaaaaaa', 2, 'update');
		insert into trade_detail values(3, 'aaaaaaaa', 3, 'commit');
		insert into trade_detail values(4, 'aaaaaaab', 1, 'add');
		insert into trade_detail values(5, 'aaaaaaab', 2, 'update');
		insert into trade_detail values(6, 'aaaaaaab', 3, 'update again');
		insert into trade_detail values(7, 'aaaaaaab', 4, 'commit');
		insert into trade_detail values(8, 'aaaaaaac', 1, 'add');
		insert into trade_detail values(9, 'aaaaaaac', 2, 'update');
		insert into trade_detail values(10, 'aaaaaaac', 3, 'update again');
		insert into trade_detail values(11, 'aaaaaaac', 4, 'commit');
		
		select d.* from tradelog l, trade_detail d where d.tradeid=l.tradeid and l.id=2;

	因为tradelog表字符集是utf8mb4,trade_detail表的字符集是utf8，所以当对trade_detail表进行搜索的时候，就会变成

	``` select * from trade_detail  where CONVERT(traideid USING utf8mb4)="aaaaaaab" /* aaaaaaab是id为2的tradeid的值 */ ```

	因为对条件字段进行函数操作，所以导致trade_detail表的索引失效

	utf8mb4是utf8的超集。类似地，在程序设计语言里面，做自动类型转换的时候，为了避免数据在转换过程中由于截断导致数据错误，也都是“按数据长度增加的方向”进行转换的。

	所以，只要我们把语句改写成

	```select d.* from tradelog l , trade_detail d where d.tradeid=CONVERT(l.tradeid USING utf8) and l.id=2```

	trade_detail就可以走索引了

## 锁
根据加锁的范围，MySQL 里面的锁大致可以分成全局锁、表级锁和行锁三类，其中全局锁、表级锁都是在server层实现的，而行级锁则是在存储引擎层实现的（myisam就没有行锁）

### 全局锁
全局锁就是对整个数据库实例加锁。MySQL 提供了一个加全局读锁的方法：

```Flush tables with read lock```

全局锁的典型使用场景是，做全库逻辑备份，但是一般用于没有提供事务功能的存储引擎，比如myisam

而有事务功能的存储引擎则可以使用官方自带的逻辑备份工具是mysqldump。当 mysqldump 使用参数–single-transaction 的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。而由于 MVCC 的支持，这个过程中数据是可以正常更新的。

### 表级锁
MySQL里面表级别的锁有两种：一种是表锁，一种是元数据锁（meta data lock，MDL)

#### 表锁
就是为表加读锁写锁，其语法是

	lock tables 表名 read // 加读锁
	lock tables 表名 write // 加写锁

如果线程A执行了```lock tables t1 read, t2 write```语句，那么其他线程写t1、读写t2的语句都会被阻塞。同时，线程A在执行```unlock tables```之前，也只能执行读t1、读写t2的操作。连写t1都不允许，自然也不能访问其他表。

#### 元数据锁（meta data lock，MDL)
MDL不需要显式使用，在访问一个表的时候会被自动加上。

当对一个表做增删改查（DML）操作的时候，加MDL读锁，当要对表做结构变更（DDL）操作的时候，加MDL写锁。

- 读锁之间不互斥，因此可以有多个线程同时对一张表增删改查

- 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的
- 安全性。因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行

##### MDL作用
防止DDL和DML并发的冲突，比如说一个查询正在遍历一个表中的数据，而执行期间另一个线程对这个表结构做变更，删了一列，那么查询线程拿到的结果跟表结构对不上了。

##### online ddl
在执行ddl语句的时候，获取到mdl写锁，那么该表的其他dml语句都会被阻塞，为了online ddl就是为了优化这种情况（截止到mysql8.0位置，创建全文索引和空间索引还是会阻塞其他dml语句）

online ddl执行流程简单概括如下：

1. 拿MDL写锁

2. 降级成MDL读锁

3. 真正做DDL

4. 升级成MDL写锁

5. 释放MDL锁

这样在做ddl这段时间内，该表的dml语句都能正常执行，因为已经降级成mdl读锁了。我们可以分析下这个流程有无DDL和DML并发冲突的问题

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220204182041509-665672210.png)

1. 在时刻t1，事务A获取mdl写锁，如果有其他mdl读锁，那么此时会阻塞。
2. 在时刻t2，事务B执行dml语句，因为事务A持有mdl写锁，所以阻塞。
3. 在时刻t3，事务A降级为mdl读锁，开始执行ddl语句
4. 在时刻t4，因为事务A降级为mdl读锁，事务B可以开始执行dml语句
5. 在时刻t5，事务A执行完ddl语句了，升级为mdl写锁，但是因为事务A持有mdl读锁，所以会阻塞
6. 在时刻t6，事务B执行完dml语句，因为此时事务A的ddl语句还没执行，所以事务A的ddl语句和事务B的dml语句并无冲突
7. 在时刻t7，因为事务B释放了mdl读锁，所以事务A能继续执行

但是，就算有了online ddl，在时刻t6，如果事务B一直在执行的话，因为事务A升级了mdl写锁，所以之后的其他语句都会被阻塞。

因此，在执行ddl语句的时候，最好用以下语句查下数据库中有无长事务在执行。

```select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60```

下面来对比下改表锁ddl和online ddl流程，分别以alter table A engine=InnoDB举例

###### 改表锁的ddl流程
MySQL5.5版本之前，每次执行dml语句的时候，都会锁ddl语句，为什么要锁？我们可以先看下ddl的流程，如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220208011739403-1080603706.png)

1. 新建一个临时表tmp，然后往临时表插入A表的数据，途中灰色空白代表内存页空洞，所以在执行alter语句的时候，相当于把原来表的数据页存在的空洞去掉
2. tmp表插入完A表数据后，然后和A表交换表名，这样tmp表就会替换掉A表,最后删除掉旧表

试想一下，假设tmp表已经插入到了R5的数据，但是现在突然之间，有来了条R7的数据，需要插入到R2和R3之间，那么这条数据在tmp表是不是就丢失了，tmp表替换掉A表后，自然而然R7这条数据就丢了，所以在执行ddl语句的时候，不得不把dml语句个阻塞掉

###### online ddl流程
而在MySQL5.6版本之后，就引入了online ddl，接下来，我们来看看online ddl是如何解决这个问题的，我们先来看下online ddl的具体流程，如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220208011758047-1569672356.png)

1. 建立一个临时文件，扫描表A主键的所有数据页
2. 用数据页中表A的记录生成B+树，存储到临时文件中
3. 生成临时文件的过程中，将所有对 A 的操作记录在一个日志文件（row log）中，对应的是图中 state2 的状态
4. 临时文件生成后，将日志文件中的操作应用到临时文件，得到一个逻辑数据上与表 A 相同的数据文件，对应的就是图中 state3 的状态
5. 用临时文件替换表 A 的数据文件

online ddl通过引入了一个row log的日志文件，最后在临时表上重放新操作来解决问题的。

online ddl其中有一步操作是把mdl写锁退化成读锁，为的就是在拷贝数据到临时表的过程中（也是ddl最耗时的过程）能进行dml操作

注意一下，改表锁的ddl流程，数据导出来的存放位置叫作tmp_table。这是一个临时表，是在server层创建的，而online ddl拷贝出来的数据是放在“tmp_file”里的，这个临时文件是 InnoDB 在内部创建出来的。整个DDL过程都在InnoDB内部完成

所以，如果现在有一个1TB的表需要做ddl操作，但是磁盘间就只有1.2TB，这种情况是不能执行ddl的，因为tmp_file也是要占用临时空间的

##### 关于online和inplace
online ddl对于 server 层来说，没有把数据挪动到临时表，是一个“原地”操作，这就是“inplace”名称的来源

在我们使用```alter table xxx engine = innodb```语句的时候，其实有一个隐藏值ALGORITHM=INPLACE（版本不同，alter的操作的这个值也不同）

如上述的两个流程中，如果是使用在server层创建临时表的方式，ALGORITHM=COPY

```alter table t engine=innodb,ALGORITHM=copy```

而如果是使用在innodb存储引擎中生成临时文件的方式的话，则是ALGORITHM=INPLACE

```alter table t engine=innodb,ALGORITHM=inplace```

具体mysql版本对应的alter操作是怎样的，可以参考官方文档

[mysql5.6](https://dev.mysql.com/doc/refman/5.6/en/innodb-online-ddl-operations.html)

[mysql5.7](https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl-operations.html)

[mysql8.0](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html)

当然，虽然看起来inplace和online是相同的，但是假如添加的是全文索引，虽然ALGORITHM=INPLACE，但是由于会阻塞dml，却不是online的

所以这里总结一下两者的关系
1. 如果ddl是online的，那么一定是inplace
2. 如果ddl是inplace的，那么有可能不是online的（比如添加全文索引和空间索引，mysql官方文档上，Permits Concurrent DML为No的，都不是online）

#####如何判断是否在server层没有新建临时表
最直观的方法就是看在执行完alter语句后，rows affected 是否为0, 不为0就是server层新建了临时表,如下入所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220211011234973-1280427830.jpg)



### 行锁
#### 两阶段锁协议
在InnoDB事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是两阶段锁协议。

#### 死锁
##### 死锁策略
- 直接进入等待，直到超时（innodb_lock_wait_timeout参数控制，默认50s）

- 发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行（innodb_deadlock_detect设置为on，该参数5.7.15才有）

##### 查看死锁
- show engine innodb status

	查看最新的死锁日志（值能查到最新一条）

- innodb_print_all_deadlocks参数设置为on

	这样就可以把产生的死锁日志记录在error_log配置的错误日志文件里面

###### 死锁检测流程
如果事务需要在访问的行上加锁，而且该行上已经有锁存在，就会发起检测。

- 一致性读不会加锁，就不需要做死锁检测

- 并不是每次死锁检测都都要扫所有事务。比如某个时刻，事务等待状态是这样的：

	B在等A
    D在等C

	现在来了一个E，发现E需要等D，那么E就判断跟D、C是否会形成死锁，这个检测不用管B和A

### 间隙锁

以如下表结构为例子

	create table t(
		`id` int(10) not null,
		`a` int(10) not null,
		`b` int(10) not null,
		primary key (`id`)
		key idx_a(`a`)
	) engine=innodb;

	insert into t values(1,1,1),values(5,5,5),values(10,10,10);

以索引idx_a为例，会产生(-∞,1)(1,5)(5,10)(10,+∞)的间隙锁，而且间隙锁前后都是开区间，也就是(5,10)这个间隙锁并不包含a=5和a=10的记录

间隙锁之间不会冲突```select * from t where a = 3 for update``` 和```select * from t where a = 3 lock in share mode```分别在不同的事务中执行，是不会发生阻塞的

### next-key lock

就是间隙锁+行锁，还是以t表的idx_a索引为例，会产生(-∞,1](1,5](5,10](10,+∞]的next-key lock，next-key lock的区间是前开后闭的

### 查找哪个线程持有锁
创建如下表：

	CREATE TABLE `t` (
  		`id` int(11) NOT NULL AUTO_INCREMENT,
  		`city` varchar(16) NOT NULL,
  		`name` varchar(16) NOT NULL,
  		`age` int(11) NOT NULL,
  		`addr` varchar(128) DEFAULT NULL,
  		PRIMARY KEY (`id`),
  		KEY `city` (`city`)
	) ENGINE=InnoDB

sessionA：执行 ``` lock table t write /*持有mdl写锁*/ ```

sessionB：执行 ``` select * from t where id = 1 /*此时会被阻塞*/ ```

sessionC：执行 ``` show full processlist ``` 查看线程情况，如下所示：

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220226080953209-834760603.jpg)

因为只有3个线程，所以我们可以很快就知道：

sessionA的线程id是2422

sessionB的线程id是2423，在等待mdl锁

sessionC的线程id是2424

但是，在生产环境中，面对show full processlist输出的大量数据，我们可以这样快速定位到：

``` select * from sys.schema_table_lock_waits ```

![](https://img2022.cnblogs.com/blog/901559/202202/901559-20220226081815513-1587596291.jpg)

- blocking_pid：持有锁的线程id

- sql_kill_blocking_connecttion：kill掉持有锁的执行线程

### sql加锁分析

#### 前提条件

- mysql版本 <= 8.0.13

- 隔离级别：RR

#### 加锁规则

两个原则，两个优化和一个bug

- 原则1：加锁的基本单位是next-key lock，next-key lock的加锁区间是前开

- 原则2：查找过程中访问到的对象才会加锁

- 优化1：索引上的等值查询，给唯一索引加锁的时候，next-key lock会退款为行锁

- 优化2：索引上的等值查询，向右遍历时且最后一个值不满足条件的时候，next-key lock退化为间隙锁

- bug1：唯一索引上的范围查询会访问到不满足条件的第一个值为止

#### sql分析

测试表结构和数据如下所示：

	CREATE TABLE `t` (
  		`id` int(11) NOT NULL,
  		`c` int(11) DEFAULT NULL,
  		`d` int(11) DEFAULT NULL,
  		PRIMARY KEY (`id`),
  		KEY `c` (`c`)
	) ENGINE=InnoDB;
 
	insert into t values(0,0,0),(5,5,5),(10,10,10),(15,15,15),(20,20,20),(25,25,25);

##### 案例1：等值查询间隙锁

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220310001056598-1716678645.png)

首先分析session A，因为是update语句，所以加的是写锁，因为只是更新d字段，所以只涉及到主键索引，不是涉及到普通索引c的加锁

根据原则1，可以得知next-key lock的范围是(5,10]

虽然主键索引是唯一索引，但是因为id=7是不存在的数据，所以优化1不适用

根据优化2，因为id=7向右遍历最后一个不满足条件的值是10，而且next-key lock需要退化为间隙锁，所以最终锁区间为（5,10）

因此session B插入id=8的记录落在锁区间(5,10)会阻塞，而session C修改id=10的记录不会阻塞

#### 案例2：非唯一索引的等值查询

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220310002855283-1273745012.png)

首先先分析session A，因为session A加的是读锁，而且走的是覆盖索引，并不需要回表查询，根据原则2，索引锁只涉及到普通索引c，主键索引不需要加锁

注意，如果加的是写锁的话（比如说for update），系统会认为你是需要更新数据，只要是更新数据，那么主键索引就必须要加写锁（因为主键上包括了所有字段，更新必然涉及到主键索引的查询）

根据原则1,索引c的next-key loc范围是(0,5]

因为c是普通索引，根据非唯一索引的查找规则，是要找到第一个不符合条件的值查询才停下来，再加上根据优化2，因为c=5向右遍历最后一个不满足的值的条件是10，而且next-key lock需要退化为间隙锁，所以最终锁区间为(0,5] (5,10)

根据优化2，因为session B只涉及到主键索引的查询，索引没有锁冲突，不会被阻塞，而session C插入c=7的记录落在锁区间(5,10)则会被阻塞

#### 案例3：主键索引范围锁

先看下下面两条查询语句

	select * from t where id=10 for update;

	select * from t where id>=10 and id<11 for update;

虽说两条语句的查询结果是一样的，但是加锁的区间范围却不同

先来看第一句，根据原则1，id=10的next-key lock锁区间为(5,10]，因为优化1，唯一索引上的等值查询，next-key lock退化为行锁，所以最终加锁范围为id=10

在来看第二句，为了方便理解，可以先把该查询看成id=10和id>10 and id < 11

id=10分析和第一句一样，最终加锁范围为id=10

而id > 10 and id < 11，因为是范围查找，所以会查找到id=15才停止，再根据原则1，此时的next-key lock加锁区间为(10,15]

所以最终第二句的加锁范围为id=10和(10,15]

#### 案例4：非唯一索引范围锁

先看下下面两条查询语句

	select * from t where c=10 for update;

	select * from t where c>=10 and id<11 for update;

与案例3相比，此时查询字段为不同索引c，不满足优化1，向右遍历最后一个值满足等值条件，不满足优化2，所以并有没有优化规则，此时加锁范围为(5,10]而不是c=10

其余分析与案例3一样

#### 案例5：唯一索引范围锁bug

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220311063145129-1060881017.png)

和案例3，4一样，可以先把sessionA的查询看成是id>10 and id<15和id=15，首先看id>10 and id<15，根据原则1，因为是范围查询，并无任何优化，所以锁区间为(10,15]

在看id=15，如果是等值查询的话，根据优化1会退化成行锁，之锁id=15，但是sessionA是范围查询，所以优化规则就不适用了，再根据一个bug，唯一索引上的范围查询会访问到不满足条件的第一个值为止，其实此时会查到id=20才会停止，所以根据原则1，此时id=15的锁范围为id=15和(15,20]

注意，这里跟案例3不同的是，id=15是实际存在的值，是因为用一个bug才导致需要查询到id=20，而案例3中id<11是不存在的值，正常查询是会查询到id=15，所以用原则1来分析。

案例3和案例5看着有点相似，实际上加锁区间分析是有点区别的

#### 案例6：非唯一索引上存在等值的例子

为了更好的说明间隙锁的概念，先插入一行数据

```insert into t values(30,10,30);```

此时索引c如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220311072811535-610774565.png)

虽然c=10的值是相同的，但是(c=10,id=10)和(c=10,id=30)两条记录之间其实也是有间隙的

先看下案例

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220311072743459-505405913.png)

session A向右遍历记录，先找到(c=10,id=10)的记录，根据原则1，此时加锁区间为((c=5,id=5),(c=10,id=10)]

接着找到(c=10,id=30)的记录，根据原则1，此时加锁区间为((c=10,id=10),(c=10,id=30)]

最后查询到(c=15,id=15)，才停止查询，根据原则1和优化2，next-key lock退化为间隙锁，所以锁区间((c=10,id=30),(c=15,id=15))

所以，sessionA的加锁区间范围是((c=5,id=5),(c=10,id=10)]和 ((c=10,id=10),(c=10,id=30)]和 ((c=10,id=30),(c=15,id=15))，如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220311074737431-1326447727.png)

#### 案例7：limit语句加锁

继续说回案例6，但是sessionA的语句改为 ```delete from t where c = 10 limit 2```，因为限制了limit2，所以只要查询到满足条数后，sql就会停止查询，所以此时sql执行只会查询到(c=10,id=30)就停止了，因此因此并不会加上(c=10,id=30)(c=15,id=15)的间隙锁

此时加锁区间如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220312000255821-1022256389.png)

此时加错区间为((c=5,id=5),(c=10,id=10)]和 ((c=10,id=10),(c=10,id=30)]

#### 案例8：一个死锁例子

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220312001138817-1086889553.png)

sessionA的加锁范围是(5,10)和c=10和(10,15)

sessionB的加锁范围是也是(5,10)和c=10和(10,15)

既然sessionB被阻塞住了，那么不是还没成功申请next-key lock吗？为啥 ```insert into t values(8,8,8)``` 会死锁

那是因为next-key lock = 间隙锁+行锁，而申请next-key lock的时候，也是分两步申请的，我们来详细分析下sessionA和sessionB的加锁过程

首先sessionA 是先申请(5,10)间隙锁，申请成功了，再申请c=10行锁，最后申请(10,15)间隙锁

而sessionB 也是先申请(5,10)间隙锁，因为间隙锁之前并不会冲突，所以不会阻塞，而当申请(5,10)间隙锁成功后，再申请c=10行锁的时候，发现sessionA持有c=10的行锁，因此进入锁等待，阻塞住了

此时sessionA再执行insert语句，因为sessionB持有(5,10)间隙锁，因此insert语句又会进入锁等待被阻塞从而导致了死锁

这个例子很好的说明了，申请next-key lock的时候，并不是一起申请，而是间隙锁和行锁分开申请

## 主从

### binlog的格式

为了便于描述，下面的讲解将用以下表和数据作为例子

	Create Table: CREATE TABLE `t11` (
  		`id` int(10) NOT NULL AUTO_INCREMENT,
  		`a` int(10) NOT NULL DEFAULT '0',
  		`b` int(10) NOT NULL DEFAULT '0',
  		PRIMARY KEY (`id`),
  		KEY `idx_a` (`a`),
  		KEY `idx_b` (`b`)
	) ENGINE=InnoDB

	insert into t11 values(null,1,100);
	insert into t11 values(null,2,99);
	insert into t11 values(null,3,98);
	insert into t11 values(null,4,97);
	insert into t11 values(null,5,96);
	insert into t11 values(null,6,95);

通过 ```show master status``` 可以获取当前的binlong文件名

#### statement

先执行

```delete from t11 where a>=4 and b<=97 limit 1;```

然后在执行

```show binlog events in mysql-bin.001013```

结果如下图所示

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220331002444348-1975916999.jpg)

## 一些经验总结

如果我们想要kill掉某个会话，那么我们怎么知道这个会话是否在执行事务呢

以一下表来举例

	CREATE TABLE `test` (
  		`id` int(11) NOT NULL,
  		`c` int(11) DEFAULT NULL,
  		`d` int(11) DEFAULT NULL,
  		PRIMARY KEY (`id`),
  		KEY `c` (`c`)
	) ENGINE=InnoDB

现在分别开启两个会话sessionA和sessionB

sessionA：

```insert into test values(50,50,50)```

sessionB:

什么也不执行

新开一个会话C，执行 ```show full processlist``` 输出如下图所示：

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220316002531353-1563415279.jpg)

会话50和51中，肯定是有一个在执行事务的，但是command都是sleep，info也没输出sql，到底怎么知道insert语句在哪个会话执行

这时候可以查询 ```information_schema.innodb_trx```，这个表记录着的是正在执行的事务

执行 ```select * from information_schema.innodb_trx where trx_mysql_thread_id in(50, 51)\G``` 输出结果如下所示：

![](https://img2022.cnblogs.com/blog/901559/202203/901559-20220316002544685-1141318811.jpg)

正在执行事务的是会话50,51是没有执行任何东西的，所以需要kill掉会话的话可以先kill掉会话51
