---
title: presto
date: 2021-03-14 00:17:26
tags:
---

`https://prestodb.io/docs/current/functions/datetime.html`

> presto时间函数
```sql
current_date current_time current_timestamp now()
```

#### 转时间戳
>字符串转时间戳
```sql
select cast('2019-04-26' as timestamp) 2019-04-26 00:00:00.000
select cast('2019-04-26 01:22:23' as timestamp) 2019-04-26 01:22:23.000
```
>按照format指定的格式，将字符串string解析成timestamp
```sql
select date_parse('2019-04-06','%Y-%m-%d') 2019-04-06 00:00:00.000
select date_parse('2019-04-06 00:03:55','%Y-%m-%d %H:%i:%S') 2019-04-06 00:03:55.000
```
>bigint 转时间戳
```sql
from_unixtime(create_time)
to_unixtime(current_date)
```

#### 转年月日
>时间戳取年月日
```sql
select date_format(current_date,'%Y-%m-%d')
select date(current_date)
select cast(current_date as date)
```
>字符串转年月日
```sql
select date(cast('2019-04-28 10:28:00' as TIMESTAMP))
select date('2019-04-28')
select date_format(cast('2019-04-28 10:28:00' as TIMESTAMP),'%Y-%m-%d')
select to_date('2019-04-28','yyyy-mm-dd');
```
>bigint 转年月日
```sql
date(from_unixtime(1556380800))
select date_format(from_unixtime(1556380800),'%Y-%m-%d')
```

#### 日期变换：间隔、加减、截取、提取
+ 求时间间隔 date_diff
```sql
date_diff(unit, timestamp1, timestamp2) → bigint

eg:select date_diff('day',cast('2019-04-24' as TIMESTAMP),cast('2019-04-26' as TIMESTAMP))  
--2
```
注：与hive差异！！！
```sql
presto中 date_diff('day',date1,date2)【后-前】
hive,mysql中 datediff(date1,date2) 【前-后】
```

+ 求几天前，几天后 interval、date_add
```sql
select current_date,(current_date - interval '7' day),date_add('day', -7, current_date)
 2019-04-28 | 2019-04-21 | 2019-04-21

select current_date,(current_date + interval '7' day),date_add('day', 7, current_date)
 2019-04-28 | 2019-05-05 | 2019-05-05
```
+ 时间截取函数 date_trunc(unit, x)
```sql
截取月初
select date_trunc('month',current_date)
2019-04-01

截取年初
select date_trunc('year',current_date)
2019-01-01
```
+ 时间提取函数 extract、year、month、day
```sql
extract(field FROM x) → bigint【注：field不带引号！】
year(x),month(x),day(x)

eg：
select 
	extract(year from current_date),
	year(current_date),
	extract(month from current_date),
	month(current_date),
	extract(day from current_date),
	day(current_date);
-------+-------+-------+-------+-------+-------
  2019 |  2019 |     4 |     4 |    28 |    28
```
+ 转int
```sql
先转timestamp，再to_unixtime转int
to_unixtime(timestamp_col)
```

