# Mysql

##  数据结构

#### Hash

通过**哈希算法**算出key的哈希值，能够快速定位对应value

```java
hash = hashfunc(key)
index = hash % array_size
```
- 缺点

  存在**哈希冲突**问题，不同的key算出的hash值可能相同。常用解决方法为**链地址法**，将冲突数据放入链表中。
  
  <font color="#E3170D">* 无法进行范围查询</font>

#### B+树
一页默认大小16kb（16384byte）int长度为 8 byte 指针为 6 byte 16384 / (8 + 6) ≈ 1170
假设一条数据1kb 一页就可以存放16 / 1 = 16条数据 那么可以算出
一颗高度为2的B+树可以存放 1170 * 16 = 18720 条数据
一颗高度为3的B+树可以存放 1170 * 1170 * 16 = 21,902,400 条数据（约为2千万）
<font color="#E3170D">* 一般单表数据达到2千万就需要考虑分库分表了</font>
![](https://github.com/710765989/learningDocument/blob/main/img/951914-20210806223911480-1865018984.png)

#### B树和B+树的区别

- B树
  1. 节点排序
  2. 一个节点可以存储多个元素，多个元素之间也进行了排序
  
- B+树
  1. 拥有B树的特点
  2. 叶子节点之间有双向指针
  3. 非叶子节点上的元素在叶子节点上做了冗余处理，因此叶子节点上存在所有元素，且已排好序
  
  索引用来加快查询速度，B+树结构通过对数据进行排序来提高查询速度，通过一个节点可以存储多个数据，使得B+树高度不会太高

***

## 三大日志(binlog、redo log和undo log)

`MySQL` 日志 主要包括错误日志、查询日志、慢查询日志、事务日志、二进制日志几大类。其中，比较重要的还要属二进制日志 `binlog`（归档日志）和事务日志 `redo log`（重做日志）和 `undo log`（回滚日志）。

### binlog与redo log的区别

- redo log属于innodb引擎，binlog属于mysql
- redo log会覆盖旧数据，binlog会一直增加（可以设置过期时间）
- redo log是用来**防止mysql异常导致修改的数据丢失**，binlog是用来**数据备份**和**主从节点数据同步**，保证数据一致性
- redo log是物理日志：记录某页做了什么修改（某个偏移量的值）。binlog是逻辑日志：记录语句原始逻辑（sql、数据行）

### redo log

`redo log`（重做日志）为`InnoDB`存储引擎独有，它让`MySQL`拥有了崩溃恢复能力。

如果 `MySQL` 实例挂了或者宕机了，重启时候，`InnoDB`存储引擎会使用`redo log`恢复数据，来保证数据的持久性与完整性。



`Mysql`中的数据以页为单位，查询一条数据时，会从硬盘中加载一页的数据，叫数据页，将其放入`buffer pool` 缓冲池中

后续查询会先在`buffer pool` 缓冲池中查询，如果未命中再去硬盘中加载，减少IO开销，提升性能。



`redo log buffer` 大小默认为16mb（`innodb_log_buffer_size`），分为很多个块，每块大小为512kb，每个块都对应磁盘上一块区域，每次`redo log`落盘是将块缓存写入对应的磁盘区域

`redo log`为一个环形数组结构，写满之后会覆盖前边的数据，又重头开始写



既然redo log也要进行IO操作，为什么不直接写入磁盘？这样有什么好处？

1. 写入log是顺序写入追加在文件末尾，而写入磁盘是随机写入，极大提升性能。
1. 数据从磁盘中加载到内存`buffer pool`中是按页为单位，而一页为16kb，一次事务可能只有不到1kb，因此如果每次事务都直接将数据落到磁盘，那么每次都至少需要写入16kb数据。

#### 刷盘策略 

`innodb_flush_log_at_trx_commit`

- 0：设置为 0 的时候，表示每次事务提交时不进行刷盘操作

- 1：设置为 1 的时候，表示每次事务提交时都将进行刷盘操作（默认值）

- 2：设置为 2 的时候，表示每次事务提交时都只把 `redo log buffer` 内容写入 `page cache`文件系统缓存

另外，`InnoDB` 存储引擎有一个后台线程，每隔`1` 秒，就会把 `redo log buffer` 中的内容写到文件系统缓存（`page cache`），然后调用 `fsync` 刷盘。

![](https://github.com/710765989/learningDocument/blob/main/img/04.png)

一个没有提交事务的 `redo log` 记录，也可能会刷盘。

因为在事务执行过程 `redo log` 记录是会写入`redo log buffer` 中，这些 `redo log` 记录会被后台线程刷盘。

![](https://github.com/710765989/learningDocument/blob/main/img/05.png)



下面是不同刷盘策略的流程图。

**innodb_flush_log_at_trx_commit=0**

![](https://github.com/710765989/learningDocument/blob/main/img/06.png)

**innodb_flush_log_at_trx_commit=1**

![](https://github.com/710765989/learningDocument/blob/main/img/07.png)

为`1`时， 只要事务提交成功，`redo log`记录就一定在硬盘里，不会有任何数据丢失。

如果事务执行期间`MySQL`挂了或宕机，这部分日志丢了，但是事务并没有提交，所以日志丢了也不会有损失。

**innodb_flush_log_at_trx_commit=2**

![](https://github.com/710765989/learningDocument/blob/main/img/09.png)

为`2`时， 只要事务提交成功，`redo log buffer`中的内容只写入文件系统缓存（`page cache`）。

如果仅仅只是`MySQL`挂了不会有任何数据丢失，但是宕机可能会有`1`秒数据的丢失。



#### 刷盘时机

  1. 缓存内容超过空间的一半（16mb/2）
  2. 后台线程每秒刷入磁盘
  3. 每次事务提交
  4. Mysql关闭时



#### 日志文件组

硬盘上存储的 `redo log` 文件不止一个，是以**日志文件组**形式出现，每个`redo log`文件大小相同

它采用环形数组结构

![](https://github.com/710765989/learningDocument/blob/main/img/10.png)

在**日志文件组**中有两个重要属性，分别是：

- **write pos** ：表示当前记录的位置，一边写入一边往后移动
- **checkpoint** ：表示当前要擦除的位置

当每次将`redo log`刷盘到硬盘中时，就会更新**write pos**的位置

当每次`Mysql`加载**日志文件组**来恢复数据时，会清空加载过的`redo log`内容，并将`checkpoint`往后移

![](https://github.com/710765989/learningDocument/blob/main/img/11.png)

如果`write pos`与`checkpoint`指针重合，则表示**日志文件组**已经满了，此时会清空一些记录，并更新`checkpoint`位置



### binlog

`binlog`属于逻辑日志，记录语句的原始逻辑，它属于`Mysql`

`MySQL`数据库的**数据备份、主备、主主、主从**都离不开`binlog`，需要依靠`binlog`来同步数据，保证数据一致性。

![](https://github.com/710765989/learningDocument/blob/main/img/01-20220305234724956.png)

`binlog`会记录所有更新操作，顺序写入。

#### 记录格式

`binlog`有三种记录格式，通过`binlog_format`参数指定

- statement 记录内容为sql原文
- row （默认）
- mixed

##### statement

`statement`模式下，记录的内容为SQL原文，比如执行一条`update T set update_time=now() where id=1`，记录的内容如下。

![](https://github.com/710765989/learningDocument/blob/main/img/02-20220305234738688.png)

同步数据时会执行SQL语句，但是存在一个问题，`update_time=now()`会获取当前系统时间，与原数据不一致

##### row

为了解决这个问题，需要指定为`row`模式，该模式下会记录所有具体数据，记录内容如下。

![](https://github.com/710765989/learningDocument/blob/main/img/03-20220305234742460.png)

`row`格式记录的内容无法直接读取，需要通过`mysqlbinlog`工具进行解析。

`row`格式为默认格式，它能为数据恢复以及同步带来更好的可靠性，但是它需要占用更多的空间，恢复与同步时也会给`IO`带来更大压力，影响执行速度。

##### mixed

因此，出现了折中方案`mixed`，`mixed`模式为前两种模式的混合。

`Mysql`会判断sql语句是否可能会引起数据不一致，如果是，则采用`row`模式，否则采用`statement`模式进行记录。



#### 写入机制

①事务执行过程中，将日志写入`binlog cache`

②事务提交时，把`binlog cache`中的内容写到`binlog`文件中去

因为一个事务的`binlog`不能被拆开，无论这个事务多大，也要确保一次性写入，所以系统会给每个线程分配一个块内存作为`binlog cache`。

我们可以通过`binlog_cache_size`参数控制单个线程 binlog cache 大小，如果存储内容超过了这个参数，就要暂存到磁盘（`Swap`）。

`binlog cache`默认大小为4kb



`binlog`日志刷盘流程如下

![](https://github.com/710765989/learningDocument/blob/main/img/04-20220305234747840.png)

- **上图的 write，是指把日志写入到文件系统的 page cache，并没有把数据持久化到磁盘，所以速度比较快**
- **上图的 fsync，才是将数据持久化到磁盘的操作**



##### sync_binlog

`write`和`fsync`的时机，可以由参数`sync_binlog`控制，在`MySQL 5.7.7`之前默认值是`0`，之后默认值为`1`。

- 0：表示每次提交事务都只`write`，由系统自行判断什么时候执行`fsync`。

  ![](https://github.com/710765989/learningDocument/blob/main/img/05-20220305234754405.png)

​	虽然性能会得到提升，但是如果机器宕机，`page cache`里缓存的`binlog`会丢失。

- 1：表示每次提交事务都会执行`fsync`，就如同 **redo log 日志刷盘流程** 一样。
- `N(N>1)`：这是一种折中的方式，表示每次提交事务都`write`，但累积`N`个事务后才`fsync`。

  ![](https://github.com/710765989/learningDocument/blob/main/img/06-20220305234801592.png)
  
  在出现`IO`瓶颈的场景里，将`sync_binlog`设置成一个比较大的值，可以提升性能。
  
  同样的，如果机器宕机，会丢失最近`N`个事务的`binlog`日志。



### 两阶段提交

`redo log`（重做日志）让`InnoDB`存储引擎拥有了崩溃恢复能力。

`binlog`（归档日志）保证了`MySQL`集群架构的数据一致性。

虽然它们都属于持久化的保证，但是侧重点不同。



在执行更新语句过程，会记录`redo log`与`binlog`两块日志，以基本的事务为单位，`redo log`在事务执行过程中可以不断写入，而`binlog`只有在提交事务时才写入，所以`redo log`与`binlog`的写入时机不一样。

 ![](https://github.com/710765989/learningDocument/blob/main/img/01-20220305234816065.png)

`redo log`与`binlog`两份日志之间的逻辑不一致，会出现什么问题？

以`update`语句为例，假设`id=2`的记录，字段`c`值是`0`，把字段`c`值更新成`1`，`SQL`语句为`update T set c=1 where id=2`。

假设执行过程中写完`redo log`日志后，`binlog`日志写期间发生了异常，会出现什么情况呢？

![](https://github.com/710765989/learningDocument/blob/main/img/02-20220305234828662.png)

由于`binlog`没写完就异常，这时候`binlog`里面没有对应的修改记录。因此，之后用`binlog`日志恢复数据时，就会少这一次更新，恢复出来的这一行`c`值是`0`，而原库因为`redo log`日志恢复，这一行`c`值是`1`，最终数据不一致。

![](https://github.com/710765989/learningDocument/blob/main/img/03-20220305235104445.png)

为了解决两份日志之间的逻辑一致问题，`InnoDB`存储引擎使用**两阶段提交**方案。

原理很简单，将`redo log`的写入拆成了两个步骤`prepare`（准备）和`commit`（提交），这就是**两阶段提交**。

![](https://github.com/710765989/learningDocument/blob/main/img/04-20220305234956774.png)

使用**两阶段提交**后，写入`binlog`时发生异常也不会有影响，因为`MySQL`根据`redo log`日志恢复数据时，发现`redo log`还处于`prepare`阶段，并且没有对应`binlog`日志，就会回滚该事务。

![](https://github.com/710765989/learningDocument/blob/main/img/05-20220305234937243.png)

`redo log`设置`commit`阶段发生异常，那会不会回滚事务呢？

![](https://github.com/710765989/learningDocument/blob/main/img/06-20220305234907651.png)

并不会回滚事务，它会执行上图框住的逻辑，虽然`redo log`是处于`prepare`阶段，但是能通过事务`id`找到对应的`binlog`日志，所以`MySQL`认为是完整的，就会提交事务恢复数据。





### undo log

`undo log`**回滚日志**用于配合隐藏字段`DB_ROLL_PTR`进行事务回滚，保证事务的**原子性**。

回滚日志会先于数据持久化到磁盘上。这样就保证了即使遇到数据库突然宕机等情况，当用户再次启动数据库的时候，数据库还能够通过查询回滚日志来回滚将之前未完成的事务。

MySQL InnoDB 引擎使用 **redo log(重做日志)** 保证事务的**持久性**，使用 **undo log(回滚日志)** 来保证事务的**原子性**。

`MySQL`数据库的**数据备份、主备、主主、主从**都离不开`binlog`，需要依靠`binlog`来同步数据，保证数据一致性。



### MVCC

MVCC（Multiversion Concurrency Control）中文全称**多版本并发控制**

#### 隐藏字段

Mysql有三个隐藏字段

- `DB_TRX_ID`：6字节，最近修改事务id，记录创建这条记录或者最后一次修改记录的id

- `DB_ROLL_PTR`：7字节，回滚指针，指向这条记录的上一个版本，用于配合`undo log`找到上一个版本的记录

- `DB_ROW_ID`：6字节，隐藏的主键，如果数据表没有主键，那么innodb会自动生成一个6字节的`row_id`

#### Read View

`Read View`是事务进行快照读操作的时候产生的读视图，用来做可见性判断

`Read View`的三个全局属性：

- `trx_lis`：一个数值列表，用来维护`Read View`生成时刻系统中正在活跃的事务ID（如：1,2,3）
- `up_limit_id`：记录`trx_lis`列表中最小的事务ID（如：1）
- `low_limit_id`：记录`Read View`生成时刻系统尚未分配的下一个事务ID（如：4）

#### 可见性规则

1. 比较`DB_TRX_ID` < `up_limit_id`

   [ **<** ]：如果小于，则当前事务能够看到`DB_TRX_ID`所在的记录，因此**可见**

   [ **>=** ]：如果大于等于，进入下一个判断

2. 判断`DB_TRX_ID` >= `low_limit_id`
   
   [ **>=** ]：如果大于等于，代表`DB_TRX_ID`所在的记录是在`Read View`生成后才出现的，那么对于当前事务肯定不可见，因此**不可见**
   
   [ **<** ]：如果小于，进入下一个判断

3. 判断`DB_TRX_ID`是否在活跃事务列表`trx_lis`中

   True：如果在，表示`Read View`生成时刻，该事务还处于活跃状态，还没有进行commit，它修改的数据，当前事务当然无法看到，因此**不可见**

   False：如果不在，则说明这个事务在`Read View`生成之前就已经开始commi，那么修改的结果应当是能够看到的，因此**可见**





## 事务隔离级别

- **READ-UNCOMMITTED(读取未提交)**：最低的隔离级别，允许读取尚未提交的操作，可能会引起**脏读**、**幻读**、**不可重复读**
- **READ-COMMITTED(读取已提交)**：允许读取已经提交的操作，解决了`脏读`问题，但是还是可能会引起**幻读**和**不可重复读**
- **REPEATABLE-READ(可重复读)** ：对同一字段多次读取结果都是一致的，除非被当前事务进行了数据修改，阻止了`脏读`和`不可重复读`，**幻读**仍有可能发生
- **SERIALIZABLE(可串行化)**：最高隔离级别，完全服从ACID的隔离级别。所有事务依次执行，阻止了`脏读`、`幻读`和`不可重复读`，但效率也是最低的。

|     隔离级别     | 脏读 | 不可重复读 | 幻读 |
| :--------------: | :--: | :--------: | :--: |
| READ-UNCOMMITTED |  √   |     √      |  √   |
|  READ-COMMITTED  |  ×   |     √      |  √   |
| REPEATABLE-READ  |  ×   |     ×      |  √   |
|   SERIALIZABLE   |  ×   |     ×      |  ×   |

- **脏读**：事务1读取到事务2已提交的数据，然后事务2发生了回归，导致事务1最终取到的数据为错误的。
- **不可重复读**：事务1两次读取到的数据不一致（事务2进行了commit）
- **幻读**：事务1两次查询到的数据量不一致（实物2做了增删操作，并commit）

`Mysql`默认隔离级别为**REPEATABLE-READ(可重复读)**，`Postgresql`默认隔离级别为**READ-COMMITTED(读取已提交)**

***

### Mysql是如何解决幻读问题的？

Mysql在**REPEATABLE-READ(可重复读)**级别下，解决了幻读问题，它是怎么做到的呢？

在**REPEATABLE-READ(可重复读)**级别下，给事务操作的表添加`Next-key Lock（Record Lock+Gap Lock）`（行锁+间隙锁）

`InnoDB`存储引擎在 RR 级别下通过 `MVCC`和 `Next-key Lock` 来解决幻读问题：

1. **执行普通 `select`，此时会以 `MVCC` 快照读的方式读取数据**

   在**快照读**的情况下，RR 隔离级别只会在事务开启后的第一次查询生成 `Read View` ，使用它直到事务提交，因此在生成`Read View`之后所做的操作是**不可见**的，实现了**可重复读**和**防止快照读情况下的幻读**。

2. **执行 select...for update/lock in share mode、insert、update、delete 等当前读**

   在**当前读**情况下，读取的都是最新的数据，如果其它事务有插入新的记录，并且刚好在当前事务查询范围内，就会产生**幻读**。

   `Innodb`通过`Next-key Lock（Record Lock+Gap Lock）`来防止这种情况，在执行当前读的情况下，会**锁定记录所在行**，以及它的**间隙**，防止其他事务在查询范围内增删数据，以此来防止**幻读**

***

## RC 和 RR 隔离级别下 MVCC 的差异

在`RC`和`RR`隔离级别下，虽然都使用了`MVCC`，但是它们生成的时机不同，因此在**RR**级别下能够实现**可重复读**

- **RC**：每次查询前都生成一个`Read View`
- **RR**：只在事务开始后，第一次查询时候生成`Read View`

***

## 索引

### 聚集索引与非聚集索引

####  聚集索引

**聚集索引即索引结构和数据一起存放的索引。主键索引属于聚集索引。**



**聚集索引的优点**

聚集索引的查询速度非常的快，因为整个 B+树本身就是一颗多叉平衡树，叶子节点也都是有序的，定位到索引的节点，就相当于定位到了数据。



**聚集索引的缺点**

1. **依赖于有序的数据** ：因为 B+树是多路平衡树，如果索引的数据不是有序的，那么就需要在插入时排序，如果数据是整型还好，否则类似于字符串或 UUID 这种又长又难比较的数据，插入或查找的速度肯定比较慢。
2. **更新代价大** ： 如果对索引列的数据被修改时，那么对应的索引也将会被修改，而且聚集索引的叶子节点还存放着数据，修改代价肯定是较大的，所以对于主键索引来说，主键一般都是不可被修改的。



#### 非聚集索引

**非聚集索引即索引结构和数据分开存放的索引。**

**二级索引属于非聚集索引。**



**非聚集索引的优点**

**更新代价比聚集索引要小** 。非聚集索引的更新代价就没有聚集索引那么大了，非聚集索引的叶子节点是不存放数据的



**非聚集索引的缺点**

1. 跟聚集索引一样，非聚集索引也依赖于有序的数据
2. **可能会二次查询(回表)** :这应该是非聚集索引最大的缺点了。 当查到索引对应的指针或主键后，可能还需要根据指针或主键再到数据文件或表中查询。

![](https://github.com/710765989/learningDocument/blob/main/img/20210420165311654.png)

### 覆盖索引

如果一个索引包含（或者说覆盖）所有需要查询的字段的值，我们就称之为“覆盖索引”。

**覆盖索引即需要查询的字段正好是索引的字段，那么直接根据该索引，就可以查到数据了，而无需回表查询。**



### 联合索引

使用表中的多个字段创建索引，就是 **联合索引**，也叫 **组合索引** 或 **复合索引**。

#### 最左前缀匹配原则

在使用联合索引时，**MySQL** 会根据联合索引中的字段顺序，从左到右依次到查询条件中去匹配，如果查询条件中存在与联合索引中最左侧字段相匹配的字段，则就会使用该字段过滤一批数据，直至联合索引中全部字段匹配完成



### 索引下推

索引下推是 **MySQL 5.6** 版本中提供的一项索引优化功能，可以在非聚簇索引遍历过程中，对索引中包含的字段先做判断，过滤掉不符合条件的记录，减少回表次数。



### 什么情况会造成索引失效

1. 查询条件中有or，且部分条件没有添加索引
2. 复合索引未遵循最左原则
3. like以 % 开头
4. 需要进行类型转换
5. where条件中，索引列有运算
6. where条件中，索引列使用了函数
7. 加了索引的列，数据重复率高



### 执行计划 explain

