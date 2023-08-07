# Lesson 01 TiDB 数据库架构概述

## 1. TiDB 体系架构

Application Via MySQL Protocol：应用程序，用户在这里输入SQL语句使用数据库。

TiDB Server：TiDB Sever主要负责SQL语句的生成及执行计划的生成，是无状态的数据，本身不存取数据。

TiKV：数据实际上存储在存储引擎StorageCluster中，其中有TiKV和TiFlash两种，其中存储的数据差不多，只不过TiKV行存，TiFlash列存。一个数据表被切割为小region，96-144M，一个region会创建多个副本，分布式存储在TiKV上，副本默认是3个。TiKV是将打散的数据存储并创建副本。TiKV还实现了分布式事务和MVCC。TiKV中的数据是行存。

TiFlash：也存储数据，存的数据本质上和TiKV一样，但是数据是列存。列存对于大表的统计分析很有帮助，所以 TiFlash适合进行大表的统计分析。

PD：整个集群的大脑，region和TiKV或TiFlash的对应关系，也就是数据库的元数据，存储在PD当中。也就是在接受SQL之后，要去找目标表在哪些TiKV上分布。还会获取SQL执行的开始时间。

TSO：TiDB当中的时间戳，随着时间的增长会增加，就像一个计时器，每一条SQL在执行时都会获得一个TSO。一个事务有两个TSO，一个是开始时间，一个是结束时间，TSO都是PD提供的。

- 水平扩容或者缩容：增加或减少节点，分布式。SQL处理能力不够？增加TiDB Server节点。存储能力不够？增加TIKV节点。
- 金融级高可用
- 实时HTAP
- 云原生的分布式数据库
- 兼容MySQL 5.7协议

![image-20230728091229114](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728091229114.png)

## 2. TiDB Server

TiDB Server服务器节点可以随时增加、回收，用户过多可以分流，可以横向进行扩展缩容操作。TiDBServer负责SQL的解析、编译、生成执行计划。TiDB当中存的是key：value键值对，所以TiDB Server还负责将数据转化为键值对，再插入到TiKV上，TiKV上负责存成region。TiDB还负责DDL语句的执行，垃圾回收机制等。

- 处理客户端的连接
- SQL语句的解析和编译
- 关系型数据与KV的转化（键值对）
- SQL语句的执行
- 执行online DDL
- 垃圾回收：数据库会存储历史版本，因此当历史数据过多会进行垃圾回收。

![image-20230728091645032](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728091645032.png)

## 3. TiKV 

rocksdbkv：通过rocksdb数据库保证数据持久化，rocksdbkv是将表数据转成键值对的形式存储再rocksdb实例中。

rocksdbraft：rocksdbraft存储指令，对于表的增删改都先存在rocksdbraft上，然后TiKV将指令应用到rocksdbkv上对表数据进行修改等。

Raft：Raft协议，一个region存在多个副本，其中有一个leader的角色，其他的副本跟着leader副本来改变，Raft协议将修改应用到leader的region之上，然后leader将自己的修改复制到其他的副本上，来实现数据一致性。也就是数据修改时修改了系统中多个region才算修改成功。

MCVV：MCVV协议，保证在修改的时候，不能进行其他操作，但是可以进行读取，但是读取的数据是修改前的数据。

Transaction：事务层，分布式事务模型，两阶段提交。

Coprocessor：算子下推，每一个节点都有CPU，所以每个节点都可以进行一部分的计算操作，并不是都要去TiDBServer上计算，在每个TiKV节点上都可以进行一些计算操作，比如过滤、投影、计算，是一个分布式计算的模型。

- 数据持久化：
- 副本的强一致性和高可用性
- MVCC（多版本并发控制）
- 分布式事务支持
- Coprocessor（算子下推）

![image-20230728091759697](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728091759697.png)

## 4. PD：Placement Driver

1. region的元数据存储在PD上，数据落地，提供很多全局的事情。
2. 分配全局ID和事务ID。
3. 在各种操作进行时会生成时间戳TSO。
4. 如果某个表的数据大部分存储在了一个TiKV上，那么性能瓶颈就会很小，导致操作时的性能较差，region和TiKV会以一定时间间隔发送信息给PD，PD会进行调度，使数据分布更平衡。
5. TiDB Dashboard：提供一些可观测的性能监控，都是在PD上完成。

- 整个集群TiKV的元数据存储
- 分配全局ID和事务ID
- 生成全局时间戳TSO
- 收集集群信息进行调度
- 提供TiDB Dashboard服务

![image-20230728092646377](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728092646377.png)

## 5. TiFlash

数据线存在TiKV中，默认存三分，有了TiFlash就会多一份，不过是列存版本，并且参与复制，对TiKV中数据的修改会同步到TiFlash中。列存为了提高分析效率。行存更多承载OLTP在线交易业务，TiFlash更多承载OLAP分析型业务。OLAP一般是暴力扫描，OLTP一般是事务。TiDB Server的SQL优化器提供智能选择功能，可以预测一个SQL是OLTP型还是OLAP型，会将SQL导向TiKV或TiFlash，也可以手动选择没目标存储器。所以TiDB支持了HTAP。

- 异步复制
- 一致性
- 列式存储提高分析查询效率
- 业务隔离
- 智能选择

![image-20230728093016643](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728093016643.png)

## 6. Summary

- 描述TiDB数据库的整体架构。
- 理解TiDB Server， TiKV，TiFlash，PD等组件的架构及功能。

## 测验

1.下列功能是由 TiKV 或 TiFlash 实现的为？（ 选 2 项 ）

A. 根据集群中 Region 的信息，发出调度指令

B. 对于 OLAP 和 OLTP 进行业务隔离

C. 将关系型数据转化为 KV 存储进行持久化

D. 将 KV 存储转化为关系型数据返回给客户端

E. 配合 TiDB Server 生成事务的唯一 ID

F. 副本的高可用和一致性

正确答案： B. 对于 OLAP 和 OLTP 进行业务隔离、F. 副本的高可用和一致性

A：PD；B：TiKV或TiFlash；C：TiDB Server；D：TiDB Server；E：PD；F：TiKV或TiFlash

2.关于 TiKV 或 TiDB Server，下列说法不正确的是？

 A. 数据被持久化在 TiKV 的 RocksDB 引擎中 

 B. 对于老版本数据的回收（GC），是由 TiDB Server 在 TiKV 上完成的 

 C. 两阶段提交的锁信息被持久化到 TiDB Server 中 

 D. Region 可以在多个 TiKV 节点上进行调度，但是需要 PD 节点发出调度指令 

正确答案： C. 两阶段提交的锁信息被持久化到 TiDB Server 中

两阶段提交的锁信息持久化到TiKV当中。TiDB Server本身不持久化信息。



# Lesson 2 TiDB Server

## 1. TiDB Server架构

![image-20230728173047874](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728173047874.png)

### 认识核心线程

Protocol Layer、Parse、Compile：负责核心SQL语句的解析、编译、优化，生成SQL语句的执行计划。

Executor、DistSQL、Transaction、KV：负责分批执行SQL执行计划，其中Transaction、KV负责和事务相关的一些操作。

PD Client、TiKV Client：负责与PD和TiKV相互之间进行交互。

schema load、worker、start job：负责online DDL（TiDB当中DDL语句不阻塞读写）

memBuffer：缓存区，用户缓存读取到的数据、元数据、登录信息等。

cache table：热点小表缓存，TiDB V6.0新特性。

## 2. TiDB Server作用&主要功能

- 处理客户端的连接 （Protocol Layer完成）
- SQL语句的解析和编译 （Parse、Compile完成）
- 关系型数据与KV的转化
- SQL语句的执行（Executor、DistSQL、KV完成）
- Online DDL的执行（schema load、worker、start job完成）
- 垃圾回收（回收过去版本的数据）
- 热点小表缓存 V6.0 （利用cache table，缓存数据量不大、不经常被修改但是经常被访问的表）

## 3. TiDB Server的进程

### SQL语句的解析和编译

#### SQL语句的解析

![image-20230728174152731](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728174152731.png)

SQL语句 -> lex词法分析 -> yacc语法分析

lex词法分析器：将语句进行抽象，解析为一个一个的token，有关键字、列名、表名等。

yacc语法分析器：将抽象过的SQL语句转化为树形结构，编译器根据AST树进行编译优化。

#### SQL语句的编译

![image-20230728174700624](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728174700624.png)

Complie环节存在三个阶段，AST作为输入，执行计划作为输出。

验证：验证SQL的合法性，验证元数据

逻辑优化：依据关系型数据库一些等价交换的规则进行逻辑变换，比如列裁剪，将不需要的列去掉，比如最大最小消除、投影消除、位次下推、子查询、外连接变为内连接等。

物理优化：根据逻辑优化的结果考虑数据的分布、大小、元数据和统计信息，决定用哪一个算子，是使用索引还是全表扫描，如果走索引是走哪一个索引。

### 关系型数据与KV的转化

以下表的编号字段作为主键。以下表过程是将一个表转化为一个聚簇表，还存在非聚簇表。聚簇表使用表原来的主键，而非聚簇表不用表中原来的主键作为K。

![image-20230728212358045](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728212358045.png)

![image-20230728212455147](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728212455147.png)

![image-20230728212545045](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728212545045.png)

加表的编号是为了让这一条记录的Key在整个数据库中唯一。

![image-20230728212637184](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728212637184.png)

![image-20230728212808580](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728212808580.png)

![image-20230728212829999](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728212829999.png)

![image-20230728212851675](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728212851675.png)

![image-20230728212902551](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728212902551.png)

如果一个Region超过144M，那么一个Region会被分为两个Region，并分布式存储。

### SQL读写相关模块

![image-20230728212940986](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728212940986.png)

编译好的执行计划会给到Executor，执行计划是树状的，Executor开始执行，每一层调用下一层的方法并获得结果，执行计划在执行时会分为两种，一种是比较复杂的SQL，比如范围查询，表连接，嵌套查询等等，为了避免这些复杂的SQL与TiKV存取相关的耦合度太高，抽象出了一个接口DistSQL模块，位于执行Executor和TiKV之间，是把对于TiKV的请求进行封装，封装一个比较简单的单表计算任务，是将一个复杂操作封装成单表简单操作的组合，发送给TiKV。所以一个DistSQL涉及到一张表但是可能会涉及到多个Region。比较复杂的SQL会走DistSQL模块，而一些点查，例如根据主键或索引的等值查询，只查一行的时候，会走KV模块。KV模块响应点查操作。

无论时复杂查询还是点查都会发送信息给TiKV Client，经过TiKV Client发送给TiKV集群。

如果语句包含一些事务相关的操作，则会启用Transaction模块，主要负责二阶段提交和一些锁的管理，Transaction相关操作都是通过KV和TiKV Client发送信息给TiKV。

PD Client是TiDB Server专门与PD交互的一个模块，从PD获取TSO时间戳，Transaction模块通过PDClient获取事务提交的时候的TSO时间戳。



### Online DDL 相关模块

![image-20230728213843598](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728213843598.png)

在整个集群中可能会有多个TiDB Server，其中的workers模块负责DDL操作，并且每个TiDB Server都可能发送DDL操作请求，但是在同一时刻只有一个TiDB Server可以进行DDL操作。

用户在发出一个DDL语句之后，由TiDB Server中的start job模块接受，并把这个DDL作为一个job放到TiKV中的一个job queue当中。

在同一时间，只有一个TiDB Server执行DDL，这个Server的角色叫做Owner，Owner在job queue中取第一个job进行执行，并且将执行过的job放到history queue历史队列当中。

每一个TiDB Server有一个任期，在某个时间段内作为Owner，不是Owner后会选取下一个Owner，谁做Owner是轮换着的。

schema load是该TiDB Server成为新的Owner之后，将最新的左右的表的schema的信息同步到其内部的缓存当中。在TiKV中做DDL的持久化是防止TiDB Server宕机，在宕机之后DDL操作仍然存在于TiKV当中。

### GC机制与相关模块

![image-20230728214632599](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728214632599.png)



TiDB事务实现了MVCC，多版本并发控制，在数据库中会存储修改过的记录的历史数据，一个记录会有多个版本，历史版本会持久化并打上时间戳，留历史版本可以撤销修改。当历史数据过多时，会进行定期的清理，这个清理过程在TiDB中就是GC。有一个TiDB Server会选为GC leader，来控制GC过程。TiDB Server会计算出一个时间戳，叫做safe point，在这个point之前的数据都会进行回收，在这个safe point之后的历史数据都会进行保留。

safe point默认数据修改开始的十分钟，也就是数据修改后的十分钟之内都会保存历史的数据。

## 4. TiDB Server的缓存

### TiDB 的缓存

- TiDB Server 缓存组成
  - SQL结果
  - 线程缓存
  - 元数据，统计信息（还有用户信息等）
- TiDB Server缓存管理
  - tidb_mem_quota_query：控制每一条SQL语句使用的缓存大小
  - oom-action：决定SQL使用缓存大小超过tidb_mem_quota_query之后要如何操作。

默认使用的是TiDB整个数据库的全部内存。

### 热点小表缓存

- 表的数据量不大
- 只读表或者修改不频繁的表
- 表的访问很频繁

如果一个表满足以上三个条件，但是存储在TiKV中，那么这个TiKV的负载会很大，存在性能瓶颈，因此把这个小表全都读出来，放到cache table当中，也就是TiDB Server的内存中，不用取TiKV中读而直接到cache table当中读。

![image-20230728215940166](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728215940166.png)

看起来很简单，但是有一个难题，就是不能保证这个表不修改，如果对在cache table中的表进行修改，要如何修改？如果先修改内存中的，那么TiKV中的就不一致了，可能会读到不一致的数据，而先修改TiKV的数据，也可能会读到cache table中不一致的数据。

**原理：**

- 租约到期，数据过期
- 写操作不再被阻塞
- 读写直接到TiKV节点上执行
- 数据更新完毕，租约继续开启

是一个循环。

开启租约，读内存，不写->租约到期，读写TiKV->数据更新，开启下一个租约，刷新缓存数据，读内存，不写

![image-20230728220421628](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728220421628.png)

使用ALTER TABLE [表名] CACHE语句将表放到cache table当中，并且该表的大小必须小于64M。

tidb_table_cache_lease参数，是缓存的租约参数。例如租房子，组一个房子给了一个开门的密码，在一年之后密码失效，房子就进不去了，房东可以租给别人。这个参数默认是5，单位是秒，也就是这一张小表到cache table中有五秒的租约，在这5s之内，TiDB Server的用户可以去缓存当中读这张表，如果用户要写这张表，就立即阻塞，等到租约到期才可以写。在租约有效的时候只能读不能写。

![image-20230728220739088](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728220739088.png)

租约到期之后，可以进行写操作，但是都是在TiKV节点上执行，也可以读。

![image-20230728220954427](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728220954427.png)

无论在租约内还是租约外，都可以进行读操作，在租约内去cache table进行读操作，租约外在TiKV上进行读操作，但是在租约内会阻塞写操作，在租约外才可以进行写操作，只会在TiKV上进行写操作，不会在缓存中进行写。

![image-20230728221202163](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728221202163.png)

在租约到期之后，放开阻塞的写操作，写操作完成后将更新后的数据refresh刷新到缓存当中，并且开启下一个租约，这个时候用户又在缓存当中读取数据，并且写操作被阻塞。

**热点小表缓存总结：**

- TiDB对于每张缓存表的大小限制为64MB
- 适用于查询频繁、数据量不大、极少修改的场景
- 在租约（tidb_table_cache_lease）时间内，写操作会被阻塞
- 当租约到期（tidb_table_cache_lease）时，读性能会下降
- 不支持对缓存表直接做DDL操作，需要先关闭热点小表缓存功能（不能在开启热点小表缓存功能时进行DDL操作，例如给表加一个字段）
- 对于表加载较慢或者极少修改的表，可以适当延长tidb_table_cache_lease保持读性能稳定

### 测验

1. 下列哪些模块直接与 TiDB 的事务处理有关？（ 选 2 项 ）

A. KV

B. Parse

C. Schema load

D. Transaction

E. GC

F. start job

正确答案： A. KV 、 D. Transaction

KV用于点查和事务有关

Transaction专门用于事务

GC是MVCC机制的历史数据回收

2. 关于关系型数据与 KV 的转化，下列说法不正确的是？

 A. 如果没有定义主键，key 中包含 RowID，Index ID 和 Table ID，都是 int64 类型 

 B. Table ID 在整个集群内唯一 

 C. 如果定义了主键，那么将使用主键作为 RowID 

 D. 不需要为每张表指定主键 

正确答案： C. 如果定义了主键，那么将使用主键作为 RowID

表分为聚簇表和非聚簇表，聚簇表需要表中有一个主键，非聚簇表可以有主键也可以没有主键，不管有没有主键都是用系统自动生成的一个row_id

# Lesson 3 TiKV

## 1. TiKV架构和作用

- 数据持久化
- 分布式一致性：数据以region为单位，分布式的存储在数据库中。 
- MVCC
- 分布式事务
- Coprocessor

在一个TiKV中存在两个rocksdb实例，一个负责存取KV，一个存取raftlog日志。数据以region为单位分布式存储在TiKV节点上，每一个region及其副本构成其raftgroup。多个raftgroup组成lucky raft。进行基于raft共识算法的写入和读取，与基于rocksdb的写入和读取不同，TiKV有两套写入读取。rocksdb 的写入和读取针对于单个TiKV节点的读写KV值，是纵向的，raft共识读写是针对多个TiKV存储节点中的region副本读写，是横向的。

Coprocessor，协同处理器，提供算子下推能力，将一部分过滤、聚合、投影交给TiKV完成，并且可以并行完成，因此不需要TiKV将所有的数据都通过网络发送给TiDB Server，缓解TiDB Server的负载。

![image-20230728235459758](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230728235459758.png)



## 2. TiKV数据持久化和读取

### Rocks DB

RocksDB针对Flash（SSD）存储进行优化，延迟极小，使用LSM存储引擎。

- 高性能的Key-Value数据库
- 完善的持久化机制，同时保证性能和安全性
- 良好的支持范围查询
- 为需要存储TB级别数据到本地FLASH或者RAM的应用服务器设计
- 针对存储在高速设备的中小键值进行优化-可以存储在FLASH或者直接存储在内存
- 性能随CPU数量线性提升，对多核系统友好

RocksDB具体负责的就是持久化存取，是开源单机存储引擎，可以看成单机的KV map。

### RocksDB 写入

![image-20230729000310310](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230729000310310.png)

数据是如何持久化到TiKV当中的呢？

RocksDB采用了LSMTree的数据结构存取KV数据，LSMTree将会把所有的数据的修改删除操作保存在内存中，MemTable中，将这些操作达到一定的数量之后，再批量的写到磁盘当中，并且会和之前的数据进行合并，并且不同于B+树，LSMTree是直接插入的，不是在数据原来的位置上做修改，避免了数据的随机写入，完全是顺序写入，对写非常友好，不需要对数据排序了。LSMTree的结构横跨的内存和磁盘，主要部分是MemTable、immutable MemTable、SSDTable。

写入流程的例子：要写入（1，Tom）数据。

首先进行WAL，因为写到内存中的数据很可能会因为掉电而丢失，因此要先预写日志，Write Ahead Log，用来保证事务写入的原子性和持久性的技术。会先把（1，Tom）写到磁盘中的一个日志文件中，再把（1，Tom）写到内存中。在宕机之后，内存中的数据会消失，但是WAL中还存在数据，可以重读数据来恢复数据，进行故障恢复。有一个参数叫sync_log，设置为True，在写入的时候会直接调用操作系统的fsync，直接将数据写入到磁盘，不会经过OS的缓存，因为在写入的时候会先写入操作系统的缓存当中，这个时候机器宕机数据也会消失，因此直接写入磁盘就可以避免这个问题。

接着（1，Tom）被写入了MemTable，是内存中的一个数据结构，保存了落盘前的数据，MemTable中一般使用跳跃表、搜索树来保证数据的有序性，在MemTable中追加到一定的数据量之后，这个数据量的大小由一个参数write_buffer_size决定，超过这个参数的大小，MemTable中的内容就会转存到immutable Memtable中，内容就不能更改了，RocksDB就会重新开辟一个MemTable。MemTable同时服务于读写，读的话也是从MemTable中读。

immutable Table就是需要刷到磁盘中的一个一个文件了，是由MemTable转存到磁盘中的SSD Table（SSD文件）的一个中间状态，使用immutable Table是为了防止IO等待导致客户端写入阻塞，防止写阻塞，会有一个flush pipeline一个写入队列。immutable Table存在一个就会刷入磁盘，如果超过5个就会默认出发rocksDB的一个自我保护，叫write stall，会限制写入MemTable的速度，从客户端反应为写入速度变慢，并且TiDB会在日志中记录write stall，可以通过优化存储或调高默认值5来避免write stall。

总而言之，数据来了，先写WAL日志，再追加入MemTable，当MemTable写到write_buffer_size大小之后，写入immutable Table，并再flush pipeline中等待刷入磁盘，如果排队的immutable Table超过默认值5，那么会触发RocksDB的自我保护机制，开启write stall，限制客户端写入速度，且TiDB会在日志当中记录这个情况。

![image-20230730115357032](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730115357032.png)

RocksDB数据落盘之后的存储方式如何？

磁盘中的文件RocksDB分成多层进行组织，每一层是Level n，Level 0比较特殊，就是immutable MemTable 的复刻，和immutable Memtable是一样的，客户端是怎么写的就怎么存，默认达到**四个**以后就会往下一层走，这个过程叫做**Compaction**。在每一层的文件都会被切分成SSDTable文件（SST文件），每一个文件都是键值对，按照Key排序，如果要在SST中查找某个Key，直接使用二分查找，因为已经排序了，如果Key在就返回，如果Key不在就看另一个文件。

当Level 0写到了四个文件，开始**Compaction**并对Key排序，形成一个SST文件，四个文件合并到一个并进入Level1。当Level1达到256MB，就继续将Level1进行Compaction，合并为Level2的一个SST文件，并继续对Key做排序，形成一个Level2的文件，以此类推。

查找的时候十分简单，因为每个文件都是有序的，直接使用二分查找。

总结写入操作：在RocksDB中，首先写入WAL，再写入MemTable，完成一次写入。需要一次磁盘IO，一次内存IO，相比于B+树，需要读多次磁盘IO，因为RocksDB的写入提高了效率。

如果是删除操作，只需要再MemTable中写一个delete key = xxx的操作，在读取的时候，只要发现xxx被删了，就不用读了。

更新的操作也是一样，仅仅需要操作MemTable就可以，只需要把操作放在MemTable当中就可以了，对写十分友好。

### RocksDB 查询

RocksDB对写操作十分友好，对查询操作就不那么友好了，比B+树要差一点。

![image-20230730120356546](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730120356546.png)

操作进入Memtable之后，要进入immutable MemTable，再在磁盘当中依次找每一层的文件，直到找到目标数据。

具体而言，在RocksDB当中存在一个叫做BlockCache的缓存，会将每次读取的最近最常读的数据先缓存到Block Cache当中，也就是读取的数据可能会在缓存中，如果在缓存中就直接读就可以了，如果要读的数据不在Block Cache当中，那么接着进入MemTable，如果在MemTable中找到了，那么就直接返回，不然就要继续进入immutable MemTable查找。如果还是没有找到，就要进入磁盘中查找。在RocksDB中，最新的数据一直在老数据之前，因此只要读到目标数据，其后可能会有之前的老数据，但是不用继续往下读了。

为了进一步提高效率，还引入了Bloom Filter布隆过滤器，1978年发明的，可以帮助判断集合中的元素，如果Bloom Filter说值在集合中，那么这个值可能不在，因为有误判率，但是如果说值不再集合中，那么久一定不在，是判断一个值在不在集合中。每一个SST文件都会安排一个Bloom Filter。

### RocksDB： Column Families 列簇CF

![image-20230730121133286](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730121133286.png)

在RocksDB当中原本只有一套Block Cache、MemTable、immutable MemTable等，现在变成了两套，相当于在一个实例当中有两套相同的内存文件和磁盘文件，一套叫做一个Column Family。

假如一张表的键值对是id - (name, age)，还有一张表是id - (addr, tel)，因此一个SST文件当中可能会存在多张表的键值对数据。有了列簇的概念，可以左边的CF专门存一张表的键值对，例如id - (name, age)，另一个CF专门存id - (addr, tel)，称为RocksDB的数据分片技术。在写数据的时候，写CF1：write(CF1, id, name, age)，写CF2：write(CF2, id, addr, tel)，写的时候可以指定某个列簇。如果写的时候没有指定列簇，在RocksDB中有一个默认的列簇，就会写到这个default列簇当中，使用列簇可以帮助进行数据分片，使某个CF当中存的都专门是一张表或几张表当中的数据。

但是日志文件WAL没有进行数据分片，无论向哪一个列簇写入，都使用同一套WAL。

一个RocksDB实例会有多个列簇，每个列簇有自己的MemTable和SST文件，但是共用一套WAL文件。



## 3. TiKV如何提供MVCC和分布式事务支持

### 分布式事务

![image-20230730122002598](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730122002598.png)

什么是分布式事务？举个例子：

```SQL
Begin
id = 1, Tom -> id = 1, Jack
id = 2, Candy -> id = 2, Andy
Commit;
```



在分布式数据库当中，这个普通的事务会有一些问题。

假设在TiDB数据库中存在两个TiKV实例存储这个数据，id=1, Tom在TiKV1中，id=2, Candy在TiKV2中，首先修改TiKV1中的数据，修改为Jack， 但是在准备修改TiKV2中的数据的时候，发现由于一些原因TiKV2无法使用，在这个时候就破坏了事务的原子性，这个就是分布式事务的问题。

TiDB的分布式事务采用了Google 的分布式事务模型。

**事务在TiDB当中的流程：**

目标需要执行的事务为：

```SQL
Begin
<3, xxx> -> <3, Frank>
Commit;
```

首先begin 的时候，从PD组件中获取一个TSO时间戳，代表事务的开始时间，称为start_ts，假设为100。接着将要修改的数据读取到TiDBServer的内存中，先进行一个修改，修改之后一旦当事务遇到Commit的时候，就进行一个持久化，进行两阶段的提交。第一阶段是prewrite阶段，将在内存中修改的数据和锁信息写入到TiKV节点中。在begin和prewrite之间，别人感知不到锁，只有在commit时候，锁才会被写入，称为乐观锁。如果想要别人在begin和prewrite之间感知到锁，那么就会提前将锁信息写入TiKV中，这个称为悲观锁。我们先使用乐观锁进行介绍。提交的第二个阶段就是到PD组件中再要一个TSO，称为commit_ts，是事务的结束时间，假设是110，并会给一些提交信息。

修改的数据在TiKV中如何存储？

TiKV中会使用三个列簇，一个是Default，存储修改的数据，一个是Lock，存储锁信息，一个是Write，存取提交信息。

prewrite阶段，将<3, Frank>中写入Default中，key不单单是原本的id，还会再加一个事务开始的时间戳，变为3_100，并且不会放之前的值。为什么呢？因为之前说的在读取的时候，仅需要读最新的值就行了，新的值在更低的Level上，老的值在更高的level上。在修改id=3的数据的时候，需要给一个锁，并且只给事务的第一行加一个主锁，其他行修改的锁都指向这个主锁，这个锁加到Lock列簇中。<3, (W, pk, 3, 100 ...)>，3是id，W代表写锁，之后是事务的一些信息，写了Default和Lock之后，prewrite阶段完成了。如果在这个阶段有别人要读或修改，都会看Lock有没有对目标数据上锁。

在Commit阶段，第一步到PD中获取一个TSO，是事务的真正的结束时间，之后会在write列簇中写入提交信息，id是原本的id加上commit时间的TSO，并且value是事务的开始时间。事务提交之后可以释放锁，还要在Lock列簇中清理锁，不是删除锁信息，而是在Lock类簇中写一个D的信息，代表delete锁。

总的来说，在整个事务中，begin获得一个时间戳，提交后进入两阶段，第一阶段，修改信息写入Default列簇，锁信息写入Lock列簇，第二阶段，获得commit事务结束时间戳，在write列簇中写入事务提交信息，在Lock列簇中写入一条记录删除锁。

如果别人想要读id=3的数据，首先去write列簇中看，id为3的最近依次修改是什么时候，获得最近一次修改时间，拿着3和100去Default中找值，可以得到值Frank。如果在Write中没有找到id为3的信息，但是在Lock列簇中找到了锁，则要阻塞。

![image-20230730124319430](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730124319430.png)

当事务修改的数据较小，小于255B的时候，则写到Write列簇中，也就是Write列簇中可能同时存在事务信息也会存在数据。Defalut列簇中只存储大小大于255B的数据。

![image-20230730124512256](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730124512256.png)

事务的原子性问题TiDB是如何解决的？

在Begin的时候，从PD获得一个TSO，并且将id为1和2的数据读到内存中进行修改，接着在Commit的时候进入两阶段提交。一阶段prewrite的时候，将修改的数据写到TiKV中的Defalut列簇或Write列簇（Write列簇存数据大小小于255B的数据），并将锁信息写到TiKV的Lock列簇中。二阶段commit的时候，从PD获得一个TSO，事务结束时间，并在write列簇写入提交信息，清除锁。

在Lock列簇当中，pk代表主锁。在第一阶段首先将事务的第一行进行处理，写Default和Lock，Lock列簇中为主锁，然后处理后面的行，并且Lock写的不是主锁，@1，存的是锁的指向，后面是其他的相关信息，@1当中的1是主锁对应数据的id，指向了主锁。

在读取的时候，先读取write，并且离现在时刻最近的修改，如果读到commit信息，那么就会根据事务开始时间寻找最新的修改数据。

如果提交信息在TiKV1上写了，但是没有在TiKV2上写，怎么办？由于已经完成了一阶段，因此会发现Lock列簇中有一把锁，这个锁指向了TiKV1中的主锁，在TiKV1中的Lock列簇中查询发现事务已经完成，因此会在TiKV2中补上之前宕机没有写入的信息。

### MVCC

<img src="C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730160348357.png" alt="image-20230730160348357" style="zoom:50%;" />

多版本并发控制，MVCC，Multi-Version Concurrency Control

许多主流数据库都支持。

**实现**

举例：Transaction2，将Jack改为Tim

在修改数据的时候，不会立即修改数据，而是会新生成一个副本，<1, Jack>原本的是未提交状态，可以随便改，并且读的时候会读到副本，如果前面修改的数据提交了，那么数据库id=1的数据就多了一个版本，数据库会给每一个版本一个TSO，在读的时候就可以根据TSO先后顺序读到最新的数据。

在以上的例子当中，Transaction2并没有提交，如果在TSO=120的时候，要读取key=1,2,4的数据，如果没有MVCC机制，1和4会阻塞，因为没有提交，只能读2，如果有MVCC，那么124都可以读出了，因为有之前的版本，1读出Jack，4读出Tony。

![image-20230730161432315](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730161432315.png)

现在的事务都是TiDB的悲观事务模式，基于悲观锁，也就是如果在事务中进行数据修改没有commit，这个锁会被所有用户感知到，也就写入阻塞。

write列簇中，看不到现在事务2中的提交信息，因为没有提交，但是由于是悲观事务，所以在Lock列簇当中可以看到事务2的锁。

![image-20230730161938362](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730161938362.png)

如果要读id=1，那么首先读到write列簇id=1的最近一次提交的记录，然后拿到了该事务开始的TSO，拿着id和TSO去Default列簇中找这条记录，可以找到Jack。

如果要写id=1，那么首先读到write列簇id=1的最近一次提交的记录，并且在Lock列簇中找到了id=1的锁记录，这个时候发现id=1的记录上锁了，因此写阻塞。

![image-20230730162547767](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730162547767.png)

如果要读id=2，那么首先读到write列簇id=2的最近一次提交的记录，然后拿到了该事务开始的TSO，拿着id和TSO去Default列簇中找这条记录，可以找到Candy。

如果要写id=2，那么首先读到write列簇id=1的最近一次提交的记录，并且在Lock列簇中没有找到id=2的锁记录，因此可以写入id=2的数据了。

![image-20230730162608731](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730162608731.png)

如果要读id=4，那么首先读到write列簇id=4的最近一次提交的记录，然后拿到了该事务开始的TSO，拿着id和TSO去Default列簇中找这条记录，可以找到Tony。

如果要写id=4，那么首先读到write列簇id=4的最近一次提交的记录，并且在Lock列簇中找到了id=4的锁记录，但是这个锁不是主锁，指向了1，因此找到了1的主锁，这个时候发现id=4的记录上锁了，因此写阻塞。

所以MVCC技术下，加锁不阻塞读，而阻塞写。



## 4. TiKV基于Raft算法的分布式一致性

### Raft 与 Multi Raft

![image-20230730162848145](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730162848145.png)

每一个Region会存在三个副本，其中有一个是Leader，其余的是Follower。

Leader：集群中的管理者，所有客户端的读写流量，都是走Leader的，Follower不参加读写。Leader会周期性的将写的数据传递给Follower。

Follower：是被管理者，接受Leader的日志并复制，如果长时间收不到Leader的信息，就会将自己的角色转换为candidate，发起投票，选出新的Leader。所以转换为candidate的条件是Leader长时间没有发送统计信息，candidate将会成为新的Leader。

一个Region插入达到96MB之后，会新开一个Region，继续插入，如此往复。

如果Region中插入数据插入太大，达到144MB后会进行分裂，如果删除数据太多使Region太小，则可以Region的合并。

### Raft 日志复制

![image-20230730164038150](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730164038150.png)

当TiDB Server发送写入数据的时候，只会发送给Leader，Leader会把数据同步到其他Region。

第一步：Propose， 操作被Leader收到，准备同步，将请求变为写入raft日志，并且存储到自己的日志文件中。日志格式：RegionId_LogId{操作 key = XXX, value = XXX}

第二步：Append，将日志存储到本地的存储日志的rocksdb当中，将本地的日志进行持久化。

第三步：replicate，Follower收到日志信息，对日志信息进行本地持久化，并向Leader发送信息表示收到日志且已经完成持久化。

第四步：committed，当大多数节点都返回了append成功的消息之后，那么Leader将会认为这一条日志已经完全成功了，也就commit了。

第五步：Apply，将日志中的操作应用到rocksdbKV数据库中，也就是根据日志的操作修改具体数据库中的数据。

一个TiKV节点存在两个rocksdb，一个rocksdb存储raft日志，一个rocksdb存储kv，具体的数据。

**再整理一遍：**

![image-20230730165146151](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730165146151.png)



Propose：外部客户端写入一条raft日志。

![image-20230730165428037](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730165428037.png)

Append：将日志持久化到本地存储日志的rocksdb当中，目前的append只在leader节点中持久化了日志。

![image-20230730165531659](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730165531659.png)

Replicate：将Leader中的日志信息分发到其他的Follower Region之中，并且在接收到数据之后，也会有一个Append操作，这个Append操作是Follower完成， Follower将收到的日志数据持久化到本地的rocksdbRaft当中。

![image-20230730165655353](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730165655353.png)

Committed：当其他的Follower Region将日志持久化到本地的rocksdb当中之后，会向Leader发送响应信息，告知已经持久化日志。根据Raft协议，收到超过一半的节点的回应，那么该日志就处于了Committed状态。这个committed代表大部分节点都对raft log进行了持久化，与SQL当中的commit不同，SQL当中的commit代表已经将数据写入且其他人可以读。

![image-20230730170104959](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730170104959.png)

raft日志处于committed状态，用户还没有committed！

TiKV是一个分成结构，Transaction是面向用户的，用户的commit是事务这一层的commit。

![image-20230730170625165](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730170625165.png)

Apply：将raft log真正的写入rocksdb kv当中，也就是真正的修改数据，真正的持久化了数据。

![image-20230730170732027](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730170732027.png)

这个时候用户committed了，才是真正事务的committed，也就是不仅raft日志被持久化了，并且数据也真正落盘了，持久化了。

### Raft Leader 选举

![image-20230730170829152](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230730170829152.png)

概念Term：时期，是raft共识算法将时间分成了一小段一小段，每个term大小不一定一样，可以看作恋爱当中的一段时间。

在集群刚开始创建的时候，集群中是没有Leader的，每个region都是follower，每个region有一个计时器，称为election timeout。假设为十秒的话，一开始的时候，每个follower都会等待集群中的leader给其发送心跳信息，都处于term=1的状态，但是如果没有等到，超过election timeout，这个follower就认为现在集群中没有leader了，这个时候谁率先超过了election timeout，谁就先成为candidate，并且进入term=2的状态，给其他的follower发送请求，告知该follower要当leader。当一个candidate的term大于其他的follower的时候，这个follower就会给这个candidate投票，如果超过半数的同意，那么其状态就会变为leader，并且与其他follower同步更新状态。

发起投票的人会递增自己的term，并成为candidate，如果多数节点同意，那么所有人都成为term2并且candidate成为leader。

![image-20230801114919343](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801114919343.png)

以上是再集群初始化的时候，每个region都是follower的情况，在运行中，如果已经存在了leader的情况，如果leader宕机了，那么也会进行相似的流程。heartbeat_time_interval参数决定了选取candidate的时间，如果一个follower超过这个参数的时间没有收到leader的信息，那么这个region就会自增term并变为candidate，发起选举，如果超过半数的region同意，那么就成为leader，并且与其他region同步term。

![image-20230801115950163](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801115950163.png)

如果有多个region成为了candidate，那么系统就会重新发起投票，选出一个新的leader，但这里可能会存在多次重复选举产生延迟，TiDB使用了一个random值来防止这个情况，election_timeout会从一个范围内随机选一个值，比如100ms-300ms，这样较好避免了重新选举。

在Raft Leader选举的时候，有两个重要参数，一个是election_timeout，一个是heartbeat_time_interval。

![image-20230801120241846](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801120241846.png)

以上是在TiKV中的具体参数表示。Election timeout对应raft-election-timeout-ticks，是一个个数，多少个ticks的时间，ticks是一个时间单位，Heartbeat time interval对应raft-heartbeat-ticks。

如何换算为实际的时间？乘上raft-base-tick-interval就可以了，也就是一个tick对应的秒数。

**注意**，以上两者的关系：raft-election-timeout-ticks的最小值不能小于raft-heartbeat-ticks



## 5. TiKV的coprocessor

### 数据对写入

在Raft日志复制的时候，说到TiKV的数据写入其实是分层实现的，有点像TCP/IP的五层协议。日志是吃就在rocksdb raft当中，kv数据是持久化在rocksdb kv中的。为了实现多个TiKV节点数据多副本的一致性，加上了Raft层，为了写不阻塞读，加了MVCC层，为了提供事务的服务，加上了Transaction层。

![image-20230801120852264](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801120852264.png)

数据的写入，其实是不包含MVCC和Transaction，在这里我们只关注数据的写入，假设写请求已经经过了Transaction和MVCC层，已经到了Raft一层了。着重关注Raft层和rocksdb持久化层上。

 ![image-20230801120646627](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801120646627.png)

当用户发出commit命令之后，开始两阶段提交。PD主要是在事务开始和提交的时候提供TSO，提供要写的数据在哪一个TiKV中的哪一个region上面的信息。TiKV中的raftstore pool是一个线程池，在收到写请求之后，将写请求转化为raft日志，也就是propose过程，之后持久化到rocksdb raft实例中，本机append之后将日志发送给其他副本所在的节点，其他节点在本地进行持久化并返回信息，当系统中的大多是TiKV节点返回持久化成功，那么commited这条日志的修改，代表这条日志在系统中持久化成功，并且最后通过apply pool线程池apply应用到kv数据上。在写的时候，有两个线程池和两个rocksdb参与。

举个例子看数据具体是如何写入的。

用户发出commit命令，在敲击回车之后，后台开始运作。假设写入的数据是key=1, value=Tom。

1. propose，将写请求转化为raft log
2. append，将日志持久化到rocksdb raft中
3. replicate，复制，将日志发送到其他节点。
   1. append，其他收到日志的节点append，持久化到本地rocksdb raft中，并返回成功信息
4. committed，提交，代表这条日志在系统中不会丢了。
5. apply，应用，将日志中的操作应用到rocksdb kv当中，修改数据。

在五个步骤全部走完之后，用户一开始输入的commit命令才算成功。

当可以读到修改后的数据时，一定是已经成功apply，将日志操作应用到数据上了。

### 数据的读取 ReadIndex Read

![image-20230801130219079](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801130219079.png)

如果要读key=1的数据，首先要去PD查询，key=1的数据在哪一个TiKV node的哪一个leader角色的region上。但是有几个问题：

1. 如何保证现在读取的leader角色的region一定是leader，不是leader是因为在读的这个段时间这个region的角色可能会改变。

在读之前，leader向其他follower发送一个心跳，并且其他的follower给出回应，确定是这个读的region是leader。

2. 读的线性一致性问题。如果<1, Tom>在十点改为<1, Jack>，在十点零五的时候，如何保证用户读取的时候一定读到Jack而不是Tom，也就是如何保证线性一致性，保证修改在时间上是一一致的。

![image-20230801130659337](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801130659337.png)

现在由于是rocksdb这一层，因此不能在写的时候读，写的时候读会阻塞。因此用户的读取如何保证一定可以读到1-95呢？在读的时候，当前的raft log是1-97，先把这个记下来，叫做readIndex，现在为97，在10：08的时候，发现95commit成功了，在10：09的时候再读，发现97 已经apply了，说明95也可以读了。

![image-20230801132242149](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801132242149.png)

在indexRead的时候，首先要去PD中确认读的region在哪，并且要确定读的时候该region是不是leader，也就是还要向其他follower发送心跳信息，这个发送心跳也会占用资源，因此在这一方面可以进行优化。Raft协议中的Lease Read对这个进行优化。

![image-20230801132428820](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801132428820.png)

假设在10：00的时候当前的leader发送一个心跳，这个心跳会在heartbeat time interval范围内发送，并且在heartbeat time interval间隔内确认当前的region是leader。  如果其他节点没有收到心跳信息，认为当前集群处于无主状态，并且超过了election timeout的情况下， 会重新进行选举。

所以在10：00到election timeout 都可以保证这个region是leader，所以在这段时间内就可以不用发送心跳信息了，叫做Lease Read，也可以叫Local Read。

### 数据的读取 Follower Read

![image-20230801140400798](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801140400798.png)

每个Follower中的数据是leader中的备份，所以其实是一样的，读Follower还可以减轻leader的压力。本质还是读rocksdb kv，问题是如何保证线性一致性。

设在10：00向leader中写入，<1, Tom> -> <1, Jack>， 

![image-20230801141109381](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801141109381.png)

![image-20230801142017142](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801142017142.png)

在10：05进行Follower Read，记住现在所有人都做了raft commit操作的日志号，也就是97号，在10：08的时候在应用到95，这个时候95的提交才成功了，在10：10的时候，97应用成功了，这个时候才可以读，因为97成功的时候95才一定成功了。

有可能在Follower读，比在leader读要快，因为有可能follower先apply成功。

### Coprocessor

![image-20230801142700599](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801142700599.png)

协同处理器

当TiDB收到来自客户端的查询请求，大量的数据都需要从TiKV中汇聚到TiDB Server中，会有网络带宽和CPU负载太高两个问题。可以在TiKV中进行一些计算操作吗？可以的，使用Coprocessor协同处理器。

![image-20230801142848540](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801142848540.png)

TiDB Server对收到的语句进行分析，计算出了物理执行计划并分解出可以由Coprocessor计算的部分，在Coprocessor接收到信息之后，根据算子进行聚合过滤等操作。

![image-20230801142944380](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801142944380.png)

比如select count(*) from T;这个语句，可以分别计算每个TiKV中的count，最后在TiDB Server中加起来，大大减轻了TiDB Server的计算压力。

### Summary

- TiKV的结构与作用
- TiKV的持久化的数据读取原理
- TiKV对于分布式事务和MVCC的支持
- TiKV基于Raft算法的分布式一致性保证
- TiKV的Coprocessor

### 练习

1. 下列属于 TiKV 相关功能的是？（ 选 4 项 ）

A. 系统参数和元数据信息的持久化

B. 产生 TSO

C. 分布式事务实现

D. MVCC

E. 生成物理执行计划

F. 表统计信息的持久化

正确答案： A. 系统参数和元数据信息的持久化 、 C. 分布式事务实现 、 D. MVCC 、 F. 表统计信息的持久化

B：PD产生TSO，E：TiDB Server生成执行计划

2. 关于 TiKV 数据持久化，下列说法不正确的是？

 A. RocksDB 有 2 个实例，分别用来持久化 raft log 和 key value 数据 

 B. RocksDB 中 WAL 用来保证写不丢失 

  C. 对于删除操作，只需要在原 key value 数据上标记已删除即可 

 D. RocksDB 中，除了 Level 0 层的数据，其他 Level 都是单一排序持久化的 

正确答案： C. 对于删除操作，只需要在原 key value 数据上标记已删除即可

C：对于删除操作，也是新加一条记录，而不是在原来的key value数据上标记删除

# Lesson 4 Placement Driver

## 1. PD的架构与功能

![image-20230801151528939](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801151528939.png)

架构：PD本身只少由三个节点组成，并且最好由奇数个组成，也会有一个Leader，Leader出现故障时重新选举。

作用：PD是整个TiDB数据库的总控，是一个大脑，主要负责元数据的存储，负责全局时钟，查询开始时间，事务开始时间，事务提交时间都由PD统一分发，还负责对于TiKV的region进行调度。

Store：指的是一个一个TiKV节点，成为Store

Region：每一个Region负责维护一份数据，默认96M大小，并且默认3副本，每一个副本成为一个Peer。每一个Region都有角色，Peer和Follower差不多，读写一般读写Leader角色的region，会把写的日志同步给另外的Peer。

**总结PD的主要功能**：

- 整个集群TiKV的元数据存储
- 分配全局ID和事务ID
- 生成全局时间戳TSO
- 收集集群信息进行调度
- 提供label，支持高可用
- 提供TiDB Dashboard（监控）

## 2. 路由功能

![image-20230801152557244](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801152557244.png)

在读写数据的时候，首先要定位数据在哪一个Region上，这个时候向PD发送路由请求，并且获得目的的region是什么，同时把这个数据对应的region信息存储到region cache中方便下次使用，再去TiKV中读写数据。使用region cache中的信息，本来的leader可能变为了follower，这个时候会有backof操作，去原本是leader现在是follower的region找数据，会导致follower向TiKV Client返回信息，告知去正确的leader上读取信息，这个backof操作会导致延迟的增加。就是Region Cache中的信息可能太久了。

## 3. TSO的分配

![image-20230801153115186](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801153115186.png)

TSO是一个int64的整型数。physical time是物理时间，也就是1ms，logical time将1ms分为了262144个TSO，

![image-20230801153352140](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801153352140.png)

在TSO分配中，提供服务的是leader角色的PD节点。具体流程如图。

由于执行的SQL可能会很多，因此PD Client可能会有一个批处理操作，也就是在一个时间范围内的TSO请求一起进行处理，一起向PD进行请求。

![image-20230801153936257](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801153936257.png)

以上的两条线，下面一条可以看为存储，上面一条可以看为缓存。

大量的TSO请求，每次接收一次请求就要写一次磁盘，因为IO压力很大。PD每次分配一段的TSO，一次分配3s的TSO，这些3s的TSO时间一下放到缓存当中，所有的TSO申请就都抢这个缓存中的TSO，如果用完了，那就再分配一个时间窗口，因此这将磁盘IO变成了3秒一次。

在PD更换leader时，可能缓存中的TSO还没有用完，那么没有用完的TSO会直接丢失，因为下一个Leader并不知道TSO具体用到哪了，因此下一个三秒钟会紧接着上一个时间窗口进行分发，导致部分TSO丢失。但是没有关系。



## 4. PD的调度原理

![image-20230801154638434](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801154638434.png)

### 调度 信息收集

![image-20230801154650352](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801154650352.png)

心跳信息分为Store Heartbeat，是TiKV的信息，Region Heartbeat是每个Region发送的信息。

PD根据收集到的信息生成调度，生成Operator。

### 调度 生成调度

- Balance 均衡：关注Leader和Region两方面
  - Leader（考虑读写均衡）
  - Region（考虑存储均衡）
- Hot Region：热点Region，读写过于频繁可以打散，降低读写频率
- 集群拓扑
- 缩容
- 故障恢复
- Region merge：合并Region，太小的或空的region进行合并



### 调度 执行调度

![image-20230801155104550](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801155104550.png)

将operator发送给region，region根据operator进行响应的操作。

## 5. Label的作用

![image-20230801155152860](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801155152860.png)

Label是PD的标签。

DC1-3分别是独立的数据中心，每个数据中心有两个机柜（Rack），一共十二台主机（TiKV）。

如果Rack4机柜坏了，那么Region1整体只有一个可用，因此region1不可用了，因为超过半数不可用了，同理，如果DC1数据中心坏了，那么region2也就不可用了，但唯独Region3，无论是坏一个机柜还是坏一个机房，都是可用的，因为只会坏一个region。不同的region对可用性是存在影响的。

 我们可以为每一个TiKV实例设置一个Label，这样PD就可以按照标签约定好的格式，将Region分配到不同的数据中心、机柜上，这个Label是用来感知整个集群的一个拓扑结构的。

![image-20230801155642077](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801155642077.png)

label的配置需要在PD和TiKV节点上，这两个地方进行配置。标识了三个隔离级别，zone代表数据中心，rack代表机柜，host代表TiKV实例。

PD使用location-labels参数指明有几个隔离级别的参数，可以根据实际情况进行更改。如果保证region的三个副本不能在同一个zone，那么就将isolation-level设置为zone，隔离级别。如果隔离级别是rack，那么一个region的副本可能会存在统一个数据中心里。

## 练习

1. 下列关于 PD（Placement Driver）架构和功能正确的是？

 A. 访问 PD 集群中的任何一个节点都可以获得 TSO 

 B. TiKV 会周期性地向 PD 上报状态 

 C. PD 会周期性地查询 TiKV 的状态，不需要 TiKV 上报，目的是为了高效 

 D. PD 的调度功能只能平衡 region 的分布，无法对 leader 进行调度 

正确答案： B. TiKV 会周期性地向 PD 上报状态

A：只有Leader可以

C：是TiKV上报的

D：心跳信息有region和leader两个，region考虑存储，leader考虑读写

2. 关于 label ，下列说法不正确的是？

 A. label 的本质是个调度系统，可以人为控制 region 副本的存放位置 

 B. label 需要在 PD 和 TiKV 上进行配置 

 C. isolation-level 要和数据中心（DC）对应，这样可以获得最大的可用性 

 D. 如果某个 region 的所有副本不可用，有可能造成整个 TiDB 数据库不可用 

正确答案： C. isolation-level 要和数据中心（DC）对应，这样可以获得最大的可用性

C：不一定，isolation-level是可以设置的

D：是的，如果这个region存储的数据库的元数据，那么这个region不可用了，那整个数据库也就不可用了。

# Lesson 5 TiDB 数据库SQL执行流程

## 1. 描述DML语句读写流程

### DML语句读流程概要

![image-20230801160823207](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801160823207.png)



首先Protocol Layer接收了用户发送的SQL，接着去PD获取一个TSO，然后进入Parse模块将SQL转化为AST，进入Compile模块进行编译，生成执行计划，进入Executor，去TiKV中读取数据。读取之后交由Execute模块，由Execute模块返回给用户。



### DML语句写流程概要

![image-20230801161127674](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230801161127674.png)

在写流程中一开始和读差不多，首先接收、解析、编译、执行，去rocksdb kv中需要修改的数据读出，然后把数据放到缓存memBuffer当中，在缓存中进行修改，如果用户需要提交修改，那么就开始两阶段提交，第一阶段是prewrite，写入修改信息和锁信息，第二阶段commit，写入提交信息并清除锁信息，这些信息不仅要持久化到本地，而且要同步到其他的TiKV中。

### SQL的解析与编译

![image-20230803224827939](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230803224827939.png)

与PD通信，都是与PD Client的，TSO是通过PD Client异步获取的。

Parse将SQL转化为AST

进入Compile模块首先预处理，检测SQL的合法性、判断是否是点查，如果是点查就直接执行，不是点查就进入优化器，分为逻辑优化（例如外连接转换为内连接）和物理优化（选择最优算子），最后都会得到执行计划。

### 读取的执行

![image-20230803225137043](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230803225137043.png)

在收到执行计划后，首先Executor要获取元数据，可以直接从TiDB Server缓存中读到，还要获取目标数据所在的TiKV和Region，这一步的时候由很大的网络负载，为了缓解这个负载，就会将读取过的信息放在TiKV Client的region Cache中，如果region的缓存信息已经过期了，那么TiKV会backoff操作，并且使TiDB Server去PD重新获取。

![image-20230803225350148](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230803225350148.png)

读数据分为两种，一个走KV模块，一个走DistSQL。如果是点查，就不需要执行计划了，直接走KV模块，执行点查就可以查询了。如果不是点查，就要走Dist SQL模块了，如果SQL十分复杂，那么DistSQL模块就会将其转化为简单的多个单表查询，再进一步查询。

TiKV收到操作后，会生成一个小快照，snapshot，保证查询的时间点。并且所有查询都会先进入一个线程池UnifyRead Pool，再进一步取数据。

DistSQL转化后的单表查询，可能会去不同的TiKV上查询，这个时候就可以并行查询。

![image-20230803225701826](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230803225701826.png)

### 写入的执行

![image-20230803225831963](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230803225831963.png)

写的时候首先将目标数据存储到memBuffer中，这里读的操作和之前读取的执行是一样的。

![image-20230803230105085](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230803230105085.png)

写请求首先发送给Scheduler模块，负责协调并发请求，如果两个请求同时要写一个数据，那么使用latch模型进行处理，拿到latch的请求先写入。Radtstore将写请求转化为raft log，并且会持久化到本地，同时发送给其他TiKV节点，在持久化之后，进行Apply操作，将操作应用到数据上，写入成功。



## 2. 描述DDL语句的执行流程

![image-20230803224619787](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230803224619787.png)

只有owner角色的workers模块执行DDL操作。TiDB Server先将所有的DDL放入一个job queue当中，owner的workers执行时从job queue中取一个，执行完后放入history queue中。owner也是会改变的。



### DDL的执行

![image-20230803230318121](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230803230318121.png)

workers不可以并行执行，在整个集群当中只有一个TiDBServer中的workers可以执行，其角色时owner。

schema load负责将最新的表的元数据提供给workers。

![image-20230803230618729](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230803230618729.png)

角色为owner的workers会定期查看job queue，如果有job就会将其执行。

owner的角色是轮询的，通过PD调度，一个一个成为owner。

add index queue是特别的，加索引这种DDL会放入add index queue中。

### 练习

1. 下列关于 DML 语句读写说法正确的是？（ 选 2 项 ）

A. Region Cache 的主要作用是缓存热数据，减少访问 TiKV 的次数

B. 二阶段提交在获取事务开始的 TSO 和提交的 TSO 时，都是由 TiDB Server 完成的

C. schedule 模块采用 latch 来控制当前正在写的数据不被读取

D. 在写操作中，锁信息也会被写入到 RocksDB KV 中

正确答案： B. 二阶段提交在获取事务开始的 TSO 和提交的 TSO 时，都是由 TiDB Server 完成的 、 D. 在写操作中，锁信息也会被写入到 RocksDB KV 中

A：Region Cache缓存的是待访问region的元数据，是为了减少访问PD的次数

C：schedule中的latch是为了方式多个请求一起写一个数据，为了排队写。防止在写的数据不被读要使用锁。

2. 关于 DDL 语句的执行流程，下列说法正确的是？

A. DDL 语句不可以在 TiDB 中并行执行

B. 同一时刻，不可以有多条 DDL 语句在等待执行

C. 同一时刻，只有一个 TiDB Server 可以执行 DDL 语句

D. 等待执行的 DDL 语句被持久化在 TiDB Server 的存储中

正确答案： C. 同一时刻，只有一个 TiDB Server 可以执行 DDL 语句

A：可以，因为还有一个add index queue，加索引操作可以和其他的DDL并行执行

B：在队列中排队，可以的

D：等待执行的DDL语句被持久化在TiKV的队列中



# Lesson 6 TiDB 数据库 HTAP 概述

## 1. 描述HTAP技术

![image-20230803231537475](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230803231537475.png)

OLTP：在线事务型，像手机支付这种。

OLAP：像报表、分析是OLAP要解决的问题，每次都要扫描大量数据，分析出趋势并做统计，使用列存储是比较好的。

![image-20230803231829864](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230803231829864.png)

传统业务队OLTP与OLAP业务支持有很大挑战，分析OLTP数据要使用ETL工具，但是有很大延迟。



**HTAP的要求：**

- 可扩展性
  - 分布式事务
  - 分布式存储
- 同时支持OLTP与OLAP
  - 同时支持行存和列存
  - OLTP与OLAP业务隔离
- 实时性
  - 行存与列存数据实时同步



## 2. 理解TiDB数据库的HTAP架构

![image-20230804103402896](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804103402896.png)

在TiFlash中进行列存，在TiKV中进行行存



## 3. 理解TiDB数据库的HTAP核心特性

**TiDB的HTAP特性：**

- 行列混合
  - 列存（TiFlash）支持基于主键的实时更新（行存怎么变，列存就怎么变）
  - TiFlash作为列存副本
  - OLTP与OLAP业务隔离
- 智能选择（CBO自动或者人工选择使用TiKV还是TiFlash）
- MPP架构（实现了在TiFlash上对聚合和连接操作的加速，智能在TiFlah中完成）



## 4. MPP架构

![image-20230804103928570](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804103928570.png)

TiFlash驱动MPP worker做计算。

计算在TiFlash内存中计算，因此数据是不落地的，宕机后计算的数据就没了。

![image-20230804104300153](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804104300153.png)

 考虑以上SQL，并且数据散落在三个TiFlash中。

![image-20230804104841247](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804104841247.png)

MPP开始工作，首先进行过滤，在每一个TiFlash中过滤出7c0和日期的数据。

![image-20230804105157951](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804105157951.png)

然后进行数据交换，让pid相等的先到一个节点上来，对每一张表的pid取一个Hash函数，值相等的放到一个TiFlash中，

![image-20230804105924819](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804105924819.png)

然后并行化进行连接，两张表就拼接起来了。

![image-20230804110212778](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804110212778.png)

如何对于group by计算的原理也是一样，进行数据交换，把一类数据放到专门一个数据节点上，聚合只要在各个节点上进行就可以了。

![image-20230804110323612](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804110323612.png)

![image-20230804110330904](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804110330904.png)

最后汇总到TiDB Server上，完成了计算。

表连接和聚合都会现在TiFlash上完成，充分发挥TiFlash并行计算的功能。

![image-20230804110412756](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804110412756.png)

MPP有很多应用场景。有OLAP业务和OLTP业务，如果没有HTAP，两个业务就要分走两个系统，但是有了TiDB就可以用一个系统，分析型业务走TiFlash，在线业务使用TiKV。

![image-20230804110517434](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804110517434.png)

流式计算场景要经过一个CDC工具，首先右侧的业务要支持CDC才可以进行，虽然有支持的，但是并不支持所有的。

![image-20230804112024123](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804112024123.png)

### 练习

1. 下面属于 HTAP 场景特点的是？（请选择 3 项）

A. 在故障恢复方面可以做到 RPO = 0

B. 支持分区特性

C. 支持在线业务高并发

D. 同时支持 OLTP 和 OLAP 业务

E. 能够读取到一致性的数据

正确答案： C. 支持在线业务高并发 D. 同时支持 OLTP 和 OLAP 业务 、 E. 能够读取到一致性的数据

A：

B：

2. 关于 MPP 架构，下列说法不正确的是？

 A. MPP 架构的中间结果都在内存中 

 B. MPP 架构可以作用于 TiKV 和 TiFlash 上的数据 

 C. MPP 架构目前不支持非等值 join 

 D. MPP 架构可以对聚合、JOIN 等操作加速 

正确答案： B. MPP 架构可以作用于 TiKV 和 TiFlash 上的数据

B：MPP现在只作用于TiFlash上的数据



# Lesson 7 TiFlash

## 1. 描述TiFlash架构

![image-20230804114614700](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804114614700.png)

在集群中加入了TiFlash，其他的组件没有变化，TiFlash就是TiKV的列存版本，他们的region是完全对应的，会随着TiKV中的region变化而变化。是根据learner来与TiKV同步的。TiFlash对于TiKV的性能影响是很低的，只是同步日志，从而做到了业务的隔离性。

TiDB Server会将适合全表扫描的业务路由到TiFlash中，在线业务路由到TiKV中。



## 2. 理解TiFlash主要功能

- 异步复制
- 一致性读取
- 引擎智能选择
- 计算加速

### 异步复制

![image-20230804115937128](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804115937128.png)

  TiFlash是以Learner的角色来接收TiKV的数据，不参与TiKV中Leader的选举和决策等待，只是接收TiKV中的raft log日志数据，所以是异步复制的。其内部是基于主键的快速更新机制。

TiFlash不能写入数据，是异步复制TiKV中的数据，但是读取的时候是可以的。主要承载OLTP业务。

TiFlash宕机的时候，不会影响TiKV的功能，因为只是复制Log。

### 一致性读取

![image-20230804131737853](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804131737853.png)

存储了历史的数据和现在的数据，但是只会看到现在的数据，不会看到历史版本了。一致性读取是一个非常重要的特性。

![image-20230804145733150](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230804145733150.png)

![image-20230805155828713](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230805155828713.png)

![image-20230806222834317](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230806222834317.png)

要队raft log index做一个确认，确认现在TiKV的raft log号以及TiFlash的region中存在的index，在确认后，TiFlash就等待到确认时的index出现。

![image-20230806222934483](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230806222934483.png)

等到确认时的index，那么可以保证在读取的时刻已经存入的数据。



![image-20230806223034981](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230806223034981.png)

如果发现有两个版本的数据，那么会看这个读取是哪一个时刻发生的，当前读取key=1的请求是T1时刻发生的，因此就会读取T1时刻及之前的版本，也就是T0时刻key=1的数据。

### 智能选择

![image-20230806223436106](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230806223436106.png)

把对product表的扫描放在TiKV上做，是因为结果中没有用到product表的数据，并且表上存在索引，是在大量数据中找少量的数据，是OLAP业务。

把队sales表的扫描放到TiFlash上做，是因为最终结果是要求sales某一列的平均值，十分适合列读，并且只扫pid和price两列。

智能选择功能选择适合的操作放到TiKV或TiFlash上操作。



### 计算加速

就是将一部分的计算放到TiFlash上做，并且将适合于列扫描的操作放到TiFlash上做。

### 练习

1. 下面属于 TiFlash 核心特性的是？（请选择 3 项）

A. 采用行存 + 列存的混合存储方式

B. region 支持 raft 投票和选举

C. TiFlash 采用异步复制来保证和 TiKV 一致

D. 在 TiKV 上写入数据成功后，在 TiFlash 上可以一致性读取

E. CBO 基于成本选择在 TiFlash 或者 TiKV 上执行 SQL（智能选择）

正确答案： C. TiFlash 采用异步复制来保证和 TiKV 一致 、 D. 在 TiKV 上写入数据成功后，在 TiFlash 上可以一致性读取 、 E. CBO 基于成本选择在 TiFlash 或者 TiKV 上执行 SQL

A：只采用列存的存储方式

B：投票和选举是TiKV的特性，TiFlash不具有，TiFlash的角色是Learner，不具有投票和选举特性。

2. 关于 TiFlash 的使用，描述不正确的是？

 A. TiFlash 不善于处理高并发，QPS 一般不应过高 

 B. SQL 语句执行中，要不然数据完全从 TiKV 中读取，要不然完全从 TiFlash 中读取 

 C. MPP 中表连接前的过滤和交换完全是在 TiFlash 节点上完成的 

 D. 在读取 TiFlash 中数据的时候，我们需要通过 TiKV 中的数据确认一致性 

正确答案： B. SQL 语句执行中，要不然数据完全从 TiKV 中读取，要不然完全从 TiFlash 中读取

B：具有智能选择的特性，而可以一部分数据从TiKV中读取，一部分从TiFlash中读取

# Lesson 8 TiDB 6.0 新特性

## 1. Placement Rules in SQL

![image-20230806224944900](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230806224944900.png)

TiDB6.0之前，分布式数据库最少一般部署在三个数据中心，如图所示。但是典型的分布式数据库会出现三个问题。

![image-20230806230637914](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230806230637914.png)

本地的用户需要访问的数据尽量放在了本地。

根据不同的业务，隔离了TiKV存储。

可以修改特定表的副本，并且可以指定表放置的具体位置，根据业务等级分配资源。

![image-20230806230834162](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230806230834162.png)

![image-20230806231012685](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230806231012685.png)

![image-20230806231210675](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230806231210675.png)

![image-20230806231255383](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230806231255383.png)



## 2. 小表缓存

 ![image-20230806231415949](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230806231415949.png)

![image-20230807104920492](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807104920492.png)

注意，热点小表缓存要在64M一下。

tidb_table_cache_lease是缓存的租约。

![image-20230807105206827](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807105206827.png)

在租约时间内，用户可以从缓存中读取这张表，不需要取TiKV中读取了。任何用户要写这张表，写不了，写阻塞了，需要等待租约到期之后。在租约中只能读不能写，

![image-20230807105350975](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807105350975.png)

租约到期之后依然可以读，但是就不是从缓存当中读了，而是去TiKV上读取了。在租约内读很快，在租约到期后就会有一些延迟。

![image-20230807105551298](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807105551298.png)

最后refresh，刷新缓存中的数据，这时候读取的话又可以去缓存读了。

![image-20230807105703139](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807105703139.png)

## 3. 内存悲观锁

![image-20230807105907631](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807105907631.png)

乐观锁是只有在commit的时候其他人才知道锁的存在，悲观锁是在修改的时候别人就会知道锁的存在，TIDB使用悲观锁机制。

悲观锁可以减少事务回滚的可能性，可以在修改的时候让别人感知到锁的存在，但是所有的锁信息都要存储到TiKV中，并且还要复制到其他TiKV节点上，因此会有磁盘IO和网络IO。

![image-20230807110704121](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807110704121.png)

内存悲观锁没有将锁信息存储到TiKV的rocksdb 当中，而是放到了内存当中，这样可以减少磁盘IO和网络IO，降低负载。这个时候只写到内存中，加快了效率，性能提升了。

![image-20230807110827457](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807110827457.png)

内存悲观锁的问题就是所丢失，宕机之后，内存中的数据就不存在了。

如果锁丢失了，那么事务就会失败。如果要完全防止锁丢失，那么还是要将锁存到rocksdb 当中。

![image-20230807111001829](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807111001829.png)

![image-20230807111012099](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807111012099.png)



## 4. Top SQL

![image-20230807111107495](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807111107495.png)

Top SQL主要针对性能可观测性的问题。

![image-20230807111241527](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807111241527.png)

主要负责找到哪个SQL导致负载很高。

![image-20230807111302197](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807111302197.png)

根据Top SQL功能，可以帮助我们选定具体是哪个TiKV或TiDB实例中，看到最前面Top5类开销最大的SQL。这样对SQL的监控就可以细化到TiKV了，而原本的对SQL的监控是对于整个集群的，不能具体到哪一个TiKV中。

![image-20230807111501654](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807111501654.png)

![image-20230807111542029](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807111542029.png)

现在暂时只支持CPU负载的查看。

![image-20230807111619869](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807111619869.png)

![image-20230807111636243](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807111636243.png)



## 5. TiDB Enterprise Manager (TiEM)

![image-20230807111717155](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807111717155.png)

TiEM为减少决数据库运维场景当中的人工成本。

![image-20230807111957194](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807111957194.png)

![image-20230807112205212](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807112205212.png)

### 练习

1. 对于 TiDB v6.0 新特性描述正确的为？（请选择 3 项）

A. 小表缓存支持 DML 和 DDL 语句操作

B. 内存悲观锁功能可以起到降低网络带宽的作用

C. 当某个 TiKV 实例的 IO 过高，我们可以通过 Top SQL 监控到其上 IO 最高的 5 类 SQL 语句

D. TiDB Enterprise Manager（TiEM）可管理多套集群

E. 我们可以通过 Placement Rules in SQL 功能增加某些重要业务表的副本数

正确答案： B. 内存悲观锁功能可以起到降低网络带宽的作用 、 D. TiDB Enterprise Manager（TiEM）可管理多套集群 、 E. 我们可以通过 Placement Rules in SQL 功能增加某些重要业务表的副本数

A：支持DML，不支持DDL。DDL语句必须要在小表缓存关闭的时候才可以做。

C：Top SQL现在只能检测CPU负载，不能检测IO、内存之类的。

2. 下列哪些情况不适宜开启小表缓存？（请选择 2 项）

A. 表数据量小于 128 MB

B. 频繁读取的热点小表

C. 只读的热点小表

D. 读取和修改都非常频繁的热点小表

正确答案： A. 表数据量小于 128 MB 、 D. 读取和修改都非常频繁的热点小表

A：只有数据量小于64MB的适宜开启

D：修改非常频繁的不适合



# Lesson 9 TiDB Cloud 简介

## 1. 为什么选择TiDB？

### TiDB的好处

**TiDB数据库作为云原生数据库的好处：**

- 分布式SQL数据库-多租户
- 混合工作负载 - 在同一个数据库中
  - 事务型：基于行的数据
  - 分析性：基于列的数据
- 弹性比例
  - 缩小 -  减少节点
  - 横向扩展 - 添加节点
- 基于 RAFT 的高可用性
  - 每个数据段在3个可用区进行复制



### 多租户

![image-20230807113644464](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807113644464.png)

传统的使用数据库软件：购买服务器，安装OS，买数据库。但是这种开销很大。

现在的云原生模式，云服务提供商购买服务器，安装OS，然后分割独立空间，提供云数据库服务给用户，这样用户只需要租用就可以了，不需要购买服务器了。这个就是多租户的概念，可以将多个用户的数据隔离开。

### TiDB 架构

![image-20230807114016665](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807114016665.png)



## 2. 什么是TiDB Cloud？

### 本地数据库与云DBaas的比较

![image-20230807120346227](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807120346227.png)



### TiDB Cloud定义

![image-20230807114117017](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807114117017.png)

用户不再具体操作TiDB了，而是使用DBaaS的数据库服务。



如果要做一个论坛，那么要自己买数据库、操作系统、软件等等，成本很高。

如果使用IaaS服务，那么服务器这种硬件就不需要考虑了，直接在云厂商提供的服务器上部署应用就可以了。

如果使用PaaS，就是云厂商不仅提供硬件，还会提供一些软件。

如果使用SaaS，那么就可以买一些基础模板服务，比如买一个论坛模板。

IaaS: Infrastructure as a Service基础设施即服务

PaaS: Platform as a Service 平台即服务

SaaS: Software as a Service 软件即服务

DBaaS：Database as a Service，对应的是PaaS，提供了一个数据库平台，用户可以直接访问提供的数据库服务。



### TiDB Cloud 供应商 Region

![image-20230807120726233](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807120726233.png)

VPC：虚拟专有网络，在VPC中可以完全控制自己的网络，可以在自己定义的网络中定义这些资源。



## 3. TiDB Cloud 实现示例

### TiDB Cloud入门

![image-20230807121206128](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807121206128.png)

![image-20230807121227971](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807121227971.png)

![image-20230807121239469](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807121239469.png)

![image-20230807121249535](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807121249535.png)

![image-20230807121309410](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807121309410.png)

![image-20230807121413245](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807121413245.png)

![image-20230807121511787](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807121511787.png)



### TiDB Cloud实现示例

![image-20230807121814602](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807121814602.png)

![image-20230807121952704](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807121952704.png)

![image-20230807121958519](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807121958519.png)

![image-20230807122005963](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807122005963.png)

![image-20230807122028762](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807122028762.png)

![image-20230807122050298](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807122050298.png)

![image-20230807122121577](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807122121577.png)

![image-20230807122132002](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807122132002.png)

![image-20230807122148814](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807122148814.png)

![image-20230807122207582](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807122207582.png)

![image-20230807122214426](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807122214426.png)

![image-20230807122230510](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807122230510.png)

![image-20230807122243433](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807122243433.png)

![image-20230807122307418](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807122307418.png)

![image-20230807122322144](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807122322144.png)

![image-20230807122345650](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807122345650.png)

![image-20230807122412104](C:\Users\Raymond\AppData\Roaming\Typora\typora-user-images\image-20230807122412104.png)

### 练习

1. 下面对于 TiDB Cloud 描述，正确的为？（请选择 4 项）

A. 属于 DBaaS 服务

B. 数据属于客户自己和云服务厂商

C. 都具有 VPC Peer

D. 属于多租户架构

E. 不仅支持自动备份还支持手动备份

F. 支持一定的删除后还原

正确答案： A. 属于 DBaaS 服务 、 D. 属于多租户架构 、 E. 不仅支持自动备份还支持手动备份 、 F. 支持一定的删除后还原

B：数据属于客户自己

C：Developer Tier不具有VPC Peer，Dedicated Tier具有VPC Peer

2. 关于 Developer Tier 和 Dedicated Tier，下面说法正确的为？（请选择 2 项）

A. 都支持 VPC Peer

B. 都支持横向扩容和缩容

C. 都支持 TiFlash 节点

D. 都具有多租户特性

E. 都具有高可用性

正确答案： C. 都支持 TiFlash 节点 、 D. 都具有多租户特性

A、B、E：Developer Tierdd都不支持，Dedicated Tierd都支持



