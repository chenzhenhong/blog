---
title: 记一次Spring事务问题
date: 2024-04-12 23:23:23
category:
- 实践
tags: 
- 事务
- Spring
---
### 原由
今天工作遇到一个事务相关的问题，在此记录一下

### 业务代码
如图所示，有三个serviceA、B、C，每个service都有各自的方法，分别是methodA、B、C，且三个方法按顺序调用。

其中methodA和methodB都有@Transactional注解，methodC往数据插入一条记录row1，methodB会查询row1记录。

![事务-业务代码](/images/202404/事务-业务代码.png)

### 问题1
methodC的是一个独立的业务，我们不希望调用方法如methodB发生异常后回滚row1记录；很容易想到此时应该为methodC使用单独的事务。

如图所示，我们给methodC使用@Transactional注解，且传播级别为REQUIRES_NEW。

![事务-新事务](/images/202404/事务-新事务.png)

### 问题2
当methodC开启新事务后，插入row1成功，但是在methodB方法中却无法读取到row1记录，因为数据库默认的隔离级别是可重复读。

考察methodB的业务后，我们决定将methodB的隔离级别设置为读已提交，如下图

![事务-读已提交](/images/202404/事务-读已提交.png)

### 问题3
一番操作下来，却发现methodB方法依然无法读取row1记录，读已提交的设置似乎没有效果。

在经过多次的demo测试后，发现遗漏的知识点，那就是methodB的事务其实是在methodA方法中开启的，其默认隔离级别是可重复读；**已经确定了隔离级别的事务后续是无法修改的**。

对于这个问题，可以将methodB的读已提交添加到methodA上面即可。

### 注意
添加@Transactional注解后开启事务是**延迟**的，即如果在methodA方法中没有进行数据库操作，是不会开启事务**。
