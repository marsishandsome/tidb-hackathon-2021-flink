# Batch 实现思路

### 流程图

![img](./images/pic1.png)  

### 原理

##### Flink 表<br/>
&ensp;&ensp;&ensp;&ensp;1.Hive 表<br/> 
&ensp;&ensp;&ensp;&ensp;2.TIDB 映射表<br/>

##### JOIN SQL（以 insert into … select … from xxx join xxx 为例）<br/>
&ensp;&ensp;&ensp;&ensp;1.Flink 通过 TiBigData 查询 TIDB 表<br/> 
&ensp;&ensp;&ensp;&ensp;2.Flink 直接查询 Hive 表<br/> 
&ensp;&ensp;&ensp;&ensp;3.结果做 join，再 insert 到 TIDB 表<br/> 
<br/><br/><br/>
# TiBigData
Flink 与 TIDB 交互，可以借助 TiBigData 项目，通过测试 TiBigData， 已有功能基本满足了我们 hack 题目的需求。
<br/>
### 基本功能
##### TIDB 表
1.建表<br/>
CREATE TABLE hack_test.tidb_stu (<br/>
 `id` int(11) DEFAULT NULL,<br/>
 `name` varchar(255) DEFAULT NULL<br/>
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;<br/>
<br/>
2.插入数据<br/>
insert into tidb_stu values(1, 'zhangsan');
<br/>
insert into tidb_stu values(2, 'lisi');
<br/><br/><br/>
##### Flink 映射表

1.建库<br/>
create database `hive`.`hack_test`;
<br/>
![img](./images/pic2.png) 

 

2.删库<br/>
drop database `hive`.`hack_test`;
<br/>
![img](./images/pic3.png) 

 

3.建映射表（properties 里制定了 TIDB 表相关的配置）<br/>
CREATE TABLE `hive`.`hack_test`.`tidb_stu`(
<br/>
id int,
<br/>
name string
<br/>
) WITH (
<br/>
  'connector' = 'tidb',
<br/>
  'tidb.database.url' = 'jdbc:mysql://${ip}:${port}/flink',
<br/>
  'tidb.username' = '${user}',
<br/>
  'tidb.password' = '${pwd}',
<br/>
  'tidb.database.name' = 'hack_test',
<br/>
  'tidb.maximum.pool.size' = '10',
<br/>
  'tidb.minimum.idle.size' = '0',
<br/>
  'tidb.table.name' = 'tidb_stu',
<br/>
  'tidb.write_mode' = 'upsert',
<br/>
  'sink.buffer-flush.max-rows' = '0'
<br/>
 );
<br/>
![img](./images/pic4.png) 

 

4.删除 TIDB 映射表（TIDB 数据库里的表不受影响）<br/>
drop table  `hive`.`hack_test`.`tidb_stu`
<br/>
![img](./images/pic5.png) 

 

5.查询 TIDB 映射表（最终查询的是 TIDB 数据库里的表)<br/>
select * from `hive`.`hack_test`.`tidb_stu`;
<br/>
![img](./images/pic6.png) 
<br/><br/><br/>
##### Hive 表

1.建表 和 准备数据<br/>
create table hive_city(id int, city string) row format delimited fields terminated by ',' stored as TEXTFILE;


2.查询 HIVE 表<br/>
select * from `hive`.`hack_test`.`hive_city`;
<br/>
![img](./images/pic7.png) 
<br/><br/><br/>
##### 联合查询

1.测试 join TIDB 表 和 Hive 表<br/>
select a.id, a.name, b.city from `hive`.`hack_test`.`tidb_stu` as a join `hive`.`hack_test`.`hive_city` as b on a.id=b.id;
<br/>
![img](./images/pic8.png) 

 

2.测试 join 后的结果 insert 到 TIDB<br/>
 1）创建 TIDB 表<br/>
CREATE TABLE hack_test.tidb_stu_details (`id` int(11) DEFAULT NULL, `name` varchar(255) DEFAULT NULL, `city` varchar(255) DEFAULT NULL) ENGINE=Inno
<br/><br/>
 2）创建 Flink 映射表<br/>
CREATE TABLE `hive`.`hack_test`.`tidb_stu_details`(
<br/>
id int,
<br/>
name string,
<br/>
city string
<br/>
) WITH (
<br/>
  'connector' = 'tidb',
<br/>
  'tidb.database.url' = 'jdbc:mysql://${ip}:${port}/flink',
<br/>
  'tidb.username' = '${user}',
<br/>
  'tidb.password' = '${pwd}',
<br/>
  'tidb.database.name' = 'hack_test',
<br/>
  'tidb.maximum.pool.size' = '10',
<br/>
  'tidb.minimum.idle.size' = '0',
<br/>
  'tidb.table.name' = 'tidb_stu_details',
<br/>
  'tidb.write_mode' = 'upsert',
<br/>
  'sink.buffer-flush.max-rows' = '0'
<br/>
 );
<br/><br/>
 3）join 结果插入 TIDB<br/>
insert into `hive`.`hack_test`.`tidb_stu_details` select a.id, a.name, b.city from `hive`.`hack_test`.`tidb_stu` as a join `hive`.`hack_test`.`hive_city` as b on a.id=b.id;
<br/>
![img](./images/pic9.png) 
<br/><br/>
 4）TIDB 端验证插入结果<br/>
![img](./images/pic10.png) 
