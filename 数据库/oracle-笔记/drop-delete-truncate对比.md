## `drop-delete-truncate`的对比

### 背景知识
[`Tablespace` `Segment` `Extent` `Block`](Oracle中的逻辑地址和逻辑文件.md)

## `drop delete truncate `对比
**他们都是为了删除数据**
### `Delete`
`delete`是`DML`，执行`delete`操作时，`DELETE`语句执行删除的过程是一行一行的删除表中的数据，并且同时将该行的的删除操作记录在`redo`和`undo`表空间中以便进行回滚`rollback`和重做操作，但要注意表空间要足够大，需要手动提交`commit`操作才能生效，可以通过`rollback`撤消操作。

`delete`语句不影响表所占用的`extent`，高水线`high watermark`保持原位置不变。

### `Truncate`
- `TRUNCATE TABLE`则一次性地从表中删除所有的数据并不把单独的删除操作记录记入日志保存，删除行是不能恢复的。
  >并且在删除的过程中不会激活与表有关的删除触发器。执行速度快。
  >但是有一些关于数据库的触发器会被触发

`truncate`会删除表中所有记录，并且将重新设置高水线和所有的索引，缺省情况下将空间释放到`minextents`个`extent`，除非使用`reuse storage`。不会记录日志，所以执行速度很快，但不能通过`rollback`撤消操作（如果一不小心把一个表`truncate`掉，也是可以恢复的，只是不能通过`rollback`来恢复）。

`TRUNCATE TABLE`删除表中的所有行，但表结构及其列、约束、索引等保持不变。

对于外键`foreign key`约束引用的表，不能使用`truncate table`，而应使用不带`where`子句的`delete`语句。

`truncate table`不能用于参与了索引视图的表。

>`Oracle`和`Mysql`中, `Truncate Table`在是否重置`Auto Increment`上存在不同：
主要原因是`Oracle`中使用的是序列`Sequence`来维持自增长，序列本身不依赖于表，所以并不会重置。


**`TTRUNCATE TABLE`表名 速度快,而且效率高,因为:**

虽然`TRUNCATE TABLE`在功能上与不带`WHERE`子句的`DELETE`语句相同：二者均删除表中的全部行。但`TRUNCATE TABLE`比`DELETE`速度快，且使用的系统和事务日志资源少。
- `DELETE`语句**每次删除一行，并在事务日志中为所删除的每行记录一项**。
- `TRUNCATE TABLE`通过释放存储表数据所用的**数据页**来删除数据，并且**只在事务日志中记录页的释放**。

当使用行锁执行`DELETE`语句时，将锁定表中各行以便删除。`TRUNCATE TABLE`始终锁定表和页，而不是锁定各行。如无例外，在表中不会留有任何页。

### `Drop`
`Drop`是`DDL`，会隐式提交，所以，不能回滚，不会触发触发器。

`Drop`语句将表所占用的空间全释放掉。

`Drop`语句将删除表的结构所依赖的约束，触发器，索引，依赖于该表的存储过程/函数将保留,但是变为`invalid`状态。

`truncate`、`drop`是`DLL`--`data define language`, 操作立即生效，原数据不放到`rollback segment`中，不能回滚。

## 总结
- 在速度上，一般来说，`drop` > `truncate` > `delete`。
- 在使用`drop`和`truncate`时一定要注意，虽然可以恢复，但为了减少麻烦，还是要慎重。
- 在没有备份情况下，谨慎使用`drop`与`truncate`。要删除部分数据行采用`delete`且注意结合`where`来约束影响范围。回滚段要足够大。

### 表和索引所占空间

- `Delete`:操作不会减少表或索引所占用的空间。
- `Truncate`:这个表和索引所占用的空间会恢复到初始大小，
- `Drop`:语句将表所占用的空间全释放掉。

### 如何合理使用这些关键字
> - 如果想删除表，用`drop`；
> - 如果想保留表而将所有数据删除，如果和事务无关，用`truncate`即可；
> - 如果和事务有关，或者想触发`trigger`，还是用`delete`；
> - 如果是整理表内部的碎片，可以用`truncate`跟上`reuse stroage`，再重新导入/插入数据。


---
 参考文章:
- [删除表数据drop、truncate和delete的用法][2]
- [drop、truncate和delete的区别][3]


[2]:https://www.cnblogs.com/fjl0418/p/7929420.html
[3]:https://www.cnblogs.com/zhizhao/p/7825469.html
