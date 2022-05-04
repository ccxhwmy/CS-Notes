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

