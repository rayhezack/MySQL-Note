在该文章中，我会介绍索引在Mysql中的运用。针对于索引的学习更多集中在概念上的理解，因为索引的使用语法并不难。

**Topics:**
- What is index?
- How index works?
- When to use index?
- Index statement


## Introduction to Index
**什么是索引？**
>"Indexes are used to find rows with specific column values quickly. Without an index, MySQL must begin with the first row and then read through the entire table to find the relevant rows." -- MySQL

> "An index is a data structure such as B-Tree that improves the speed of data retrieval on a table at the cost of additional writes and storage to maintain it." --MySQLTutorial

基于上述两条定义，索引的含义为: *索引本质是一种额外创建需要定期维护的数据结构，其主要目是提升数据检索速度。对于有索引的字段，Mysql会迅速定位到被查找的数据而无需从第一行开始扫描整列，直到找相应的数据为止。*

通过一个例子来清晰地理解什么是索引。假设有一个记载了名字和电话的电话本。现在这个电话本没有按照名字的字母顺序排序，如果我们想要找到名叫James的人的电话，就只能从头到尾一个一个扫描，直到找到James为止。但是如果，我们将电话按照名字的字母顺序排序，我们可以直接找到J对应的区域，然后找到James。所以，索引就类似于字典。

其次，***对于primary key和unique key，MySQL会自动为其创建索引。这种索引被称为clustered index。***


**什么是clustered index?**
Clustered index就是表格本身。其次，Clustered index是一种将表格的所有记录按照关键字段进行物理排序的一种索引，所以每张表格都只有一个clustered index。


## Mechanism behind Index
索引会将所有记录排序，然后再去迅速扫描找到相应的数据。其本质上是一种例如二叉树的数据结构，通过二叉树结构，大幅度缩减了扫描范围，从而增加数据检索速度。

InnoDB存储引擎下的表格，MySQL会默认创建B-Tree index。下表显示了不存储引擎下的默认索引类型：

|Storage Engine | Allowed Index Types  |
|-----------|------------|
|InnoDB | BTREE |
|MyISAM|BTREE|
|MEMORY/HEAP| HASH,BTREE  |

**Example**
举个具体的例子来解释索引的原理。如果我们有以下虚构的表格，ID为主键。
`users`
|ID | Name  |
|-----------|------------|
|100 | Jay |
|110|James|
|99| Robert  |
|88| John|
|101| Mary |
|130| Patricia  |
|55| Jennifer|
因为ID为主键，所以MySQL会自动为其创建索引，并进行物理排序，如下图所示：
![B-TREE](https://upload-images.jianshu.io/upload_images/10429581-c02109c2fcbb5031.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
假设我们的查询语句为：
```sql
select * from users where id=130
```
如果没有索引，MySQL会自动从第一行开始扫描所有记录，总计7行。但是如果添加了索引，创建出二叉树结构后，MySQL只需要迭代两次就能找出id=130对应的记录。


## 什么时候对某个字段创建索引？
- 该字段经常出现在where子句中
- 不会经常对该字段执行DML语句
- 数据量庞大，需要提升检索效率

## Index Statement

**创建Index**
通常，我们会在创建表格的时候创建索引。
```sql
CREATE TABLE table_name(
  col1 INT PRIMARY KEY,
  col2 INT NOT NULL,
  col3 INT NOT NULL,
  col4 VARCHAR(10),
  INDEX(col2,col3))
```
此外，针对某个column创建index的语句为：
```sql
CREATE INDEX index_name ON table_name(column_name)
```

**删除索引**
```sql
DROP INDEX index_name ON table_name
```

**显示索引**
```sql
SHOW INDEXES FROM table_name
```

**MySQL CREATE INDEX example**
现有一张员工表`emp`记载了员工个人信，如下表所示：
| EMPNO | ENAME  | JOB       | MGR  | HIREDATE   | SAL     | COMM    | DEPTNO |
|-------|--------|-----------|------|------------|---------|---------|--------|
|  7369 | SMITH  | CLERK     | 7902 | 1980-12-17 |  800.00 |    NULL |     20 |
|  7499 | ALLEN  | SALESMAN  | 7698 | 1981-02-20 | 1600.00 |  300.00 |     30 |
|  7521 | WARD   | SALESMAN  | 7698 | 1981-02-22 | 1250.00 |  500.00 |     30 |
|  7566 | JONES  | MANAGER   | 7839 | 1981-04-02 | 2975.00 |    NULL |     20 |
|  7654 | MARTIN | SALESMAN  | 7698 | 1981-09-28 | 1250.00 | 1400.00 |     30 |
|  7698 | BLAKE  | MANAGER   | 7839 | 1981-05-01 | 2850.00 |    NULL |     30 |
|  7782 | CLARK  | MANAGER   | 7839 | 1981-06-09 | 2450.00 |    NULL |     10 |
|  7788 | SCOTT  | ANALYST   | 7566 | 1987-04-19 | 3000.00 |    NULL |     20 |
|  7839 | KING   | PRESIDENT | NULL | 1981-11-17 | 5000.00 |    NULL |     10 |
|  7844 | TURNER | SALESMAN  | 7698 | 1981-09-08 | 1500.00 |    0.00 |     30 |
|  7876 | ADAMS  | CLERK     | 7788 | 1987-05-23 | 1100.00 |    NULL |     20 |
|  7900 | JAMES  | CLERK     | 7698 | 1981-12-03 |  950.00 |    NULL |     30 |
|  7902 | FORD   | ANALYST   | 7566 | 1981-12-03 | 3000.00 |    NULL |     20 |
|  7934 | MILLER | CLERK     | 7782 | 1982-01-23 | 1300.00 |    NULL |     10 |

query objective: 找出员工名称为`KING` 的员工信息。
如果没有给`ename`列创建索引的情况下，MySQL会对ename列从头到尾扫描，总共14行。来看以下代码：
*用explain语句查看数据检索信息*
```sql
explain select * from emp where ename = 'KING';
```
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
|----|-------------|-------|------------|------|---------------|------|---------|------|------|----------|-------------|
|  1 | SIMPLE      | emp   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   14 |    10.00 | Using where |
从rows这一列中，我们可以看出，没有创建索引的情况下，MySQL扫描了整张表14行记录。

*对ename列创建索引*
```sql
CREATE INDEX index_ename ON emp(ename);
```
*用explain解释查询语句*
```sql
explain select * from emp where ename='KING';
```
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
|----|-------------|-------|------------|------|---------------|------|---------|------|------|----------|-------------|
|  1 | SIMPLE      | emp   | NULL       | ref  | index_ename          | index_ename | 43    | const |   1 |    10.00 | NULL |
创建索引后，MySQL只扫描了一行就找出了员工King的数据。



**Reference**
- [tutorialspoint](https://www.tutorialspoint.com/mysql/mysql-indexes.htm)
- [MySQL Server](https://dev.mysql.com/doc/refman/8.0/en/create-index.html)

Welcome to Access My Github: [rayhezack](https://github.com/rayhezack)
