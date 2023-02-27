# 说过的话一定要做到——Redo日志

> **redo 日志是为了在系统因崩溃而重启时恢复崩溃前的状态而提出的**

## redo日志是什么

​	对于一个已经提交的事务，在提交以后即使系统发生了崩溃，这个事务对数据库所作的更改也不能丢失，如果我们只在内存的Buffer Pool中修改了页面，假设在事务提交后突然发生了某个故障，导致内存中的数据都失效了，那么这个已经提交的事务在数据库中所作的更改也就跟着丢失了。

​	因此我们需要保证这个数据的持久性存储，在事务提交完成之前通过写入redo日志来保证，这样即使之后系统崩溃了，重启以后只要按照redo日志中的相关内容所记录的步骤更新一下数据页，那么之前的事务对数据库中所做的修改就可以被恢复出来。

​	相较于在事务提交时将所有修改过的内存中的页面刷新到磁盘中，只将该事务执行过程中产生的redo日志刷新到磁盘具有以下的好处：

		- redo日志占用空间非常小
		- redo地址是按顺序写入磁盘中的，执行事务过程中，每执行一条一条语句，就可能产生若干条redo日志，按照产生的顺序写入磁盘

## redo日志格式

![](D:\Program Files\电子书\go\md\图片\image-20221230160018242.png)

## redo日志类型

	### 简单日志类型

![](D:\Program Files\电子书\go\md\图片\image-20221230160225391.png)

### 复杂日志类型

有时候，一条INSERT语句除了向B+树的页面中插入数据外，还需要考虑页分裂和向页面头部信息写入相关信息等过程，因此一条INSERT语句可能包括多条redo日志

![](D:\Program Files\电子书\go\md\图片\image-20221230160410004.png)

![](D:\Program Files\电子书\go\md\图片\image-20221230160415773.png)

- 从物理层面看，这些日志都指明了对哪个表空间的页进行修改
- 从逻辑层面看，系统崩溃后重启时，需要调用一些事先准备好的函数，在执行完这些函数后才可以将页面恢复成系统崩溃前的样子

## Mini-Transcation

### 以组的形式写入redo日志

语句在执行过程中可能会修改若干个页面，修改完以后需要记录相应的redo日志，在执行语句中产生的redo日志，被人为划分成了若干个不可分割的组，且规定向某个索引对应的B+树种插入一条记录的过程必须是原子的

![](D:\Program Files\电子书\go\md\图片\image-20221230160846448.png)

- 乐观插入：数据页剩余空间充足
- 悲观插入：插入数据的页面已满，需要进行页分裂，需要对多个页面进行修改，会产生多条redo日志

### Mini-Transcation概念

- 对底层页面进行一次原子访问的过程称为一个Mini-Transcation（**MTR**）

- 一个MTR可以对应一组redo日志，进行崩溃恢复时，需要把这一组redo日志作为一个不可分割的整体来处理

- 一个事务包含若干条语句，每条语句包含若干个MTR，每个MTR又包含若干条redo日志

  ![](D:\Program Files\电子书\go\md\图片\image-20221230161317366.png)

## redo日志的写入过程

### redo log block

为了更好地管理redo日志，设计InnoDB的大叔把通过MTR生成的redo日志都放在了大小为512字节的block中，一个redo log block如图所示

<img src="D:\Program Files\电子书\go\md\图片\image-20221230165126773.png" style="zoom: 67%;" />

真正的redo日志都是存储到占用496字节的log block body中，header和trailer存储一些管理信息

### redo地址缓冲区

写入redo日志时页不能直接写入磁盘中，而是在服务器启动时就向操作系统申请了一大片称为**`redo log buffer`**，这片内存空间被划分为若干个连续的redo log block，如图所示

<img src="D:\Program Files\电子书\go\md\图片\image-20221230165408947.png" style="zoom:67%;" />

可以通过启动选项`innodb_log_buffer_size`来指定`log buffer`的大小，默认值为16MB

### redo日志写入log buffer

​	一个全局变量`buf_free`，它指明了当前需要写入的redo日志应该写入到log buffer中的第几个字节去（偏移量），**一个MTR执行过程中产生若干条redo日志，这些redo日志是一个不可分割的组，所以将每个MTR运行过产生的日志先暂时存到一个地方；当该MTR结束的时候，再将过程中产生的一组redo日志全部复制到`log buffer`中**

<img src="D:\Program Files\电子书\go\md\图片\image-20221230165612745.png" style="zoom:67%;" />

## redo日志文件

### redo日志刷盘时机

在log buffer中的redo日志在某些情况下会被刷新到磁盘中，例如：

	- log buffer空间不足时
	- 事务提交时
	- 后台有一个线程以1s1次的频率将log buffer中的redo日志刷新到磁盘
	- 正常关闭服务器时

### redo日志文件组

​	MySQL的数据目录中默认有名为ib_logfile0和ib_logfile1的两个redo日志文件，logbuffer中的日志默认情况下刷新到这两个磁盘文件中，可以通过启动选项来调节相关的日志文件

		- `innodb_log_group_home_dir`：指定redo日志文件所在目录，默认为数据目录
		- `innodb_log_file_size`：指定了磁盘中每个redo日志文件的大小，默认为48MB
		- `innodb_log_files_in_group`：指定了redo日志文件的个数，默认为2，最大为100

![](D:\Program Files\电子书\go\md\图片\image-20221230170529522.png)

### redo日志文件格式

​	redo日志文件也是和log buffer一样由若干个512字节大小的block组成

​	在redo日志文件组中，每个文件的大小都一样，都由两部分组成：

  - 前2048字节（前4个block）用来存储一些管理信息

    ![](D:\Program Files\电子书\go\md\图片\image-20221230170808539.png)

    ![](D:\Program Files\电子书\go\md\图片\image-20221230170837701.png)

  - 从第2048字节往后的字节用来存储log buffer中的block镜像

    <img src="D:\Program Files\电子书\go\md\图片\image-20221230170740415.png" style="zoom: 67%;" />

### log sequence number

​	全局变量`lsn`：用来记录当前总共已经写入的redo日志量，起始值为8704

​	全局变量`flushed_to_disk_lsn`：用来标记当前log buffer中已经有哪些日志被刷新到磁盘中了