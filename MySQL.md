# MySQL

[TOC]

## 基础架构

![架构](https://static001.geekbang.org/resource/image/0d/d9/0d2070e8f84c4801adbfa03bda1f98d9.png)

- Server层：涵盖所有核心功能和内置函数
- 存储层：负责数据存储，其中存储引擎为插件模式，常用的有：InnoDB（默认）、MyISAM、Memory

### 实际语句执行过程分析

1. 连接器

   - 客户端与服务端连接

   - 用户权限获取

   - 维持和管理连接

     连接语句：```mysql -h$ip -P$port -u$user -p```（密码一般不写在命令中，防止泄露）

     完成TCP握手之后开始认证身份，同时获取用户权限（该权限的值会维持到连接结束）

     显示当前所有连接：```show processlist;```

     **由于建立连接过程较为复杂，因此尽量使用长连接；然而长连接带来的另一个问题就是内存占用过大，解决方案有两种**：

     **1.  定期断开长连接，或者执行完一个大查询之后断开连接**

     **2. MySQL5.7以上版本可以执行mysql_reset-connection来重新初始化连接资源，这个过程会释放内存且不需要重连**

2. 查询缓存

   执行查询语句的时候，MySQL会先去缓存中查询是否执行过相同的语句，如果有的话则直接返回结果；**但是最好不要使用这种缓存，因为缓存失效十分频繁，大部分缓存几乎没有作用，只有某些特殊的静态表才适合使用这种缓存；**MySQL也支持按需使用查询缓存，可以用关键字SQL_CACHE指定，如：```select SQL_CACHE * from table;```**另外，MySQL8.0以上将不再支持查询缓存**

3. 分析器

   执行词法分析和语法分析，根据结果执行对应查询逻辑

4. 优化器

   同一个语句有多种执行方法，优化器会对其效率进行优化

5. 执行器

   执行器负责具体的查询执行逻辑。执行操作前，先验证用户是否有操作权限；

   以基本的查询语句为例：

   ```select * from student where id=1;```

   如果id没有索引，执行器会调用存储引擎的获取每一行数据的接口，如果行满足查询条件那么将结果保存下来，直到取完整张表格，最后将结果返回客户端；

   如果id有索引，那么执行过程也类似，只不过执行器调用的是存储引擎另一个按索引取数据的接口；

   **在一些慢查询中，会有一个row_examined字段，这个值代表执行器调用了多少次存储引擎接口，然而不同存储引擎在一个接口中可能不止扫描一行数据，因此row_examined和世纪扫描行数不完全相同。**

# 日志系统

```create table T(ID int primary key, c int);```

```update T set c=c+1 where ID=2;```

这是一条普通的更新语句，执行流程和之前的查询语句基本类似：

1. 连接器连接数据库
2. 分析器分析为更新语句
3. 优化器优化
4. 执行器具体执行
5. 记录日志

与查询语句的不同在于更新语句需要记录日志，而两个日志模块最重要：redo log、binlog

## redo log

在MySQL中，为了数据持久化，所有数据都需要存入磁盘中，日志也一样，但如果每次记入磁盘IO压力过大，因此考虑使用WAL(Write Ahead Logging)技术进行优化。具体来说，以InnoDB为例，当有记录需要更新时，InnoDB会把记录写到redo log中，并更新内存；之后当InnoDB空闲时，将操作记录更新到磁盘。redo log的大小是固定的，可以配置为一组4个文件，每个文件1GB，当记录超过4GB，那么写入指针指向开通从头开始写，如下图所示：

![图1](https://static001.geekbang.org/resource/image/b0/9c/b075250cad8d9f6c791a52b6a600f69c.jpg)

write pos为写指针的位置，check point为擦除指针的位置，两个指针都向后移动，在check point移动时需要擦除数据同时把数据更新到数据文件。如果write pos和check point之间没有额外空间了，就需要停止更新，执行擦除。有了redo log，InnoDb就可以保证即使数据库异常重启，提交记录也不会丢失，称之为**crash-safe**。

## binlog

redo log是存储层InnoDB引擎的日志，而server层的日志就是binlog。两者有几点不同：

1. redo log是InnoDB引擎特有的，binlog是MySQL中server层实现的，所有引擎都可以使用。
2. redo log是物理日志，记录的是数据的具体修改；而binlog是逻辑日志，记录语句的原始逻辑；
3. redo log固定大小，空间写完要执行擦除；binlog写完一个文件会切到下一个文件

## 过程分析

以之前简单的Update语句为例进行分析：

![过程](https://static001.geekbang.org/resource/image/2e/be/2e5bff4910ec189fe1ee6e2ecc7b4bbe.png)

**浅色为执行器中执行，深色为存储引擎中执行**

1. 执行器首先找到ID=2这一行。
2. 执行器得到值后将值加1，再调用写入接口将结果写回数据库
3. 存储引擎将数据更新到内存
4. 存储引擎将更新操作记录到redo log里面，此时redo log处于prepare，告知执行器随时可以提交事务
5. 执行器生成这次操作的binlog，将binlog写入磁盘
6. 执行器调用引擎的提交事务接口，引擎将redo log改成commit状态

## 两阶段提交

可以看到，整个更新过程redo log日志是两阶段提交，首先置为prepare状态，最后在提交事务的时候变为commit状态。这是为了保证两份日志之间的逻辑一致，而如果不这么做会产生一些问题：

1. 先写redo log再写binlog。假设redo log先写完，而binlog还没有写完，MySQL进程异常重启。重启之后，redo log能把数据恢复，此时数据库不会有问题。但是如果当你需要通过binlog来恢复数据库的时候，由于binlog没有记录到这次操作，那么恢复出来的数据就是有问题的。
2. 先写binlog再写redo log。假设binlog写完，而redo log还没写，MySQL进程异常重启。重启之后，崩溃前的事务执行无效，因此update操作没有成功，但是binlog已经记录下了这次没成功的操作，之后恢复出来的数据同样有问题。

## Tips

**innodb_flush_log_at_trx_commit**这个参数设为1的时候代表每次redo log都直接持久化到硬盘，建议设置，保证每次异常重启后数据不丢失；

**sync_binlog**这个参数设为1的时候代表每次binlog都持久化到硬盘，建议设置，保证异常重启后binlog不丢失