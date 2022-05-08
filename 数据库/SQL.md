## SQL

### 存储过程

​	存储过程可以看成是对一些列 SQL 操作的批处理。

​	使用存储过程的好处：

- 代码封装，保证了一定的安全性；
- 代码复用；
- 由于是预先编译，因此具有很高的性能。

​    命令行中创建存储过程需要自定义分隔符，因为命令行是以 ; 为结束符，而存储过程中也包含了分号，因此会错误把这部分分号当成是结束符，造成语法错误。

​	包含 in、out 和 inout 三种参数。

​	给变量复制都需要用 select into 语句。

​	每次只能给一个变量复制，不支持集合操作。

```sql
delimiter //

create procedure myprocedure( out ret int )
	begin
		declare y int;
		select sum(col1)
		from mytable
		into y;
		select y*y into ret;
		
delimiter ;
```

```sql
call myprocedure(@ret);
select @ret;
```

### 四种语言

**DDL，DML，DCL，TCL**

#### 1.DDL

​	DDL（Data Definition Language）数据库定义语言statements are used to define the database struccture or schema.

​	DDL 是 SQL 语言的四大功能之一。

​	用于定义数据库的三级结构，包括外模式、概念模式、内模式及其相互之间的映像，定义数据的完整性、安全控制等约束

​	DDL 不需要 commit。

​	CREATE、ALTER、DROP、TRUNCATE、COMMENT、RENAME

#### 2.DML

​	DML（Data Manipulation Language）数据操纵语言statements are used for managing data within schema objects.

​	由 DBMS 提供，用于让用户或程序员使用，实现对数据库中数据的操作。

​	DML 分成交互型 DML 和嵌入型 DML 两类。

​	依据语言的级别，DML 又可分成过程性 DML 和非过程性 DML 两种。

​	需要 commit。

​	SELECT、INSERT、UPDATE、DELETE、MERGE、CALL、EXPLAIN PLAN、LOCK TABLE

#### 3.DCL

​	DCL（Data control Language）数据库控制语言 授权，角色控制等

​	GRANT 授权

​	REVOKE 取消授权

#### 4.TCL

​	TCL（Transaction Control Language）事务控制语言

​	SAVEPOINT 设置保存点

​	ROLLBACK 回滚

​	SET TRANSACTION

### 四部分

（1）数据定义。（SQL DDL）用于定义 SQL 模式、基本表、视图和索引的创建和撤销操作。

（2）数据操纵。（SQL DML）数据操纵分成数据查询和数据更新两类。数据更新又分成插入、删除和修改三种操作。

（3）数据控制。包括对基本表和视图的授权，完整性规则的描述，事务控制等内容。

（4）嵌入式 SQL 的使用规定。涉及到 SQL 语句嵌入在宿主语言程序中使用的规则。



















































