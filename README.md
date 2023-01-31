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

    ``` explain select uid,count(*) from user group by uid```



