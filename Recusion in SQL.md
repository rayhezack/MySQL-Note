## An Introduction to Recursion in MySQL

In some situations, we probably need to draw upon recursion to product results of queries that meet the business needs. 

**So what is recursion in MySQL?**

*Recursion is a process that a defined function references /call itself within its function body*. For better understanding this conception, we can look at an example of *Fibonacci sequence* created by python
```
def fibonacci_sequence():
  """create a fibonacci array"""
  if n==0:
    return 0
  elif n==1:
    return 1
  elif n>1:
    return fibonacci_seq(n-1) + fibonacci_seq(n-2)
```
We can find that this function `fibonacci_sequence` is called by itself in the function body in the form of `fibonacci_seq(n-1) + fibonacci_seq(n-2)`. This is a recursion.

For MySQL, a typical recursive common table expression(CTE) is a CTE that has a subquery referring to its own name. This is the exactly same logic as what happened above. The syntax of of recursive CTE is:
```
WITH RECURSIVE cte_define_your_own_name(arg*) AS (
  initial_query
  union
  recursive query  # This query must includes a termination condition
)
SELECT * FROM cte_define_your_own_name
```
So we can summarize the main components of a recursive cte table as follows:
- To create a recursive cte table, we must add `RECURSIVE` keyword.
- An initial query will form the base result of the cte table for which you can give any name you want
- recursive query is a query that refers to its own name, whose logic is very similar to the `fibonacci_seq(n-1) + fibonacci_seq(n-2)`.  The result of the recursive query is concatenated with the initial query by `union` or `union all` keyword.
- A termination condition must be included in recursive query to ensure that the recursion stops.


**What are the scenarios in which we can use recursive cte table in MySQL?**
Recursive table can be highly useful in the following situations:
- Create an array
- Hierarchical or tree-structured data traversal


**How recursive cte table works?**

There is a fixed execution order of recursive cte table:
- First, the initial query is executed, and the result of it forms the base of the cte table; in other words, the cte table already has a result from the initial query when we move into the recursive query
- Then, the recursive cte query is executed and new result is produced using the base result
- Repeat the second step until the termination condition meets


Now let's start using it in MySQL

## Create Array
There must be a situation in which we need to create an array with a list of numbers. We would produce a number of duplicated codes and spend a huge amount of time if we did this without a recursive table.

**For example, please form an array from 1 to 100 using MySQL.**

This is the code without using recursion
```
select 1 as num
union
select 2 as num
union
select 3 as num
union
...
select 99 as num
union
select 100 as num
```
This is quite a boring job and time-consuming.

In contrast to this approach, a recursive cte table can simplify the job a lot.
```
WITH RECURSIVE numbers(n) AS (
SELECT 1
UNION
SELECT n+1 FROM numbers
WHERE n<100
)
SELECT * FROM numbers;
```
[output]
|n|
|-|
|1|
|2|
|.|
|.|
|.|
|100|

Now let's split the code into two parts for explanation:
First, the recursive cte table will execute the initial query `select 1`
```
WITH RECURSIVE numbers(n) AS (
SELECT 1
)
```
As a result of this, the table `numbers ` has one record 1 before the recursive query is executed. So the numbers table looks like this
|n|
|-|
|1|

Then the recursive query is executed, and new records are stored in the table until the recusion `n+1` creates 100 to terminate the recursion. So we have an array starting with 1 and ending at 100.


### Exercise of Array Generation
Now let's look at one difficult question from Leetcode. In this question, we need to use the method we just learned.

#### [Number of Transactions per Visit](https://leetcode-cn.com/problems/number-of-transactions-per-visit/)

A bank wants to draw a chart of the number of transactions bank visitors did in one visit to the bank and the corresponding number of visitors who have done this number of transactions in one visit.

**Write an SQL query to find how many users visited the bank and didn't do any transactions, how many visited the bank and did one transaction and so on.**

The result table will contain two columns:

transactions_count which is the number of transactions done in one visit.
visits_count which is the corresponding number of users who did transactions_count in one visit to the bank.
**transactions_count should take all values from 0 to max(transactions_count) done by one or more users.**

Order the result table by transactions_count.

`visits`

| user_id | visit_date |
|---------|------------|
| 1       | 2020-01-01 |
| 2       | 2020-01-02 |
| 12      | 2020-01-01 |
| 19      | 2020-01-03 |
| 1       | 2020-01-02 |
| 2       | 2020-01-03 |
| 1       | 2020-01-04 |
| 7       | 2020-01-11 |
| 9       | 2020-01-25 |
(user_id, visit_date) is the primary key for this table.

`Transactions`

| user_id | transaction_date | amount |
|---------|------------------|--------|
| 1       | 2020-01-02       | 120    |
| 2       | 2020-01-03       | 22     |
| 7       | 2020-01-11       | 232    |
| 1       | 2020-01-04       | 7      |
| 9       | 2020-01-25       | 33     |
| 9       | 2020-01-25       | 66     |
| 8       | 2020-01-28       | 1      |
| 9       | 2020-01-25       | 99     |
There is no primary key in this table

**Explanation**
The question requires us to compute two columns, one for a possible number of transactions and the other one for a corresponding number of customers who have done such number of transactions.
So first we need to know what are the **possible number of transactions** for each customer. To compute this, we need to know the relationship between the two tables.
- Just look at the `visit` table(). (user_id, visit_date) is the primary key for this table, suggesting that one customer may visit a bank at a different date. 
- And there is no primary key in the `transactions` table, suggesting that one customer may carry out complete multiple transactions at the same day.
- One customer may complete transactions in one visit to bank or may not do any transactions in one visit. This directly indicates that there is one situation in which records in the `visit` table do not appear in the `transactions` table because some customers do not do any transaction, so we have one possible number of 0 transactions.

After organizing the relationship, we should know that we have outer join the two tables use `visits` table as the main table so that we can compute the number of transactions per day for **each customer** who either do transactions or do not. So we have
```
select
    v.user_id,
    v.visit_date,
    count(t.transaction_date) as transaction_cnt 
from visits v 
left join transactions t
on v.user_id = t.user_id
and v.visit_date = t.transaction_date
group by v.user_id,v.visit_date
```
Here's the result:
![temp](https://upload-images.jianshu.io/upload_images/10429581-dd31a96981779527.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
*Note: We name the result table as `temp`.*

We can find that the possible number of transactions includes 0, 1, and 3. But remember there is one requirement that **transactions_count should take all values from 0 to max(transactions_count) done by one or more users.** So we need to create an array starting from 0 and ending at 3. And to generate a series, we definitely need to write a recursive cte table.  
To create a recursive cte number, we need to know the termination condition. The condition should be `n<max(number_of_transactions)`.  And the maximum number can be accessed by using `max()` function in the result table we just query. So we have:
```
with recursive cte(n) as(
    select 0 
    union
    select n+1
    from cte
    where n<(
        select max(transaction_cnt)
        from (select
                v.user_id,
                v.visit_date,
                ifnull(count(t.transaction_date),0) as transaction_cnt 
            from visits v 
            left join transactions t
            on v.user_id = t.user_id
            and v.visit_date = t.transaction_date
            group by v.user_id,v.visit_date
        ) t
    )
)
SELECT * FROM cte;
```
So we get one table with 0,1,2,3
|n|
|-|
|0|
|1|
|2|
|3|

So we can compute the corresponding number of customers who have done these transactions. How to compute the numbers?

- Outer join the `cte` table and `temp` based on `cte`, so that we can look up the transaction information for each number of transaction, including 2 that does appear in `temp` table. In this setting, all transaction information for transaction number 2 will be NULL.
- `Group by` the transactions count.
- `Count` the transaction date because each transaction date corresponds to one transaction.

Based on the code we have written, We can get the final answer:
```
with recursive cte(n) as(
    select 0 
    union all
    select n+1
    from cte
    where n<(
        select max(transaction_cnt)
        from (select
                v.user_id,
                v.visit_date,
                ifnull(count(t.transaction_date),0) as transaction_cnt 
            from visits v 
            left join transactions t
            on v.user_id = t.user_id
            and v.visit_date = t.transaction_date
            group by v.user_id,v.visit_date
        ) t
    )
)

select
    c.n as transactions_count,
    count(t.visit_date) as visits_count
from cte c
left join (select
            v.user_id,
            v.visit_date,
            ifnull(count(t.transaction_date),0) as transaction_cnt 
        from visits v 
        left join transactions t
        on v.user_id = t.user_id
        and v.visit_date = t.transaction_date
        group by v.user_id,v.visit_date) t
on c.n = t.transaction_cnt
group by c.n
order by transactions_count;
```
[OUTPUT]
| transactions_count | visits_count |
|--------------------|--------------|
| 0                  | 4            |
| 1                  | 5            |
| 2                  | 0            |
| 3                  | 1            |


## Hierarchical or tree-structured data traversal 
Let's take a look at the other situation, organization structure.
![image.png](https://upload-images.jianshu.io/upload_images/10429581-d6ac0c90fa06bff0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Perhaps, your boss requires you to produce one report of the organizational hierarchy: who is the boss of the organization, who should directly report to the boss, and who should report to the managers reporting to boss? Just as the picture above shows: the Mattchew is the boss of the company and Carolline and Tom need to directly report to Matthew. For Carolline and Tom, there are also some employees who need to report to them.....
To find the relationship, we can use a recursive cte table to solve this.

**Example**
`tree`
|node|parent|
|----|----|
|1|NULL|
|2|1|
|3|1|
|4|2|
|5|2|
|6|3|
|7|3|

Here is the organizational hierarchy
![hierarhcy](https://upload-images.jianshu.io/upload_images/10429581-46daef485e90c194.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**Solution**

First, we need to write the initial query. Here, the initial query is the root node of the tree. We can easily find from the graph that the root node is 1 because 1 does not have a parent node. 
```
with recursive cte(node,path,level) as (
select node,cast(node as char(30)),1
from tree
where parent is null
)
```
The initial query will return the record 1. Then we write the recursive query. Note that to find the managers who directly report to boss 1 we need to join tables on the condition that `node` of cte table = `parent` of tree table because the node_id of 1 equals the parent if of node 2 and node 3. So we have
```
with recursive cte(node,path,level) as (
select node,cast(node as char(30)),1
from tree
where parent is null
union all
select
	t.node,concat(c.path,"->",t.node),c.level+1
from cte c
join tree t
on c.node=t.parent
)
select * from cte;
```
In this case, the recursion will execute until it cannot find any direct relationship.

Here's the result
![image.png](https://upload-images.jianshu.io/upload_images/10429581-7f072986177dc107.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
