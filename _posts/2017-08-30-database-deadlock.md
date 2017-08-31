---
layout: post
title: MySQL死锁分析
categories: mysql
description:
keywords: database, deadlock, mysql
---
## 背景 ##
&emsp;&emsp;同事在一个nodejs写的项目中，出现了一个 <strong>在网页中删除一个数据，然后不断刷新，会在某一刻，被删除的数据被恢复回来</strong> 的问题。<br/>
&emsp;&emsp;因为自己对nodejs不敏感，没有先去分析代码，只是根据同事的描述情况，做了分析和验证，下面描述当时自己分析的过程。

## 分析过程 ##
### 1. 首先判断是数据库隔离级别的问题<br/>
&emsp;&emsp;当前的数据库隔离级别为RR,猜测是否因为多个数据库连接之中，1已经提交了事务，但是2仍然处在事务当中，所以2不能够读到1操作中所做的删除，当前台获取的连接为2连接的时候就会产生<strong>恢复</strong>的问题。<br/>
&emsp;&emsp;验证自己的想法，先修改级别：
```
MariaDB [test]> select @@tx_isolation;
+-----------------+
| @@tx_isolation  |
+-----------------+
| REPEATABLE-READ |
+-----------------+
1 row in set (0.00 sec)

MariaDB [test]> set global transaction isolation level Read committed;
Query OK, 0 rows affected (0.00 sec)

MariaDB [test]> exit
Bye

MariaDB [test]> select @@tx_isolation;
+----------------+
| @@tx_isolation |
+----------------+
| READ-COMMITTED |
+----------------+
1 row in set (0.00 sec)
```
&emsp;&emsp;然后重启nodejs程序，重启完成之后，验证，发现问题确实被解决了。

### 2. 进行压测，出现数据库死锁 ###
&emsp;&emsp;建表语句：
```
CREATE TABLE `T_COMP_PARAM` (
  `COMP_ID` varchar(32) DEFAULT NULL,
  `PARAM_NAME` varchar(32) DEFAULT NULL,
  `PARAM_VALUE` varchar(255) DEFAULT NULL,
  `PARAM_TYPE` varchar(32) DEFAULT NULL,
  `VALID` varchar(16) DEFAULT NULL,
  `INSERT_TIME` datetime DEFAULT NULL,
  `UPDATE_TIME` datetime DEFAULT NULL,
  `REMARK` varchar(255) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```
&emsp;&emsp;发生数据库死锁之后，首先查看数据库日志：
```
MariaDB [test]> show engine innodb status \G;
------------------------
LATEST DETECTED DEADLOCK
------------------------
2017-08-17 13:37:20 7fcfe20ff700
*** (1) TRANSACTION:
TRANSACTION 2335476, ACTIVE 1 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s)
MySQL thread id 11768, OS thread handle 0x7fcfe1f38700, query id 1636448 127.0.0.1 r7 updating
update T_COMP_PARAM set PARAM_VALUE = 'database_280', UPDATE_TIME = '2017-08-17 13:37:19' where COMP_ID = 'capture_282' and PARAM_NAME = 'source_db'
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 533 page no 4 n bits 216 index `GEN_CLUST_INDEX` of table `dip`.`T_COMP_PARAM` trx id 2335476 lock_mode X waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 11; compact format; info bits 0
 0: len 6; hex 000000000fd8; asc       ;;
 1: len 6; hex 000000005c5f; asc     \_;;
 2: len 7; hex f2000001ad0270; asc       p;;
 3: len 6; hex 73796e635f31; asc sync_1;;
 4: len 8; hex 7265736572766564; asc reserved;;
 5: len 3; hex 796573; asc yes;;
 6: len 6; hex 4e4f524d414c; asc NORMAL;;
 7: len 3; hex 796573; asc yes;;
 8: len 5; hex 999d50fa18; asc   P  ;;
 9: len 5; hex 999d50fa18; asc   P  ;;
 10: SQL NULL;

*** (2) TRANSACTION:
TRANSACTION 2335459, ACTIVE 2 sec starting index read
mysql tables in use 1, locked 1
20 lock struct(s), heap size 2936, 1429 row lock(s), undo log entries 3
MySQL thread id 11762, OS thread handle 0x7fcfe20ff700, query id 1636453 127.0.0.1 r7 updating
delete from T_COMP_PARAM where COMP_ID = 'apply_287' and PARAM_NAME = 'INCLUDE' and PARAM_TYPE = 'EXTERNAL'
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 533 page no 4 n bits 216 index `GEN_CLUST_INDEX` of table `dip`.`T_COMP_PARAM` trx id 2335459 lock mode S
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
 
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 533 page no 4 n bits 216 index `GEN_CLUST_INDEX` of table `dip`.`T_COMP_PARAM` trx id 2335459 lock_mode X waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 11; compact format; info bits 0
```

&ensp;&ensp;看起来挺乱的，但是可以从中提取几个信息：<br/>
&ensp;&ensp;1. [2]事务持有了一个S锁，在等待一个X锁,SQL语句为：<br/>
```delete from T_COMP_PARAM where COMP_ID = 'apply_287' and PARAM_NAME = 'INCLUDE' and PARAM_TYPE = 'EXTERNAL'```<br/>
&ensp;&ensp;2. [1]事务在等待一个X锁，SQL语句为：<br/>
```update T_COMP_PARAM set PARAM_VALUE = 'database_280', UPDATE_TIME = '2017-08-17 13:37:19' where COMP_ID = 'capture_282' and PARAM_NAME = 'source_db'```<br/>

&ensp;&ensp;首先这两个无论哪一条sql语句都不可能产生共享锁，所以[2]事务得到的共享锁莫名其妙。难道MySQL中有其它产生共享锁的方式了？
自己不断的验证，打开2个命令行不断的尝试，试来试去，只能得到锁超时，根本不可能得到死锁。但是日志说死锁了，一定是死锁了。问题出在了哪里？
万般无奈，最后只好叫着同事一起<strong>看！代！码！</strong>

&ensp;&ensp;能够产生共享锁，应该是查询相关的操作，所以自己就找找执行delete之前有什么查询：
```
"DELETE FROM " + hdrcfg.cfg.table_name.T_COMP_DEPEND_SETS + " WHERE ID IN" +
"(SELECT b.PARAM_VALUE FROM " + hdrcfg.cfg.table_name.T_COMP_INFO + " a, " + hdrcfg.cfg.table_name.T_COMP_PARAM + " b " +
"WHERE a.GROUP_ID = ? AND a.ID = b.COMP_ID AND b.PARAM_TYPE = 'EXTERNAL')",
```
&ensp;&ensp;怀疑是不是这种子查询得到了共享锁，于是自己赶紧再命令行中验证，确实，经过这种操作之后确实能够得到死锁。


### 3. 静下心来仔细分析，再找到疑点 ###
&emsp;&emsp;仔细回想自己当时的逻辑，虽然情况是这样发生的，但是怎么可能产生这么长时间的事务？并且为什么这个事务还能执行其它的工作呢？<br/>
&emsp;&emsp;首先看删除的逻辑：
```
   let DOJob=async function () {
        try{
            await hdrcom.pub.checkMd5(body);
            db =await hdrcom.db.openDb();
            console.info("[delete_project], conn db ok.");
            //这里发现关闭了自动提交功能
            await hdrcom.pub.setAutoCommit(db);
            await hdrcom.db.dbTransaction(db);
            await getAllGroups();
            await checkGroupState();
            await deleComp();
            await deleGroup();
            await deleProject();
            await hdrcom.db.dbCommit(db);
            delDir();
            hdrcom.pub.processResult(res,"SUCCESS", true, body);
            console.info("end delete_project database.\n");
            return 'SUCCESS';
        }catch(err){
            db && await hdrcom.db.dbRollback(db).catch(err=>{
                console.error(err);
            });
            hdrcom.pub.processResult(res, err, false, body);
            return err;
        }finally {
            db && hdrcom.db.closeDB(db);
        }
    };
```
&emsp;&emsp;原来问题是经过删除操作的所有连接，都被关闭了自动提交，那么这个连接再之后产生的操作都会汇聚到一个事务里，占有的锁也不会被释放，
这时候如果有一个新事务正好执行卡在这个事务的中间了，那一定会产生死锁。<br/>
&emsp;&emsp;还原数据库隔离级别，在commit时候加上开启自动提交，进行压测，问题已经被解决。