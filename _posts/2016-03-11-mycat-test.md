---
layout: postlayout
title: MyCAT默认表测试
description: MyCAT默认表测试
keywords: MyCAT
categories: [MySQL]
tags: [MySQL, MyCAT]
---

## 一、测试环境

| 属性          | 值           |
| ------------- |:-------------|
|Ubuntu版本     |Ubuntu 14.04.2 LTS|
|服务器IP       |10.1.1.167|
|MySQL版本      |10.0.21-MariaDB-log|
|Mycat版本      |Mycat-server-1.5-GA-20160215160037-linux.tar.gz|
|jdk版本        |jdk-7u71-linux-x64.tar.gz|
|Mycat-MySQL版本|Server version: 5.5.8-mycat-1.5-GA-20160215160037 MyCat Server (OpenCloundDB)|
|MySQL账号密码  |gaizai/123|
|Mycat账号密码  |test/test|

## 二、测试

连接到Mycat进行测试
``` shell
/usr/local/mariadb/bin/mysql -h10.1.1.167 -P8066 -utest -p
```

#### 1、测试 `travelrecord` 表

``` xml
<!-- auto sharding by id (long) -->
<table name="travelrecord" dataNode="dn1,dn2,dn3" rule="auto-sharding-long" />
```

<!-- more -->

travelrecord表，是根据ID主键的范围进行分片，分布在dn1,dn2,dn3三个节点，对应着conf/autopartition-long.txt文件

``` html
# range start-end ,data node index
# K=1000,M=10000.
0-500M=0
500M-1000M=1
1000M-1500M=2
```

```sql
(product)test@10.1.1.167 [(none)]> use TESTDB;
Database changed

(product)test@10.1.1.167 [TESTDB]> create table travelrecord (id bigint not null primary key,user_id varchar(100),traveldate DATE, fee decimal,days int);
Query OK, 0 rows affected (0.04 sec)

(product)test@10.1.1.167 [TESTDB]> explain insert into travelrecord (id,user_id,traveldate,fee,days) values(1000000,'abc','2016-01-02',100.01,3);
+-----------+-------------------------------------------------------------------------------------------------------+
| DATA_NODE | SQL                                                                                                   |
+-----------+-------------------------------------------------------------------------------------------------------+
| dn1       | insert into travelrecord (id,user_id,traveldate,fee,days) values(1000000,'abc','2016-01-02',100.01,3) |
+-----------+-------------------------------------------------------------------------------------------------------+
1 row in set (0.04 sec)

(product)test@10.1.1.167 [TESTDB]> explain insert into travelrecord (id,user_id,traveldate,fee,days) values(7000000,'abc','2016-01-02',100.01,3);
+-----------+-------------------------------------------------------------------------------------------------------+
| DATA_NODE | SQL                                                                                                   |
+-----------+-------------------------------------------------------------------------------------------------------+
| dn2       | insert into travelrecord (id,user_id,traveldate,fee,days) values(7000000,'abc','2016-01-02',100.01,3) |
+-----------+-------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

(product)test@10.1.1.167 [TESTDB]> explain select * from travelrecord;
+-----------+--------------------------------------+
| DATA_NODE | SQL                                  |
+-----------+--------------------------------------+
| dn1       | SELECT * FROM travelrecord LIMIT 100 |
| dn2       | SELECT * FROM travelrecord LIMIT 100 |
| dn3       | SELECT * FROM travelrecord LIMIT 100 |
+-----------+--------------------------------------+
3 rows in set (0.03 sec)

insert into travelrecord (id,user_id,traveldate,fee,days) values(1000000,'abc','2016-01-02',100.01,3);
insert into travelrecord (id,user_id,traveldate,fee,days) values(7000000,'abc','2016-01-02',100.01,3);

(product)test@10.1.1.167 [TESTDB]> select * from travelrecord;
+---------+---------+------------+------+------+
| id      | user_id | traveldate | fee  | days |
+---------+---------+------------+------+------+
| 7000000 | abc     | 2016-01-02 |  100 |    3 |
| 1000000 | abc     | 2016-01-02 |  100 |    3 |
+---------+---------+------------+------+------+
2 rows in set (0.01 sec)
```

#### 2、测试 `company` 表

``` xml
<!-- global table is auto cloned to all defined data nodes ,so can join
     with any table whose sharding node is in the same data node -->
<table name="company" primaryKey="ID" type="global" dataNode="dn1,dn2,dn3" />
```

company表，定义为全局表，在dn1,dn2,dn3 三个节点都有相同的记录

``` sql
create table company(id int not null primary key,name varchar(100));

(product)test@10.1.1.167 [TESTDB]> explain insert into company(id,name) values(1,'hp');
+-----------+---------------------------------------------+
| DATA_NODE | SQL                                         |
+-----------+---------------------------------------------+
| dn1       | insert into company(id,name) values(1,'hp') |
| dn2       | insert into company(id,name) values(1,'hp') |
| dn3       | insert into company(id,name) values(1,'hp') |
+-----------+---------------------------------------------+
3 rows in set (0.00 sec)

insert into company(id,name) values(1,'hp');
insert into company(id,name) values(2,'ibm');
insert into company(id,name) values(3,'oracle');
(product)test@10.1.1.167 [TESTDB]> select * from company;
+----+--------+
| id | name   |
+----+--------+
|  1 | hp     |
|  2 | ibm    |
|  3 | oracle |
+----+--------+
3 rows in set (0.00 sec)
```

#### 3、测试 `goods` 表测试测试测试测试

#### 3、测试 `goods` 表测试测试测试测试

#### 3、测试 `goods` 表测试测试测试测试


``` xml
<table name="goods" primaryKey="ID" type="global" dataNode="dn1,dn2" />
```

goods表，定义为全局表，在dn1,dn2两个节点都有相同的记录

``` sql
create table goods(id int not null primary key,name varchar(200),good_type tinyint,good_img_url  varchar(200),good_created date,good_desc varchar(500), price double);

(product)test@10.1.1.167 [TESTDB]> explain insert into goods(id,name,good_type,good_img_url,good_created,good_desc,price) values(1,'hp',1,'http://a.jgp',now(),'惠普',100.00);
+-----------+--------------------------------------------------------------------------------------------------------------------------------------+
| DATA_NODE | SQL                                                                                                                                  |
+-----------+--------------------------------------------------------------------------------------------------------------------------------------+
| dn1       | insert into goods(id,name,good_type,good_img_url,good_created,good_desc,price) values(1,'hp',1,'http://a.jgp',now(),'惠普',100.00)   |
| dn2       | insert into goods(id,name,good_type,good_img_url,good_created,good_desc,price) values(1,'hp',1,'http://a.jgp',now(),'惠普',100.00)   |
+-----------+--------------------------------------------------------------------------------------------------------------------------------------+
2 rows in set (0.02 sec)

insert into goods(id,name,good_type,good_img_url,good_created,good_desc,price) values(1,'hp',1,'http://a.jgp',now(),'惠普',100.00);

(product)test@10.1.1.167 [TESTDB]> select * from goods;
+----+------+-----------+--------------+--------------+-----------+-------+
| id | name | good_type | good_img_url | good_created | good_desc | price |
+----+------+-----------+--------------+--------------+-----------+-------+
|  1 | hp   |         1 | http://a.jgp | 2016-02-23   | 我      |   100 |
+----+------+-----------+--------------+--------------+-----------+-------+
1 row in set (0.00 sec)
```

#### 4、测试 `hotnews` 表

``` xml
<!-- random sharding using mod sharind rule -->
<table name="hotnews" primaryKey="ID" dataNode="dn1,dn2,dn3" rule="mod-long" />
```

hotnews表，基于主键取摸的方式分配到dn1,dn2,dn3上

``` sql
create table hotnews(id int  not null primary key ,title varchar(400) ,created_time datetime);

(product)test@10.1.1.167 [TESTDB]> explain insert into hotnews(id,title,created_time) values(1,'first',now());
+-----------+--------------------------------------------------------------------+
| DATA_NODE | SQL                                                                |
+-----------+--------------------------------------------------------------------+
| dn2       | insert into hotnews(id,title,created_time) values(1,'first',now()) |
+-----------+--------------------------------------------------------------------+
1 row in set (0.00 sec)

(product)test@10.1.1.167 [TESTDB]> explain insert into hotnews(id,title,created_time) values(2,'two',now());
+-----------+------------------------------------------------------------------+
| DATA_NODE | SQL                                                              |
+-----------+------------------------------------------------------------------+
| dn3       | insert into hotnews(id,title,created_time) values(2,'two',now()) |
+-----------+------------------------------------------------------------------+
1 row in set (0.00 sec)

(product)test@10.1.1.167 [TESTDB]> explain insert into hotnews(id,title,created_time) values(3,'three',now());
+-----------+--------------------------------------------------------------------+
| DATA_NODE | SQL                                                                |
+-----------+--------------------------------------------------------------------+
| dn1       | insert into hotnews(id,title,created_time) values(3,'three',now()) |
+-----------+--------------------------------------------------------------------+
1 row in set (0.00 sec)

insert into hotnews(id,title,created_time) values(1,'first',now());
insert into hotnews(id,title,created_time) values(2,'two',now());
insert into hotnews(id,title,created_time) values(3,'three',now());
insert into hotnews(id,title,created_time) values(5,'five',now());

(product)test@10.1.1.167 [TESTDB]> select * from hotnews;
+----+-------+---------------------+
| id | title | created_time        |
+----+-------+---------------------+
|  3 | three | 2016-02-23 14:34:19 |
|  2 | two   | 2016-02-23 14:34:15 |
|  5 | five  | 2016-02-23 14:34:22 |
|  1 | first | 2016-02-23 14:34:11 |
+----+-------+---------------------+
4 rows in set (0.00 sec)
```

- id为1，1%3=1，则到dn2上；
- id为2，2%3=2，则到dn3上；
- id为3，3%3=0，则到dn1上；
- id为5，5%3=2，则到dn3上，即对应dn3的index；


#### 5、测试 `employee` 表

``` xml
<table name="employee" primaryKey="ID" dataNode="dn1,dn2" rule="sharding-by-intfile" />
```

employee表，根据sharding-by-intfile （分片字段为sharding_id）规则进行分片，只在dn1,dn2存储数据，对应着conf/partition-hash-int.txt文件

``` html
10000=0
10010=1
```

``` sql
create table employee (id int not null primary key,name varchar(100),sharding_id int not null);

insert into employee(id,name,sharding_id) values(1,'leader us',10000);
insert into employee(id,name,sharding_id) values(2,'me',10010);
insert into employee(id,name,sharding_id) values(3,'mycat',10000);
insert into employee(id,name,sharding_id) values(4,'mydog',10010);

(product)test@10.1.1.167 [TESTDB]> select * from employee;
+----+-----------+-------------+
| id | name      | sharding_id |
+----+-----------+-------------+
|  1 | leader us |       10000 |
|  3 | mycat     |       10000 |
|  2 | me        |       10010 |
|  4 | mydog     |       10010 |
+----+-----------+-------------+
4 rows in set (0.00 sec)
```

#### 6、测试 `orders` 表

``` xml
<table name="customer" primaryKey="ID" dataNode="dn1,dn2" rule="sharding-by-intfile">
    <childTable name="orders" primaryKey="ID" joinKey="customer_id" parentKey="id">
    <childTable name="order_items" joinKey="order_id" parentKey="id" />
    </childTable>
    <childTable name="customer_addr" primaryKey="ID" joinKey="customer_id" parentKey="id" />
</table>
```

 `customer表`，根据sharding-by-intfile （分片字段为sharding_id）规则进行分片，只在dn1,dn2存储数据
 `orders表`，ER分片，根据customer表的ID

首先创建customer表

``` sql
create table customer(id int not null primary key,name varchar(100),company_id int not null,sharding_id int not null);

(product)test@10.1.1.167 [TESTDB]> explain insert into customer (id,name,company_id,sharding_id)values(1,'wang',1,10000);
+-----------+-------------------------------------------------------------------------------+
| DATA_NODE | SQL                                                                           |
+-----------+-------------------------------------------------------------------------------+
| dn1       | insert into customer (id,name,company_id,sharding_id)values(1,'wang',1,10000) |
+-----------+-------------------------------------------------------------------------------+
1 row in set (0.00 sec)

(product)test@10.1.1.167 [TESTDB]> explain insert into customer (id,name,company_id,sharding_id)values(2,'xue',2,10010);
+-----------+------------------------------------------------------------------------------+
| DATA_NODE | SQL                                                                          |
+-----------+------------------------------------------------------------------------------+
| dn2       | insert into customer (id,name,company_id,sharding_id)values(2,'xue',2,10010) |
+-----------+------------------------------------------------------------------------------+
1 row in set (0.00 sec)

(product)test@10.1.1.167 [TESTDB]> explain insert into customer (id,name,company_id,sharding_id)values(3,'feng',3,10000);
+-----------+-------------------------------------------------------------------------------+
| DATA_NODE | SQL                                                                           |
+-----------+-------------------------------------------------------------------------------+
| dn1       | insert into customer (id,name,company_id,sharding_id)values(3,'feng',3,10000) |
+-----------+-------------------------------------------------------------------------------+
1 row in set (0.00 sec)

insert into customer(id,name,company_id,sharding_id)values(1,'wang',1,10000);
insert into customer(id,name,company_id,sharding_id)values(2,'xue',2,10010);
insert into customer(id,name,company_id,sharding_id)values(3,'feng',3,10000);
insert into customer(id,name,company_id,sharding_id)values(4,'wo',4,10000);

(product)test@10.1.1.167 [TESTDB]> select * from customer;
+----+------+------------+-------------+
| id | name | company_id | sharding_id |
+----+------+------------+-------------+
|  2 | xue  |          2 |       10010 |
|  1 | wang |          1 |       10000 |
|  3 | feng |          3 |       10000 |
|  4 | wo   |          4 |       10000 |
+----+------+------------+-------------+
4 rows in set (0.01 sec)
```

测试orders表

``` sql
create table orders (id int not null primary key ,customer_id int not null,sataus int ,note varchar(100) );

//stored in db1 because customer table with id=3 stored in db1
insert into orders(id,customer_id) values(1,3);

//stored in db1 because customer table with id=1 stored in db1
insert into orders(id,customer_id) values(2,1);

//stored in db2 because customer table with id=2 stored in db2
insert into orders(id,customer_id) values(3,2);

(product)test@10.1.1.167 [TESTDB]> select * from orders;
+----+-------------+--------+------+
| id | customer_id | sataus | note |
+----+-------------+--------+------+
|  1 |           3 |   NULL | NULL |
|  2 |           1 |   NULL | NULL |
|  3 |           2 |   NULL | NULL |
+----+-------------+--------+------+
3 rows in set (0.00 sec)

(product)test@10.1.1.167 [TESTDB]> select customer.name ,orders.* from customer ,orders where customer.id=orders.customer_id;
+------+----+-------------+--------+------+
| name | id | customer_id | sataus | note |
+------+----+-------------+--------+------+
| wang |  2 |           1 |   NULL | NULL |
| feng |  1 |           3 |   NULL | NULL |
| xue  |  3 |           2 |   NULL | NULL |
+------+----+-------------+--------+------+
3 rows in set (0.05 sec)
```
这里无法使用explain来查看insert执行分布，如果加上explain，直接被当成insert了；


## 三、参考文献

> _[MyCAT安装与部署][1]
> _[Mycat 安装配置][2]

[1]: http://valleylord.github.io/post/201601-mycat-install/
[2]: http://jicki.blog.51cto.com/1323993/1658603


* any list
{:toc}