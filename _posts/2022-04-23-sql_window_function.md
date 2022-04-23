---
layout:     post
title:      窗口函数
category:   [sql]
tags:       [sql]
description: 最近业务开发中，需要取top10的数据，就专门去研究了下窗口函数的实现
---

## 窗口函数介绍

### 语法

```
<窗口函数> over (partition by <用于分组的列名> order by <用于排序的列名>）
```

- 专用窗口函数，比如 rank，dense_rank，row_number 等
- 聚合函数，如 sum，avg，count，max，min 等

### 功能

- 不减少原表的行数，所以经常用来在每组内排名
- 同时具有分组（partition by)和排序（order by)的功能

### 使用场景

业务需求“在每组内排名”，比如：

- 排名问题：每个部门按业绩来排名
- topN 问题：找出每个部门排名前 N 的员工进行奖励

### 注意事项

- 窗口函数原则上只能写在 select 子句中
- partition by 子句可以省略，省略就是不指定分组

## 标准聚合函数

标准的聚合函数有 avg、count、sum、max 和 min，接下来分别介绍这些聚合函数的窗口函数形式。

### 移动平均窗口函数

移动平均值的定义：若依次得到测定值（x1,x2,x3,...,xn）时，按顺序取一定个数所做的全部算数平均值。例如（x1+x2+x3）/3，（x2+x3+x4）/3，（x3+x4+x5）/3，...就是移动平均值。其中，x 可以是日或者月，以上的可以成为 3 日移动平均，或 3 月移动平均，常用于股票分析中

```
#语法结构

avg(字段名) over(
        partition by 用于分组的列名
        order by 用于排序的列名 asc/desc
        rows between A and B)#A和B是计算的行数范围
		rows between 2 preceding and current row #取当前行和前面两行
		rows between unbounded preceding and current row #包括本行和之前所有的行
		rows between current row and unbounded following #包括本行和之前所有的行
		rows between 3 preceding and current row #包括本行和前面三行
		rows between 3 preceing and 1 following #从前面三行和下面一行，总共五行
		当order by 后面缺少窗口从句条件，窗口规范默认是 rows between unbounded prceding and current row.
		当order by 和窗口从句都缺少，窗口规范默认是 rows between unbounded preceing and unbounded following
```

例子：

```
select *, avg(score) over (
           order by score desc rows between 2 preceding and current row
           ) as '平均分'
from student_score
```

![1](/images/sql_window_func/1.jpg)

- 对于第一行来说，没有前面两行，所以值就是当前行
- 对于第二行来说，前面只有一行，所以值就是第一行和第二行的平均值
- 影响行数范围的语句在标准的聚合函数中都适用

### 计数（count）窗口函数

> 窗口函数 count(\*) over()对于查询返回的每一行，它返回了表中所有行的计数

```
语法结构：
count(字段名1) over(
        partition by 字段名2
        order by 字段名3 asc/desc)

```

例子 1：

查询出成绩在 90 分以上的人数

```
select *,
       count(*) over () as 'cnt'
from student_score
where score >= 90
```

例子 2:

按照班级进行分组，找出成绩大于等于 80 分的学生人数

```
select *,
       count(*) over (partition by class_no) as 'cnt'
from student_score
where score >= 80
```

![2](/images/sql_window_func/2.jpg)

### 累计求和（sum）窗口函数

```
语法结构：

sum(字段名1) over (
        partition by 字段名2
        order by 字段名3 asc/desc)

#按照字段1进行累计求和，按照字段2进行分组，在组内按照字段3进行排序
```

例子 1：

根据分数排序，对学生的成绩进行累计求和

```
select *,
       sum(score) over (order by score desc) as 'sum_score'
from student_score

```

![3](/images/sql_window_func/3.jpg)

### 例子 2:

按照班级分组，然后根据分数对成绩进行累计求和

```
select *,
       sum(score) over (partition by class_no order by score desc) as 'sum_score'
from student_score

```

![4](/images/sql_window_func/4.jpg)

**注：一定要选择根据学号排序，要不然的出来的是最终的累计求和结果，如下图：**

```
select *,
       sum(score) over (partition by class_no) as 'sum_score'
from student_score

```

![5](/images/sql_window_func/5.jpg)

### 最大（max）、最小值（min）窗口函数

```
语法结构
max(字段名1) over(patition by 字段名2 order by 字段名3 asc/desc)

min(字段名1) over(patition by 字段名2 order by 字段名3 asc/desc)
```

例子 1:

求成绩的累计最大值和累计最小值

```
select *,
       max(score) over (order by student) as 'max_score',
       min(score) over (order by student) as 'min_score'
from student_score
```

![6](/images/sql_window_func/6.jpg)

按照姓名进行排序，在累积最大值中，会依次找最大值，如果有比当前值大的，就更新，没有就保持；最小值同理

例子 2：

按照班级进行分组，再求最大、最小值

```
select *,
       max(score) over (partition by class_no order by student) as 'max_score',
       min(score) over (partition by class_no order by student) as 'min_score'
from student_score
```

![7](/images/sql_window_func/7.jpg)

例子 3:

根据班级求成绩的累积最小值

```
select *,
       max(score) over (partition by class_no) as 'max_score',
       min(score) over (partition by class_no) as 'min_score'
from student_score
```

![8](/images/sql_window_func/8.jpg)

## 排序窗口函数

> row_number()、rank()、dense_rank()，这三个函数的作用都是返回相应规则的排序序号。

### row_number()

为查询出来的每一行记录都会生成一个序号，依次排序且不会重复。1，2，3，4

```
语法：
row_number() over(partition by 字段1 order by 字段2) #字段1是分组的字段名称
```

### rank()

使用 rank 函数来生成序号，over 子句中排序字段值相同的序号是一样的，后面字段值不相同的序号将跳过相同的排名排下一个，rank 函数生成的序号有可能是不连续的，即排名可能为 1，1，3，是跳跃式排名，有两个第一名时接下来就是第三名。1，1，1，4

```
语法：
rank() over(partition by 字段1 order by 字段2)
```

### dense_rank()

dense_rank 函数在生成序号时是连续的，当出现相同排名时，将不跳过相同排名号，有两个第一名时仍跟着第二名，即排名为 1，1，2 这种。1，1，1，2.

```
语法：
dense_rank() over(partition by 字段1 order by 字段2)
```

```
select *,
       row_number() over (order by score) as 'row_number',
       rank() over (order by score) as 'rank',
       dense_rank() over (order by score) as 'dense_rank'
from student_score
```

![9](/images/sql_window_func/9.jpg)

## 分组排序窗口函数

可以按照销售额的高低、点击次数的高低，以及成绩的高低为对用户和学生进行分组，这里的考点是：取销售额最高的 25%的用户（将用户分成 4 组，取出第一组）、取成绩高的前 10%的学生（将学生分成 10 组，取出第一组）等等。

```
语法结构：
ntile(n) over(partition by 字段名2 order by 字段名3 asc/desc)

#n表示要切片的分数，如需要取前25%的用户，则需要分4组，取前10%的用户，则需要分10组
```

- ntile(n)，用于将分组数据按照顺序切分成 n 片，返回当前切片值
- ntile 不支持 rows between 的用法
- 切片如果不均匀，默认增加第一个切片的分布

例子 1:

取出成绩前 25%的学生信息

1. 第一步：按照成绩的高低，将学生按照成绩进行切片

```
select *,
       ntile(4) over (order by score desc ) as 'rank'
from student_score
```

![10](/images/sql_window_func/10.jpg)

2. 第二步：按照 rank 筛选出第一组，则得到最终的结果如下

```
select a.*
from (select *,
             ntile(4) over (order by score desc ) as 'rank'
      from student_score) as a
where a.rank = 1
```

![11](/images/sql_window_func/11.jpg)

## 偏移分析窗口函数

- lag() over()和 lead() over()窗口函数，lag 和 lead 分析函数可以在同一次查询中取出同一个字段的前 N 行数据（lag）和后 N 行（lead）作为独立的列。
- 在实际应用当中，若要用到取今天和昨天的某字段的差值时，lag 和 lead 函数的应用就显得尤为重要了
- 适用场景：获取用户在某个页面停留的起始与结束时间
- 注意：LEAD()和 LAG()函数始终与 OVER()一起使用。缺少 over 子句将引发错误。

```
#语法结构

lag(exp_str,offset,defval) over(partition by ... order by ..)
lead(exp_str,offset,defval) over(partition by ... order by..)

#exp_str 表示字段名称
#offset偏移量，假设当前行在表中排在第5行，offset为3，则表示我们所要找的数据行就是表中的第2行（即5-3=2）
#offset默认为1
```

例子 1:

按班级分组，分数从高到低排序后，向前推 1 个分数

```
select *,
       LAG(score, 1, 0) over (partition by class_no order by score desc) as 'lag_score'
from student_score
```

![12](/images/sql_window_func/12.jpg)

例子 2：

按班级分组，分数从高到低排序后，向后推 1 个分数

```
select *,
       lead(score, 1, '无') over (partition by class_no order by score desc) as 'lead_score'
from student_score
```

![13](/images/sql_window_func/13.jpg)

例子 3:

统计每天符合以下条件的用户数：A 操作之后是 B 操作，AB 操作必须相邻。

用户行为表 racking_log(user_id,operate_id,log_time)

- 先根据用户 ID 和日期，用 LEAD()窗口函数向后获取下一步的步骤
- AB 必须相邻，则表明当前的步骤为 A，而下一个步骤为 B，即 A 向下移的步骤是 B；
- “每天”，根据日期进行分组

```
select a.log_date,count(distinct a.user_id)
from (select user_id,operate_id,
                    date_format(log_time,'%Y-%m-%d') as log_date,
                    lead(operate_id,1,null) over(partition by user_id,date_format(log_time,'%Y-%m-%d') order by log_time) as 'next_operate'
        from tracking_log) as a
where a.operate_id = A and a.next_operate = B
group by a.log_date
```

## 参考文档

- [SQL之窗口函数](https://www.cnblogs.com/xcyjblog/p/14868069.html?ivk_sa=1024320u)
