### innodb 关键特性
* **Buffer Pool**
  - 缓冲池是mysql中一大块的内存区域，主要用来缓存各种数据，以提高性能，要为加载的数据分配页，有的页刷到磁盘可以释放，对于这么大的一片内存区域如何管理分配和释放页呢。mysql 中使用改造之后的LRU算法来管理这些页，这个LRU算法中有一个midpoint，热点数据存在new区中，不活跃的数据存在在old区中。新分配的页就放在old中，只要当在一定时间内被再次访问（是因为查询而不是数据库的预读），就会被加入到new中，这个时间可以通过全局的时间来控制
* **insert buffer**
  - 插入缓冲是提高插入性能的一种措施。数据插入的时候数据是按照主键顺序存放的，也就是存在一级索引里，数据是按照一级索引的顺序来存储的。在大多数情况下主键都是自增长的，插入的过程其实就是个顺序写，性能没有什么问题。但是对于二级索引来说，写入往往都是随机的，经常遇到要写的页不在缓冲池中，这是就要缺页中断，从磁盘调入页进内存，然后在写，这样一来就很慢了。为了解决二级索引写入慢的问题，就有了insert buffer，insert buffer 自身也是个b+ 树，当写二级索引的时候如果页不在缓冲池中，那么就直接写insert buffer，这样就快的很多了。但是这只是个临时的解决办法，最终还是要把insert buffer中的数据写入到页里面。
  - insert buffer bitmap 记录了二级索引页空间占用情况，以及哪些页的记录存在于insert buffer 中。当因为一些外部的原因导致页被加载到了缓冲池中（比如一个select），而恰好bitmap中记录了这个页有数据在insert buffer里面，那么就可以合并了。如果bitmap里面发现整体二级页空闲空间不多了，也会强制的加载页到内存进行合并。还有就是master线程在每秒或每10秒进行的合并。
  - insert buffer 不适用于唯一性索引，因为为了去检查是否违反唯一性约束，有可能要去检查和之前的记录是否重复，有可能要去磁盘上加载页，这样就违背了insert buffer 的设计初衷。
  - insert buffer 为什么加快了二级索引的插入速度，就是因为他减少了磁盘IO的操作次数。
* **doublewrite**
  - 部分写失效问题（partial page write）。innodb中的page大小是16k，而磁盘写入是512byte为单位的，也就是说512 bytes 可以原子写入，但是16k的page 无法保证原子写入，因此crash的时候有可能16k只写入一半。
  - 主要解决的是刷新脏页到磁盘过程中，如果os或者存储系统或者进程 crash 之后，double write 可以在后续的恢复数据过程中提供一定的保障能力, 假如说double write 写成功了，但是刷到各个表空间磁盘的时候crash了，在系统的恢复过程中可以使用double write备份的页，然后再结合redolog就可以完成恢复了。脏页不是直接被刷到磁盘中，先把准备flush到磁盘的脏页先写入到 double write buffer，然后立即sync到磁盘。这样就保证了也谢脏页在磁盘上有了一份备份了，然后在把double write buffer 中的数据flush到磁盘。其中double write buffer 写到磁盘是顺序写到系统表空间中，第二次write到磁盘是写到各个表空间中，是个随机写过程。
  - 疑问是：如果double write 写的过程中也没有成功，也就是说连备份都没有成功，那么岂不是还是白搭？
  - double write 只是面对部分写失效问题的被动解决方案，不是什么优点
  - 当double write 写成功后，页写入磁盘发生crash的情形，double write 方案才会起作用，可以利用double write 恢复 page
  - 当double write 自身也没有写成功，那么恢复的时候不会用double write，这个时候会用 事务日志(这个事务日志是什么？) 来恢复double write，然后才能用redo log 来恢复page, 有 double write 后，恢复时就简单多了，首先检查数据页，如果损坏，则尝试从 double write 中恢复数据；然后。检查 double writer 的数据的完整性，如果不完整直接丢弃，重新执行 redo log；如果 double write 的数据是完整的，用 double buffer 的数据更新该数据页，跳过该 redo log。
  - 疑问： 磁盘扇区是512bytes，传输也是以扇区为单位，但为什么可以保证扇区的写入是原子呢？会不会一个扇区只写入了一半？
    http://jm.taobao.org/2013/07/09/2923/ 这篇文章解释的比较清楚，磁盘的磁头其实一次只能写一个bit,所以 512byte 也是一位一位写的，完全有可能写一半的情况。但是磁盘每次都是在写完了一个512byte后给操作系统应答说写成功，写一半的情况就是没有写成功，写一次系统启动会把这些写一半的块清除掉，就相当于没有写。也就是说磁盘回答说这个扇区写成功了就一定是写到了物理介质上了。

* **redo总结**
  - redo 的主要功能是什么？
    - redo 是当数据库宕机的时候用来恢复数据库的。无论是否是crash，每次启动数据库的时候都会进行修复，当然根据LSN来判断如果没有什么要修复的就跳过去了
  - redo 是什么时候产生的？
    - 进行数据库修改时产生redo
  - redo 什么时候提交的？
    - 每秒钟都会提交（master线程），也就是说redo的每时每刻都在提交而不是等到最后一次性提交，这就解释了为什么一个事务里可能操作了很多的数据，但是事务commit的时候很快。
  - redo 和事务的关系
    - redo 保证了事务的原子性和持久性。redo log 使用 force log at commit 技术给我们保证了：如果事务提交成功了，那么数据一定是持久化到了数据库中了。当事务提交的时候首先写redolog 到磁盘，然后fsync强制刷盘，然后才返回事务提交成功的状态。这里面有个参数来控制：innodb_flush_log_at_tx_commit =1 来保证强制刷盘，如果改为了0或者2就不能保证了
  - redo log 的整体架构是怎样的？
    - redo log 分为redo log buffer 和redo log 文件，redo log buffer 会不断的刷新到文件中去，文件是按照组来管理的，可以有多个文件组，默认是一个文件组，里面有两个文件，在数据库data目录的 ib_logfile0、ib_logfile1 两个文件，两个文件是循环写的，如果文件写满了会触发checkpoint，因为马上一部分的redolog 就要被覆盖了，所以必须要保证对应这部分脏页要写到硬盘里。redo log 写到硬盘里的方式时以512byte为单位的，而不是像数据页、索引页等都是16k都基本单位，这样就保证了redolog写入过程是原子性的，不会出现部分写失效问题（这是硬盘给我们的保证），因此这个过程不需要double write 
  - redo log 的文件结构是怎样的？
    - redolog记录的是对页的操作，例如对 page(2,3) offset=10 setvalue 11 ，一个redo块是512byte，去掉头尾所占的20个字节，一个日志块可用的空间是492byte，relog 每个日志组中的第一个文件前2k字节不是用来存日志块的，是用来存放log file header 、checkpoint 信息的，其余的日志文件前2k是空着的
    - 因此redolog日志的写入也不是完全顺序写的，因为虽然是日志块是顺序写入的，但是每次还得去更新前面2k的信息。

  - redo 是如何进行数据库恢复的？
    - 使用LSN 进行判断恢复，通过比较页的LSN和redolog的LSN来确定要重做的日志范围，从而恢复数据
    - 页的LSN、redolog的LSN都存在哪里？
* **undo**
  - 为什么undo可以实现MVCC？
    - 因为undo可以恢复之前版本的数据，因此可以实现多版本的并发控制


* **好的资料**
  - [阿里云mysql专家总结了innodb的基础知识故障恢复,这个家伙的关于innodb的文章非常不错](https://www.cnblogs.com/coderyuhui/p/7191413.html)
  - [这篇文章解释了如果doublewrite也没有写完整就不直接丢弃直接执行redolog](https://jin-yang.github.io/post/mysql-innodb-double-write-buffer.html)
  
