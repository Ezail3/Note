# SQL基础

## Ⅰ、表
### 1.1 什么是表？
行的集合

### 1.2 创建表
- 常规：create table t(id int, name varchar(10));
- cats：create table t1 as select * from t where xxx;
- 常用字段类型：int，bigint，varchar

### 1.3 查看表
- desc	查看字段和字段属性
- show create table t\G	比desc多看到索引情况
- show status like 'xxx'\G	看表的物理大小，索引大小和行数，优化sql看物理io

### 1.4 表特性
- myisam	忽略不看
- innodb	最大特点，二级索引包含主键，主键作为回表的key point，所以主键不能太长

## Ⅱ、SQL基础语法
咱们重点是select
### 2.1、单表查询
- 单列、多列、*
- 别名	select emp_no d from t_group;	别名必须英文开头，特殊字符禁用，有坑
- 有条件查询	select * from t_group where emp = 22744;两边需要满足数据类型相同
	- in必须写相同类型数据，in有去重作用
	- 如果两个以上的等号条件需要用and(不同字段)，如果两个等号条件针对同一个字段则结果为空(and缩小范围)
	- or对同一个列使用时等同于in，对不同列进行使用时候特别注意必须要加括号
- where 1=1的作用		开发拼接sql
- group by分组，分组之后的数据每个组只能出一行，分组之后可以结合聚合函数使用(count,min,max,sum等)
	- group by分组5.6的时候经常出现不完全分组，这是一种错，比如：select a, b from t group by a;除a以外的字段在select后面只能以聚合函数形式出现(ONLY_FULL_GROUP_BY)
	- 5.7的group by会排序，8.0不会排序
- order by 对结果集进行排序，可升序可降序(以前可以使用group by等省略order by，为了数据准确必须用order by)
	- order by a, b 先根据a排序好，相同的a再根据b排序，不要用concat(a,b)，走不到索引
- limit 主要用于分页，返回结果集的行数 limit m, n 从第m-1个开始取n个
- having 一般用于group by之后对结果进行二次过滤
- 子查询 子查询是指from后面使用括号，括起来的select部分需注意必须要有别名
- 标量子查询 select和from之间使用的子查询，标量子查询结果不能超过一行数据

### 2.1、多表查询
- 笛卡儿积 S × R
	- join的本质就是形成笛卡儿积，然后按连接条件进行过滤
	- 复杂sql排查笛卡儿积
		- 执行计划中type为all，extra为using join buffer(Block Nested Loop)
- 等价join
	- select a.\*, b.\* from a, b where a.id = b.id;
- between join
	- 一般不会使用，没搞懂
- in的半连接
	- select * from a where id in (select id from b);  只返回a表数据
- exsists
	- 结果和in一样
	- select * from a where exists(select b.id from b where b.id=a.id);
- not exsists
	- not in都改为not exsists，not in碰到null比较麻烦
	- select * from a where id not in (select id from b where id is not null); <==>  select * from a where not exists(select b.id from b where b.id=a.id);
- n:n join
	- select a_t.name, b_t.name from a a_t join (select id, name from b) b_t on a_t.id = b_t.id;
	- 尽量无重复值join，即1:1，不要join完了用distinct和group by去重，先去重好了再join


创建唯一索引用的sql
select xxx, count(1) from t group by xxx having count(1) > 1
为空则可以创建

```
优化sql要关注的一些东西
show create table
show table status
show variables like '%heap%'
show variables like '%tmp_table_size%'
show variables like '%innodb_buffer_pool_size%'
show variables like '%join%_size%'
```
