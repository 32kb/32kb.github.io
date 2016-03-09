---
layout: post
title: Java编程Tips
description: Java编程中“为了性能”尽量要做到的一些地方
keywords: java
categories: [java]
tags: [java, 性能]
---

MyCAT安装
===============

[TOC]

---------------

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

## 二、安装步骤

#### 1. 安装jdk

``` shell
mv jdk-7u71-linux-x64.tar.gz /home/ubuntu/
tar -zvxf jdk-7u71-linux-x64.tar.gz
ln -s jdk1.7.0_71 jdk

vi /etc/profile
export JAVA_HOME=/home/ubuntu/jdk
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
source /etc/profile
java -version
```

#### 2. 安装Mycat

``` shell
mkdir -p /data/
cd /data
rz
tar -zvxf  Mycat-server-1.5-GA-20160215160037-linux.tar.gz
vi /etc/profile
export MYCAT_HOME=/data/mycat
export PATH=$PATH:$MYCAT_HOME/bin
source /etc/profile
echo $MYCAT_HOME
```

#### 3. 配置Mycat

- **`修改my.cnf`**

``` shell
vim /etc/my.cnf
[mysqld]
lower_case_table_names = 1
```

- **`修改schema.xml`**

Schema中主要配置 Mycat 数据库，MySQL表，分片规则，分片类型

``` shell
vim $MYCAT_HOME/conf/schema.xml
```

``` xml
<writeHost host="hostM1" url="localhost:3306" user="root"
                        password="123456">
```
修改为
``` xml
<writeHost host="hostM1" url="10.1.6.101:3307" user="gaizai"
                        password="123">
```

#### 4. 创建MySQL数据库

``` shell
# 启动MySQL
/usr/local/mariadb/bin/mysqld --defaults-file=/data/mariadb/mariadb3307/my3307.cnf &
# 进入MySQL
/usr/local/mariadb/bin/mysql -S /tmp/mariadb3307.sock
# 关闭MySQL
/usr/local/mariadb/bin/mysqladmin -S /tmp/mariadb3307.sock shutdown
```

``` sql
CREATE database db1;
CREATE database db2;
CREATE database db3;
```

#### 5. 启动Mycat

``` shell
# 启动mycat
$MYCAT_HOME/bin/mycat start
# 关闭mycat
$MYCAT_HOME/bin/mycat stop
```

#### 6. 测试Mycat

连接到Mycat进行测试

``` shell
/usr/local/mariadb/bin/mysql -h10.1.1.167 -P8066 -utest -p
```

- **`测试travelrecord表`**

``` xml
<!-- auto sharding by id (long) -->
<table name="travelrecord" dataNode="dn1,dn2,dn3" rule="auto-sharding-long" />
```

travelrecord表，是根据ID主键的范围进行分片，分布在dn1,dn2,dn3三个节点，对应着conf/autopartition-long.txt文件
``` txt
# range start-end ,data node index
# K=1000,M=10000.
0-500M=0
500M-1000M=1
1000M-1500M=2
```

``` sql
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
更多表测试可以参考【1.4 Mycat默认表测试.md】

## 三、补充说明

1. 如果没有配置java的环境变量，需要修改wrapper.conf文件

```shell
vim $MYCAT_HOME/conf/wrapper.conf
# Java Application
wrapper.java.command=wrapper.java.command=/usr/local/java/jdk1.7.0_67/bin/java
```
2. 在mycat上看到了很多表，其实在db1、db2、db3是没有这些表的，mycat看到的只是配置文件中标签中设置的表；

2. 标签中primaryKey的作用
程序在后台根据 primaryKey 配置的主键列，自动生成主键的 sequence 值并替换原SQL中相关的列和值；

2. 温馨提示
explain可以用于任何正确的SQL上，其作用是告诉你，这条SQL会路由到哪些分片节点上执行，这对于诊断分片相关的问题很有帮助。另外，explain可以安全的执行多次，它仅仅是告诉你SQL的路由分片，而不会执行该SQL。

3. mycat文件列表
MyCAT的目录下主要包括以下文件

``` shell
root@db_test1:/data/mycat# ls | xargs ls
version.txt

bin:
init_zk_data.sh  mycat  rehash.sh  startup_nowrap.sh  wrapper-linux-ppc-64  wrapper-linux-x86-32  wrapper-linux-x86-64  xml_to_yaml.sh

catlet:

conf:
autopartition-long.txt   index_to_charset.properties  partition-hash-int.txt   rule.xml                  sequence_db_conf.properties    wrapper.conf
cacheservice.properties  log4j.xml                    partition-range-mod.txt  schema.xml                sequence_time_conf.properties  zk-create.yaml
ehcache.xml              myid.properties              router.xml               sequence_conf.properties  server.xml

lib:
curator-client-2.9.0.jar     guava-18.0.jar              libwrapper-linux-x86-32.so    netty-3.7.0.Final.jar                            wrapper.jar
curator-framework-2.9.0.jar  jline-0.9.94.jar            libwrapper-linux-x86-64.so    sequoiadb-java-driver-1.0-20150615.070208-1.jar  xml-apis-1.0.b2.jar
dom4j-1.6.1.jar              json-20151123.jar           log4j-1.2.17.jar              slf4j-api-1.7.12.jar                             zookeeper-3.4.6.jar
druid-1.0.14.jar             leveldb-0.7.jar             mapdb-1.0.7.jar               slf4j-log4j12-1.7.12.jar
ehcache-core-2.6.11.jar      leveldb-api-0.7.jar         mongo-java-driver-2.11.4.jar  snakeyaml-1.16.jar
fastjson-1.2.7.jar           libwrapper-linux-ppc-64.so  Mycat-server-1.5-GA.jar       univocity-parsers-1.5.4.jar

logs:
```

其中：
- lib目录下是MyCAT的主要的依赖库文件；
- bin目录下是MyCAT的运行脚本文件；
- conf目录下是MyCAT是主要配置文件；

本次安装过程中需要改到的相关MyCAT配置主要在schema.xml和server.xml。
 - conf/schema.xml
 - conf/rule.xml
 - conf/server.xml

- **`schema.xml`**
Schema中主要配置 Mycat 数据库，MySQL表，分片规则，分片类型
``` shell
vim $MYCAT_HOME/conf/schema.xml
```
``` xml
<schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100">
<!-- auto sharding by id (long) -->
<table name="travelrecord" dataNode="dn1,dn2,dn3" rule="auto-sharding-long" />
<table name="company" primaryKey="ID" type="global" dataNode="dn1,dn2,dn3" />
```
\# Mycat 数据库 TESTDB
\# MySQL表 travelrecord
\#  MySQL 节点dn1,dn2,dn3
\# 分片规则  auto-sharding-long
\# rule分片规则 具体在 conf/rule.xml 中定义
\# global表示全局表，即每个库的数据都是一样的
``` xml
<dataNode name="dn1" dataHost="localhost1" database="db1" />
<dataNode name="dn2" dataHost="localhost1" database="db2" />
<dataNode name="dn3" dataHost="localhost1" database="db3" />
<dataHost name="localhost1" maxCon="1000" minCon="10" balance="0"
                writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
```
\# 以上为MySQL节点信息
\# dn1，dn2，dn3为分片的MySQL节点，既分片会存放到 3个MySQL或者群集中
\# db1，db2，db3为 MySQL数据库中三个库
``` xml
<writeHost host="hostM1" url="localhost:3306" user="root"
                        password="123456">
```
MySQL节点连接，用户名，密码

- **`rule .xml`**
``` shell
vim $MYCAT_HOME/conf/rule.xml
```
\# auto-sharding-long（基于主键的范围分片）
\# mod-long（基于主键的取模分片）
\# sharding-by-intfile（基于sharding_id的哈希分片）
\# ER分片

- **`server.xml`**
``` shell
vim $MYCAT_HOME/conf/server.xml
```

``` xml
<property name="serverPort">8066</property>
<property name="managerPort">9066</property>
   <user name="test">
      <property name="password">test</property>
      <property name="schemas">TESTDB</property>
   </user>
```
\# serverPortMycat登录端口默认为 8066
\# managerPort管理端口 默认为 9066
\# username 为登录Mycat 用户
\# password 为登录 密码
\# schemas 为上面schema name= 中设定的 Mycat 数据库名

## 四、错误处理
1. 错误1：
ERROR 3009 (HY000): java.lang.IllegalArgumentException: Invalid DataSource:0
> **解决：**
在示例的2个数据hostM1和hostS1上，新建3个数据库db1,db2,db3，如不新建，可能提示找不到数据

2. 错误2：
ERROR 3009 (HY000): java.lang.IllegalArgumentException: Invalid DataSource:1
> **解决：**
这个有可能是Mycat和MySQL部署在同一台机器上，而在schema.xml是使用了IP的，但是账号只能使用localhost登陆，所以会出现本地的Mycat无法连接MySQL

3. 错误3：
mysql> create table abc (id bigint not null primary key, name varchar(100));
ERROR 1064 (HY000): op table not in schema----ABC
> **解决：**
表abc如果建立的表之前没有在schema.xml中定义，那么不可以建立此表。

## 五、参考文献
> [MyCAT安装与部署][1]
> [Mycat 安装配置][2]

[1]: http://valleylord.github.io/post/201601-mycat-install/
[2]: http://jicki.blog.51cto.com/1323993/1658603