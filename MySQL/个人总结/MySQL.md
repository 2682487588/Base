## MVCC

### MVCC

> MVCC,多版本并发控制，MVCC是一种并发控制的方法，实现对数据库的并发访问

**MVCC**在 **InnoDB**中实现主要是对实现对数据库的性能提升，做到即使有**读写冲突也能做不加锁，非阻塞并发读**

### 当前读和快照读

- **当前读**

  读取的是当前最新版本的记录，**而且读取的时候需要保证其他并发事务不能修改本地记录，对读取的记录进行加锁**

- **快照读**

  1 `不加锁`的select操作，即不加锁的非阻塞读，**快照读的前提隔离级别不是串行**，如果串行级别下快照会退化为当前读，**快照读是为了提高并发效率**，  快照读实现是基于MVCC，避免了加锁，节省开销，**不过既然是多版本，那么快照读实现的可能就不是最新数据*

**MVCC实现 读-写冲突不加锁，这个读是`快照读`，当前读是加锁的操作，是悲观锁的实现**

### 当前读和快照读和MVCC的关系

- MVCC多版本并发控制室 **维持一个数据多个版本，使得读写操作没有冲突**
- MVCC是抽象概念，但是要实现这个抽象概念，MySQL就需要提供具体功能实现  **快照读就是实现MySQL中MVCC的非阻塞读的功能**
-  MVCC在MySQL中的具体实现是由，`3个隐式字段`，`undo log`, `Read view`实现的

### MVCC可以解决什么，好处是什么

**数据库并发场景有三种**

- `读-读`：不存在问题，不需要并发控制
- `读-写`：有线程安全问题，可能会造成事务隔离性问题，如 脏读，不可重复度，幻读
- `写-写`：有线程安全问题，可能会存在更新丢失问题， 如第一类更新丢失，第二类更新丢失

**MVCC带来的好处是**

MVCC是解决`读-写冲突`的无锁并发控制，为事务分配一个增长的时间戳，为每次修改保存一个版本，每个版本对应一个时间戳，解决的问题有

- 在并发读写数据库，读操作不影响写操作，写操作不影响读操作，提高数据库并发性能
- 解决脏读，不可重复读，幻读等操作，**但是不能解决更新丢失问题**

**MVCC的两种组合**

- `MVCC+悲观锁`

  MVCC解决读写冲突，悲观锁解决写写冲突

- `MVCC+乐观锁`

  MVCC解决读写冲突，乐观锁解决写写冲突

## MVCC的实现原理

MVCC为了解决`读写冲突`，实现原理主要由 `3个隐式字段`，`undo日志`, `Read view实现的`

#### **隐式字段**

除了自定义字段，还有数据库隐式定义的`DB_TRX_ID`,`DB_ROLL_PTR`,`DB_ROW_ID`

- `DB_TRX_ID`

  6byte,最近修改(修改/插入)事务ID,  记录创建/最后一次修改记录的ID

- `DB_ROLL_PTR`

  7byte,回滚指针,记录这条记录的上一个版本（配合undo日志）

- `DB_ROW_ID`

  6byte,隐含的自增ID(隐含主键)，如果数据表没有主键，InnoDB会自动以`DB_ROW_ID`形式创建聚簇索引

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190313213705258.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NuYWlsTWFubg==,size_16,color_FFFFFF,t_70)

#### **undo日志**

undolog负责回滚

undo log 主要分为两种:

- **insert undo log**  
  代表事务在`insert`产生新纪录时候的`undo log`，只有回滚时候需要，事务提交后可以立即删除

- **update undo log**

  事务在进行`update`或者`delete`产生`undo log`,不仅在事务回滚需要，在快照读也需要，只有在快照读或回滚事务不涉及日志，对应的日志被**purge**线程统一删除

> **purge**
>
> - 实现InnoDB的MVCC，更新或删除都是设置老记录的delete_bit并不是真正删除过时就
> - 为了节省磁盘，InnoDB会专门有purge来delete_bit的记录

对MVCC有帮助的是 `update undo log` , undo-log实际是 `rollback segment`记录链

**一 比如说有一个事务插入 persion 表插入了一条新记录**,记录如下

- name : Jerry
- age : 24 
- `隐式逐渐`: 1
- `事务ID`：null
- `回滚指针`:null

![img](https://img-blog.csdnimg.cn/20190313213836406.png)

**二，现在来了一个`事务 1`对该记录的 `name` 做出了修改，改为 Tom**

- `事务1`修改的时候，数据库会对该事务先加`排它锁`
- 把改行拷贝到`undo log`  作为旧的记录
- 拷贝完成，修改 `name`为Tom，  修改当前事务ID为  `事务1`的ID
- 事务提交后，释放锁

![img](https://img-blog.csdnimg.cn/20190313220441831.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NuYWlsTWFubg==,size_16,color_FFFFFF,t_70)

**三、又来了个`事务 2`修改`person 表`的同一个记录，将`age`修改为 30 岁** 

- 将`事务2` 修改该行数据，数据库先加行锁
- 把数据拷贝到`undo log`,发现记录已经在`undo log`,    最新的旧数据作为链表的表头，作为链表的表头，插在该行记录的 `undo log` 最前面
- 修改该行 `age` 为 30 岁，并且修改隐藏字段的事务 ID 为当前`事务 2`的 ID, 那就是 `2` ，回滚指针指向刚刚拷贝到 `undo log` 的副本记录
- 事务提交，释放锁















