### SQL
1）group by
分组查询
例子：
` select student_id, count(course_id), sum(score) from score group_by student_id;`
` select s.student_id, stu.name, count(s.course_id), sum(s.score) from score s, student stu where s.student_id = stu.student_id group_by student_id;`
语法：
* 列函数对于group by子句定义的每个组各返回一个结果。
* 如果用到group by，那么select语句中选出的列要么是 group by里用到的列，要么就是带有聚合函数的列，或者是别的表的列。
* 出现在同一SQL的顺序：Where > Group by > Having

2）having 
通常与group by子句连用
where过滤行，having过滤组
例子：
` select student_id, avg(score) from score group by student_id having avg(score) > 60;`

