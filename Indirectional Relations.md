# Indirectional Relations

In a table of database, the relation of two fields, in some cases, is indirectional. 

## Understanding of Indirectional Relation

**What does the word 'indirectional' mean?**

Suppose that there are two fileds user1_id and user2_id and that each row of record indictates a friendship, 
indirectional means that the fact that user1_id has a friend called user2_id can be also understanded as user2 has a friend, user1. 
We may encounter this indirectional relationship in many situations, such as a match between two teams, and a call between two persons.
To better undetstand the kind of association, let us look at the table below:

| user1_id | user2_id |
|----------|----------|
| 1        | 2        |
| 1        | 3        |
| 2        | 3        |
| 1        | 4        |
| 2        | 4        |
| 1        | 5        |
*Table1*

**Explanation:**

The first row indicate that user 1 has a friend user 2, and the relation also means that user 2 has a friend user 1.
Analogously, user 1 has a friend user 3 also implies that user 3 has a friend user 1.


**What is different for the indirectional relationship?**

We will first write `union` or `union all` to find all possible mutual relationships if we encounter a business context in which 
the boss requires us to write a report based on a table with indirectional relationship. The explanation might look very
abstract, so let me introduce the difference using the table1 above:

`Friendship`
| user1_id | user2_id |
|----------|----------|
| 1        | 2        |
| 1        | 3        |
| 2        | 3        |
| 1        | 4        |
| 2        | 4        |
| 1        | 5        |
| 2        | 5        |
| 1        | 7        |
| 3        | 7        |
| 1        | 6        |
| 3        | 6        |
| 2        | 6        |

If the question requires us to count the number of friends for each user in user1_id field, we may code like the following thing if we 
do not consider `union`.
```
select
   user1_id,
   count(user2_id) as friend_cnt
from friendship
group by user1_id
```
The result is definitely incorrect because the solution can only take the relation from user1_id to user2_id,
into account. Take the user 2 as an example, the third, fifth, seventh, and last row in the left column(user1_id) 
tell us that user 2 has four friends, user 3, user 4, user 5 and user 6. But user 1 is also a friend for user 2,
because 2 also appears in the right column(user2_id).

So I believe that you have seen what is wrong here. The code above will give us a number smaller than the true figure.
This is why we need to union the two fields to produce one full list of friendships. Please consult the code below:
```
select * from friendship
union all
select user2_id as user1_id, user1_id as user2_id from friendship
```
The code gives us a table displaying all friendships. *Note: the table is very long, so I just list the section for user 2
for demonstration.*

| user1_id | user2_id |
|----------|----------|
| 2        | 3        |
| 2        | 4        |
| 2        | 5        |
| 2        | 6        |
| 2        | 1        |



## Practice

Now let's move onto the practice section to apply the knowledge we just learned into practice. Here I will list some 
questions from Leetcode, and use them for the purpose of only demonstration.



**Question 1:** [**Strong Friendship**](https://leetcode-cn.com/problems/strong-friendship/)

| Column Name | Type |
|-------------|------|
| user1_id    | int  |
| user2_id    | int  |

(user1_id, user2_id) is the primary key for this table.
Each row of this table indicates that the users user1_id and user2_id are friends.
*Note that user1_id < user2_id.*

A friendship between a pair of friends x and y is strong if x and y have at least three common friends.
Write an SQL query to find all the strong friendships. Note that the result table should not contain 
duplicates with user1_id < user2_id.

**Solution:**

The key to resolving the question lies in the understanding of common friends. 

```
# list all possible friendships
with temp as (select user1_id, user2_id from friendship
            union all
            select user2_id user1_id, user1_id user2_id from friendship
    )

select
    user1_a user1_id,
    user2_a user2_id,
    count(*) common_friend
from (select
        t1.user1_id user1_a,t1.user2_id user1_b,
        t2.user1_id user2_a,t2.user2_id user2_b
    from temp t1
    join temp t2
    on t1.user2_id = t2.user1_id
    and t1.user1_id<>t2.user1_id
    ) t
join temp f
on t.user1_a = f.user1_id
and user2_b = f.user2_id
where user1_a<user2_a
group by user1_a,user2_a
having count(*)>=3;
```
