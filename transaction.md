## 事务介绍

### 1.概念与特性
定义：事务是作为一个逻辑单元执行的一系列操作。
事务特性：
原子性: 要么完全回滚，全部不保留。 
一致性: 任何时候都应该处于一致的状态。
隔离性: 多个事务同时进行，它们之间应该互不干扰。
持久性: 事务提交以后，所做的工作就被永久的保存下来

### 2.Oracle事务的特性
原子性
存储过程: Oracle将存储过程视为单一的语句，并在存储过程的执行前后都建立了SAVEPOINT。
事务级: 事务启动会自动建立SAVEPOINT 
DDL：执行前自动提交已存在的事务
持久性
一般而言，COMMIT是同步调用，此过程中数据库会将redo日志写入磁盘。
 COMMIT WRITE NOWAIT，此命令将采取异步写redo日志的方式，但是极端情况下持久性会受影响。
 
事务的使用不再迁就于软硬件的实现，不必为了提高并发而人为缩短事务边界；
事务的首要目标是确保数据一致性和完整性，事务边界应该根据业务需要而设定。
错误的观点-为了提高效率和速度而频繁地提交小事务

事务隔离级别标准

基于是否允许下列三种现象发生， SQL标准中定义了四类事务隔离级别：
脏读：允许读取未提交数据，即脏数据。
不可重复读：This simply means that if you read a row at time T1 and attempt to reread that row at time T2, the row may have changed, disappeared, updated, and so on.
幻读：execute a query at time T1 and re-execute it at time T2, additional rows may have been added to the database, which will affect your results - 已读的数据没有变化，但是结果集返回更多的数据

Oracle 支持下列三种事务隔离级别
READ COMMITTED 
Oralce的默认事务隔离级别。没有脏读（dirty read）, 可重复读, 未提交的修改不会block其他事务的查询。
SERIALIZABLE
事务串行执行（逻辑上），不存在并行的事务同时更新数据。 
Oracle利用回滚段重建查询数据，从逻辑上看，所有查询数据在事务开始时就已经确定，而不是statement执行时。
READ ONLY
事务隔离级别和SERIALIZABLE 相似，但是不允许inserts, updates, and deletes等操作

Oracle 事务锁

事务锁的生成与销毁：acquired when a transaction initiates its first change, and it is held until the transaction performs a COMMIT or ROLLBACK.

如何锁定行：the locked row points to a copy of the transaction ID that is stored with the data  block, and when the lock is released that transaction ID is left behind.

事务ID：is unique to your transaction and represents the undo segment number, slot, and sequence number.

Autonomous Transactions（自治事务）
简单来说，自治事务就是事务中嵌套事务，两个事务相互独立运行，互不干扰。
由于处于不同的事务上下文，查询数据会有不一致

Deadlock（死锁）
死锁形成的原因是资源的无序竞争，并最终形成环路。
Deadlocks occur when you have two sessions, each of which is holding a resource that the other wants.
如何避免死锁
按照既定顺序获取资源
尽量缩短事务时间
一次性获取资源
设置相应的超时机制
The number one cause of deadlocks in the Oracle database is unindexed foreign keys.

事务管理方式
本地事务
也称为JDBC事务或数据库事务，是指将事务的处理交由数据库来管理，适用于单一的增删改查操作。

Spring事务管理
基于AOP技术实现的声明式事务管理，即在方法执行前后进行拦截，然后在目标方法开始之前创建并加入事务，执行完目标方法后根据执行情况提交或回滚事务






