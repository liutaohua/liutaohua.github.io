---
layout: post
title: mysql binlog日志存储格式
categories: mysql
description:
keywords: mysql, binlog
---
# 获取日志事件 #
&emsp;&emsp;MySQL对于binlog的处理是以事件为单位的，每一次DML操作可能会产生多次事件，例如对于 innodb 存储引擎，会额外产生一条 query_event（事务的begin语句）和 xid_event（事务提交）。
那么我们如何获取到这些事件呢?

## 1. 发送DUMP包 ##
&emsp;&emsp;首先，我们知道 MySQL 本身就带有 replication 的机制，我们需要伪造一个 slave，向 master 注册，这样的话 master 才会发送 binlog event。
注册很简单，通过调用 limysql.so 中的 cli_advanced_command（），指定 binlog filename + position，向 master 发送 COM_BINLOG_DUMP 命令。
在发送 dump 命令的时候，我们可以指定 flag 为 BINLOG_DUMP_NON_BLOCK，这样 master 在没有可以发送的 binlog event 之后，就会返回一个 EOF 的包。

数据包的具体格式如下：<br/>
![dump-package](/images/mysql/dump-package.png)

例如 COM_BINLOG_DUMP 类型的数据包 payload（数据包体）是这样的：
![dump-package-payload](/images/mysql/dump-package-payload.png)

## 2. 接收事件 ##
&emsp;&emsp;通过调用 libmysql.so 库中的 cli_safe_read（）函数，获得 master 发过来的数据，每次获得一个事件记录的数据，cli_safe_read（）的返回值标示了从 master 发送过来的数据的数据字节数。
而发送过来的数据保存在 mysql->net->read_pos 数组中。
 
#### Binlog 事件 ####

&emsp;&emsp;通过调用 libmysql.so 库中的 cli_safe_read（）函数可以获取一次 binlog 事件。
MySQL 的 Binlog 事件类型有27种，在 MySQL 5.6之后增加到 38 种，但是我们只介绍与 ROW 模式相关的事件，所有的 event 都含有如下通用的事件结构：
```
+===============================+
|     event header   |
+===============================+
|   event data     |
+===============================+
```
&emsp;&emsp;分别为事件头和事件体组成，而事件的内部结构随MySQL版本的不同而变化着，我们需要用到的版本为 v4，用于 mysql5.1 及以上，其他版本就不做介绍了，想要了解的朋友可以参考官方文档。
下图为v4版本的 event header 格式。
![event-header-v4](/images/mysql/event-header-v4.png)

&emsp;&emsp;由于各个事件的事件头一致，这里我们就不重复介绍了，后面各个事件我们也将忽略对事件头字段的描述。
 
#### ROTATE_EVENT ####
ROTATE_EVENT,记录了接下来要请求的 binlog 的信息。格式如下：
![rotate-event](/images/mysql/rotate-event.png)

&emsp;&emsp;它里面其实就是标明下一个 event 所在的 binlog filename 和 position。
这里需要注意的是，当 slave 发送 binlog dump 之后，master会首先发送一个 ROTATE_EVENT，用来告知 slave 下一个 event 所在的位置，然后才跟着其他事件。

#### QUERY_EVENT ####

&emsp;&emsp;QUERY_EVENT, 存储的是SQL，主要是一些与数据无关的操作，如 begin，alter table，drop table, create table 等。格式如下<br/>
![query_event](/images/mysql/query_event.png)

#### TABLE_MAP_EVENT ####

&emsp;&emsp;TABLE_MAP_EVENT,记录了某个 table 所对应的表信息，在其中存储了数据库名和表名等。格式如下：<br/>
![table-map-event.png](/images/mysql/table-map-event.png)
 
&emsp;&emsp;在这个事件中我们需要注意的是，虽然我们可以用表的 id 标识符 table_id 来代表一个特定的表，但是因为 alter table 或 rotate binlog event等原因，master 会改变某个 table 的 table_id，
所以我们在外部不能使用这个 table_id 来索引某个 table。<br/>
&emsp;&emsp;还需要关注的就是里面的 column meta 信息，后续我们解析 ROW_EVENT 的时候会根据这个来处理不同类型的数据。

#### ROWS_EVENT ####

&emsp;&emsp;ROWS_EVENT,包含了 insert，update，以及 delete 三种 event ，并且有 v0，v1，v2 三个版本。
ROWS_EVENT 的基本格式如下：<br/>
![rows_event](/images/mysql/rows_event.png)

&emsp;&emsp;在这里 MySQL5.6 开始 event 的 type 发生变化了，所以 MySQL5.6 需要改一下，
所以为了我们软件的版本兼容性，我们需要在软件中支持不同的版本。
``` 
MySQL 5.1 ~ MySQL 5.6
	TABLE_MAP_EVENT: 19
	WRITE_ROWS_EVENT : 23
	UPDATE_ROWS_EVENT : 24
	DELETE_ROWS_EVENT :  25
MySQL 5.6 以上
	TABLE_MAP_EVENT: 19
	WRITE_ROWS_EVENT : 30
	UPDATE_ROWS_EVENT : 31
	DELETE_ROWS_EVENT :  32
```

&emsp;&emsp;ROWS_EVENT 的 table_id 跟 TABLE_MAP_EVENT 一样，虽然 table_id 可能变化，但是 ROWS_EVENT 和 TABLE_MAP_EVENT 的 table_id 是能保证一致的，所以我们也是通过这个来找到 TABLE_MAP_EVENT 的。<br/>
&emsp;&emsp;为了节省空间，ROWS_EVENT 里面对于各列状态都是采用 bitmap 的方式来处理的。首先我们需要得到 column present bitmap 的数据，
这个值用来表示当前列的一些状态，如果没有设置，也就是某列对应的 bit 为0，表名该 ROWS_EVENT 里该列的数据，外部用 null 代替就可以了。<br/>
&emsp;&emsp;然后就是每个 record 里的 bitmap，这个是用来表明一行实际的数据里面有哪些列是 NULL 的，如果该列为 NULL，则为 1。<br/>
&emsp;&emsp;在得到 column_bitmap 和 null_bitmap 后，我们就可以实际解析这行对应的数据了，对于每一列，首先判断是否 column_bitmap 标记了，如果为 0，这跳过用 null 表示，
然后再看是否 null_bitmap 里面标记了，如果为 1，则表明为 null。<br/>
&emsp;&emsp;但是，因为我们得到的是一行数据的二进制流，又如何将一列数据解析出来呢，这里就需要靠 TABLE_MAP_EVENT 里的 column_meta 了。
Column_def 中定义了该列的数据类型，对于一些特定的类型，譬如 MYSQL_TYPE_LONG，MYSQL_TYPE_TINY 等，长度都是固定的，所以我们可以直接读取对应长度的数据得到实际的值，
但对于一些类型，则没有这么简单。这时候就需要通过meta来辅助计算了。<br/>
&emsp;&emsp;举个例子：<br/>
&emsp;&emsp;MYSQL_TYPE_BLOB 类型，meta为 1 表示的为 tinyblob，第一个字节解释 blob 的长度，2 表名的是 shortblob，前两个字节为 blob 的长度等，而对于 MYSQL_TYPE_VARCHAR，meta 存储的则是 string 的长度，下一小节我们介绍每一种类型的计算方法。

#### XID_EVENT ####

&emsp;&emsp;XID_EVENT 一般出现在一个事务操作之后或者其他语句提交之后。它的主要作用提交事务操作和把事件刷新至 binlog 文件中。里面存的是 8 个字节的事务 ID 号。
 
## 3. 不同的数据类型的长度 ##

&emsp;&emsp;上一节我们介绍了在不同事件的事件格式以及需要注意的事项，在其中 ROWS_EVENT 事件中，在 record 中 bit_map 之后的列数据中，针对不同的数据类型，可能在 record 中占用不同的字节，因从需要针对每种数据类型进行处理，
下面列出大部分常用数据类型的字节数和解析方法：
#### MYSQL_TYPE_INT 家族 ####
![mysql_type_int](/images/mysql/mysql_type_int.png)

#### MYSQL_TYPE_DOUBLE 家族 ####
![mysql_type_double](/images/mysql/mysql_type_double.png)


#### MYSQL_TYPE_NEWDECIMAL 类型 ####
声明语法为 decimal(M,D),decimal 参量的取值范围如下：

* M 为数字的最大数(精度),其范围为 1 ~ 65，默认为 10
* D 是小数点后面的数目(标度)。范围为 0 ~ 30，但不得超过 M
* table_map_event 中的 metadata 为 2 个字节，它会记录该类型的精度和标度，其中第一个字节表示精度，第二个字节表示精度。那么数据的长度是如何计算的呢？
* 比如说 decimal(15, 5) a = 1234567890.12345，那么 a 的精度为 15，标度为 5，所以 a 的整数位最多为 10，mysql 中每 9 个数字用一个 32 位 4 个字节的整形来存储，剩下不够 9 个字节的按照如下方式：
* int dig2byte[10] = { 0, 1, 1, 2, 3, 3, 3, 4, 4, 4 };
* 该数组表示剩下几个字节的数字用几个字节存储。为什么这样做呢，我们知道一个 int 型变量能存储的最大整数位 2^32，是一个十位数的整数，所以最多可以表示9位数的数字。
binlog 中的 decimal 按照小数点前后分别存储，所以a的整数位需要2个 int 型来存储，小数后的计算方式也一样，需要3个字节存储，所以 decimal(15, 5)在 binlog 中需要 11 个字节来存储数据


#### MYSQL_TYPE_STRING ####
![MYSQL_TYPE_STRING](/images/mysql/MYSQL_TYPE_STRING.png)

&emsp;&emsp;处理该类型数据时，需要利用 metadata 辅助计算，metadata 的前一个字节表示类型，后一个字节表示长度，如果类型为 MYSQL_TYPE_SET 和 MYSQL_TYPE_EUM 时，数据的长度则为后一个字节的大小。<br/>
&emsp;&emsp;如果类型为 MYSQL_TYPE_STRING，第一个字节的数字即为该类型数据的长度。
 
#### MYSQL_TYPE_BIT ####
![MYSQL_TYPE_BIT](/images/mysql/MYSQL_TYPE_BIT.png)

#### MYSQL_TYPE_DATA 家族 ####
![MYSQL_TYPE_DATA](/images/mysql/MYSQL_TYPE_DATA.png)

#### MYSQL_TYPE_BLOB 家族 ####
![MYSQL_TYPE_BLOB](/images/mysql/MYSQL_TYPE_BLOB.png)

&emsp;&emsp;Metadata 为 1，第一个字节表示长度，metadata 为 2，前两个字节表示长度，metadata 为 3，前三个字节表示长度，metadata 为 4，前四个字节表示长度。

#### MYSQL_TYPE_VARCHAR ####
![MYSQL_TYPE_VARCHAR](/images/mysql/MYSQL_TYPE_VARCHAR.png)

&emsp;&emsp;如果 metadata 小于 256，则第一个字节表示数据的长度，如果 metadata 大于 256 则前两个字节表示数据的长度。
