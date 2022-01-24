# Introduction to the solution to Query of Consecutive Records

When writing queries based on business needs, we are very likely to create some queries, in which a certain number/ID appears multiple times consecutively, or ones, in which a number/date is increasing in ascending order. We need to write a query to filter out these records. 

For example, our boss requires us to find out active users that log in to our game, let's say Hornor of Kings. In this setting, a particular user, let's call him user 1, may log in in our game for consecutive 5 days. Then the data will look like this.
|User_id|date|
|-----|-----|
|1|2021-07-01|
|1|2021-07-02|
|1|2021-07-03|
|1|2021-07-04|
|1|2021-07-05|

Assume that (user_id,date) is the primary key. We can call such kind of user an active user.

The solution to this type of question can be summarized as follows:
- Create a column, which we can call tag, to determine whether a number/id appears consecutively
  - To add such column, we can define variables and use the window function
- Create a temporary table wrapping the query result. For example `with temp as (query)`
- Filter out records that question asks us to find out based on the `temp` table.



*Any method is unjustified and unclear if there is no question demonstrating it.* 

**Question 1: Consecutive Numbers**
Let's look at one problem from Leetcode: Please find numbers that appear at least three times consecutively.
`Logs`
| Id | Num |
|----|-----|
| 1  | 1   |
| 2  | 1   |
| 3  | 1   |
| 4  | 2   |
| 5  | 1   |
| 6  | 2   |
| 7  | 2   |

Id is the primary key of this table. Write an SQL query to find all numbers that appear at least three times consecutively. Return the result table in any order.

**Solution1: Variables**
Let me explain how to solve this question using variables. Remember that our objective is to create one column that serves the purpose of automatically calculating the frequency.
- First define variable `@num:=0,@cnt` where `:=` is assignment and `@` is a variable keyword immediately followed by variable_name. So `@num:=0` means that we assign 0 to the variable `@num`
  - `@num` is used to record which number we are checking. It needs to be assigned to a value of new record each time the SQL select a new record. For example, when SQL moves from the first row to the second row, @id should be assigned to 1 in the second row. 
  - `@cnt` is used to record the frequency that a number appears. For example, for the first three rows, we can find that Num has records 1, 1, and 1. So `@cnt` should be 1,2,3. 
```Mysql
SELECT
  *
FROM Logs,(select @num:=0,@cnt:=0) t
```
In this sql, `select @num:=0,@cnt:=0` is the syntax of defining variables. We have created two variables, num and cnt. Then we need to figure out the logic: @cnt can increase by 1 only when @num equals a record of Num.
```Mysql
SELECT
  *,
  case when @num=num then @cnt:=@cnt+1
           else @cnt:=1
  end as tag
FROM Logs,(select @num:=0,@cnt:=0) t
```
|Id|Num|tag|
|---|---|---|
|1|1|1|
|2|1|2|
|3|1|3|
|4|2|1|
|5|1|1|
|6|2|1|
|7|2|2|

Then we create a temporary table and find out the number that appears three times consecutively through `tag` column.
```Mysql
select
  distinct num ConsecutiveNums
from (SELECT
  *,
  case when @num=num then @cnt:=@cnt+1
           else @cnt:=1
  end as tag
FROM Logs,(select @num:=0,@cnt:=0) t) t1
where t1.tag>=3
```
*Note: we must use `distinct` to drop any duplicated records. The reason for duplication is that a particular number may appear three times consecutively for id 1, 2, 3, and this can be the same case for id 7,8,9.*


**Solution2: Window Function**
`lag(exp,n)` is used to return the value from the row that precedes the current row
`lead(exp,n)` is used to return the value from the row that succeeds the current row
So the general idea of using window function is that we find out the value from the row that precedes the current row and the value from the row that succeeds the current row. If the value of the three columns is equal, suggesting that the value appears at least three times consecutively.

- *First step: create two columns for the value of the previous row and succeeding row*
```Mysql
select
  num,
  lag(num,1) over (order by id asc)  as preceding_num,
  lead(num,1) over (order by id asc) as succeeding_num
from logs
```

- *Second Step: create a temporary table that stores the query result*
```Mysql
with temp as (select num,
  lag(num,1) over (order by id asc)  as preceding_num,
  lead(num,1) over (order by id asc) as succeeding_num
from logs)
```

- *Third Step: filter out values from the temp table by num=preceding_num and num=succeeding_num*
```Mysql
with temp as (select num,
  lag(num,1) over (order by id asc)  as preceding_num,
  lead(num,1) over (order by id asc) as succeeding_num
from logs)

select
  distinct num as ConsecutiveNums
from temp
where num = preceding_num
and num = succeeding_num;
```

**Question 2: Active Users**
`Accounts`
| id | name     |
|----|----------|
| 1  | Winston  |
| 7  | Jonathan |


`Logins` 
| id | login_date |
|----|------------|
| 7  | 2020-05-30 |
| 1  | 2020-05-30 |
| 7  | 2020-05-31 |
| 7  | 2020-06-01 |
| 7  | 2020-06-02 |
| 7  | 2020-06-02 |
| 7  | 2020-06-03 |
| 1  | 2020-06-07 |
| 7  | 2020-06-10 |

No primary key in this table. (This suggests that there might be duplicated records. So we need to be cautious)


First, let's find out the difference between the question and the first question.
When we try to compute the frequency of a number appearing in the first question, we only need to examine whether the value of the current row is the same as that from the preceding row and succeeding row. However, the second question requires us not only to examine this situation but also to check whether the data is consecutive.

*Then, let me give the solution using window function.*
If we try to find out numbers appearing in consecutive dates, there is one fixed thought on the type of question: calculate the row_number for each record, and then subtract the date, which can be either date or number, from the row_number. If a number appears for consecutive days, meaning that a number appears multiple times at a given number/date increasing by fixed step, **the subtraction will be equal for this number.** 
For example, we want to find out users log in for consecutive three days
| id | login_date |
|----|------------|
| 7  | 2020-05-31 |
| 7  | 2020-06-01 |
| 7  | 2020-06-02 |

We can find that the login_date, for id 7, is consecutive. In this case, the subtraction will be 2020-05-30 for the three records, if I add one column row_number for id using the window function.
| id | login_date |row_number|Diff|
|----|------------|------------|------------|
| 7  | 2020-05-31 |1|2020-05-30|
| 7  | 2020-06-01 |2|2020-05-30|
| 7  | 2020-06-02 |3|2020-05-30|

Then we can group the data by `id,Diff` and filter out any id with frequency(`count(*)`) greater than or equal 3.

So the procedures of solving this question can be summarized as follows:
- Create `row_number` to compute the difference between row_number and login_date
```Mysql
select
  num,
  row_number() over (partition by id order by login_date asc)
from logins
```
- Substrat the `login_date` from `row_number` to create one tag that help determine which user consecutively log in. Here we need to use `date_sub(exp, interval x date_type)` function when working with datetime data.
```Mysql
select
        *,
        date_sub(login_date,Interval row_number() over (partition by id order by login_date asc) DAY ) tag 
from (select distinct id,login_date from logins)
```

- `Group` the table by `id,diff` to filter out active users
```Mysql
select
    distinct t1.id,a.name
from (select
        *,
        date_sub(login_date,Interval row_number() over (partition by id order by login_date asc) DAY ) tag 
    from (select distinct id,login_date from logins)t ) t1
join accounts a
on a.id = t1.id
group by t1.id,t1.tag
having count(t1.tag)>=5;
```
*Note: we must use the `distinct` keyword to drop duplicated records because of no primary key in the login table.*


There is one a little bit difficult question from Leetcode, You can use this question for practice this kind of question.
## [601\. Q3: Human Traffic of Stadium](https://leetcode-cn.com/problems/human-traffic-of-stadium/)
Write an SQL query to display the records with three or more rows with consecutive id's, and the number of people is greater than or equal to 100 for each.

Return the result table ordered by visit_date in ascending order. The query result format is in the following example.
`Stadium`
| id   | visit_date | people    |
|------|------------|-----------|
| 1    | 2017-01-01 | 10        |
| 2    | 2017-01-02 | 109       |
| 3    | 2017-01-03 | 150       |
| 4    | 2017-01-04 | 99        |
| 5    | 2017-01-05 | 145       |
| 6    | 2017-01-06 | 1455      |
| 7    | 2017-01-07 | 199       |
| 8    | 2017-01-09 | 188       |

visit_date is the primary key for this table.
Each row of this table contains the visit date and visit id to the stadium with the number of people during the visit.
No two rows will have the same visit_date, and as the id increases, the dates increase as well.

Here's the solution.(You can refer)
```Mysql
with temp as (select
                *,
                id - row_number() over (order by visit_date asc) as flag
            from stadium
            where people>=100
),
temp1 as (select
            flag
        from temp
        group by flag
        having count(*)>=3)
select
    id,
    visit_date,
    people
from temp
where flag in (select flag from temp1)
order by visit_date asc;
```
