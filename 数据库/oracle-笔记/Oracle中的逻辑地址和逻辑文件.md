# Oracle中的逻辑结构

## `Tablespace` `Segment` `Extent` `Block`

一个数据库是由多个表空间`tablespace`组成，一个表空间又由多个段`segment`组成，一个段又由多个区`extent`组成，一个区则由多个块`block`组成。
一个数据库中，`UNDO`和`SYSTEM`表空间是必须存在的。

#### 简述
对于我们在数据库里新建数据库`database`，在数据库中建立多个表空间`tablespace`，在每个表空间内建表。例如我们可以分配多个用户，在`user1`用户下建立`table1`和`table2`,`user2`下建立`table3`和`table4`，`user1`、`user2`就等于不同的表空间，`table1`、`table2`是建立在表空间下的不同段`segment`，而每张表的每个数据就是块`block`，一列数据可看做一个区`extent`，区满了之后不断扩展就组成了表。

![逻辑体系结构][1]

### `Segment`
如果一个表不进行分区，那么一个表就是在一个`Segment`中。如果一个表进行多个分区，那么每一个分区就在一个`Segment`。
每一张表(未分区)都会有一个逻辑单元`Segment`，他代表了一张未分区表所存储的所有的数据。一个`Segment`可能存在于多个数据文件中，因为物理的数据文件是组成逻辑表空间的基本物理存储单位。
`Segment`由一组`extent`构成，其中存储了表空间`tablespace`内各种逻辑存储结构的数据。例如，`Oracle`能为每个表的数据段`data segment`分配`extent`，还能为每个索引的索引段`index segment`分配`extent`。

### `Extent`
`extent segment`由一个或多个`extent`组成，`extent`是文件中一个逻辑上连续分配的空间（一般来讲，文件本身在磁盘上并不是连续的）。每个`segment`都至少有一个`extent`，有些对象可能还需要至少两个`extent`（回滚段就至少需要两个`extent`）。

如果一个对象超出了其初始`extent`，就会请求再为它分配另一个`extent`。第二个`extent`不一定就在磁盘上第一个`extent`旁边，但`extent`内的空间总是文件中的一个逻辑连续空间。

每个段`segment`的定义中都包含了`extent`的存储参数`storage parameter`。存储参数适用于各种类型的段。这个参数控制着`Oracle`如何为段分配可用空间。例如，用户可以在`CREATE TABLE `语句中使用`STORAGE`子句设定存储参数，决定创建表时为其数据段`data segment`分配多少初始空间，或限定一个表最多可以包含多少`extent`。如果用户没有为表设定存储参数，那么表在创建时使用所在表空间`tablespace`的默认存储参数。

用户既可以使用数据字典管理的表空间`dictionary managed tablespace`，也可以使用本地管理的表空间`locally managed tablespace`。除了`SYSTEM`之外的所有永久表空间`permanent tablespace`默认使用本地管理方式。

在一个本地管理的表空间中，其中所分配的`extent`的容量既可以是用户设定的固定值，也可以是由系统自动决定的可变值。当用户创建表空间`tablespace`时可以使用`UNIFORM`用户指定或`AUTOALLOCATE`由系统管理子句设定`extent`的分配方式。s

- 对于固定容量`UNIFORM`的`extent`，用户可以为`extent`设定容量或使用默认大小`1MB`。用户须确保每个`extent`的容量至少能包含5个数据块`database block`。本地管理`locally managed`的临时表空间`temporary tablespace`在分配`extent`时只能使用此种方式。
- 对于由系统管理`AUTOALLOCATE`的`extent`，由`Oracle`决定新增`extent`的最佳容量，其最小容量为`64KB`。如果创建表空间时使用了`“segment space management auto”`子句，且数据块容量大于等于`16 KB`，`Oracle`扩展一个段时`segment`所创建的`extent`的最小容量为`1MB`。对于永久表空间`permanent tablespace`上述参数均为默认值。

在本地管理的表空间`locally managed tablespace`中，`INITIAL`，`NEXT`，`PCTINCREASE`，和` MINEXTENTS` 这四个存储参数可以作用于段`segment`，但不能作用于表空间。`INITIAL`，`NEXT`，`PCTINCREASE`，和 `MINEXTENTS` 相结合可以用于计算段的初始容量。

一般来说，在用户将一个段`segment`对应的方案对象`schema object`移除使用`DROP TABLE`或`DROP CLUSTER`语句之前，此段的`extent`不会被回收到表空间`tablespace`中，但是以下情况例外：

* 表，簇表的所有者`owner`或拥有`DELETE ANY`权限的用户， 可以使用`TRUNCATE...DROP STORAGE`语句将表，簇表的数据清除
* `DBA`可以使用以下语法收回一个段中未使用的`extent`：
  ```
  ALTER TABLE table_name DEALLOCATE UNUSED;
  ```
* 如果用户为回滚段`rollback segment`设定了`OPTIMAL`参数，`Oracle`将周期性地从其中回收`extent`。


### `Data Block`
`block`是`oracle`中最小的空间分配单位。数据行、索引或者临时排序结果就存储在块中。通常`oracle`从磁盘读写的就是块。常见大小有4种：`2kb`、`4kb`、`8kb`或者`16kb`，但这并不意味着块的大小是2的幂，而是可以任意2~32之间的数值。

数据库中是允许有多种块大小，目的是为了可以在更多的情况下使用可传输的表空间。而有多种块大小的表空间主要用于传输表空间，一般没有其他用途。

数据库还有一个默认的块大小。`system`表空间总是使用这个默认块大小。

在所给定的表空间内部，块大小都是一致的。对于一个多段对象，如一个包含`LOB`列的表，可能每个段在不同的表空间中，而这些表空间分别有不同的大小。可无论大小如何，每个块的格式是一致的。

- 块首部`block header`包含块类型的有关信息（表块、索引块）、块上发生的活动事务和过去事务的相关信息，以及块在磁盘上的地址。
- 表目录`table directory`，包含把行存储在这个块上的表的有关信息。
- 行目录`row directory`包含块中行的描述信息，这是一个指针数组，指向块数据部分中的行。

以上3部分称为块开销`block overhead`，这部分空间并不用于存放数据，而是由`oracle`用来管理块本身。接下来的两部分就是存储数据了：块上可能有一个空闲空间`free space`，通常还会有一个目前已经存放数据的已用空间`used space`。


对于块空间的管理，主要有两个参数，`pctfree`和`pctused`。

`pctfree`值是指达到该值后，数据将无法继续插入数据，同时将该块从`free list`上移走；
`pctused`值是指如果因删除更新等操作使得实际值低于`pctused`值，则表示数据可以继续插入，同时将该块移入`free list`，表示该块是空闲的。

系统默认值是`pctfree =10` ,`pctused=40`


---
 参考文章:
- [Block，extent,segment介绍][2]
- [Oracle逻辑结构 TableSpace→Segment→Extent→Block][3]


[1]:https://images0.cnblogs.com/i/368951/201405/022207599866905.png
[2]:https://blog.csdn.net/luoyuchuan/article/details/7208426
[3]:https://www.cnblogs.com/yongjian/p/3704592.html
