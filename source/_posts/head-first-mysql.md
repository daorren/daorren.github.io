---
title: 《深入浅出MySQL》笔记
date: 2017-04-10 10:16:37
tags:
- MySQL
- 读书笔记
---
TL;DR

> 《深入浅出MySQL》是讲MySQL不可多得的好书，做法和背后的原理都涉及。索引和锁，show variables 查看参数，优化SQL的几大步骤等，是这本书的核心。不过这篇文章作为记录性笔记，参考意义更大些，可读性可能不是很强。

## 第一部分 基础篇
### 支持的数据类型
- 数值类型
  - 整数类型。按照取值不同，有tinyint、smallint、mediumint、int、bigint。
    指定显示宽度，如int(5)，默认为int(11)，常与zerofill同用，也就是不够长用0填空。存储数据超过宽度限制时，宽度限制不再有意义。
    属性 UNSIGNED（无符号）。tinyint有符号范围是 -127~+127，而无符号范围是 0~255。
    属性 AUTO_INCREMENT。插入NULL到此列时，MySQL插入一个比该列当前最大值大1的值。
  - 小数
    - 浮点数。包括float单精度和double双精度。
    - 定点数。只有decimal一种表示。（注：这里我不懂linux的数值表示，浮点定点是基于二进制表示）
    小数可以在类型名称后加(M, D)，表示该值一共显示M位数字，其中D位位于小数点后面。M和D称为精度和标度。
    浮点数不指定精度时，会按照实际精度（硬件和操作系统决定），而定点数不指定精度时，默认为(10, 0)。
    注意，超过精度的数值，会被截断；而在传统的SQL Mode下，记录将无法插入。
  - BIT位类型。存放二进制数，使用时需要 `select bin(column)` 或者 `select hex(column)`。
- 日期时间类型
  - Date只保存日期，Time只保存时分秒，Datetime保存日期加时分秒。Year只保存年。
  - Timestamp。支持范围较小，表中第一个Timestamp列自动设置为系统时间。Timestamp与时区有关，插入查询都会先转换成本地时区。
- 字符串类型
  - CHAR与VARCHAR。用户保存较短字符，主要区别是存储方式。CHAR列的长度固定为创建表时声明的长度，VARCHAR列保存可变长字符串。
    另外，查询的时候，CHAR列删除末尾的空格，而VARCHAR则保留空格。（改小VARCHAR限制，性能不会提高）
  - ENUM。枚举类型，255个成员以下用1个字节，65535个成员以下用2个字节。一次只能取单个值
  - SET。类似ENUM，存储64个成员以下，占用字节数也大，但一次能取多个值

<!-- more -->

### 常用函数
使用时都要带括号，如 `select curdate()`
- 字符串函数。concat、insert（从指定位置开始子串替换）、lower、upper、left、right（返回左右指定长度字符）、lpad、rpad（填充到左右直到指定长度）、
  ltrim、rtrim（去除左右空格）、repeat、replace（替换指定字符串）、trim（去掉头尾空格）、substring（返回子串）
- 数值函数。abs、ceil、floor、mod、rand（0-1内随机值）、round（四舍五入）、truncate。
- 日期函数。curdate、curtime、now、unix_timestamp、week、year、monthname、date_formate、datediff（起止时间之间的天数）
- 流程函数。if(value, t, f)、case [expr] when [value1] then [result1]...else [default] end

## 第二部分 开发篇
### 表类型（存储引擎）的选择
MySQL的插件式存储引擎是其最重要的特性之一。其中InnoDB和BDB提供事务安全表，其他存储引擎提供非事务安全表。
5.5之前默认引擎是 MyISAM，5.5之后是InnoDB。 查看 `show engines;`
- 存储引擎特性
  - MyISAM。不支持事务，不支持外键，优势是访问速度快。
    每个MyISAM表在磁盘上存储成三个文件，文件名与表名相同，扩展名是 .frm（表定义），.MYD（表数据），.MYI（表索引）。
    数据文件和索引文件可以放在不同目录（建表时指定），平均分布IO，获得更快的速度。
    MyISAM表可能会损坏，损坏表可能不能被访问。 check table 来检查MyISAM表的健康，repair table 来修复损坏。
    支持三种存储格式，静态表、动态表、压缩表。
    - 静态表。默认存储格式。字段都是非变长字段，记录都是定长，优点是存储快速易缓存，缺点是占用空间比动态表多。存储时会用空格补足列宽度，访问时去掉空格。
      注意，保存的内容自身后面带有空格，也会被去掉，开发人员需要注意。
    - 动态表。表中包含变长字段，记录不是定长的，优点是存储占用小。缺点是频繁更新删除记录会产生碎片，需要定期执行 optimize table 来优化，出问题时恢复较困难
    - 压缩表。由myisampack工具创建，占据非常小的空间。
  - InnoDB。提供了提交、回滚和崩溃恢复能力的事务安全。对比MyISAM引擎，写的效率较低，且会占用更多内存和硬盘空间。
    - 自动增长列。`alter table XX auto_increment=n` 修改初始值。
      对于InnoDB，自动增长列必须是索引，在组合索引中必须是第一列。
      但是对于MyISAM表，自动增长列可以是组合索引其他列。这样插入后，自动增长列是按照组合索引前面几列进行排序后递增的。
    - 外键约束。MySQL支持外键的引擎只有InnoDB。实际上是在数据库层做了表关联，至少Rails中没看到使用。
      创建索引时，能够指定更新父表时子表相应的操作。restrict限制在有子表关联的情况下父表不能更新，cascade表示父表更新删除时子表也更新删除，set null表示父表更新删除时子表相应字段设为null。
    - 存储方式。存储表和索引有两种方式。
      - 共享表空间存储。表结构保存在.frm文件中，数据和索引保存在 innodb_data_home_dir 和 innodb_data_file_path 定义的表空间中，可以是多个文件。
      - 多表空间存储。表结构仍在.frm文件中，但是每个表的数据和索引单独保存在.ibd文件中。
  - memory。使用存在于内存中的数据来创建表。默认使用hash索引，也能指定btree。
    每个memory表只对应一个磁盘文件，.frm文件。数据都在内存中，所以访问很快，但是服务关闭数据就丢失。
    主要适用于内容变化不频繁的表。
  - merge。是一组MyISAM的组合，这些MyISAM表结构必须完全相同。
    merge表本身没有数据，对其进行RUD操作都是对内部的MyISAM表的操作。插入是通过insert_method子句定义插入的表。drop操作删除merge的定义，对内部表无影响。
    磁盘保留两个文件，.frm文件存储表定义，.mrg文件包含组合表信息。
    `create table payment_all (...) engine=merge union=(payment_1,payment_2) insert_method=last`。
  - TokuDB。第三方存储引擎，高写性能高压缩。
    适用于，范围查询、存储量大的场景，如日志数据、历史数据等。
### 选择合适的数据类型
- CHAR与VARCHAR。CHAR是定长的，处理速度较快，但是比较浪费空间。长度变化不大且对查询速度较高的可以使用CHAR。
  但是随着版本升级，VARCHAR的性能也在提高。下面是不同存储引擎使用原则。
  - InnoDB。建议使用VARCHAR。对于InnoDB表，内部行存储格式不区分定长变长（所有数据行使用指向该数据列值的头指针），因此使用CHAR不一定就性能好。而存储量角度看，使用VARCHAR更好。
  - MyISAM。建议使用定长代替可变长数据列。（没有理由）
  - memory。目前使用定长的数据行存储，也就是内部都是用CHAR类型处理。
- text与blob。blob能保存二进制数据，text只能保存文本数据。这里只是介绍两者常见的问题。
  - 性能问题，特别是执行了大量删除时。
    删除操作会在表中留下大量空洞（内容删除但是存储空间没有被回收），所以需要定期执行 optimize table 进行碎片整理。
  - 使用合成索引提高查询性能。
    简单说，合成索引就是根据大文本字段的内容建立散列值（md5、sha1等），存储在一个新字段中。以后只要检索散列值，而不用搜索大文本字段，但适用于精确检索。
  - 不必要的时候避免检索大型text或blob值。
    select * 不是个很好的做法，除非能确定where子句只会找到所需数据行。
  - 分离到单独的表中。
    如果移出后，原表能转变成定长的数据行格式，那么就是有意义的。
- 浮点数与定点数
  浮点数会被四舍五入，而定点数其实是以字符串的形式存储的。
  单精度浮点数误差会经常发生，精度要求较高的应用中（如货币）应该使用定点数表示和保存。
  （编程中的浮点数比较，也是个常见的问题来源。应尽量进行范围比较，而不是断言等于或者不等于某个值）
- 日期类型选择
  要记录年月日时分秒，且年代久远，最好使用 datetime，它的范围远大于 timestamp。
  但是如果要表示时区，只有timestamp支持时区。
### 字符集的选择
以前遇到一个问题，需要支持emoji的话，必须使用 UTF-8 Unicode(utf8mb4)。
MySQL字符集包括字符集（character）和校对规则（collation）两个概念。一个是存储字符串的方式，一个是比较字符串的方式。
校对规则命名约定：以字符集名开始，包括语言名，以 `_ci`(大小写不敏感)， `_cs`(大小写敏感)，`_bin`(二元，比较是基于字符编码而与语言无关)结尾。
- 字符集和校对规则设置4级别：服务器级，数据库级，表级，字段级。
  原则是不指定时，使用默认的或者继承上一级的。
- 修改字符集
  alter操作修改字符集，但是这并没有更新已有记录的字符集，只是对新创建的有效。
  第一步，导出表结构。`mysqldump -uroot -p -d dbname tb.sql`
  第二步，手工修改sql文件中的字符集为新的字符集
  第三步，导出所有记录。`mysqldump -uroot -p --no-create-info ...`。打开文件，把 SET NAME latin1 修改为 SET NAME GBK。
  第四步，创建表导入数据。
### 索引的设计和使用
不使用索引，MySQL将会扫描全表，直到找到相关行。
大多数索引类型 primary key, unique, index, fulltext等在btree中存储，空间列类型使用rtee。
- MyISAM和InnoDB默认创建的是btree索引（B树，不是二叉树，是一种一般化的自平衡的二叉查找树）
  MyISAM还支持全文索引。但是应用较少。因为MyISAM自身锁表不锁行，以及复杂全文查询支持差。
  memory默认是hash索引。
  `explain select * from city where city= 'fuzhou' \G`，使用explain语句进行查看SQL是否使用了索引。`\G` 后面不需要分号。
- 索引设计原则
  搜索的索引列，不一定是要选择的列。最适合索引的列是出现在where子句中的列。
  使用短索引。如果对字符串列进行索引，应该制定一个前缀长度，只要有可能就应该这么做。只对前10到20个字符进行索引能够节省大量空间，查询更快。
- 补充
  有时候你创建了索引但是没有被使用，因为MySQL内部有个优化机制，它可能认为不使用更好。如果你想要强制使用，`force index(index_name)`
### 视图
视图是一种虚拟表。相对普通表的优势：简单（视图是已经过滤好的结果集），安全（视图用户只能访问被允许访问的），数据独立（屏蔽表变化对视图的影响）
```sql
create or replace view payment_view as
select (select city from city where id = 1)
```
定义视图时，MySQL不允许from关键字后面有子查询。另外，使用了聚合函数、groupby、union、join，select中包含子查询 等的视图，不能更新。
with [cascaded|local] check option 决定是否允许更新数据使记录不再满足视图的条件。local只要满足本视图条件即可更新，cascaded必须满足所有针对该视图的视图的条件
查看视图， show tables 即可
### 存储过程与函数
存储过程和函数是事先经过编译并存储在数据库的一段SQL的集合。调用存储过程和直接执行SQL的效果是一致的，好处是逻辑封装在数据库端。哪怕需要修改，对于调用者没有影响。
函数和存储过程的区别是函数必须有返回值，存储过程没有。存储过程的参数可以是in、out、inout，函数只能用in。
不过，它们能将数据处理在数据库服务器中间进行，减少数据的传输。但实际上大量复杂运算也会拖累数据库服务器，所以大量复杂运算还是尽量分摊到应用服务器中。
```sql
delimiter $$
create procedure film_in_stock(in p_film_id int, in p_store_id int, out p_film_count int)
read sql data
begin
  select inventory_id from inventory where film_id = p_film_id and store_id = p_store_id and inventory_in_stock(inventory_id);
  select found_rows into p_film_count;
end $$
delimiter ;
call film_in_stock(2, 2, @a)
```
通常执行创建之前，会通过delimiter命令将语句结束符修改为其他符号，这里是 $$。这样过程和函数中的 ; 就不被MySQL解释成语句结束而提示错误。创建完毕后再改回来。
characteristic特征值
  - {contains sql | no sql | reads sql data | modifies sql data }
    子程序包含sql，不包含sql，只包含读数据的语句，包含写语句。一般只是提供给服务器，并没有约束效力。
  - sql security {definer | invoker} 指定子程序该用创建子程序者的许可来执行，还是使用调用者的许可来执行。默认definer。
子程序中，还可以使用变量，条件（处理异常），光标（循环结果集），流程控制，
- MySQL定时任务：事件调度器，将数据库按自定义的时间周期触发操作，可以理解为时间触发器，类似crontab。
  调度器创建后需手动开启才能生效。非常适合定时清空临时表或日志表。
### 触发器
触发器是与表有关的数据库对象，在满足定义条件时触发，执行触发器中定义的语句集合。协助应用在数据库端确保数据完整性。
```sql
delimiter $$
CREATE TRIGGER ins_film AFTER INSERT ON film FOR EACH ROW BEGIN
  insert into film_text(film_id, title, description) values (new.film_id, new.title, new.description);
END; $$
delimiter ;
```
使用old，new别名来引用触发器中发生变化的记录内容。
注意：包含insert或update的SQL，并不只是单纯触发insert或update相关的触发器。
比如对于有重复记录需要进行update的insert，`insert...on duplicate key update...`，按顺序触发 before insert, before update, after update。
### 事务控制和锁定语句
- lock tables 与 unlock table。
  `lock table film_text read;`, 其他线程不能读；`unlock tabls;`，其他线程获得锁。
- 事务。
  所谓事务，不提交，行记录是无法被其他session查询到的。默认情况下，MySQL是自动提交Autocommit的
  start transaction 或者 begin 开始一项事务。commit 和 rollback 提交和回滚事务。savepoint
  补充，MySQL还支持分布式事务，只有InnoDB支持。但是服务器异常重启、客户端连接异常中止，都可能导致事务不完整（prepare状态的分支事务没有写到binlog），总之不推荐。
### SQL安全问题
SQL注入：将用户输入，插入到实际的SQL中执行。
办法：进行各种API对特殊字符进行转换。
### SQL MODE 及相关问题
SQL MODE的严格模式，提供了很好的数据校验功能。查看SQL MODE，使用 `select @@sql_mode`。
指定 `set sql_mode='ansi'`可以获得对应组合模式。常用mode值：ANSI（等同于多个小模式的组合模式）、STRICT_TRANS_TABLES、TRADITIONAL。
另外数据迁移时，还有 ORACLE、DB2、POSTGRESQL 这些模式组合，这样导出的数据更容易导入异构的目标数据库。
### MySQL分区
- 额外发现：MySQL的元数据库 **information_schema**，有个partitions表（其实很多信息都在这个数据库中，包括支持的引擎、字符集、processlist等很多知识）。
  这个表中，能发现到不分区的表其实被看做一个默认分区。能看到表中的行数、数据大小（单位byte）、索引大小等信息。
- MySQL支持大部分引擎（InnoDB、MyISAM、memory等）创建分区表。不能使用主键/唯一键之外的字段分区（除非没有主键/唯一键）。
  - NULL的处理。Range中被看做最小值，Hash/Key中被看做0，List中Null必须出现在枚举列表中。
  主要四种分区类型，Range、List、Hash、Key。前三种要求分区键必须是INT类型，或者通过表达式返回INT。Key分区支持其他键。
  - Range。原本只支持整数列，但是用函数转换也可以。但是如果查询不使用函数，就无法利用分区特性提高性能。
    适用于：删除过期数据，alter table emp drop partition p0 即可删除分区及其数据；经常运行包含分区键的查询，能够确定只有哪些分区要扫描。
    ```sql
    create table emp(...)
    partition by range (year(created_at)) (
      partition p0 value less than ('1992-01-02'),
      partition p0 value less than maxvalue -- 不设上限
    )
    ```
  - List。类似Range，但是是离散值分区。所有分区键值必须在可枚举的集合中，如 `partition p0 values in (2, 4)`。
  - Columns。是5.5新引入的分区类型，解决了Range、List只支持整数分区，从而导致额外的函数计算的问题。
    细分为 Range Column和List Column，支持整数、日期、字符串，不支持表达式作为分区键。
    Column 另一个亮点是，支持多列分区，如 `partition by range columns(a,b)`。实际插入的时候，会先看a是否小于界定值，如果a等于，再看b是否小于界定值。
  - Hash。分区键必须是整数。
    支持两种分区，常规hash分区和linear hash分区。前者使用取模算法，后者使用线性的2的幂的运算法则（线性哈希，网上搜的也看不太懂）。
    - 普通hash分区。`partition by hash(<expr>) partitions <num>`
      原理比较简单，就是把分区键对分区的数目进行取模。
      主要用来分散热点读，使数据在分区中尽可能平均分布。问题：一旦要增加分区或合并分区时，大部分数据都要重新计算分区。
    - 线性hash分区。`partition by linear hash(<expr>) partition <num>`
      原理中有个 `234 & (4-1) = 2` 这个运算看不懂，网上搜的东西就更难看懂了。优势是分区维护时MySQL处理更加迅速，缺点是数据分布不太均衡。
  - Key。类似Hash，但不能使用表达式，并且分区键除了Text和Blob以外都支持。
- 常用SQL
  查看记录在哪个分区中 `explain partition select * from emp where store_id=234\G`。
  查看各个分区中聚合参数。
  `select partition_name part, partition_expression expr, partition_description desc, table_rows from information_schema.partitions where table_schema=schema() and table_name='emp'`
- 分区管理
  - Range/List。删除操作之前提到过。
    增加分区，`alter table emp add partition (partition p4 values less than (2013))`。新增Range分区，必须在分区最大端往更大；新增List分区，不能包含现有值。
    重新分区，`alter table emp reorganize partition p1,p2 into (partition p1 values less than (2015))`。重新定义用来拆分和合并，据说数据不丢失，没试过。
  - Hash/Key。
    减少分区，`alter table emp coalesce partition 2`。减少为2个分区，这个语句不能用来增加分区。
    增加分区，`alter table emp add partition 8`。这是增加8个分区，不是增加到8个，所以现在有10个分区。

## 第三部分 优化篇
### SQL优化
- 优化SQL的一般步骤
  - show status 查看执行频率。
    如 `show global status like 'Com_%'` 查看数据库启动后语句的执行统计结果，`show status like 'Slow_queries'` 查看当前session慢查询次数。
  - 定位执行效率低的SQL语句。
    一般两种方式：查看慢查询日志，以及 `show processlist` 查看当前线程状态。
  - explain 分析低效SQL执行计划
    - explain一个SQL语句后，会有个表展示相关信息
      - select type。表示select类型，常见的有 simple（简单表，不用表连接或子查询）、primary（主查询）、UNION（union中第二个开始的查询）、subquery（子查询第一个select）
      - type。MySQL找到所需行的方式，或叫访问类型。效率 all < index < range < ref < eq_ref < const,system < NULL。
        - type=all。遍历全表
        - type=index。遍历整个索引
        - type=range。索引范围扫描，常见于 <、>、between等操作符
        - type=ref。使用非唯一索引或者唯一索引的前缀扫描，返回匹配某个单独值的记录行。也常见于join操作中。
        - type=eq_ref。使用唯一索引，也就是表中只有一条记录匹配。常见于多表连接中使用主键或unique index作为关联条件
        - type=const/system。单表中最多一个匹配行，这一行的其他列值可以被优化器当做常量来处理。如根据主键或唯一索引查询的是const，再从其中检索时是system。
        - type=null。不用访问表或者索引
      - possible_keys。查询时可能用的索引
      - key。实际使用的索引。
      - key_len。使用到的索引长度
      - Extra。额外信息，这个也很有用，下一节索引会提到。
    - explain extended 加上 show warnings 查看SQL真正执行前优化器做了哪些改写。
      `explain extended select ...` `show warnings \G`。
  - show profile 分析SQL，定位问题
    `select @@have_profiling` 查看是否支持profile，`select @@profiling` 查看开闭状态，`set profiling=1` 开启。
    ```SQL
    show profiles;            -- 查看SQL的 Query_Id。
    show profile for query 4; -- 查看所需 Query_Id的SQL的执行过程中线程的每个状态和消耗时间。
    -- 上面这里常见的是 sending data 状态最耗时。它表示开始访问数据行并把结果返回客户端，而不仅仅是返回结果给客户段，往往需要大量磁盘读取操作。
    -- 此时也可以，通过查询 information_schema.profiling 表，并按照消耗时间倒序排列进行查看。
    show profile cpu for query 4; -- 支持进一步选择 all、cpu、block io、context switch、page fault等明细类型查看
    ```
  - 通过 trace 分析优化器如何选择执行计划
    `set optimizer_trace="enabled=on", end_markers_in_json=on` 打开trace，设置格式为json
    `set optimizer_trace_max_mem_size=1000000` 设置最大能使用的内存大小，避免不能完整显示。（不过书中没有对trace文件内容做详细说明）
  - 确认问题并采取优化措施
    比如全表扫描的话，就创建索引
- 索引问题（指B-Tree索引）
  带索引的查询可以理解为有两类，第一类在索引中查询，第二类从索引回表查询。
  - 索引能够使用的典型场景
    - 匹配全值。查询时，对索引中所有列都指定具体值
    - 匹配值的范围查询。对索引的值进行范围查找
      `explain select * from rental where customer_id >= 373 and customer_id < 400 \G`。注意到 Extra列为 Using Where，表示还需要根据索引回表查询数据
    - 匹配最左（leftmost）前缀。仅仅使用索引最左边的列进行查找。————最重要的原则
      比如 col1 + col2 + col3 字段的联合索引，能够被包含 col1、(col1 + col2)、(col1+col2+col3)的等值查询利用到，可是不能被 col2、(col2+col3)等值查询利用
    - 仅仅对索引进行查询。查询的列都在索引的字段中时，甚至都不用回表。
      `explain select last_update from payment where customer_id=12`。last_update字段也在索引中。注意到 Extra列为 Using index，意味着只要访问索引即可获得所需数据。
    - 匹配列前缀。仅仅使用索引中第一列的开头一部分
      `explain select title from film_text where title like 'African%'`。
    - 索引匹配部分精确而其他部分范围匹配。
      `explain select inventory_id from rental where date = '2006-12-23' and customer_id >= 373 and customer_id < 400 \G`
    - 下面是补充部分
      - 列名是索引，`where <column> is null` 也会使用索引。区别于Oracle。
      - Index Condition Pushdown特性，部分情况下的条件过滤下放到存储引擎。简单说，根据复合索引回表获取记录后，再过滤一次的操作，能够被优化到，不再进行第二次过滤。
  - 存在索引但不能被使用的典型场景
    - 以%开头的like查询。
      比如 `explain select * from actor where last_name like '%NI%' \G`。
      如果我们有个索引存储last_name和主键actor_id，可以首先扫描索引获得满足 last_name like '%NI%' 条件的主键actor_id列表，再根据主键回表检索记录，避开全表扫描
      `explain select * from (select actor_id from actor where last_name like '%NI%')a, actor b where a.actor_id = b.actor_id`。
      这里的第一次查询，我们使用了“仅仅对索引进行查询”，所以条件里面虽然有索引列的非开头部分，也能使用索引进行查询。
    - 数据类型隐式转换时。
      比如对于字符串类型的字段，你的条件是数值类型，将不会使用索引。（这个感觉比较少见，更像是编程错误）
    - 复合索引时，查询条件不包含索引列最左部分。也就是不满足最左原则leftmost，之前提过了
    - MySQL估计使用索引比全表扫描慢时。
      通过trace，可以看到优化器的选择过程，如果使用索引代价cost比扫描全表高，将不会使用
    - 用or分割开的条件，如果or前的条件有索引，后面的列没有索引。
      or后面列没有索引，肯定要全表扫描。在存在全表扫描的情况下，就没必要多一次索引扫描了。
  - 查看索引使用情况
    `show global status like 'Handler_read%'`。
    Handler_read_key表示一个行被索引值读的次数，越高越好；Handler_read_rnd_next表示数据文件中读下一行的请求数，表扫描越多，值越高，意味着查询低效。
- 简单优化方法
  - 分析表和检查表
    `analyze table <table>` 用于分析和存储表的关键字分布，分析结果使系统得到准确的统计信息，使SQL生成正确的执行计划。如果感觉执行计划生成不是预期，分析表可能可以解决
    `check table <table>` 用于检查表是否有错误。
  - 定期优化表
    `analyze table <table>`。常用于删除大量数据后，合并空间碎片，前面提过。
- 常用SQL优化
  - 大批量插入数据
    - MyISAM
      `alter table tbl_name disable keys`，导入到非空MyISAM表前，先关闭非唯一索引的更新，之后再 enable。（导入空表，默认就是先导入后创建索引的，所以不需要设置）
    - InnoDB
      InnoDB类型表是按照主键顺序保存的，所以将导入的数据按照主键顺序排列，能够提高导入效率
      `set unique_checks=0`，导入前，关闭唯一性校验，之后再打开。
      如果使用自动提交的方式，导入前，`set autocommit=0` 关闭自动提交，之后再打开。
  - 优化insert语句
    同一用户插入多行，尽量使用一次插入多值，`insert into test values(1,2), (1,3), (1,4)...`
    不同用户插入多行，可以用insert delayed。含义是insert语句立刻执行，其实数据都被放在内存队列中，没有写入磁盘；low_priority相反，在所有其他用户读写完成后才插入
    索引文件和数据文件在不同磁盘上存放（建表时）
    批量插入MyISAM时，增加 bulk_insert_buffer_size 变量值
    从文本装载时，load data infile，比使用许多insert语句快得多。
  - 优化order by语句
    MySQL有两种排序方式，
      - 一种索引直接返回，`explain select customer_id from customer order by store_id \G`。
      - 凡是非索引直接返回的都叫Filesort排序。
        Filesort不代表通过磁盘进行排序，而是说明进行了一个排序操作，算法马上说明。
        两种算法：
        - 两次扫描。第一次取出排序字段和行指针信息，之后在sort buffer中排序（不够的话，在临时表中存储排序结果），完成后回表读取记录。效率低，但是占用内存小
        - 一次扫描。一次取出所有满足条件行的所有字段，然后在sort buffer中排序。效率高，但是占内存。
        `max_length_for_sort_data` 和query语句取出的字段总大小比较，如果max_length更大，就用效率更高的第二种算法。但不能设置过大，否则造成CPU利用率过低和磁盘IO过高。
        适当增大 `sort_buffer_size`，让排序在内存中完成，而不是通过创建临时表在文件中完成。但也不能设置过大，否则导致服务器SWAP严重（？）
          select 具体字段名称，可以减少排序区的使用。
    优化目标：减少额外排序，索引直接返回有序数据。where条件和order by使用相同索引，且order by的顺序和索引顺序相同，且order by的字段都是升序或降序。否则肯定需要Filesort
    - 对索引使用的影响。（使用Filesort与不使用索引并不严格相关）
      order by中字段混合升序降序，where与order by使用不同索引，order by中使用不同关键字：都会导致不使用索引。
      order by中字段属于同一索引且都是同序，where与order by使用相同索引：都会使用索引。
  - 优化group by语句
    默认情况下，MySQL对所有group by字段进行排序，与order by字段差不多。不过如果想要避免排序结果的消耗，可以 order by null 禁止排序。
  - 优化嵌套查询
    子查询技术，可以用select语句创建一个单列的查询结果，作为过滤条件用在另一个查询中。
    有些情况下，可以使用join替代。join之所以更有效率一些，是因为MySQL不需要在内存中创建临时表来完成这个逻辑上需要两步的查询工作。
  - 优化OR条件
    OR前后的条件列都必须用到索引，OR前后使用同一复合索引是不行的。执行计划中能看到实际是对OR前后分别查询再UNION。
  - 优化分页查询
    常见的头痛场景是，`limit 1000,20`。排序出前1020条记录后仅需返回最后20条，前面1000条都被舍弃，代价很高。
    - 第一种思路：在索引上完成排序分页，最后根据主键关联回表查询所需其他列的内容
      `explain select film_id, description from film order by title limit 50,5 \G`
      `explain select a.film_id, a.description from film inner join (select film_id from film order by title limit 50,5)b on a.film_id=b.film_id \G`
    - 第二种思路：翻页时增加一个参数，用来记录上一页最后一行的id。这样只要找到比这个id大（或者小）的几个即可。
      将`limit m,n` 转换成了 `limit n` 的查询。适合排序字段不会出现重复值的情况
  - 使用SQL提示
    都是写在表明之后，如 `select * from rental force index (idx_fk_inventory_id) where inventory_id > 1 \G`
    - use index。提供希望MySQL参考的索引列表
    - ignore index。忽略索引
    - force index。强制使用索引。
### 优化数据库对象
- 优化表数据类型。
  `select * from tbl_name procedure analyse()`。分析当前应用表，并查看优化建议。
- 拆分表。这里提到的是MyISAM，InnoDB应该一样。
  - 垂直拆分。主码和一些列放到一个表，主码和另外的列放到另一个表中。常用于，某些列常用，另一些不常用。数据行变小，一个数据页能存放更多数据，减少IO。缺点是查询所有数据时需要join操作。
  - 水平拆分。多列放到两个独立表中。常用于，表很大减少数据量，本身就有一定独立性如不同月份的话单。
- 逆规范化
  规范化越高，产生的关系就越多，导致的就是表之间连接操作越频繁。所以对于查询较多的应用，需要根据实际情况进行逆规范化设计提高性能。
  （增加冗余列操作跟垂直拆分其实恰好相反。使用情景也相反，垂直拆分是表中不常用的字段过多，逆规范化是表中缺失少量字段需要连表）
  常用的操作有，增加冗余列，增加派生列（计算生成），重新组表（经常要连表干脆组合起来），分割表（就是拆分）
  逆规范化技术需要维护数据完整性。常见的有在应用逻辑实现，使用触发器等。
### 锁问题
- MyISAM表锁
  - 查询表级锁争用情况。 `show status like 'table_lock%'`。
  - 锁模式。两种：表共享读锁，表独占写锁。读操作与写操作之间、以及写操作之间是串行的。
  - 如何加表锁。
    MyISAM执行查询前，自动给涉及表加读锁；执行更新操作（增删改）前，自动加写锁。一般不用用户显式地lock table。（书中主要是为了方便说明）
    显式lock tables时，当前线程只能访问显式加锁的这些表，不能访问未加锁表。MyISAM必须一次获得所有涉及表的锁，所以不会出现死锁。
  - 并发插入。
    一定条件下，MyISAM也支持查询和插入操作并发进行。加锁时带上local选项，如 `lock tabls film read local;`
    该存储引擎系统变量 concurrent_insert 用于控制。设为0，不允许并发插入；设为1，表没有空洞时允许另一进程在表尾插入记录；设为2，不管有没有空洞都允许在表尾并发插入
  - 锁调度。
    MySQL认为写请求比读请求重要，所以写进程优先获得写锁，读进程不要写锁只要读锁也在后面。所以MyISAM表不太适合大量更新和查询操作应用，读操作可能永远阻塞。
    通过设置调节调度行为：`set low_priority_updates=1` 降低该连接的更新请求优先级；指定增删改语句的low_priority属性，降低语句优先级。
- InnoDB表锁
  - 背景知识
    - 事务是由一组SQL语句组成的逻辑处理单元，具有以下ACID属性
      - 原子性Atomicity。事务是一个原子操作单元，对数据的修改，要么都执行，要么都不执行；
      - 一致性Consistent。事务开始和完成时，数据必须保证一致状态。意味着事务过程中，不能有其他事务修改数据
      - 隔离性Isolation。数据库系统提供一定的隔离机制，保证事务在不受外部并发操作影响的环境中执行。事务过程中，中间状态对外部是不可见的，反之亦然。
      - 持久性Duration。事务完成之后，对于数据的修改是永久性的，即使系统故障也能保持。
    - 并发事务带来的问题
      - 更新丢失。多个事务选择同一行并进行更新时，每个事务都不知道其他事务的存在，就会发生更新丢失——最后的更新覆盖其他事务的更新
      - 脏读。一个事务正在对一行进行修改，事务完成提交之前，这行数据就处于不一致状态。这时如果另一个事务来读取同一行，如果不加控制，就读取了脏数据。
      - 不可重复读。一个事务过程中，两次读同样的数据，却发现数据已经发生改变甚至被删除
      - 幻读。一个事务过程中，两次用同样的条件进行查询，第二次发现有其他事务插入的符合条件的新数据。
    - 事务隔离级别
      - 解决问题，防止更新丢失是应用的责任（数据库事务具有隔离性）；而另外三种都是数据库读一致性问题，数据库通过提供事务隔离机制来解决，一般有两种方式
        - 读取数据前，对其加锁，防止其他事务对数据进行修改
        - 不加锁，通过一定机制生成一个数据请求时间点的一致性数据快照，并用这个快照提供一定级别（语句或事务级）的一致性读取。叫做数据多版本并发控制MVCC。
      - 隔离与并发的矛盾，SQL92 定义了4个事务隔离级别。分别是上述三个一致性问题的反映，
        未提交读，全部都会发生；已提交读，脏读不发生；可重复读，只有幻读发生；可序列化，全部不发生。
      MySQL中查看隔离级别，`select @@tx_isolation;`
  - 获取InnoDB行锁争用情况。`show status like 'innodb_row_locks%'`。如果比较严重，则需要查看详细
    - 查看information_schema.innodb_locks表。
    - 设置InnoDB Monitors。按下面操作，默认15秒向日志中记录监控内容，完成后务必删除监控表。
      ```sql
      create table innodb_monitors(a INT) engine=innodb;
      show engine innodb status \G
      drop table innodb_monitors;
      ```
  - 行锁模式和加锁方法
    - 行锁
      两种行锁。共享锁S：组织其他事务获得相同数据集；排它锁X：阻止其他事务取得相同数据集的读锁和写锁。
    - 意向锁。允许行锁与表锁共存，意向锁都是表锁。
      两种意向锁。意向共享锁IS：事务在给行加S锁前必须获得表的IS锁；IX锁同样。
    意向锁是自动加的：对于增删改语句，InnoDB自动加涉及数据集的X锁；对于查询语句，不加任何锁。
    也可以显式地给数据集加锁：S锁，`select * from tbl_name where... lock in share mode`；X锁，`select * from tbl_name where... for update`。
    如果 `select * in share mode`获得共享锁，如果还要进行更新操作，则很容易造成死锁。要进行更新操作时，务必使用排它锁。（？意向锁是自动加的，但是显式加锁并不常见）
  - InnoDB行锁实现方式。
    InnoDB行锁是通过给索引上的索引项加锁来实现的，如果没有索引，将通过隐藏的聚簇索引来对记录进行加锁。
    - 如果不通过索引条件检索数据，那么InnoDB将对表中所有记录进行加锁，效果等同于表锁。
    - 访问不同行数据，但是如果使用相同索引键（查询使用的其他键没有索引），是会出现锁冲突的。
    - 如果使用不同索引，但是访问相同数据集，前面查询的排它锁没有释放，后面的查询也无法获得数据。（只要使用索引，通常就是行锁而不锁表；访问相同的数据，肯定是被锁的。）
    - 注意，分析锁冲突的时候，记得检查SQL的执行计划。如果MySQL认为不需要使用索引，也将锁表。
  - Next-Key锁。
    使用范围条件检索并请求锁时，InnoDB会给键值在条件范围内但并不存在的记录（Gap）加锁，这个锁机制叫做Next-Key锁。作用是防止幻读。
    但是会阻塞范围内的并发插入，所以在实际应用中尽量使用相等条件来访问更新数据。（相等条件请求给一个不存在的记录加锁，也会使用Next-Key锁）
  - 恢复和复制的需要，对InnoDB锁机制的影响
    MySQL通过binlog记录执行成功的增改删等更新数据的SQL语句，并由此实现数据库的恢复和主从复制。有三种日志格式，基于语句的日志格式SBL，基于行的RBL和混合格式。
    - 对基于语句日志格式SBL而言，要求一个事务未提交之前，并发事务不能插入满足其锁定条件的任何记录，也就是不允许出现幻读。这就是InnoDB需要使用Next-Key锁的原因。
    - 对于`insert into target_tbl select * from source_tbl where...` 和 `create table new_tbl ...select * from source_tbl where...`
      这两个语句，只是简单读source_tbl表的数据，相当于一个普通的select。InnoDB有多版本数据，对普通select一致性地读，不加任何锁；但在这里却给source_tab加共享锁。
      因为在语句执行过程中，其他事务对source_tab做了更新操作，如果不加锁，就可能导致数据恢复的结果错误。（binlog中保存事务顺序不对）
      所以推荐的不加锁做法是，select into outfile 和 load data infile的语句合用；或者使用基于行的binlog格式和基于行的复制。
  - InnoDB使用表锁
    特殊事务中，考虑表级锁：事务需要更新大部分或全部数据，事务执行效率低；事务涉及多个表比较复杂，容易引起死锁，造成大量事务回滚。
    - 表锁不是由InnoDB存储引擎层控制的，而是由其上一层MySQL Server负责，仅当 autocommit=0、innodb_table_locks=1（默认设置）时，InnoDB才能识别表级锁的死锁。
    - lock tables 事务结束时，不要 unlock tables释放表锁，因为它会隐含地提交事务；而commit或rollback不能释放表锁。两个都是不可省略的，unlock tables在后。
  - 关于死锁
    MyISAM总是一次获得所需全部锁，要么全部满足，要么等待，不会出现死锁。而InnoDB，除单个SQL组成的事务外，锁是逐步获得的，所以可能发生死锁。
    比如两个事务，都需要获得对方持有的排它锁才能完成事务，就是典型的死锁。设置锁等待超时参数 innodb_lock_waite_timeout 解决死锁问题，以及高并发下的性能问题。
    一般来说，死锁都是应用设计的问题，下面介绍避免死锁的常用方法
    - 应用中，如果不同程序并发存取多个表，应尽量约定以相同的顺序来访问表。（比如刚才的典型死锁，就是互相需求对方所执的锁）
    - 程序以批量方式处理数据的时候，尽量事先对数据进行排序，保证每个线程按固定顺序来处理记录。（跟上面这条差不多）
    - 事务中更新记录，直接申请足够高级别的锁，即排它锁。而不应先申请共享锁，更新时再申请排它锁。因为此时其他事务可能又获得了相同记录的共享锁，造成锁冲突甚至死锁。
    - repeated-read隔离级别下，两线程同时对相同记录加排它锁，在没有符合该条件记录情况下，两个线程都会加锁成功。这时如果两线程都试图插入新记录，就会死锁。隔离界别改为read commited 可解。
    - read-commited隔离级别下，两线程都执行加排它锁，判断是否存在条件的记录，如果没有就插入记录。此时只有一个能成功，另一个锁等待，等第1个提交之后，第2个线程会主键重复出错，但它会获得一个排它锁。这时第3个线程申请排他锁也会出现死锁。
      这时应先做插入，再捕获主键重复异常，或者遇到主键重复异常时，总是rollback释放排它锁。
    最后，如果出现死锁，`show innodb status` 可以查看最后一个死锁的产生原因。
### 优化MySQL Server
- 内存管理及优化
  - MyISAM
    MyISAM存储引擎使用key buffer缓存索引块；而对于数据块，MySQL没有特别的缓存机制，完全依赖于系统的IO缓存。
    - key_buffer_size。决定MyISAM索引块缓存区大小
    - 使用多个索引缓存。key buffer是共享的，所以增大并不能解决多session的竞争。
      `set global hot_cache.key_buffer_size=128*1024` 新建索引缓存（设为0即是删除）。`cache index sales in hot_cache` 指定sales表的索引缓存，否则使用默认。
    - 调整“中点插入策略”。
      默认情况下，MySQL使用LRU（Least Recently Used）策略来选择要淘汰的索引数据块。但是这个算法不精细，某些情况下会导致真正的热块被淘汰。
      所以这里引入中点插入策略，其关键是把LRU链分成两部分。通过 key_cache_division_limit 来控制。默认是100，也就是不启用此策略；设为70，也就是用30%缓存来cache最热索引
  - InnoDB内存优化
    InnoDB用一块内存区做IO缓存池，不仅缓存InnoDB索引块，也用来缓存InnoDB数据块。内部，缓存池逻辑上由free list、flush list和LRU list构成。
    free list是空闲缓存列表，flush list是需要刷新到磁盘的缓存块列表，LRU list是正在使用的缓存块，是 InnoDB buffer pool 的核心。
    优化性能：可以通过调整InnoDB buffer pool大小、改变中点插入策略分配比例、控制脏缓存的刷新活动、使用多个InnoDB缓存池等方法来。
    - innodb_buffer_pool_size。表数据和索引数据的最大缓存区大小。可以将80%物理内存分配过去，但是不能设置过大导致页交换（？待查询）
    - innodb_old_blocks_pct，控制中点插入策略old sublist的分配比例。可通过show variables like查看。
    - innodb_old_blocks_time，决定了缓存数据块由old sublist转移到young sublist的快慢。
    - innodb_buffer_pool_instance，调整缓存池数量，将缓存池大小平均分配给这么多个缓存池，减少内部争用。
    - 缓存池中数据停留时间尽可能长，从而减少磁盘IO次数，磁盘IO是数据库系统的最主要瓶颈。通过延迟缓存刷新来减少IO系统的压力，缓存池刷新快慢主要取决于下面两个参数
      - inndb_max_dirty_pages_pct，控制了缓存池脏页的最大比例
      - innodb_io_capacity，磁盘IO能力
    - InnoDB doublewrite。MySQL数据页与操作系统数据页大小不一致，无法保证InnoDB缓存页被完整一致地刷新到磁盘。双写开启对性能影响不大，但如果对一致性要求不高，也可以关闭。
- InnoDB log机制及优化
  支持事务的数据库系统，内部使用redo log来保证事务的ACID。
  [MySQL redo log及recover过程浅析](http://www.cnblogs.com/liuhao/p/3714012.html)
  更新数据时内部流程：数据读入缓存池，相关记录加独占锁；将UNDO信息写入undo表空间的回滚字段；更改缓存页中数据，将更新记录写入redo buffer；提交时将redo buffer中更新记录刷新到InnoDB redo log file中，然后释放独占锁；最后后台IO线程根据需要择机将缓存中更新过的数据刷新到磁盘文件中。
  （`show engine innodb status` 查看日志写入情况。LSN（log sequence number）日志序列号，老的LSN = 新的LSN + 写入的日志大小）
  优化性能：除缓存池外，InnoDB log buffer大小、redo日志文件大小以及 innodb_flush_log_at_trx_commit（提交时刷新到log file方式）等参数设置
  - innodb_flush_log_at_trx_commit。设为0，InnoDB每秒触发一次缓存日志回写磁盘操作，并调用fsync刷新IO缓存；设为1，事务提交时立刻回写刷新；设为2，立即回写但不刷新IO。
    默认是1，最安全但性能最差。修改为0，如果数据库崩溃，最后1秒重做日志丢失，最不安全；修改为2，如果崩溃，数据不会丢失，安全性能的折中。
  - innodb_log_file_size。一个日志文件写满之后，InnoDB会切换到另一个日志文件，切换时触发数据库检查点，导致缓存脏页小批量刷新，明显降低数据库性能。
    一般来说每半小时写满一个日志文件比较合适。书中给了两种估算方法，其中一个是通过information_schema.global_status表的innodb_os_log_written来计算。
  - innodb_log_buffer_size，重做日志缓存池
- MySQL并发相关参数
  max_connections，提高并发连接（connection_errors_max_connection）；table_open_cache，控制所有SQL执行线程可打开表缓存的数量
### 磁盘IO问题
前面提到的SQL优化、数据库对象优化、数据库参数优化、应用程序优化等，通过减少或延缓磁盘读写来减轻IO压力。这里从磁盘阵列、符号链接、裸设备等底层技术提高磁盘IO能力
- 磁盘阵列 RAID，按照一定策略将数据分布到若干物理磁盘上。
- 虚拟文件卷或软RAID，操作系统模拟了一些RAID特性，改善性能和可靠性。
- 符号链接 Symbolic Links。利用操作系统符号链接（ln -s），将不同数据库、表或索引的datadir指向不同物理磁盘，从而分布磁盘IO。
- IO调度算法。操作系统对于磁盘IO请求，有四种调度算法。MySQL数据库环境设为Deadline算法更稳定；对于SSD设备，采用NOOP或者Deadline更好。
### 应用优化
- 使用连接池。不是直接访问数据库，而是从这个连接池中获取连接来使用。
- 减少对SQL的访问：避免重复检索。增加cache层。
- 负载均衡
  - 利用MySQL的主从复制，可以有效分流更新和查询操作。具体是主服务器承担更新，多台从服务器承担查询，主从之间通过复制实现数据同步。
  - 分布式数据库架构。具体可以使用MySQL的Cluster功能。

## 管理维护篇
### MySQL高级安装和升级
高级安装，即除RPM包以外的，二进制包和源码包安装两种方式
- 升级：
  - 安装新版本MySQL
  - 建表导数据，两种方式。
    - 第一种，`mysqladmin -h host -u user -p pwd create db_name`，`mysqldump --opt db_name | msyql -u user -p pwd db_name`
    - 第二种
      ```shell
      mysqldump --tab=DUMPDIR db_name   # --tab不会生成SQL文本，而是对每个表生成.sql和.txt文件，分别保存表创建语句，纯数据文本
      # 将DUMPDIR目录文件转移到目标服务器上
      mysqladmin create db_name         # 创建数据库
      cat DUMPDIR/*.sql | mysql db_name # 创建表
      mysqlimport db_name DUMPDIR/*.txt # 加载数据
      ```
  - 旧版本mysql数据库目录全部cp过来覆盖新版本，如 `cp -R /home/mysql_old/data/mysql /home/mysql_new/data/mysql`
  - 新版本MySQL shell中执行 mysql_fix_privilege_tables 升级权限表
  - 重启，完成
### MySQL常用管理工具（全部都是命令行，而不是在shell内）
mysql、myisampack（MyISAM表压缩）、mysqladmin（侧重管理）、mysqlbinlog（日志管理）、mysqlcheck（MyISAM表维护）、mysqldump（数据导出）、
mysqlhotcopy（MyISAM热备份）、mysqlimport（数据导入）、mysqlshow（数据库对象查看）、perror（错误代码查看）、replace（文本替换工具，类似sed）。
### MySQL日志
MySQL有四种日志，错误日志、二进制日志、查询日志、慢查询日志。日志默认似乎都在DATADIR目录下，并且除了错误日志，其他都默认关闭。
查看 DATADIR 命令：`SHOW VARIABLES LIKE "%dir"`。Mac上默认值是，`/usr/local/var/mysql/`。
- 错误日志
  记录了mysqld启动和停止时，以及服务器运行过程中发生任何严重错误时的相关信息。数据库出现故障无法使用时，可首先查看错误日志。
  错误日志默认在DATADIR（数据目录）下，文件名为 host_name.err。比如我的Mac下文件名叫 jojo.local.err。
- 二进制日志
  记录了所有DDL和DML语句，但是不包含查询语句。对于灾难时数据恢复起着极其重要的作用。
  `show binary logs`查看日志文件名，默认是host_name-bin.num。二进制日志默认是关闭的，`SHOW VARIABLES LIKE "%log_bin%"`，打开使用 `--log-bin`。
  - 三种日志格式：
    statement语句格式，每一条对数据造成修改的SQL都会被记录在日志中。清晰易读，日志量小，对IO影响小，但是有时用于复制会出错。
    row行格式，将每一行的变更记录都记录下来。比如全表更新的SQL，statement格式只记录一句，row格式会记录总行数的日志。缺点是日志量大，对IO影响大。
    mixed格式，默认格式，混合了上述两种。默认情况下采用statement，但在特殊情况下采用row，比如客户端使用临时表、使用不确定函数等。
  - 读取日志。二进制方式存储，所以要使用mysqlbinlog工具。如果是row格式，记得加上-v翻译字符。
  - 删除日志。
    - `reset master`。全部清除
    - `purge master logs to 'mysql-bin.000006'`。删除某个编号之前的日志。
    - `purge master logs before '2007-08-10 04:07:00'`。删除某个日期之前的日志。
    - 启动参数`--expire_logs_days=3`。重启后即可 `mysqladmin flush-log` 触发日志文件更新。
- 查询日志
  查询日志记录了客户端所有语句，而二进制不包含只查询的语句。
  可以设置 `--log-output=TABLE,FILE|NONE`，表示将日志保存在表和文件中（或不保存）。这个对查询日志、慢查询日志都管用。输出到表时，慢查询精确到秒，输出到文件时精确到微妙
  host_name.log 日志文件存储的是纯文本，可以直接读取。对于访问频繁的系统，此日志对性能影响较大，一般建议关闭。
- 慢查询日志
  记录了所有，执行时间超过 long_query_time（默认10秒）设置值且扫描记录不小于 min_examined_row_limit的所有SQL语句。
  默认情况下，管理语句 和 不使用索引查询 的语句都不会被记录。
  默认日志名，host_name-slow.log，纯文本保存可直接查看。打开慢查询日志 `--slow_query_log`，指定日志路径 `slow_query_log_file`。
  如果慢查询日志记录有很多，可使用 mysqldumpslow 工具，对慢查询日志进行分类汇总。
- 第三方日志分析工具（笔记中除了特别注明第三方，其他都是自带工具）
  mysqlsla 可以分析多种日志，不限于某种类型。具体使用看文档，这里不记录。
  还有些其他日志分析工具，myprofi、mysql-explain-slow-log、mysqllogfilter等。
### 备份与恢复
分为逻辑备份与物理备份，逻辑备份不分引擎，而对于物理备份，不同引擎有不同的备份方法
- 逻辑备份
  将数据库中的数据备份为一个文本文件，备份的文件可以被查看和编辑。通常使用mysqldump来备份。
  为了保证数据一致性，MyISAM备份时加上-l参数，给所有表加读锁；对于InnoDB，可用--single-transaction，得到一个快照（dump过程中不会看到其他会话提交的数据）。下面详细说恢复
  - 完全恢复
    ```shell
    mysql -uroot -p test < test.dmp                           # 恢复某个时间的备份
    mysqlbinlog localhost-bin.000015 | mysql -uroot -p test   # 使用mysqlbinlog恢复自备份以来的binglog（前提是你知道从哪开始）
    ```
  - 基于时间点恢复（假设10点进行了误操作，所以需要按照时间跳过误操作语句）
    ```shell
    mysqlbinglog --stop-date="2005-04-20 9:59:59" localhost-bin.000015 | mysql -uroot -p
    mysqlbinglog --start-date="2005-04-20 10:01:00" localhost-bin.000015 | mysql -uroot -p
    ```
  - 基于位置恢复（更精确，因为同一时间可能有多个SQL）
    ```shell
    mysqlbinglog --stop-date="2005-04-20 9:55:50" --start-date="2005-04-20 10:05:00" localhost-bin.000015 > /tmp/mysql_restore.sql
    # 根据导出的文件，找到出问题语句前后的位置号。如前后位置号分别为 368312 和 368315。
    mysqlbinglog --stop-position="368312" localhost-bin.000015 | mysql -uroot -p
    mysqlbinglog --start-position="368315。" localhost-bin.000015 | mysql -uroot -p
    ```
- 物理备份与恢复
  - 冷备份。其实就是停掉数据库服务，cp数据文件的方法。很少使用，因为应用一般不允许长时间停机
  - 热备份。
    - MyISAM。
      操作方法的本质是，将要备份的表加读锁，然后cp数据文件到备份目录。
      使用mysqlhotcopy工具；手工备份：`flush tables for read`，然后cp数据文件到备份目录即可。
    - InnoDB。介绍了收费第三方工具ibbackup。
    - Xtrabackup热备工具。这个第三方包包含两个工具，xtrabackup和innobackupex。
      xtrabackup只能备份InnoDB和XtraDB两种，不能备份MyISAM；innobackupex是封装前者的Perl脚本，能备份InnoDB和MyISAM，同样备份MyISAM的时候加全局读锁。
      参考： [innobackupex在线增量备份及恢复mysql数据库](http://devliangel.blog.51cto.com/469347/1374232)，[innobackupex在线增量备份及恢复](http://blog.csdn.net/dbanote/article/details/13295727)
      - 全量备份
        ```shell
        # >>>>>>>> 进行全备
        innobackupex --user=back --password=123 --defaults-file=/etc/my.cnf /data/backup/hotbackup/full --no-timestamp
        # 恢复之前，停止数据库，并删除数据和日志文件（可以备份）。备份删除这两步也可以变成，mv命令重命名，然后mkdir原名空目录。
        mysqld stop               # 停掉服务
        cp -a mysql/ mysql.bak    # archive备份要删除的文件目录
        cd mysql && rm -rf *      # 执行删除
        # >>>>>>>> 恢复全备，分两步，第一步 apply-log，第二步 copy-back。
        innobackupex --defaults-file=/etc/my.cnf --use-memory=20G --apply-log /data/backup/hotbackup/full
        innobackupex --defaults-file=/etc/my.cnf --user=back --copy-back /data/backup/hotbackup/full
        # 修改权限
        chown -R mysql:mysql /var/lib/mysql
        mysqld start
        ```
      - 增量备份
        增量备份时，要先进行一次全量备份，第一次增量是基于全量的，之后的增量是基于上一次增量备份
        ```shell
        # 假设已经全量备份过一次，目录为 /data/backup/hotbackup/base，创建第一次增量 incremental_one
        innobackupex --defaults-file=/etc/my.cnf --user=back --incremental /data/backup/hotbackup/incremental_one --incremental-basedir=/data/backup/hotbackup/base
        # 第二次备份 incremental_two
        innobackupex --defaults-file=/etc/my.cnf --user=back --incremental /data/backup/hotbackup/incremental_two --incremental-basedir=/data/backup/hotbackup/incremental_one
        # >>>>>>>> 增量恢复。 注意一开始要加 --redo-only 参数，只应用xtrabackup日志中已提交的事务数据，不回滚未提交的数据。最后一次不加，回滚未提交。
        innobackupex --apply-log --redo-only /data/backup/hotbackup/base
        # 两次增量备份应用到基础备份，最后一次去掉 --redo-only
        innobackupex --apply-log --redo-only /data/backup/hotbackup/base --incremental-dir=/data/backup/hotbackup/incremental_one/
        innobackupex --apply-log /data/backup/hotbackup/base --incremental-dir=/data/backup/hotbackup/incremental_two/
        # 所有合在一起的基础备份整体一次apply，回滚未提交数据
        innobackupex --apply-log /data/backup/hotbackup/base
        # 后面依旧是停止数据库，复制数据，赋权，启动数据库。这里不赘述
        ```
      - 不完全恢复
        innobackupex备份的文件夹中，有些文件包含有用的信息。如 xtrabackup_binlog_info 能查看备份结束时binglog的名称和位置，根据这个能利用binlog进行不完全恢复。
      - 克隆slave。
        在线添加从库时，常用的参数是，--slave-info 和 --safe-slave-backup。
- 表的导入导出。
  导出：`select * from tbl_name into outfile 'target_file' [options]`；或者mysqldump。
  导入：`load data infile 'source_file' into table tbl_name [options]`；或者mysqlimport。
### MySQL权限与安全
权限的存取，会用到mysql数据库的user、host和db三张表，主要是user。账号的增删改查不记录了。
安全上，避免以root权限运行MySQL。以root用户启动（Linux的用户）MySQL，任何具有FILE权限的用户（MySQL用户）都能够读写root用户的文件。
### MySQL监控
主要是讲Zabbix，配合使用Fromdual插件，用于监控MySQL的MPM插件
### MySQL常见问题
- 忘记root密码
  ```shell
  # 关闭MySQL服务后，带 --skip-grant-tables 参数重启，跳过权限表验证
  mysqld_safe --skip-grant-tables --user=mysql
  # 不用密码以root登录MySQL
  mysql -uroot
  mysql> update user set password=password('123') where user='root' and host='localhost';
  mysql> flush privileges;    # 刷新权限表，权限认证重新生效。重新正常以root登录即可。
  ```
  [官方文档说明](https://dev.mysql.com/doc/refman/5.7/en/resetting-permissions.html)
- mysql.sock 丢失后如何连接数据库
如果指定localhost作为主机名，则默认使用UNIX套接字文件连接，而不是TCP/IP。而这个套接字文件（一般叫mysql.sock）经常会被删除。
这时候，手动指定连接协议即可：`mysql --protocol=TCP -uroot -p -hlocalhost`。

## 架构篇
### MySQL复制（Replication）——《高性能MySQL》主要内容之一
- 复制概述
  原理：MySQL主库把数据变更记录在binlog中，主库推送binlog中的事件到从库的中继日志Relay Log，从库对这些日志重新执行（也叫重做），以保持数据一致
  3个线程：从库启动复制时，创建IO线程连接主库，主库随后创建binlog dump线程读取数据库事件并发送给IO线程，IO线程更新从库relay log，从库SQL线程读取中继日志并应用。
  relay log格式内容等基本和binlog一样，只不过会被SQL线程删除。另外，从库还会创建 maser.info 和 relay-log.info 保存从库IO线程读取主库binlog和SQL线程应用中继的进度
  三种binlog格式，对应三种复制模式。基于语句的复制SBR，基于行的复制RBR，混合模式复制。
  - 复制的常见架构
    - 一主多从复制架构。
      主库读取压力很大时，配置这个架构实现读写分离。把读请求通过负载均衡分布到多个从库上。主库宕机时，可把一个从库切换为主库继续提供服务。
    - 多级复制架构。
      一主多从架构，由于复制是主库主动推送，主库IO压力和网络压力会随从库增加而增加。
      多级架构，就是在一主多从的架构下，主库和从库之间增加二级主库。这样主库只要负责写请求和推送给一个二级主库，减轻主库压力。缺点是，两次复制延时比一次复制延时要大。
      可在二级主库上选择表引擎为BLACKHOLE，来降低多级复制的延时。写入BLACKHOLE表的数据并不会写到磁盘上，永远是空表，数据变更仅仅是在binlog中记录。
      注意，从库默认是不记录binlog的，如果它本身作为下一级的主库的话（如二级主库），需要打开 log-slave-updates 配置。
    - 双主复制/Dual Master 架构
      两个主库互为主从，避免了对主库进行维护带来的额外搭建从库的麻烦。
      双主复制可以和主从复制联合使用，双主多级架构。
- 复制搭建
  - 异步复制
    在主库上锁表备份，将主库备份恢复到从库上；主从库在配置文件中设置唯一的server-id的值，都重启，从库重启时使用--skip-slave-start，这样不会立即启动复制进程，方便进一步配置
    在从库上配置 `change master to ...`（其中file和position信息在主库上`show master status`查看，是从库复制的目的坐标），然后启动salve线程，`start slave`。`show processlist` 查看进程，看是否连接上。
    （`change master to`的配置参数包括，master_host、master_port、master_user、master_password、master_log_file、master_log_pos 等）
    数据完整性依赖主库binlog，即使主库宕机，也可以把丢失的部分同步到从库上：高可用MHA架构自动抽取缺失部分补全从库；或者5.6的global transaction identitifier特性抽取。
    为保护binlog，MySQL引入sync_binlog参数控制binlog刷新到磁盘（写binlog）的频率。为0时表示不控制，由文件系统自己控制文件的缓存刷新；为n时，表示每n次事务提交，刷新到磁盘
  - 半同步复制
    5.5以前的异步复制，有延迟隐患：从库尚未得到主库binlog，主库宕机且binlog丢失，从库就损失了这个事务，造成主从不一致。
    异步复制，主库执行完提交操作、写入binlog后，即可返回客户端；而半同步复制，必须等待其中一个从库也接收binglog事务并写入中继日志，主库才返回提交操作成功给客户端
    ```sql
    -- 假设是在异步复制的基础上，修改为半同步复制 semi-sync。半同步是基于插件的
    select @@have_dynamic_loading;                                      -- 是否支持动态增加插件
    -- 检查 $MYSQL_HOME/lib/plugin 目录下是否存在插件 semisync_master.so 和 semisync_slave.so。
    install plugin rpl_semi_sync_master soname 'semisync_master.so';    -- 主库安装插件，从库改成slave不赘述
    select * from mysql.plugin;                                         -- 检查是否安装成功
    set global rpl_semi_sync_master_enabled=1;                          -- 开启半同步，从库改成slave。
    set global rpl_semi_sync_master_timeout=30000;    -- 主库在等待这个参数毫秒超时后，自动转为异步复制。在此基础上从库正常连上后，又会自动切换回半同步
    stop slave io_thread; start slave io_thread;                        -- 从异步复制切换过来，需要重启从库io线程。如果是新配置的半同步不需要
    show status like "%semi_sync%";                                     -- 查看状态值
    ```
- 日常管理维护
  - 查看从库状态。`show slave status \G`，关注IO和SQL两个参数，代表相应线程运行状况
  - 主从维护同步
    OLTP这样的系统，主库更新频繁，从库因各种原因导致更新缓慢，使得主从库之间数据差距越来越大，所以需要定期进行主从数据同步。做法是负载较低时阻塞主库更新，强制同步
    ```sql
    flush tabls with read lock;                       -- 主库阻塞更新
    show master status;                               -- 主库日志名和偏移量，是从库复制的目的坐标（同异步复制配置部分）
    select master_pos_waite('msyql-bin.123456','973') -- 从库执行，会阻塞直到从库达到指定日志文件和偏移量后，返回0。返回1表示超时退出。
    unlock tabls;                                     -- 主库解锁
    ```
  - 从库复制出错
    这里书里面似乎写错了。没有处理复制出错的问题，变成跳过来自主库的语句了，使用 `set global sql_slave_skip_count=n`。
  - 查看从库复制进度
    这个进度，帮助我们判断是否需要手工进行主从同步，也可以判断从库中统计数据的覆盖范围。
    可以通过查看 show processlist 的slave_sql_running（SQL线程用于读取中继日志）的 time 值，它记录了从库当前执行SQL的时间戳与系统时间的差距，单位秒。
  - 提高复制性能
    高并发下主库是多线程写入，而从库默认只有一个SQL线程在应用日志，容易出现从库追不上主库。 show slave status 的 seconds_behind_master是预估落后的秒数，不准
    - 方案一：一主多从下，通过设置replicate-do-db、replicate-ignore-db等参数，使每个从库上只同步部分表，减少需要写入的数据。
      优点是自由拆分从库，分散热点数据；缺点是维护起来不够简洁，且一旦主库宕机不好处理，其他从库上没有完整数据。
    - 方案二：5.6提供了基于schema的多线程复制，设置 slave_parallel_workers 参数，启动多个SQL线程。书中建议具体内容查看文档。
- 切换主从库
  假设有个复制环境，主库M，两个从库S1、S2。当M故障时，需要将S1切换成主库，同时将S2的主库修改为S1。
  ```sql
  -- 首先关闭IO线程，确认从库中所有的relay log都执行完成
  stop slave io_thread;
  show processlist;             -- 状态时 has read all relay log 即可。
  -- 将S1切换成主库
  stop slave; reset master;
  -- 修改S2的配置
  stop slave;
  change master to master_host=<S1>;
  start slave;
  ```
  之后，通知客户端程序将应用指向S1，并删除S1的 master.info 和 relay-log.info，否则重启还会按照从库启动。
  （注意：这里默认S1是打开log-bin的，重置成主库后就有binlog；且关闭 log-slave-updates，否则重置成主库后，可能把已执行过的binlog传给S2）
### MySQL Cluster（集群）
集群是一组节点的集合。节点是逻辑概念，一台计算机上可以有一至多个节点。节点功能各不相同，有的存放数据（数据节点），有的存放表结构（SQL节点），有的管理其他节点（管理节点）
MySQL使用NDB引擎来对数据节点的数据进行存储，NDB引擎以前支持基于内存的数据表，5.1开始支持基于磁盘的数据表。MySQL Cluster的SQL节点，可以理解为应用与数据节点的桥梁
启动时，按照管理节点、数据节点、SQL节点的顺序，依次启动服务，然后再使用集群客户端，查看相关信息。
使用时，在任意一个SQL节点中，建表插入数据（具体怎么存储书中没说），在另一个节点中查询时正常。（必须使用NDB引擎）
### 高可用架构
具体安装测试这里不记录，安装成功后内部是故障自动切换的。
- MMM架构
  MMM（Master-Master replication manager for MySQL）是一套支持双主故障切换和双主日常管理的脚本程序。
  名为双主，同一时刻只允许对一台写入，另一台备选主上提供部分读。MMM主要是实现故障切换的功能，另外其内部附加的工具脚本也可实现多slave的读负载均衡。
  MMM适合于对数据一致性要求不高、但想最大程度保证业务可用性的场景。
- MHA架构（书中评价更高）
  MHA（Master High Availability）是比较成熟的高可用方案，且能最大程度上保证数据一致性。
  它由两部分组成，MHA Manager管理节点和MHA Node数据节点。管理节点可以部署在独立机器上或某个slave上，管理主从复制；数据节点运行在每台MySQL服务器上。
  目前MHA主要支持一主多从架构，至少有三台数据库服务器，一主两从，一个master，一个备用master，一个从。淘宝改造的TMHA支持一主一从。
  工作原理：从宕机master保存binglog，识别含有最新更新的slave，应用差异的中继日志到其他slave，应用从master保存的binlog，提升一个slave为新master，其他slave连接新master进行复制。


---

总结：使用MySQL的核心是索引（index only查询）、锁。多使用show variable可以查看参数（@@开头的是系统参数，@开头的是用户定义参数，使用select查看）。优化SQL的步骤。
