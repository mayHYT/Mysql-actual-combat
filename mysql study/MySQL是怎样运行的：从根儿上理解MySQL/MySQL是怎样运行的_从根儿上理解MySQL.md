

## InnoDB记录结构

MySQL 服务器上负责对表中数据的读取和写入工作的部分是 存储引擎 ，而服务器又支持不同类型的存储引擎，真实数据在不同存储引擎中存放的格式一般是不同的。

InnoDB存储引擎最终会把数据存储在磁盘中，数据处理的过程是将数据从磁盘加载到内存，在内存中对数据进行操作后，再将数据写入磁盘。InnDB采用分页的形式对数据进行读写，将数据划分为若干个页，以页作为磁盘和内存之间交互的基本单位，InnoDB中页的大小一般为 16 KB。

### InnoDB的行格式

我们平时是以记录为单位来向表中插入数据的，这些记录在磁盘上的存放方式也被称为 行格式 或者 记录格式 。主要有四种行格式：

- Compact
- Redundant
- Dynamic
- Compressed

创建表时指定行格式：

CREATE TABLE 表名 (列的信息) ROW_FORMAT=行格式名称 

ALTER TABLE 表名 ROW_FORMAT=行格式名称

示例：

mysql> CREATE TABLE record_format_demo ( 
 -> c1 VARCHAR(10), 
 -> c2 VARCHAR(10) NOT NULL, 
 -> c3 CHAR(10), 
 -> c4 VARCHAR(10) 
 -> ) CHARSET=ascii ROW_FORMAT=COMPACT; 


#### Compact格式


![alt text](compact.png)


##### 记录的额外信息

- 变长字段长度列表

在 Compact 行格式中，把所有变长字段的真实数据占用的字节长度都存放在记录的开头部位，从而形成一个变长字段长度列表，各变长字段数据占用的字节数按照列的顺序逆序存放，我们再次强调一遍，是逆序存放！

变长字段长度列表中只存储值为 非NULL 的列内容占用的长度，值为 NULL 的列的长度是不储存的 。

- NULL值列表

1. 首先统计表中允许存储 NULL 的列有哪些。
2. 如果表中没有允许存储 NULL 的列，则 NULL值列表 也不存在了，否则将每个允许存储 NULL 的列对应一个

    二进制位，二进制位按照列的顺序逆序排列，二进制位表示的意义如下：

- 二进制位的值为 1 时，代表该列的值为 NULL 。
- 二进制位的值为 0 时，代表该列的值不为 NULL 。


- 记录头信息

它是由固定的 5 个字节组成。 5 个字节也就是 40 个二进制位，不同的位代表不同的意思，如图：

![alt text](header.png)



######  记录的真实数据

MySQL 会为每个记录默认的添加一些列（也称为 隐藏列 ），具体的列如下：


#### Redundant行格式

![alt text](redundant.png)


#### Dynamic和Compressed行格式

这两种行格式类似于 COMPACT行格式 ，只不过在处理行溢出数据时有点儿分歧，它们不会在记录的真实数据处存储字符串的前768个字节，而是把所有的字节都存储到其他页面中，只在记录的真实数据处存
储其他页面的地址。

另外， Compressed 行格式会采用压缩算法对页面进行压缩。


## InnoDB数据页结构


### 数据也结构

![alt text](page.png)


![alt text](page_info.png)

我们自己存储的记录会按照我们指定的 行格式 存储到 User Records 部分。但是在一开始生成页的时候，其实并没有 User Records 这个部分，每当我们插入一条记录，都会从 Free Space 部分，也就
是尚未使用的存储空间中申请一个记录大小的空间划分到 User Records 部分，当 Free Space 部分的空间全部被 User Records 部分替代掉之后，也就意味着这个页使用完了，如果还有新的记录插入的话，就需要去申请新的页了，这个过程的图示如下：

![alt text](insert.png)

用户记录头信息

![alt text](user_head.png)

![alt text](head_info.png)

- record_type

这个属性表示当前记录的类型，一共有4种类型的记录， 0 表示普通记录， 1 表示B+树非叶节点记录， 2 表
示最小记录， 3 表示最大记录。

- next_record

它表示从当前记录的真实数据到下一条记录的真实数据的地址偏移量。


## 事务基础语法

### 开启事务

- BEGIN [WORK]

- START TRANSACTION

可以在 START TRANSACTION 语句后边跟随几个 修饰符：

1. READ ONLY ：标识当前事务是一个只读事务，也就是属于该事务的数据库操作只能读取数据，而不能修
改数据。

2. READ WRITE ：标识当前事务是一个读写事务，也就是属于该事务的数据库操作既可以读取数据，也可
以修改数据。
3. WITH CONSISTENT SNAPSHOT ：启动一致性读

如果我们想在 START TRANSACTION 后边跟随多个 修饰符 的话，可以使用逗号将 修饰符 分开，比如开启一个只读事务和一致性读，就可以这样写：
START TRANSACTION READ ONLY, WITH CONSISTENT SNAPSHOT;

**如果我们不显式指定事务的访问模式，那么该事务的访问模式就是 读写 模式**

### 提交事务

COMMIT [WORK]


### 手动终止事务

如果我们写了几条语句之后发现上边的某条语句写错了，我们可以手动的使用下边这个语句来将数据库恢复到事务执行之前的样子：

ROLLBACK [WORK]

这里需要强调一下， ROLLBACK 语句是我们程序员手动的去回滚事务时才去使用的，如果事务在执行过程中遇到了某些错误而无法继续执行的话，事务自身会自动的回滚。


MySQL 中并不是所有存储引擎都支持事务的功能，目前只有 InnoDB 和 NDB 存储引擎支持（NDB存储引擎不是我们的重点），如果某个事务中包含了修改使用不支持事务的存储引擎的表，那么对该使用不支持事务的存储引擎的表所做的修改将无法进行回滚。

### 自动提交
mysql默认是自动提交模式，如果我们不显式的使用 START TRANSACTION 或者 BEGIN 语句开启一个事务，那么每一条语句都算是一个独立的事务，这种特性称之为事务的 自动提交 。

mysql> SHOW VARIABLES LIKE 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+
1 row in set (0.01 sec)

关闭自动提交：
1. 显式的的使用 START TRANSACTION 或者 BEGIN 语句开启一个事务。这样在本次事务提交或者回滚前会暂时关闭掉自动提交的功能。
2. 把系统变量 autocommit 的值设置为 OFF ，就像这样：SET autocommit = OFF;

### 隐式提交

1. 定义或修改数据库对象的数据定义语言（Data definition language，缩写为： DDL ）。所谓的数据库对象，指的就是 数据库 、 表 、 视图 、 存储过程 等等这些东西。当我们使用 CREATE 、
ALTER 、 DROP 等语句去修改这些所谓的数据库对象时，就会隐式的提交前边语句所属于的事务，就像这
样：

 BEGIN; 
 SELECT ... # 事务中的一条语句 
 UPDATE ... # 事务中的一条语句 
 ... # 事务中的其它语句 
 CREATE TABLE ... # 此语句会隐式的提交前边语句所属于的事务

2. 隐式使用或修改 mysql 数据库中的表。当我们使用 ALTER USER 、 CREATE USER 、 DROP USER 、 GRANT 、 RENAME USER 、 REVOKE 、 SET PASSWORD 等语句时也会隐式的提交前边语句所的事务。

3. 事务控制或关于锁定的语句
当我们在一个事务还没提交或者回滚时就又使用 START TRANSACTION 或者 BEGIN 语句开启了另一个事务时，会隐式的提交上一个事务，比如这样：

BEGIN; 
 SELECT ... # 事务中的一条语句 
 UPDATE ... # 事务中的一条语句 
 ... # 事务中的其它语句 
BEGIN; # 此语句会隐式的提交前边语句所属于的事务


或者当前的 autocommit 系统变量的值为 OFF ，我们手动把它调为 ON 时，也会隐式的提交前边语句所属的事务。

或者使用 LOCK TABLES 、 UNLOCK TABLES 等关于锁定的语句也会隐式的提交前边语句所属的事务。

4. 加载数据的语句
比如我们使用 LOAD DATA 语句来批量往数据库中导入数据时，也会隐式的提交前边语句所属的事务。

5. 关于mysql复制的一些语句

使用 START SLAVE 、 STOP SLAVE 、 RESET SLAVE 、 CHANGE MASTER TO 等语句时也会隐式的提交前边语句所属的事务。

6. 其他一些语句

使用 ANALYZE TABLE 、 CACHE INDEX 、 CHECK TABLE 、 FLUSH 、 LOAD INDEX INTO CACHE 、 OPTIMIZE TABLE 、 REPAIR TABLE 、 RESET 等语句也会隐式的提交前边语句所属的事务。

### 保存点

在事务对应的数据库语句中打几个点，我们在调用 ROLLBACK 语句时可以指定会滚到哪个点，而不是回到最初的原点。定义保存点的语法如下：

SAVEPOINT 保存点名称;

当我们想回滚到某个保存点时，可以使用下边这个语句（下边语句中的单词 WORK 和 SAVEPOINT 是可有可无的）：

ROLLBACK [WORK] TO [SAVEPOINT] 保存点名称;

不过如果 ROLLBACK 语句后边不跟随保存点名称的话，会直接回滚到事务执行之前的状态。

如果我们想删除某个保存点，可以使用这个语句：

RELEASE SAVEPOINT 保存点名称;

示例：

mysql> SELECT * FROM account; 
+----+--------+---------+ 
| id | name | balance | 
+----+--------+---------+ 
| 1 | 狗哥 | 11 | 
| 2 | 猫爷 | 2 | 
+----+--------+---------+ 
2 rows in set (0.00 sec) 

mysql> BEGIN; 
Query OK, 0 rows affected (0.00 sec) 

mysql> UPDATE account SET balance = balance - 10 WHERE id = 1; 
Query OK, 1 row affected (0.01 sec) 
Rows matched: 1 Changed: 1 Warnings: 0 

mysql> SAVEPOINT s1; # 一个保存点 
Query OK, 0 rows affected (0.00 sec) 

mysql> SELECT * FROM account; 
+----+--------+---------+ 
| id | name | balance | 
+----+--------+---------+ 
| 1 | 狗哥 | 1 | 
| 2 | 猫爷 | 2 | 
+----+--------+---------+ 
2 rows in set (0.00 sec) 

mysql> UPDATE account SET balance = balance + 1 WHERE id = 2; # 更新错了 
Query OK, 1 row affected (0.00 sec) 
Rows matched: 1 Changed: 1 Warnings: 0 

mysql> ROLLBACK TO s1; # 回滚到保存点s1处 
Query OK, 0 rows affected (0.00 sec) 
mysql> SELECT * FROM account; 
+----+--------+---------+ 
| id | name | balance | 
+----+--------+---------+ 
| 1 | 狗哥 | 1 | 
| 2 | 猫爷 | 2 | 
+----+--------+---------+ 
2 rows in set (0.00 sec)


