1. 主从复制的原理
    - 简述

      MySql主库在事务提交时会把数据变更作为事件记录在二进制日志Binlog中；
    
      主库推送二进制日志文件Binlog中的事件到从库的中继日志Relay Log中，之后从库根据中继日志做数据变更操作，通过逻辑复制来达到主库和从库的数据一致性；
    - 分述
    
      MySql通过三个线程来完成主从库间的数据复制，其中Binlog Dump线程跑在主库上，I/O线程和SQL线程跑着从库上。

      当在从库上启动复制时，首先创建I/O线程连接主库，主库随后创建Binlog Dump线程读取数据库事件并发送给I/O线程，I/O线程获取到事件数据后更新到从库的中继日志Relay Log中去，之后从库上的SQL线程读取中继日志Relay Log中更新的数据库事件并应用。

2. 事务

    - 操作

      ```sql
      SET AUTOCOMMIT=0;  //设置mysql的提交模式，默认是0，自动提交
      START TRANSACTION;
      SELECT @A:=SUM(salary) FROM table1 WHERE type=1;
      UPDATE table2 SET summary=@A WHERE type=1;
      COMMIT;
      ROLLBACK;
      ```

    - 事务的ACID特性 

      - 原子性：事务中的全部操作在数据库中是不可分割，要么全部执行，要么不执行。
      - 一致性：几个并行执行的事务，其执行结果必须和按照某一顺序串行执行的结果相一致。
      - 隔离型：事务的执行不受其他事务的干扰，当数据库被多个客户端并发访问时。隔离他们的操作，防止出现脏读、幻读、不可重复读。
      - 持久性：对于已经提交的事务，它对数据的变更是持久的，不会丢失的

    - 事务的隔离级别

      ```
      1.未提交读（read uncommitted）
      2.已提交读（read committed）
      3.可重复读（repeatable read）
      4.串行化（serializable）
      对于不同的事务，采用不同的隔离级别分别有不同的结果。不同的隔离级别有不同的现象。
      mysql 默认的事务隔离级别是 repeatable read
      SELECT @@GLOBAL.tx_isolation, @@tx_isolation;  //查看数据库的默认隔离级别
      SET TRANSACTION ISOLATION LEVEL
      ```

    - 使用时注意

      隔离性设置时要考虑多个线程操作同一资源造成的多线程并发安全问题

      加锁可以完美的保证隔离性，但是会造成数据库性能大大下降

      - 如果两个事务并发修改	必须隔离
      - 如果两个事务并发查询    完全不用隔离
      - 一个查询、一个修改    根据需求，看对不同现象的接受程度，使用不同的隔离级别

3. 一条sql的执行过程

    - Mysql 主要分为Server层和引擎层，Server层主要包括连接器、查询缓存、分析器、优化器、执行器，同时还有一个日志模块（binlog），这个日志模块所有执行引擎都可以共用。

    - 引擎层是插件式的，目前主要包括，MyISAM,InnoDB,Memory等。

    - 查询语句的执行流程如下：权限校验（如果命中缓存）---》查询缓存---》分析器---》优化器---》权限校验---》执行器---》引擎

    - 更新语句执行流程如下：分析器----》权限校验----》执行器---》引擎---redo log(prepare 状态---》binlog---》redo log(commit状态)

4. 如何做查询优化-定位和分析

   - 开启慢查询日志

   ```sql
   Show status like ‘slow_queries’;
   Show variables like ‘long_query_time’;
   //更改慢查询设置
   Set long_query_time=1;
   //到bin目录下 执行 
   \mysql.exe --safe -mode --slow-query-log
   注意：
       不要直接打开慢查询日志进行分析 使用pt-query-digest工具进行分析
   ```
   - show profile
   ```
   set profiling=1开启 会把所有的语句和对应的消耗时间存到临时表中
   show profiles
   show profile for query 临时表id
   ```
   - show processlist
   ```
   观察是否有连接（线程）处于不正常状态
   ```
   - explain查看执行计划
   ```
   explain + sql //检查sql查询索引的使用情况 
   
   EXPLAIN列的解释
   table 查询所涉及到的表
   type  使用何种方式连接表  查询类型  const  all  ref
   possible_keys  可能用到的索引
   key  实际用到的索引
   key_len 索引的长度
   extra  额外的信息
   rows   检查返回数据的记录数
   //type 查询类型（官方术语是 连接类型）
   参考: https://blog.csdn.net/dennis211/article/details/78170079
   ```

5. 如何做查询优化-优化

   - 更改数据表范式

   ```
   冗余数据，用空间来换取时间
   ```
   - 重写、拆分sql
   ```
   复杂的查询拆分成简单的查询，帮助优化器以更优的方式查询
   ```
   - 优化关联查询
   ```
   确定on或者using子句的列有索引
   确保group by 和order by 中只有一个表中的列，这样才能用到索引
   ```

6. 索引

   - 索引的基础知识

     Mysql的基本存储结构是**页**(记录都存在页里边)

     各个数据页可以组成一个双向链表，而每个数据页中的记录又可以组成一个单向链表

   - 索引是如何提高查询速度

     索引本质是将无序的数据变成有序的，对于innoDB和MyIsam存储引擎的表，它的底层结构是一个b+树，

     能够帮助我们快速的定位数据。

   - 为什么使用b+树而不使用二叉树、红黑树、avl树以及其他树

     对于普通的二叉树来说，在极端情况下会退化成链表，查询速度就会变慢

     对于AVL树来说，虽然不会退化成链表，一个节点只存储一个元素，随着数据量变大，树的高度会越来越大，

     造成磁盘的io的消耗就会变多，查询自然也会变慢

   - 为什么索引会降低增删改的速度

     因为DML操作时，如果有索引，会重新构建索引树，消耗一部分数据库性能

   - hash索引、以及hash索引的缺陷

     hash索引就是通过一定的哈希算法，把键值转化成哈希值，查找时不需要像B+树一样从根节点到叶子节点逐级索引，

     只需要一次哈希算法，就能快速定位数据

     缺点: 没有办法排序，不支持范围查找

     ​		  不支持最左匹配

     ​		  如果有大量重复的键值，效率是极低的，而且会有哈希碰撞的问题

   - mysql的索引原理和数据结构

     这种问题比较扯淡、很宽泛，遇到这种问题，就尽量的往宽的说，抓住以下几点

     ​	 数据结构	

     ​	 存储引擎	

     ​	 索引的分类	

     ​	 索引涉及到的常问的几个名词（索引覆盖，索引下推，最左匹配，回表）

      	执行计划 

     ​	 索引优化

     比如按照以下话术来说：

     索引的目的是将无序数据变得有序化，Mysql的底层数据结构是B+树或者hash表，不同的存储引擎，对应不同的数据结构，比如innoDB和MyIsam存储引擎是B+树，memery的存储引擎是哈希表

     存储引擎决定数据在磁盘上的不同组织形式，同时区分出不同功能，比如innoDB支持事务、支持数据行锁、支持外键；MyIasm支持表锁、支持全文索引。

     为什莫采用B+树，因为对于key-value结构的数据来说，检索查询，不管是采用二叉树、红黑树、AVL树来说，随着数据量的变多，都会导致树的高度不断变大，导致mysql在io操作（将索引分片的加载到内存进行寻址）的消耗变多。而B+树会在一个节点尽可能多的存储数据，只存储键值，这样就会使树的高度变低。

     同时Mysql里有主键索引，唯一索引，组合索引，全文索引，日常工作中用的比较多的就是主键和组合索引，而在使用这些索引的时候，会有索引覆盖、回表、索引下推、最左匹配等一系列的细节点，通过这些细节点，我们可以通过查看执行计划，对sql进行优化。

   - B树和B+树的区别

     - 磁盘预读

       分块的将数据读入内存，针对mysql来说每次从磁盘中读取的数据页是固定的，针对索引就是每次io只能读取一个节点数据。这样我们就必须降低树的高度。来达到减少io的目的。

     - B树

       b树是一个多叉树，一个节点可以存储多个列值，并且会存储整个记录，造成每个磁盘块存储的数据不多  三层的限制是4096条记录

     - B+树

       一个节点可以存储多个列值，并且不存储整条记录，可以存储更多的键值和指针数据，叶子结点存储的是整个记录 三层的限制是 4096000 千万级别的记录

       两种查找方式：可以从根节点依次查找、也可以在叶子节点查找

   - 对于主键索引要不要自增

     ​	这种情况建议自增，在插入新数据时，更新构建索引树时，如果不是自增的值，

     ​	会造成叶子节点页分裂，也就是重新组织数据块

   - 聚簇索引和非聚簇索引的区别

     以主键创建的索引就是聚簇索引， 数据跟索引绑定在一起的索引

     以非主键创建的索引就是非聚簇索引  叶子节点存储的是主键和索引列

     如果没有主键的情况

     ​	innoDB存储引擎，在插入数据的时候，必须将数据跟某一个索引列绑定到一起，索引列可以是主键，也可以是唯一键，也可以是6字节的rowid。

     innoDB 即存在聚簇索引又存在非聚簇索引

     myisam只存在非聚簇索引

   - 使用索引的原则

     - 回表

       从某一个索引的子节点中获取聚簇索引的id值，然后在根据id去聚簇索引中获取全量记录的过程

       比如id name age gender 四个列 id主键索引，name是普通索引     select * from table where name = 'zhangsan'

     - 索引覆盖

       从索引的叶子节点中能够获取全量查询的列的过程

       比如  select id,name table where name = 'zhangsan'

     - 最左匹配

       针对复合索引，如果是复合索引则会根据最左侧的列创建索引，最左侧的列是有序的，针对第一列，第二列也是有序的，节点中存储的键值也会是多个列的数据。可以和邮件的地址类比

       注意：

       ​	当表中的列都是索引的时候，无论怎么写查询条件都会用到索引

     - 索引下推

       对于符合索引来说，如果没有索引下推，则会先根据第一列难道全部数据，然后在server进行过滤，有索引下推之后就会在存储层直接查询出数据。

     综上所述：

     ​	使用索引时注意：

     ​	尽量避免回表，使用索引覆盖

     ​	遵循最左匹配原则

     ​	索引列尽量保持干净，不要使用函数过滤

     ​	避免连表查询

     ​	创建索引时注意：

     ​	where 子句的列适合创建索引

     ​	索引的基数越大，索引的效果越好

     ​	适当的创建索引，不要大量的创建索引

     ​	尽量的拓展索引，不新建索引

   - 参考

     [数据库中的两个神器](https://mp.weixin.qq.com/s/d7LIlMYAl5dZKv5o3btjxw)

     [索引的底层原理](https://zhuanlan.zhihu.com/p/113917726)

     [Mysql索引十连问](https://www.bilibili.com/video/BV1bv411x7cM?p=42&spm_id_from=pageDriver)

7. 锁

8. 自增的主键用完了会出现的问题

   写入时会报主键冲突