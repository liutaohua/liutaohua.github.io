---
layout: post
title: oracle压缩表
categories: oracle
description:
keywords: oracle, compress
---
&emsp;&emsp;oracle 提供压缩表技术的目的是为了节省磁盘空间，减少 data buffer cache 的空间使用量，从而在涉及大量数据查询时，减少io的读取量，提高查询效率。
压缩的过程对上层应用是透明的，只需要在新建表时指定 compress 属性，或者对已经存在的表执行 alter table 操作。数据库会在后台将数据按照压缩后的格式进行存储。压缩不会影响 redo 和 undo 日志的记录格式。

oracle 的表压缩有3中方法
### 1. Basic compress ###
   basic compress 从 oracle 9i 就已经引入，对应普通的dml操作，数据不会被压缩，只有通过 direct path 进行装载时，数据才会被压缩。
#### 启用表压缩 ####
```sql
	--新建压缩表
	create table test_compress 
	(
	  id int, 
	  name varchar(40)
	) compress
	--或
	create table test_compress compress 
	as select * from user_tables where rownum < 1


	--修改表为压缩表
	alter table test_compress compress 

	--对分区表的某个分区启用压缩
	alter table test_compress  modify partition part_1 compress
```
####  取消表压缩 ####
```sql
alter table test_compress nocompress
```
&emsp;&emsp;当对表取消压缩后，表中之前已经压缩的数据还将处于压缩状态。只是后进入的数据不进行压缩。调用 ```alter table test_compress move```, 表将按照现有的属性重新组织。

#### 压缩原理 ####

&emsp;&emsp;表压缩的原理是数据中会存在大量的重复列值，如果把这些重复的列值用一个简称进行代替，数据行内存储这些简称就会大大的减少存储空间，oracle就是通过这种重新组织数据的方式实现压缩的。
basic compress 的压缩单位是 block ，把一个 block 上的数据分析重复值，然后建立重复数据与简称的映射关系，将这个映射关系存储在 block 头后，然后再每一行数据重复的值替换成其简称。

例如：下图描述的表accounts 和数据<br/>
![accounts](/images/oracle/accounts-data.png)
		  
假设上面的这些记录在数据库中都存储在同一个 block 中，那么 block 内的存储结构类似于下图所示。<br/>
![block](/images/oracle/block-struct.png)
		
&emsp;&emsp;first_name和last_name列有很多重复的数据，所以可以进行压缩，将重复的值建立简称，最后压缩后的结构类似于<br/>
![zip-duplate](/images/oracle/zip-duplate.png)
	  
以上例子只是从原理上对表压缩的描述，oracle 真正的存储要比例子中所描述的复杂和聪明的多，它会更大的减少空间的使用。

### 2. compress for oltp ###
  &emsp;&emsp;该压缩方式在oracle 11g中引入， 但是需要拥有 Advanced Compression Option 的 license 才可以使用，和 basic compress 压缩方式的不同在于，执行普通的dml产生的数据也会按照压缩的格式进行存储。
  但是压缩的时间不是在 dml 执行后马上对数据进行压缩，而是在满足一定条件后，触发进行 block 的压缩，具体的触发条件还不清楚，但是当新 insert 记录到一个 block，如果 block 的剩余空间小于记录的大小时，会触发对该 block 已存在数据的压缩。
  
  <strong>SQL语句</strong>
```sql
	--创建表
	create table test_compress
	(id int,
	name varchar(50)
	) compress for oltp

	--或
	create table test_compress compress for oltp
	as select * from user_tables where rownum < 1

	--修改表
	alter table test_compress compress for oltp
```

### 3、Hybrid Columnar Compression ###
&emsp;&emsp;该压缩方式只有在 oracle exadata v2  11g release 2 的版本上才能够支持，它的原理同样是为重复列建立简称，然后重新组织行数据，实现压缩。
但是它打破了block的限制，上面说的 basic 和 oltp 压缩都是在一个 block 内建立重复列的简称,而 hybrid columnar compression 则采用单独的 block 存储表中所有列的简称，相当于表中的全部数据块都统一用一份简称列表。

例如：下图是表中的两个数据块，没有进行压缩的存储方式
![data-block](/images/oracle/data-block.png)

&emsp;&emsp;经过 hybrid columnar compression 压缩后，数据会按照下图进行组织，所有的列值及简称会组成 compression unit,实际的数据行（data block）通过简称引用 compression unit 中的列值。

hybrid columnar compression 有两种应用场景

1. Warehouse compression<br/>
主要用于查询比较频繁的应用场景<br/>
<strong>sql语句</strong>	
```sql
	 --创建表
	create table test_compress compress for query as select * from user_tables;
```

1. Archive compression<br/>
主要用于数据归档，查询较少的应用场景，对cpu的占用较高<br/>
<strong>sql语句</strong> 
```sql
	---创建表
	create table test_compress compress for archive as select * from user_tables;
```