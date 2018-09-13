-   [InterviewFAQ-database](#interviewfaq-database)
    -   [数据库](#数据库)
        -   [索引](#索引)
        -   [乐观锁和悲观锁](#乐观锁和悲观锁)
        -   [多版本并发控制（MVCC）](#多版本并发控制mvcc)
        -   [存储引擎](#存储引擎)
        -   [优化步骤](#优化步骤)
        -   [主从复制](#主从复制)
        -   [高可用架构（MHA）](#高可用架构mha)
        -   [读写分离](#读写分离)
        -   [配置参数与调优](#配置参数与调优)
        -   [锁](#锁)
        -   [备份](#备份)
        -   [常用的SQL语句](#常用的sql语句)
        -   [Memcached](#memcached)
        -   [Redis](#redis)

InterviewFAQ-database
=====================

数据库
------

### 索引

- 特点

    -   a）避免进行数据库全表的扫描，大多数情况，只需要扫描较少的索引页和数据页，而不是查询所有数据页。而且对于非聚集索引，有时不需要访问数据页即可得到数据。

        b）聚集索引可以避免数据插入操作，集中于表的最后一个数据页面。

        c）在某些情况下，索引可以避免排序操作。

- MySQL数据库支持多种索引类型，如BTree索引，哈希索引，全文索引

    -   MySQL就普遍使用B+Tree实现其索引结构。

- 为什么索引提高查询效率，一定提升吗？

    -   因为索引虽然加快了查询速度，但索引也是有代价的：索引文件本身要消耗存储空间，同时索引会加重插入、删除和修改记录时的负担，另外，MySQL在运行时也要消耗资源维护索引，因此索引并不是越多越好。
    -   一般两种情况下不建议建索引。表记录比较少 、索引的选择性较低
        （指不重复的索引值（也叫基数Cardinality）与表记录数（\#T）的比值
        ）的情况下不建议建索引 。

- 建立索引的几条规则，索引的数据结构？

    -   在使用InnoDB存储引擎时，如果没有特别的需要，请永远使用一个与业务无关的自增字段作为主键。
    -   不建议使用过长的字段作为主键，因为所有辅助索引都引用主索引，过长的主索引会令辅助索引变得过大。

- b类树的特点

    -   首先从根节点进行二分查找，如果找到则返回对应节点的data，否则对相应区间的指针指向的节点递归进行查找，直到找到节点或找到null指针，前者查找成功，后者查找失败。
    -   由于插入删除新的数据记录会破坏B-Tree的性质，因此在插入删除时，需要对树进行一个分裂、合并、转移等操作以保持B-Tree性质。
    -   B+Tree比B-Tree更适合实现外存储索引结构
    -   索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储的磁盘上。索引的结构组织要尽量减少查找过程中磁盘I/O的存取次数。

- InnoDB存储引擎的索引和其他的索引有什么区别？

    - MyISAM索引文件和数据文件是分离的，索引文件仅保存记录所在页的指针（物理位置），通过这些地址来读取页，进而读取被索引的行。

    - InnoDB存储引擎采用"聚集索引"的数据存储方式实现B-Tree索引，所谓"聚集"，就是指数据行和相邻的键值紧凑地存储在一起，注意InnoDB只能聚集一个叶子页（16K）的记录（即聚集索引满足一定的范围的记录），因此包含相邻键值的记录可能会相距甚远。

    - 主键：

        - 主键索引既存储索引值,又在叶子中存储行的数据
        - 如果没有主键,则会Unique key做主键 
        - 如果没有unique,则系统生成一个内部的rowid做主键
        - 像innodb中,主键的索引结构中,既存储了主键值,又存储了行数据,这种结构称为"聚簇索引"

    - 推荐: http://tech.meituan.com/mysql-index.html

        [MySQL索引背后的数据结构及算法原理](http://blog.codinglabs.org/articles/theory-of-mysql-index.html)

### 乐观锁和悲观锁

悲观锁：假定会发生并发冲突，屏蔽一切可能违反数据完整性的操作

乐观锁：假设不会发生并发冲突，只在提交操作时检查是否违反数据完整性。

乐观锁与悲观锁的具体区别: [看这里](http://www.cnblogs.com/Bob-FD/p/3352216.html)

### 多版本并发控制（MVCC）

-   MySQL的innodb引擎是如何实现MVCC的？
    -   innodb会为每一行添加两个字段，分别表示该行**创建的版本**和**删除的版本**，填入的是事务的版本号，这个版本号随着事务的创建不断递增。在repeated
        read的隔离级别（[事务的隔离级别请看这篇文章](http://blog.csdn.net/chosen0ne/article/details/10036775)）下，具体各种数据库操作的实现：
        -   select：满足以下两个条件innodb会返回该行数据：
        -   该行的创建版本号小于等于当前版本号，用于保证在select操作之前所有的操作已经执行落地。
        -   该行的删除版本号大于当前版本或者为空。删除版本号大于当前版本意味着有一个并发事务将该行删除了。
    -   insert：将新插入的行的创建版本号设置为当前系统的版本号。
    -   delete：将要删除的行的删除版本号设置为当前系统的版本号。
    -   update：不执行原地update，而是转换成insert +
        delete。将旧行的删除版本号设置为当前版本号，并将新行insert同时设置创建版本号为当前版本号。
    -   其中，写操作（insert、delete和update）执行时，需要将系统版本号递增。
    -   由于旧数据并不真正的删除，所以必须对这些数据进行清理，innodb会开启一个后台线程执行清理工作，具体的规则是将删除版本号小于当前系统版本的行删除，这个过程叫做purge。通过MVCC很好的实现了事务的隔离性，可以达到repeated
        read级别，要实现serializable还必须加锁。参考：[MVCC浅析](http://blog.csdn.net/chosen0ne/article/details/18093187)

### 存储引擎

-   MyISAM
    适合于一些需要大量查询的应用，但其对于有大量写操作并不是很好。甚至你只是需要update一个字段，整个表都会被锁起来，而别的进程，就算是读进程都无法操作直到读操作完成。另外，MyISAM
    对于 SELECT COUNT(\*) 这类的计算是超快无比的。
-   InnoDB 的趋势会是一个非常复杂的存储引擎，对于一些小的应用，它会比
    MyISAM 还慢。他是它支持"行锁"
    ，于是在写操作比较多的时候，会更优秀。并且，他还支持更多的高级应用，比如：事务。
-   [mysql 数据库引擎](http://www.cnblogs.com/0201zcr/p/5296843.html)
-   [MySQL存储引擎－－MyISAM与InnoDB区别](https://segmentfault.com/a/1190000008227211)
-   InnoDB引擎
    -   关键特性
        -   支持事务，行锁设计、支持外键，并支持类似于Oracle的一致性非锁定读，即默认操作不会产生锁。
        -   通过使用所版本并发控制（MVCC）来获得高并发性，并且实现了SQL标准的4种隔离级别，默认为REPEATABLE级别。同时使用next-key locking的策略来避免幻读现象的产生。
        -   插入缓冲：提高非聚集索引插入的性能，应用于非唯一辅助索引的插入操作
            -   对于非聚集索引的插入或更新操作，不是每一次直接插入到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，若在，则直接插入，若不在，则先放入到一个Insert Buffer对象中，然后再以一定的频率和情况进行Insert Buffer和辅助索引页子节点的合并操作，这时通常能将多个插入操作合并到一个操作中。
        -   两次写：可靠性的提升，在应用重做日志前，用户需要一个页的副本，当写入失效发生时，先通过页的副本还原该页，再进行重做。
            -   组成部分
                -   内存中的doublewrite buffer，大小为2MB
                -   物理磁盘上共享表空间中连续的128个页，大小为2MB
            -   实现步骤
                -   在对缓冲池的脏页进行刷新时，并不直接写磁盘，而是通过memcpy函数将脏页复制到内存中的doublewrite buffer；
                -   通过doublewrite buffer分两次，每次1MB顺序地写入共享表空间的物理磁盘上，然后马上调用fsync函数，同步磁盘，避免缓冲写带来的问题。（doublewrite页是连续的，因此这个过程是顺序写的，开销不是很大）
                -   完成doublewrite页的写入后，再将doublewrite buffer中的页写入各个表空间文件中。（此时的写入是离散的）
        -   自适应哈希索引：
            -   InnoDB存储引擎会监控对表上各索引页的查询，如果观察到建立哈希索引可以带来速度提升，则建立哈希索引。
            -   通过缓冲池的B+树页构造而来，因此建立的速度很快。
            -   存储引擎会自动根据访问的频率和模式来自动地为某些热点页建立哈希索引。
            -   哈希索引只能用来搜索等值的查询，范围查找不适用
        -   预读
        -   聚集
        -   异步IO
        -   刷新邻接页
            -   当刷新一个脏页时，存储引擎会检测该页所以在的区的所有页，如果是脏页，那么一起进行刷新。
    -   体系架构
        -   后台线程
            -   功能
                -   刷新内存池中的数据，保证缓冲池中的内存缓存的是最近的数据
                -   将已经修改的数据文件刷新到磁盘文件
                -   在数据库发生异常的情况下InnoDB能恢复到正常状态
            -   类型
                -   Master Thread：主要负责将缓冲池中的数据异步刷新到磁盘，具有最高的线程优先级
                    -   脏页的刷新（1.2版本之前）
                    -   合并插入缓冲
                    -   UNDO页的回收（1.1版本之前）
                -   IO Thread：主要负责异步IO请求的回调处理
                    -   insert buffer thread
                    -   log thread
                    -   read thread
                    -   write thread
                -   Purge Thread：回收已经使用并分配的undo页
                -   Page Cleaner Thread：脏页的刷新，减轻原Master Thread的工作及对于用户查询线程的阻塞。
        -   内存池
            -   功能
                -   维护所有进程/线程需要访问的多个内部数据结构
                -   缓存磁盘上的数据
                -   重做日志缓冲
            -   组成
                -   缓冲池
                    -   索引页
                    -   数据页
                    -   undo页
                    -   插入缓冲
                    -   自适应哈希索引
                    -   锁信息
                    -   数据字典信息
                -   重做日志缓冲
                    -   会将重做日志缓冲刷新到日志文件
                        -   一般情况下Master Thread每一秒钟刷新一次
                        -   每个事务提交时会刷新
                        -   重做日志缓冲池剩余空间小于1/2时刷新
                -   额外的内存池
                    -   通过内存堆的方式进行内存管理
                    -   在对一些数据结构本身的内存进行分配时，需要从额外的内存池中进行申请，当该区域的内存不够时，会冲缓冲池中进行申请。
            -   LRU算法（最近最少使用算法）
                -   midpoint：在LRU列表长度的5/8处
                -   LRU列表：管理缓冲池中页的可用性
            -   checkpoint机制
                -   Flush列表：管理将页刷新回磁盘
                -   作用
                    -   缩短数据库的恢复时间：数据库只需要对checkpoint后的重做日志进行恢复
                    -   缓冲池不够用时，将脏页刷新到磁盘
                    -   重做日志不可用时，刷新脏页
                -   关键点
                    -   每次刷新多少页
                    -   每次从哪里取脏页
                    -   什么时间触发Checkpoint
                -   类型
                    -   Master Thread Checkpoint
                    -   FLUSH_LRU_LIST Checkpoint：LRU列表空闲页不足时
                    -   Async/Sync Flush Checkpoint：重做日志文件不可用时，保证重做日志循环使用的可用性
                    -   Dirty Page too much Checkpoint：缓冲池中的脏页数量达到阈值时
        -   文件
            -   参数文件
            -   日志文件
            -   socket文件
            -   pid文件
            -   MySQL表结构文件
            -   存储引擎文件
    -   MyISAM引擎
        -   不支持事务
        -   缓冲池只缓冲索引文件，而不缓存数据文件
-   该如何选用两个存储引擎?
    -   因为MyISAM相对简单所以在效率上要优于InnoDB.如果系统读多，写少。对原子性要求低。那么MyISAM最好的选择。且MyISAM恢复速度快。可直接用备份覆盖恢复。
    -   如果系统读少，写多的时候，尤其是并发写入高的时候，InnoDB就是首选了。

### 优化步骤

-   通过数据库分片来解决数据库写扩展的问题，避免单点故障以及写操作成为瓶颈
-   利用主从复制解决读的问题，通过多个从服务器来应对读操作

### 主从复制

+ 主从复制的原理：
  + 主服务器把数据更新记录到二进制日志(binary log)中；
  + 从服务器把主服务器的二进制日志复制到自己的中继日志(relay log)中（由从服务器的IO线程负责）；
  + 从服务器重做中继日志中的事件，把改变应用在自己的数据库上（由从服务器的SQL线程负责）。
+ 主从复制过程：
  - 从数据库，执行start slave开启主从复制。
  - 从数据库IO线程会通过主数据库授权的用户请求连接主数据库，并请求主数据库的binlog日志的指定位置，change master命令指定日志文件位置。
  - 主数据库收到IO请求，负责复制的IO线程跟据请求读取的指定binlog文件返回给从数据库的IO线程，返回的信息除了日志文件，还有本次返回的日志内容在binlog文件名称和位置。
  - 从数据库获取的内容和位置（binlog），写入到（从数据库）relaylog中继日志的最末端，并将新的binlog文件名和位置记录到master-info文件，方便下次读取主数据库的binlog日志，指定位置，方便定位。
  - 从数据库SQL线程，实时检测本地relaylog新增内容，解析为SQL语句，执行。
+ 如何判断MySQL主从是否同步？
  - 获取到MySQL的主从状态信息
    - SHOW SLAVE STATUS\G
  - 查看以下三个参数的值
    - Slave_IO_Running    I/O线程是否被启动并成功地连接到主服务器上。（状态信息为Yes No）
    - Slave_SQL_Running    SQL线程是否被启动（状态信息为Yes No）
    - Seconds_Behind_Master    测量SQL线程和I/O线程的时间差（即延迟，单位为秒）
    - 对于简单的监测SQL是否同步，只需要第一和第二个参数都为Yes的情况下，并且延迟小于一定数值的时候，我们就可以认定mysql主从是同步的
+ MySQL如何减少主从复制延迟？
  - 主服务器方面：
    - 主库读写压力大，导致复制延迟 --> 架构的前端要加buffer及缓存层
    - 主从复制单线程，如果主库写并发太大，来不及传送到从库，就会导致延迟 --> 更高版本的mysql可以支持多线程复制
    - 慢SQL语句过多 --> 在架构上做优化，尽量让主库的DDL快速执行
  - 从服务器方面：
    - 从库硬件比主库差，导致复制延迟 --> 升级从服务器的硬件设备
    - 从服务器负载大 --> 使用多台slave来分摊读请求，再从这些slave中取一台专用的服务器，只作为备份用，不进行其他任何操作
    - 牺牲写从服务器的数据安全，将sync_binlog设置为0或者关闭binlog，将innodb_flushlog设置为0来提高sql的执行效率。
    - 两个可以减少延迟的参数:
      - slave-net-timeout=seconds 单位为秒 默认设置为 3600秒 #参数含义：当slave从主数据库读取log数据失败后，等待多久重新建立连接并获取数据
      - master-connect-retry=seconds 单位为秒 默认设置为 60秒 #参数含义：当重新建立主从连接时，如果连接建立失败，间隔多久后重试。
  - 网络延迟 --> 改变网络架构，或使用更好的网络服务

### 高可用架构（MHA）

-   管理节点：定时探测集群中的master节点，当master出现故障时，它可以自动将最新数据的slave提升为新的master，然后将所有其他的slave重新指向新的master。
-   数据节点：
-   选举过程
-   错误恢复：MHA试图从宕机的主服务器上保存二进制日志，最大程度的保证数据的不丢失，但这并不总是可行的。例如，如果主服务器硬件故障或无法通过ssh访问，MHA没法保存二进制日志，只进行故障转移而丢失了最新的数据。使用MySQL
    5.5的半同步复制，可以大大降低数据丢失的风险。MHA可以与半同步复制结合起来。如果只有一个slave已经收到了最新的二进制日志，MHA可以将最新的二进制日志应用于其他所有的slave服务器上，因此可以保证所有节点的数据一致性。

### 读写分离

-   MySQL Proxy
-   MyCat
    -   MyCat基于Cobar，MySQL通讯协议，代理服务器，无状态，容易部署，负载均衡。
    -   原理：应用服务器传SQL语句，路由解析，转发到不同的后台数据库，结果汇总，返回MyCat把逻辑数据库和数据表对应到物理真实的数据库、数据表，遮蔽了物理差异性。
    -   MyCat工作流程：
        - 应用服务器向MyCat发送SQL语句select * from user where id in(30, 31, 32)。
        - MyCat前端通信模块与应用服务器通信，交给SQL解析模块。
        - SQL解析模块解析完交给SQL路由模块。
        - SQL路由模块，id取模，余数为0：db1，余数为1：db2……
        - 把SQL拆解为select * from user where id in 30……交给SQL执行模块，对应db1 db2 db3。
        - SQL执行模块通过后端，分别在db1 db2 db3执行语句，返回结构到数据集合合并模块，然后返回给应用服务器。

### 配置参数与调优

- 连接数 、会话数和线程数
  - max_connections 
  - max_connect_errors
    - 日志中会出现类似信息：blocked because of many connection errors 
    - 执行`mysqladmin flush-hosts`或者重启 MySQL服务，将错误计数器清零
  - thread_concurrency 
    - CPU核数的2倍 
  - max_allowed_packet 
  - key_buffer_size
    - 根据增大`Key_reads / Uptime` 来优化这个参数 
    - `mysqladmin ext -ri10 | grep Key_reads`
  - thread_cache_size
    - 此参数用来缓存空闲的线程，以至不被销毁 
    - `SHOW STATUS WHERE Variable_name LIKE '%Thread%';`
    - 建议设置成与threads_connected一样 。 
  - sort_buffer_size 
  - join_buffer_size 
  - query_cache_size 
  - read_buffer_size 
  - read_rndbuffer_size 
  - myisam_sortbuffer_size 
  - innodb_buffer_pool_size 
- 日志和事务
  - innodb_log_file_size 
  - innodb_log_buffer_size 
  - innodb_flush_log_at_trx_commit 
  - innodb_lock_wait_timeout 
- 软件优化
  - 选择合适的引擎
    - MyISAM 索引顺序访问方法 
    - InnoDB 事务型存储引擎 
  - 正确使用索引
    - 给合适的列表建立索引，给where子句，连接子句建立索引
  - 避免使用SELECT
    - 返回结果过多，降低查询的速度，会增大服务器返回给APP端的数据传输量
  - 字段尽量设置为NOT NULL
- 硬件优化
  - Linux内核用内存开缓存存放数据
    - 写文件：文件延迟写入机制，先把文件存放到缓存，达到一定程度写进硬盘。
    - 读文件：同时读文件到缓存，下次需要相同文件直接从缓存中取，而不是从硬盘取。
  - 增加应用缓存
    - 本地缓存：数据放到服务器内存的文件中。
    - 分布式缓存：Redis, Mencache 读写性能非常高，QPS（每秒查询请求数）每秒达到1W以上；数据持久化用Redis，不持久化两者都可以。
  - 用SSD代替机械硬盘
    - 日志和数据分开存储，日志顺序读写 – 机械硬盘，数据随机读写 – SSD
- 架构优化
  - 分表
    - 水平拆分：数据分成多个表拆分后的每张表的表头相同。
    - 垂直拆分：字段分成多个表。
    - 插入数据、更新数据、删除数据、查询数据时：MyISAM MERGE存储引擎，多个表合成一个表，InnoDB用alter table，变成MyISAM存储引擎，然后MEGRE。
    - 面试题：MERGE存储引擎将N个表合并，数据库中如何存储？答： 真实存储为N个表，表更大的话就需要分库了。
  - 读写分离
    - 把读和写拆开，对应主从服务器，主服务器写操作、从服务器是读操作。
  - 分库
- SQL慢查询分析、调参数
  - 慢查询：指执行超过一定时间的SQL查询语句记录到慢查询日志，方便开发人员查看日志
    - long_qeury_time：定义慢查询时间
    - slow_query_log：设置慢查询开关
    - slow_query_log_file：设置慢查询日志文件路径
    - /etc/sysconfig/network-scripts/ifcfg-eth0
    - IPADDR
    - GATEWAY
    - BOOTPROTO
    - /etc/sysconfig/network

### 锁

+ MySQL的innodb如何定位锁问题?
  + 通过查询information_schema数据库中innodb_locks表了解锁等待情况。
    + SELECT * FROM  Innodb_trx  \G;  # 当前运行的所有事务
    + SELECT * FROM  Innodb_locks  \G; # 当前出现的锁
    + SELECT * FROM Innodb_locks_waits \G ; # 锁等待的对应关系
  + 通过设置InnoDB Monitors观察锁冲突情况。
    + CREATE TABLE innidb_monitor(a INT) ENGINE=INNODB;
    + SHOW ENGINE INNODB STATUS

### 备份

-   mysql表备份(mysqldump)
-   表类型
    -   支持事务
        -   innodb可以在mysqladmin命令中加入single-transaction选项，生成一个快照来保证数据备份期间的一致性。
        -   停止从库，然后进行备份
    -   不支持事务
        -   MyISAM中为了保持数据的一致性，需要在备份之前加读锁操作，flush
            table with read lock
        -   mysqlhotcopy mydatabase backupdir
-   备份形式
    -   全备份
        -   mysqldump -u root -p -all -database \> all.sql
        -   mysqldump -u root -p mydatabase \> mydatabase.sql
        -   mysqldump -u -root -p mydatabase tablename1 tablename2 \>
            tablename.sql
    -   增量备份
        -   通过备份二进制日志来实现
-   备份时机
    -   选择应用负担小的时候
-   恢复测试

### 常用的SQL语句

- describe tablesname
- distinct
- order by xxx desc
- creat user identified by "123"
- gread/revoke xxx on xxx.* to xxx
- set password for xxx=password('123')
- select xxx from xxx
  - where
    - between xxx and xxx
    - is Null
    - and  or
    - like
      - % --> *
      - _ --> .
    - regexp 'xxx|xxx'
  - group by
  - having
  - order by limit
- inner join xxx on
- insert into xxx (xxx, xxx, ...)
- update xxx set xxx where
- delete xxx from xxx where
- creat table xxx (auto_increment/xxx)

### Memcached

-   一套分布式的快取系统
-   Memcached的API使用三十二位元的循环冗余校验（CRC-32）计算键值后，将资料分散在不同的机器上。
-   当表格满了以后，接下来新增的资料会以LRU机制替换掉
-   应用首先从Memcached中获得数据，获取不到再从数据库中获得并保存在Memcached中。
-   好的应用95%的数据从Memcache中获得，3%的数据来自MySQL的query
    cache中获得，剩下的2%采取查表。Cache is King

### Redis

-   Redis是什么？

    -   是一个完全开源免费的key-value内存数据库
    -   通常被认为是一个数据结构服务器，主要是因为其有着丰富的数据结构
        strings、map、 list、sets、 sorted sets

-   Redis数据库

    -   通常局限点来说，Redis也以消息队列的形式存在，作为内嵌的List存在，满足实时的高并发需求。在使用缓存的时候，redis比memcached具有更多的优势，并且支持更多的数据类型，把redis当作一个中间存储系统，用来处理高并发的数据库操作

        -   速度快：使用标准C写，所有数据都在内存中完成，读写速度分别达到10万/20万

        -   持久化：对数据的更新采用Copy-on-write技术，可以异步地保存到磁盘上，主要有两种策略，一是根据时间，更新次数的快照（save
            300 10 ）二是基于语句追加方式(Append-only file，aof)
        -   自动操作：对不同数据类型的操作都是自动的，很安全
        -   快速的主--从复制，官方提供了一个数据，Slave在21秒即完成了对Amazon网站10G
            key set的复制。
        -   Sharding技术：
            很容易将数据分布到多个Redis实例中，数据库的扩展是个永恒的话题，在关系型数据库中，主要是以添加硬件、以分区为主要技术形式的纵向扩展解决了很多的应用场景，但随着web2.0、移动互联网、云计算等应用的兴起，这种扩展模式已经不太适合了，所以近年来，像采用主从配置、数据库复制形式的，Sharding这种技术把负载分布到多个特理节点上去的横向扩展方式用处越来越多。

-   Redis缺点

    -   是数据库容量受到物理内存的限制,不能用作海量数据的高性能读写,因此Redis适合的场景主要局限在较小数据量的高性能操作和运算上。

    -   Redis较难支持在线扩容，在集群容量达到上限时在线扩容会变得很复杂。为避免这一问题，运维人员在系统上线时必须确保有足够的空间，这对资源造成了很大的浪费。
