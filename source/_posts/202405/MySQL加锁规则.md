---
title: MySQL加锁规则
date: 2024-05-10 22:23:20
category:
- 实践
tags: 
- MySQL
---
#### 一、前言

MySQL不同隔离级别、索引加锁的情况特别复杂，但是本质是为了确保数据在一定的隔离级别下的一致性，本文通过实践，分析加锁情况。

PS：没有参考任何MySQL源码，本文列举的情况不一定准确。

一些版本信息：
```shell
mysql> show variables like '%version%';
+-----------------------------+------------------------------+
| Variable_name               | Value                        |
+-----------------------------+------------------------------+
| admin_tls_version           | TLSv1.2,TLSv1.3              |
| explain_json_format_version | 1                            |
| immediate_server_version    | 999999                       |
| innodb_version              | 8.4.3                        |
| original_server_version     | 999999                       |
| protocol_version            | 10                           |
| replica_type_conversions    |                              |
| slave_type_conversions      |                              |
| tls_version                 | TLSv1.2,TLSv1.3              |
| version                     | 8.4.3                        |
| version_comment             | MySQL Community Server - GPL |
| version_compile_machine     | x86_64                       |
| version_compile_os          | Linux                        |
| version_compile_zlib        | 1.3.1                        |
+-----------------------------+------------------------------+
14 rows in set (0.00 sec)
```
使用的一些语句
```shell
# 设置锁等待时间，默认才50s, 不适合我们实践 (需重启)
set global innodb_lock_wait_timeout = 999999; 

# 查看当前会话的隔离级别
SELECT @@SESSION.transaction_isolation;

# 设置当前会话的隔离级别 - 读已提交
set session transaction isolation level read committed;

# 设置当前会话的隔离级别 - 可重复读
set session transaction isolation level repeatable read;

# 查看当前锁情况
select * from performance_schema.data_locks;

# 查看锁等待情况（上面data_locks其实也会显示等待状态的锁）
select * from performance_schema.data_lock_waits;
```
#### 二、基础信息
##### 1.行锁的类型
- **记录锁**

    锁住单条记录
- **间隙锁（Gap Lock）**

    锁住一个范围，左开右开的区间（2, 5），相同范围的间隙锁是兼容的
- **Next-Key Lock**

    间隙锁 + 记录锁的组合，左开右闭的区间（2, 5]，相同范围的Next-Key Lock
    锁不是是兼容的

- **插入意向锁**

    插入新数据时，先生成一个插入意向锁记录，相当于gap lock，不过不是范围，而是一个点

##### 2.data_locks展示的锁类型
- LOCK_MODE 为 X -- next-key 锁

- LOCK_MODE 为 X, REC_NOT_GAP -- 记录锁

- LOCK_MODE 为 X, GAP -- 间隙锁

##### 3.演示表结构
```shell
mysql> show create table t_student\G;
*************************** 1. row ***************************
       Table: t_student
Create Table: CREATE TABLE `t_student` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `age` int NOT NULL,
  `name` varchar(32) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_age_id` (`age`,`id`) 
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)
```

### 三、读已提交
**总结：**

读已提交的隔离级别下，锁情况比较简单，只会出现表级别意向性和行级别记录锁。

#### 1.普通读
```shell
set session transaction isolation level read committed;
begin;
select * from t_student;
```

```shell
# 不会加锁
mysql> select * from performance_schema.data_locks\G
Empty set (0.00 sec)
```

#### 2.加锁读
##### （1）唯一索引记录存在
```shell
set session transaction isolation level read committed;
begin;
select * from t_student where id = 1 for update;
```
```shell
mysql> select * from performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:136:1070:139937483641824
ENGINE_TRANSACTION_ID: 4001
            THREAD_ID: 54
             EVENT_ID: 196
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 139937483641824
            LOCK_TYPE: TABLE
            LOCK_MODE: IX      # 表级别的X意向锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:136:4:4:2:139937483638832
ENGINE_TRANSACTION_ID: 4001
            THREAD_ID: 54
             EVENT_ID: 196
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 139937483638832
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP    # id=1的X记录锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: 1
2 rows in set (0.00 sec)
```
##### （2）唯一索引记录不存在
```shell
set session transaction isolation level read committed;
begin;
select * from t_student where id = 999 for update;
```
```shell
mysql> select * from performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:138:1070:139937483641824
ENGINE_TRANSACTION_ID: 4002
            THREAD_ID: 54
             EVENT_ID: 201
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 139937483641824
            LOCK_TYPE: TABLE
            LOCK_MODE: IX      # 表级别的X意向锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
1 row in set (0.01 sec)
```
可以看到仅有表级别的X意向锁，并没有任何行锁，此时是可以插入新数据的（会有唯一键校验）。

| 当前锁 \ 请求锁 | IS (意向共享) | IX (意向排他) | S (共享锁) | X (排他锁) |
| --------- | --------- | --------- | ------- | ------- |
| **IS**    | ✅         | ✅         | ✅       | ❌       |
| **IX**    | ✅         | ✅         | ❌       | ❌       |
| **S**     | ✅         | ❌         | ✅       | ❌       |
| **X**     | ❌         | ❌         | ❌       | ❌       |

这是表级锁的冲突情况，在我们这个场景中，顺序是这样的：

1.事务A获取IX锁

2.事务B获取IX锁（兼容）

3.事务B插入新数据，需获取X锁（注意这里的X锁是行锁，所以与事务A的表级别IX锁不冲突）

##### （3）二级索引记录存在
```shell
set session transaction isolation level read committed;
begin;
select * from t_student where age = 10 for update;
```
```shell
mysql> select * from performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:142:1070:139937483641824
ENGINE_TRANSACTION_ID: 4009
            THREAD_ID: 54
             EVENT_ID: 209
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 139937483641824
            LOCK_TYPE: TABLE
            LOCK_MODE: IX     # 表级别的X意向锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:142:4:5:2:139937483638832
ENGINE_TRANSACTION_ID: 4009
            THREAD_ID: 54
             EVENT_ID: 209
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: idx_age_id
OBJECT_INSTANCE_BEGIN: 139937483638832
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP   # X记录锁，锁的是age=10,id=1的记录
          LOCK_STATUS: GRANTED
            LOCK_DATA: 10, 1
*************************** 3. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:142:4:4:2:139937483639176
ENGINE_TRANSACTION_ID: 4009
            THREAD_ID: 54
             EVENT_ID: 209
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 139937483639176
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP   # X记录锁，锁的是id=1的记录
          LOCK_STATUS: GRANTED
            LOCK_DATA: 1
3 rows in set (0.00 sec)
```
可以看到二级索引不仅会锁二级索引，还会锁主键索引

##### （4）二级索引记录不存在
```shell
set session transaction isolation level read committed;
begin;
select * from t_student where age = 999 for update;
```
```shell
mysql> select * from performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:146:1070:139937483641824
ENGINE_TRANSACTION_ID: 4012
            THREAD_ID: 54
             EVENT_ID: 216
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 139937483641824
            LOCK_TYPE: TABLE
            LOCK_MODE: IX     # 表级别的X意向锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
1 row in set (0.00 sec)
```

##### （5）范围查询
```shell
set session transaction isolation level read committed;
begin;
select * from t_student where id >= 10 for update;
```
```shell
mysql> select * from performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:152:1070:139937483641824
ENGINE_TRANSACTION_ID: 4015
            THREAD_ID: 54
             EVENT_ID: 226
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 139937483641824
            LOCK_TYPE: TABLE
            LOCK_MODE: IX     # 表级别的X意向锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:152:4:4:5:139937483638832
ENGINE_TRANSACTION_ID: 4015
            THREAD_ID: 54
             EVENT_ID: 226
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 139937483638832
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP    # X记录锁，锁的是id=10的记录
          LOCK_STATUS: GRANTED
            LOCK_DATA: 10
2 rows in set (0.00 sec)
```


### 四、可重复读
#### 1.普通读
**总结：**

普通读通过MVCC读取快照数据，不需要加锁。

##### （1）记录存在
```shell
set session transaction isolation level repeatable read;
begin;
select * from t_student where id = 10;
```
```shell
mysql> select * from performance_schema.data_locks\G
Empty set (0.00 sec)
```

##### （2）记录不存在
```shell
set session transaction isolation level repeatable read;
begin;
select * from t_student where id = 999;
```
```shell
mysql> select * from performance_schema.data_locks\G
Empty set (0.00 sec)
```

#### 1.加锁读
这是我们演示可重复读的数据
```shell
mysql> select * from t_student;
+----+-----+-------+
| id | age | name  |
+----+-----+-------+
|  2 |  12 | name1 |
|  6 |  13 | name2 |
| 10 |  20 | name3 |
+----+-----+-------+
```
![数据实例](/images/202405/数据实例.png)

##### （1）唯一索引记录存在
```shell
set session transaction isolation level repeatable read;
begin;
select * from t_student where id = 2 for update;
```
```shell
mysql> select * from performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:168:1070:139937483641824
ENGINE_TRANSACTION_ID: 4023
            THREAD_ID: 54
             EVENT_ID: 266
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 139937483641824
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:168:4:4:3:139937483638832
ENGINE_TRANSACTION_ID: 4023
            THREAD_ID: 54
             EVENT_ID: 266
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 139937483638832
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP
          LOCK_STATUS: GRANTED
            LOCK_DATA: 2
2 rows in set (0.00 sec)
```

##### （2）唯一索引记录不存在
```shell
set session transaction isolation level repeatable read;
begin;
select * from t_student where id = 3 for update;
```
```shell
mysql> select * from performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:170:1070:139937483641824
ENGINE_TRANSACTION_ID: 4024
            THREAD_ID: 54
             EVENT_ID: 271
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 139937483641824
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:170:4:4:6:139937483638832
ENGINE_TRANSACTION_ID: 4024
            THREAD_ID: 54
             EVENT_ID: 271
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 139937483638832
            LOCK_TYPE: RECORD
            LOCK_MODE: X,GAP   # 间隙锁，右边是6，左边是上一条记录2，即(2, 6)
          LOCK_STATUS: GRANTED
            LOCK_DATA: 6
2 rows in set (0.00 sec)
```
```shell
set session transaction isolation level repeatable read;
begin;
select * from t_student where id = 100 for update;
```
```shell
mysql> select * from performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:172:1070:139937483641824
ENGINE_TRANSACTION_ID: 4025
            THREAD_ID: 54
             EVENT_ID: 275
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 139937483641824
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:172:4:4:1:139937483638832
ENGINE_TRANSACTION_ID: 4025
            THREAD_ID: 54
             EVENT_ID: 275
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 139937483638832
            LOCK_TYPE: RECORD
            LOCK_MODE: X   # next-key lock，右边是无穷大，左边是上一条记录10，即(10, +inf)
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record
2 rows in set (0.00 sec)
```
##### （3）唯一索引范围查询
```shell
set session transaction isolation level repeatable read;
begin;
select * from t_student where id > 7 for update;
```
```shell
mysql> select * from performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:174:1070:139937483641824
ENGINE_TRANSACTION_ID: 4026
            THREAD_ID: 54
             EVENT_ID: 280
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 139937483641824
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:174:4:4:1:139937483638832
ENGINE_TRANSACTION_ID: 4026
            THREAD_ID: 54
             EVENT_ID: 280
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 139937483638832
            LOCK_TYPE: RECORD
            LOCK_MODE: X  # next-key lock，右边是无穷大，左边是上一条记录10，即(10, +inf)
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record
*************************** 3. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:174:4:4:7:139937483638832
ENGINE_TRANSACTION_ID: 4026
            THREAD_ID: 54
             EVENT_ID: 280
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 139937483638832
            LOCK_TYPE: RECORD
            LOCK_MODE: X  # next-key lock，右边是10，左边是上一条记录6，即(6, 10)
          LOCK_STATUS: GRANTED
            LOCK_DATA: 10
3 rows in set (0.00 sec)
```

如果记录存在，则只加记录锁
- 删除、更新：不能，有记录锁
- 新增：不能，有唯一键校验

如果记录不存在，优先加next-key lock，但是可以退化为gap lock
- 删除、更新：不能，范围已被加锁
- 新增：不能，范围已被加锁

主要还是抓住一点：加什么锁能保证可重复读，不会因为删除、更新、新增而导致俩次读的结果不一样

##### （4）非唯一索引
非唯一索引会同时对二级索引和主键加锁，不再赘述。

##### （5）二级索引插入特殊情况
```shell
set session transaction isolation level repeatable read;
begin;
select * from t_student where age = 15 for update;
```
```shell
mysql> select * from performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:178:1070:139937483641824
ENGINE_TRANSACTION_ID: 4028
            THREAD_ID: 54
             EVENT_ID: 289
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 139937483641824
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 139937551555800:178:4:5:7:139937483638832
ENGINE_TRANSACTION_ID: 4028
            THREAD_ID: 54
             EVENT_ID: 289
        OBJECT_SCHEMA: students
          OBJECT_NAME: t_student
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: idx_age_id
OBJECT_INSTANCE_BEGIN: 139937483638832
            LOCK_TYPE: RECORD
            LOCK_MODE: X,GAP  # age(13, 20)
          LOCK_STATUS: GRANTED
            LOCK_DATA: 20, 10
2 rows in set (0.00 sec)
```
![非唯一不存在](/images/202405/非唯一不存在.png)

查询age=15，一开始对age（13, 20]加锁，但是新增和删除20并不影响可重复读，所以退化回间隙锁age（13,20）。

age=13和age=20，看起来并没有被锁住，那么是否可以插入age=13或age=20的记录？

###### 1.插入age=13，id=5的记录
可以插入
###### 2.插入age=13，id=7的记录
不能插入
###### 3.插入age=20，id=7的记录
不能插入
###### 2.插入age=20，id=12的记录
可以插入

**可以看到如果新数据的插入点在（13, 20）之间，是不允许的**