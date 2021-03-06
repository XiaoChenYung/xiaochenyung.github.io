---
layout: post
title:  "Sqlite3 DB 文件格式"
date:   2019-08-21 21:45:21 +0800
categories: SwiftUI Swift
---
## 该文档描述和定义了自从 SQLite 3.0.0 （2004-06-18）版本以来的数据库磁盘文件的格式。
# 1、数据库文件
一个完成状态的 sqlite 数据库通常指的是一个我们称之为 "主数据库文件" 的文件。在事务当中，sqlite 会存储一些额外的信息在一个日志文件里，我们称之为 “回滚日志”。如果 sqlite 是 WAL 模式下，那么该额外信息是以预写式日志存在的。

### 1-1、热日志

如果应用程序在一个事务完成之前发生崩溃，那么这个回滚日志文件或者预写式日志就包含了主数据库文件恢复一致性所包含的信息。当回滚日志包含有恢复数据库状态所需要的信息的时候，那么该回滚日志则被称为是热日志。热日志只是数据库错误回复数据方案中的一个因素，它是数据库状态的一部分，但我们还是要把重点放在主数据库文件中。

### 1-2、页数

主数据库文件包含有一个或者多个页，每一页的大小介于 512 到 65536 字节之间，同一个数据库中所有页的大小相同，每一页的大小是由主数据库文件头部偏移 16 字节的两字节大小标识的。数据库中的页从第一页开始，最大页数为 2147483645（2的31次幂减2）即21亿多页，如果每页取最大值 65536 字节，然后页数取最大 2147483645 字节，那么单个数据库文件的理论最大值为 140,737,488,224,256 字节，即 140 tb 左右，当然数据库的大小是受制于磁盘系统的限制的，一般不会使用到这么大的数据库。通常数据库的大小在几 kb 到 几 gb 之间，虽然也有一些生产环境的数据库是在 tb 级的。

在任何情况下，数据库中的每一页都是以下格式中的其中一个

* Btree 页

Btree 页分为 B 树和 B+ 树，每一种树又分为内部页和叶子页。

* 空闲页
* 溢出页
* 锁页

当数据库大于 1G 大小的时候，在偏移为 1GB 的地方会有一个锁页，该页是数据库加锁的区域，不能存储数据。

* 指针位图页

这种页用于 auto-vacuum 数据库中，本文不做讨论

数据库在每一页的读写都是从该页的末尾进行的，读数据库通常是一块读几页，有个例外是当数据库第一次打开的时候，数据库的前 100 个字节会按照一个子页大小的单元来处理。

在修改数据库的任何信息页面之前，该页面的原始未修改的内容都会先写入回滚日志。如果事务被中断需要回滚，则可以使用回滚日志将数据库回滚回之前的状态。为了降低磁盘 IO 操作，空闲叶子页在修改的时候不会将其原始信息写入回滚日志中。

# 2、文件头格式
### 2.1、文件头前 100 字节

db 文件第一页的前一百个字节的格式

| Offset   | Size   | Description   | 
|:----|:----|:----|:----:|
| 0   | 16   | The header string: "SQLite format 3\000"  固定格式的字符串   | 
| 16   | 2   | The database page size in bytes. Must be a power of two between 512 and 32768 inclusive, or the value 1 representing a page size of 65536.  数据库中每一页的大小，如果是1表示一页大小是 65536 个字节   | 
| 18   | 1   | File format write version. 1 for legacy; 2 for [WAL](https://www.sqlite.org/wal.html).  数据库写的格式, 1 是普通模式，2 是 WAL 模式,如果读的标志位是 1 或者 2 但是写标志位大于2，则表示该数据库是只读的，如果读标志位大于2则表示该数据库损坏了   | 
| 19   | 1   | File format read version. 1 for legacy; 2 for [WAL](https://www.sqlite.org/wal.html).  数据库读的格式, 1 是普通模式，2 是 WAL 模式   | 
| 20   | 1   | Bytes of unused "reserved" space at the end of each page. Usually 0.  每一页的保留字节数。sqlite 支持在每一页的末尾区域保留少量的字节区域供扩展使用。比如 sqlite 的加密扩展使用该段来存储一些加密信息，该字节的值就是每一页末尾要保留的供扩展使用的字节数量。数据库中可用的字节大小是页大小减去保留字节数的大小。每一页的可用大小有个限制，即不能少于 480 字节，如果，一个数据库的页大小是 512 字节，那么该保留自己数就不能大于 32   | 
| 21   | 1   | Maximum embedded payload fraction. Must be 64.  内部页一个单元最大使用空间的百分比，即25%  B-tree 内部页一个单元最多使用的空间，该值是固定值不可修改,该值保证了一个内部页至少包含有四个单元。   | 
| 22   | 1   | Minimum embedded payload fraction. Must be 32.  内部页一个单元最小使用空间的百分比，即12.5%   | 
| 23   | 1   | Leaf payload fraction. Must be 32.  叶子页中一个单元使用空间的最小值。默认值为 12. 5% 老版本中这些百分比的数值是可以更改的以修改 B 数算法的存储格式，当前版本已经固定，无法修改。   | 
| 24   | 4   | File change counter.  该值会在每次修改完数据库文件对其进行解锁的时候进行加一的操作。   | 
| 28   | 4   | Size of the database file in pages. The "in-header database size".  数据库有多少页，这样就可以知道整个数据库文件的大小   | 
| 32   | 4   | Page number of the first freelist trunk page.  空闲页链表首指针(页号)   | 
| 36   | 4   | Total number of freelist pages.  空闲页总数量   | 
| 40   | 4   | The schema cookie.  Schema 版本：每次 schema 改变(创建或删除表、索引、视图或触发器等对象，造成 sqlite_master 表被修改)时，此值+ 1   | 
| 44   | 4   | The schema format number. Supported schema formats are 1, 2, 3, and 4.  schema 格式，随着 sqlite 对于一些 sql 语法的支持而增加，比如格式3增加了ALTER TABLE ... ADD COLUMN添加的额外列的功能，使其具有非NULL默认值。在2005-03-11的SQLite版本3.1.4中添加了此功能   | 
| 48   | 4   | Default page cache size.  sqlite 默认高速缓存的大小   | 
| 52   | 4   | The page number of the largest root b-tree page when in auto-vacuum or incremental-vacuum modes, or zero otherwise.  auto-vacuum 类型数据库使用，不做研究   | 
| 56   | 4   | The database text encoding. A value of 1 means UTF-8. A value of 2 means UTF-16le. A value of 3 means UTF-16be.  数据库编码   | 
| 60   | 4   | The "user version" as read and set by the [user_version pragma](https://www.sqlite.org/pragma.html#pragma_user_version).   | 
| 64   | 4   | True (non-zero) for incremental-vacuum mode. False (zero) otherwise.  对于 auto-vacuum 类型数据库中的 incremental-vacuum 模式，该值非0，其他情况下都是0   | 
| 68   | 4   | The "Application ID" set by [PRAGMA application_id](https://www.sqlite.org/pragma.html#pragma_application_id).  应用程序可以自己设定   | 
| 72   | 20   | Reserved for expansion. Must be zero.  扩展字节   | 
| 92   | 4   | The [version-valid-for number](https://www.sqlite.org/fileformat2.html#validfor).   | 
| 96   | 4   | [SQLITE_VERSION_NUMBER](https://www.sqlite.org/c3ref/c_source_id.html)   | 

### 2.2、WAL 模式

SQLite在3.7.0版本引入了WAL ([Write-Ahead-Logging](https://www.sqlite.org/wal.html)),WAL的全称是Write Ahead Logging，它是很多数据库中用于实现原子事务的一种机制,引入WAL机制之前，SQLite使用rollback journal机制实现原子事务。

      rollback journal机制的原理是：在修改数据库文件中的数据之前，先将修改所在分页中的数据备份在另外一个地方，然后才将修改写入到数据库文件中；如果事务失败，则将备份数据拷贝回来，撤销修改；如果事务成功，则删除备份数据，提交修改。

      WAL机制的原理是：修改并不直接写入到数据库文件中，而是写入到另外一个称为WAL的文件中；如果事务失败，WAL中的记录会被忽略，撤销修改；如果事务成功，它将在随后的某个时间被写回到数据库文件中，提交修改。

      同步WAL文件和数据库文件的行为被称为checkpoint（检查点），它由SQLite自动执行，默认是在WAL文件积累到1000页修改的时候；当然，在适当的时候，也可以手动执行checkpoint，SQLite提供了相关的接口。执行checkpoint之后，WAL文件会被清空。

      在读的时候，SQLite将在WAL文件中搜索，找到最后一个写入点，记住它，并忽略在此之后的写入点（这保证了读写和读读可以并行执行）；随后，它确定所要读的数据所在页是否在WAL文件中，如果在，则读WAL文件中的数据，如果不在，则直接读数据库文件中的数据。

      在写的时候，SQLite将之写入到WAL文件中即可，但是必须保证独占写入，因此写写之间不能并行执行。

      WAL在实现的过程中，使用了共享内存技术，因此，所有的读写进程必须在同一个机器上，否则，无法保证数据一致性。

# 3、Btree 页格式
### 3.1 Btree 页分区

| 1、页头   | 
|:----|
| 2、单元指针数组   | 
| 3、未分配空间   | 
| 4、单元内容区   | 

Btree 内部页以单元（cell）来组织数据，一个单元包含一个或者部分 playload（也叫做 Btree 记录）。由于各类数据大小不同，所以每个单元的大小也是可变的，这就要求 Btree 内部页的空间进行动态的分配。

页内所有单元的内容是集中在页的底部，称为单元内容区，有下向上增长。因为单元的大小是可变的，所以就需要保存每个单元的起始指针。单元指针数组包含 0 或多个指针，位于页头之后，单元指针由上往下增长，这样的好处是比较容易增加新的单元，而不需要整理碎片。单元不需要相邻和有序，但是单元指针是相邻和有序的。每个指针占用 2 个字节，表示该单元在单元内容区距离页开始处的偏移。页中单元的数量保存在页头中。

### 3.2 页头格式

| 偏移量   | 大小   | 说明   | 
|:----|:----|:----|
| 0   | 1   | 页类型   | 
| 1   | 2   | 第一个自由块的偏移量   | 
| 3   | 2   | 本页的单元数   | 
| 5   | 2   | 单元内容区的起始地址   | 
| 7   | 1   | 碎片的字节数   | 
| 8   | 4   | 最右儿子的页号   | 

### 3.2.0 页类型

B-tree

![图片](https://uploader.shimo.im/f/wiXNeTaJDmAe21BI.png!thumbnail)

B+tree

内部页：B+tree 的内部页不用来存储数据，只用来存储导航信息，所有叶子页位于同一层级上。，并按照关键字进行排序，所有关键字必须唯一。

![图片](https://uploader.shimo.im/f/L48A19s5tRQtnsYW.png!thumbnail)

0x0D：B+tree 叶子页（存放具体的数据第27页）

0x05：B+tree 内部页（存放数据叶子页的位置,第一页）

0X0A：B-tree 叶子页   (存放索引）

0x02： B-tree 内部页  （存放索引，与B-tree 内部页区别不大）

![图片](https://uploader.shimo.im/f/0U8FmkmooiUf7ctf.png!thumbnail)

### 3.2.1 自由块

由于单元内容区是随机的进行插入和删除单元。将会导致一个页上面单元和内容区之间相互交错。这些没有使用的空间于是就被收集起来形成一个空闲块链表。这些空闲块按照他们的地址升序排列。页头中偏移量为 1 的 2 个字节指向这些空闲块·链表的头指针。每个空闲块至少包含有 4 个字节。空闲块的前四个字节存储控制信息。前两个字节指向下一个空闲块的地址，后两个字节表示该空闲块的大小，再往后则是该空闲块的空间。

### 3.2.2 碎片字节数

由于空闲块大小至少为 4 个字节，那么对于小于 4 个字节的一系列的空间块的总的大小则位于页头偏移量为7的位置上，碎片的总的大小最多为255 个字节，在它达到最大值之前，页会被整理

### 3.2.3 单元内容区地址

单元内容区是页中真正用来存放数据的地方，单元内容区的地址是区分未使用空间和单元内容区的分界线

### 3.2.4 最右儿子的页号

如果本 Btree 页是叶子页，则无此区域，页头长为8 个字节，如果本 Btree 页为内部页，则有该域，页头长 12 个字节，页头偏移为 8 的 4 个字节则包含指向最右儿子的指针。

### 3.2.5 可变长整形

说明：SQLite 中，所有整数都采用大端格式，即高位字节在前。

可变长整形由 1 - 9 个字节组成，每个字节的低七位有效，第八位是标志位，在组成可变长整形的各字节中，前面字节的第八位置 1，最后一个字节的第八位置 0，这样就标识这个整形结束了。

比如 

| 数据   | 二进制   | 按照格式转化后   | 最终结果   | 
|:----|:----|:----|:----|
| 0x81 0x00   | **~~1~~**0000001 **~~0~~**0000000   | 00000000 10000000   | 0x80   | 
| 0x80 0x7f   | **~~1~~**0000000 **~~0~~**1111111   | 00000000 01111111   | 0x7f   | 

### 3.3 sqlite_master 表

sqlite_master 表是一个系统表，保存了数据库的 scheme 信息，sqlite_master 包含五个字段，分别为

| 编号   | 字段   | 说明   | 
|:----|:----|:----|
| 1   | type   | 值为 "table"、"index"、"trigger"、"view"   | 
| 2   | name   | 对象的名称，值为字符串   | 
| 3   | tbl_name   | 如果是表或者视图对象，此字段值与字段2相同。瑞国是索引或者触发器对象，此字段为其关联的表名   | 
| 4   | rootpage   | 对触发器或者视图对象，此字段值为0，对表或者索引对象，此字段值为其根页的编号   | 
| 5   | SQL   | 创建该对象所使用的的 SQL 语句   | 

### 3.3.1 索引

限制数据库查找速度的主要制衡原因一个是查找次数，一个就是 IO 次数。索引是为了进行数据库的快速查找而建立的一个存储结构。如果没有索引，则使用 where 语句有可能需要遍历整个表才能找到对应的对象，创建索引则可以使用二分法查找，极大地提高效率。

这是在一个表中对 fruit 建立索引后快速查找的例子

![图片](https://uploader.shimo.im/f/B4SHFO5VGKkdj5rb.png!thumbnail)


# 4 单元格式
### 4.1 B+tree 叶子页单元格式

单元是变长的字节串，一个单元包含一个或者多个 playload (实际上就是表中的一条记录),B+tree叶子页单元的结构如下

| 大小   | 说明   | 
|:----|:----|
| var(1-9)   | playload 大小，以字节为单位   | 
| var(1-9)   | 数据库记录的 rowId 的值   | 
| *   | playload 的内容，存储着数据库中某个表中一条记录的数据   | 
| 4   | 溢出页链表中第一个溢出页的页号，如果没有溢出页，无此域   | 

### 4.2 playload 格式

| header-size   | Type1   | Type2   | ...    | TypeN   | Data1   | Data2   | DataN   | 
|----|----|----|----|----|----|----|----|
每个 playload 由两部分组成，第一个部分是记录头，由 N+1 个可变长整数组成，N 为记录中的字段数。第一个可变长整数的值是记录头的字节数，跟着的 N 个可变长整数与记录的各个字段一一对应，表示各字段的数据类型和宽度。规则如下：

为了使用一个类型值既表示该数据的类型，又能表示数据的宽度，有以下的约定

| 类型值   | 含义   | 数据宽度（字节数）   | 
|:----|:----|:----|
| 0   | NULL   | 0   | 
| N in 1..4   | 有符号整数   | N   | 
| 5   | 有符号整数   | 6   | 
| 6   | 有符号整数   | 8   | 
| 7   | IEEE浮点数   | 8   | 
| 8-11   | 未使用   | N/A   | 
| N>=12的偶数   | BLOB   | (N-12)/2   | 
| N>=13的奇数   | TEXT   | （N-3)/2   | 

h5 表为例

| 数据 | 二进制 | 按照格式转化后 | 最终结果 | 
|:----:|:----:|:----:|:----:|
| 0x81 0x33 | **~~1~~**0000001 **~~0~~**0110011 | 00000000 10110011 | 0xa3(163) | 
| 0x01 | **~~0~~**0000001 | 00000001 | 0x01 (1) | 
| 0x0c   | **~~0~~**0001100   | 00001101   | 0x0c(13)   | 


























