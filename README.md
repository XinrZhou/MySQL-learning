

# MySQL学习笔记

### 索引

存储引擎用来快速查找记录的一种数据结构，不使用索引必须从第一条记录开始读完整个表。默认为树型索引

#### 分类

- 按实现方式：
  - Hash索引
  - B+Tree索引
- 按功能：
  - 单列索引
    - 普通索引：关键字index
    - 唯一索引：列的值必须唯一但允许为空，关键字unique
    - 主键索引：MySQL自动在主键列上建一个索引
  - 组合索引：复合最左原则，条件中必须包含索引前面的字段才能进行匹配
  - 全文索引：主要用来查找文本中的关键字，而不直接与索引中的值比较，关键字fulltext
  - 空间索引

#### 索引的原理

- 索引以索引文件的形式存储在磁盘上，索引的结构组织要尽量减少查找过程中磁盘I/O的存取次数
- HASH算法
  - 优点：通过字段值计算hash值，定位数据快
  - 缺点：不能进行范围查找，散列表中的值无序
- BTREE树：目前大部分DBS及FS都采用B-TREE或B+TREE作为索引结构
  - MylSAM引擎：叶子节点data域存放数据记录的地址
  - InnoDB：data域存放数据，效率比MylSAM引擎高，但占硬盘内存大

#### 创建索引的原则

- 更新频繁的列不设置索引
- 数据量小的表不设置索引
- 重复数据多的字段不应设置为索引
- 首先考虑对where和order by涉及的列上建索引

### MySQL的优化

#### 查看SQL的执行频率

查看当前会话SQL执行类型的统计信息：```show session status like 'Com_______';  ```

查看全局（从上次MySQL服务器启动至今）执行类型的统计信息：```show global status like 'Com_______';  ```

#### 定位低效率执行SQL

慢日志查询

- 查看慢日志配置信息：```show variables like '%slow_query_log%';```
- 开启慢日志查询：```set global slow_query_log = 1;```
- 查看慢日志记录SQL的最低阈值时间，默认为10s：```show variables like '%long_query_time%';```
- 修改最低阈值时间：```set global long_query_time = 5;```

show processlist：查看当前客户端连接服务器的线程执行信息，包括线程的状态，是否锁表等

#### explain分析执行计划

通过explain命令获取MySQL执行select语句的信息，包括select语句执行过程中表如何连接以及连接的顺序  

explain + select语句，如```explain select * from user where uid = 1;  ```

explain返回结果字段详解：

- id：select 查询序列号。id相同，优先级相同，执行顺序由上至下；id不同，id值越大优先级越高，越先被执行

- select_type：select的类型，常见取值

  - simple：简单查询，不包含子查询或 union
  - primary：包含复杂的子查询，最外层查询标记为该值（主查询）
  - subquery：在 select 或 where 包含子查询，被标记为该值
  - derived：在 from 列表中包含的子查询被标记为derived（衍生），MySQL 会递归执行这些子查询，把结果放在临时表中
  - union：若第二个 select 出现在 union 之后，则被标记为该值。若 union 包含在 from 的子查询中，外层 select 被标记为 derived
  - union result：从 union 表获取结果的 select

- table：输出结果集的表

- partitions：匹配的分区

- type：表的连接类型，性能由高到低，**通常优化至少到range级别，最好能优化到 ref**

  - null：不访问任何表，索引，直接返回结果

  - system：系统表，少量数据，不需要进行磁盘IO，5.7及以上版本不再显示system，而是直接现实all
  - const：通过索引一次就找到，只匹配一行数据
  - eq_ref：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常用于主键或唯一索引扫描
  - ref：非唯一性索引扫描，返回匹配某个单独值的所有行（左表有普通索引，和右表匹配时可能会匹配多行）
  - range：只检索给定范围的行，使用一个索引来选择行。一般适用between、>、<情况
  - index：需要扫描索引上的全部数据
  - ALL：全表扫描，性能最差

- possible_keys：查询时可能使用的索引

- key：实际使用的索引

- key_len：索引字段最大可能长度，并非实际使用长度

- ref：该表的索引字段关联了哪张表的哪个字段

- rows：扫描行的数量，值越小越好

- filtered：返回结果的行数占读取行数的百分比，值越大越好

- extra：执行情况的说明和描述

  - using filesort：对数据使用外部的索引排序，称为”文件排序“，效率低
  - using temporary：建立临时表来暂存中间结果，常见于order by和group by，效率低
  - using index：SQL需要返回的列所有数据均在一棵索引树上，避免访问表的数据行，效率不错

  ```mysql
  explain select uid,count(*) from user group by uid;
  ```

#### show profile 分析SQL

show profiles做SQL优化时能了解时间消耗  

是否支持profile：``` select @@hanve_profiling;```  

开启profiing：```set profiling=1;```  

执行完SQL命令之后，再执行```show profiles```指令，查看SQL语句执行的耗时  

查看SQL执行过程中每个线程的状态和消耗时间``` show profile for query query_id;```

#### 索引优化

通过索引可以帮助用户解决大多数的MySQL性能优化问题  

全值匹配：和字段匹配成功即可，该情况下索引生效，执行效率高

```mysql
create index idx_name_addr on user(name,address);
explain select * from user where name='zhangsan' and address='杭州';
```

如果索引为多列，要遵守**最左前缀法则**（查询从索引的最左列开始，并且不跳过索引中的列）

- ``` explain select * from user where name='zhangsan'; ``` 
- 违反最左前缀法则，索引失效：```explain select * from user where address='杭州';```
- 符合最左前缀法则但出现跳跃某一列，只有最左列索引生效

其他匹配原则：

- 范围查询右边的列，不使用索引
- 在索引列上进行运算操作，索引失效
- 字符串不加单引号，索引失效
- 尽量使用覆盖索引，避免select *
  - select * 需要从原表及磁盘上读取数据
  - 而覆盖索引从索引树中就可以查询到所有数据，效率高
- 用or分隔的条件，索引失效
- 以%开头的like模糊查询，索引失效
  - ```explain select * from user where name like '%赵';```失效
  - ```explain select * from user where name like '赵%';```使用索引
- 不用*，使用索引列```explain select name from user where name like '%赵%'```
- 如果MySQL评估使用索引比全表更慢，则不使用索引
- 尽量使用复合索引，如果一个表有多个单列索引，及时使用了多个索引列，也只有一个索引列生效（由MySQL优化器决定）

#### SQL优化

大批量插入数据

- 当通过load向表加载数据时，尽量保证文件中的主键有序，提高执行效率
- 关闭唯一性校验：导入数据前执行```set unique_checks=0```，关闭唯一性校验，结束后执行```set unique_checks=1```，恢复校验

insert优化

- 如果需要同时对一张表插入多行数据，应尽量使用多个值表的insert语句，这种方式将大大**降低客户端与数据库之间的连接、关闭等消耗**，效率比执行单个insert语句快

  ```insert into user values(1,'Tom‘),(2,'Jerry'),(3,'Mike');```

- 在事务中进行数据插入

  ```mysql
  begin;
  insert into user values(1,'Tom‘);
  insert into user values(2,'Jerry‘);
  insert into user values(3,'Mike‘);
  commit;
  ```

- 数据有序插入

#### order by优化

order by 后面的多个排序字段排序方式尽量相同且排序字段顺序尽量和组合索引字段顺序一致

```mysql
create index idx_emp_age_salary on emp(age,salary);
explain select * from emp order by age;  -- using filesort;
explain select id from emp order by id,age; -- using index;
```

通过创建合适的索引，能够减少filesort出现，但在无法让filesort消失的情况下，就需要加快filesort的排序操作。对于filesort，MySQL有两种排序算法

- 两次扫描算法：在MySQL4.1之前，使用该方式排序，首先根据条件取出排序字段和行指针信息，在sort buffer完成排序后再根据行指针回表读取记录，该操作可能会导致大量随机I/O操作
- 一次扫描算法：一次性取出满足条件的所有字段，在排序区sort buffer中排序后直接输出结果集，排序时内存开销大，但是效率比两次高

#### 子查询优化

使用子查询可以避免事务或者表死锁，但在一些情况下，子查询可以被更高效的连接（JOIN）代替  

连接查询之所以更高效，是因为MySQL不需要在内存中创建临时表来完成逻辑上需要两个步骤的查询工作

#### limit优化

一般分页查询时，通过创建覆盖索引能够比较好地提升性能，但在记录太多时，查询排序的代价非常大 

```mysql
select * from user limit 9000000,10;
```

优化思路

- 在索引上完成排序分页操作，最后根据主键关联回原表查询所需要的其他列内容

```mysql
select * from user a, (select id from user order by id limit 9000000,10) b where a.id = b.id;
```

- 对于主键自增的表，可以把limit查询转换成对某个位置的查询

```mysql
select * from user where id > 900000 limit 10;
```

