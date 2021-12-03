When we need to reshape the dataframe such that the rearranged data frame can satisfy the needs for information dissemination or for data analytics, there is two directions of the pivoting table:

- Pivoting from rows in a dataset into columns
- Pivoting from columns in a dataset into rows

The former rearrangement of the dataset is referred to *Pivoting Long to Wide Format*, and the latter one is referred to *Pivoting Wide to Long Format*. 

**So what is a wide table and what is a long table?**

## Long Table and Wide Table

### Long Table
A long table is an unprocessed, original dataset, so it is the best candidate for data analytics. 

`Scores`
|student	|school	|class	|grade|
|------------|------------|------------|------------|
|Andy    |	Z            |	english|	90|
|	Bernie|	Y|	english|	80|
|	Cindy|	Z|	english|	70|
|	Deb|	Y|	english|	65|
|	Andy|	Z|	math|	60|
|	Bernie|	Y|	math|	55|
|	Cindy|	Z|	math|	100|
|	Deb|	Y|	math|	98|
|	Andy|	Z|	physics|	87|
|	Bernie|	Y|	physics|	100|
|	Cindy|	Z|	physics|	45|
|	Deb|	Y|	physics|	99|

`Products`
| product_id  | store  | price |
|-------------|--------|-------|
| 0           | store1 | 95    |
| 0           | store3 | 105   |
| 0           | store2 | 100   |
| 1           | store1 | 70    |
| 1           | store3 | 80    |



### Wide Table
A wide table is used for disseminating information because the data is usually processed in a way that can increase people's knowledge of something. Also, the wide table is a particular form of dataframe containing one column by one dimension and indexed by another dimension in the column. For example, there are two typical wide tables:

`Scores`
| product_id  | English | Math | Physics |
|-------------|--------|--------|--------|
| Andy           | 90     | 60    | 87    |
| Bernie           | 80     | 55   | 100     |
| Cindy           | 70     | 100   | 45     |
| Deb           | 65     | 98   | 99     |

The table shows scores per subject indexed by each student. 

`Products`
| product_id  | store1 | store2 | store3 |
|-------------|--------|--------|--------|
| 0           | 95     | 100    | 105    |
| 1           | 70     | null   | 80     |

The Products table contains each column per store indexed by per product. Looking at the table, we can easily know what are the prices of each product across all stores.

## Converting between Long Table and Wide Table
Let's say we have the two unprocessed dataframes mentioned above, `products` and `scores`. Now we can convert them into wide table at will and perform the reverse operation, coverting the wide table to long format.


### Converting to Wide Format
First, Let's take the `products` table as an example to demonstrate how the transformation works in Python and MySQL.

#### MySQL
`products`
| product_id  | store  | price |
|-------------|--------|-------|
| 0           | store1 | 95    |
| 0           | store3 | 105   |
| 0           | store2 | 100   |
| 1           | store1 | 70    |
| 1           | store3 | 80    |

Now we need to write an SQL query to find the price of each product in each store.

First let's quickly review the essence of a wide table, which shows information per column and is indexed by a particular dimension. So our goal is to **group the price data by product_id** and **create 3 new files** for store1, store2, and store3.

**So how to create fields that can automatically identify which record is store1 or store2?**
The answer is to write a control flow. In MySQL, we need to use `case when / if` to create the required fields. Either way works in this case.

First, we group the dataset by `product_id` so that we can index each record by product_id.
```
select
  xx
from products
group by product_id
```

Then, based on the code chunk above, we need to write control flow to create new fields for store1, store2, and store3. Here I use `case when` as the demonstration:
```
select
  product_id,
  case when store="store1" then price else null end as `store1`,
  case when store="store2" then price else null end as `store2`,
  case when store="store3" then price else null end as `store3`,
from products
group by product_id
```

Are we done? Absolutely not, we must use the aggregation function after we group by the dataset; otherwise, it only returns one record. And it does not matter what kinds of `aggregate function` we use, we can either write `max` or `min`.
```
select
  product_id,
  max(case when store="store1" then price else null end) as `store1`,
  max(case when store="store2" then price else null end) as `store2`,
  max(case when store="store3" then price else null end) as `store3`,
from products
group by product_id
```
The output is as follows:
| product_id  | store1 | store2 | store3 |
|-------------|--------|--------|--------|
| 0           | 95     | 100    | 105    |
| 1           | 70     | null   | 80     |

#### Python
In python, the question becomes extremely simple because pandas have its built-in function especially for pivoting data frame. We can either use the top-level function `pd.pivot_table` or the instance function `df.pivot_table`. Either way works.

Here is the syntax for pivotting long to wide format.
`pd.pivot_table(df,index,columns,values,margins,aggfunc,fill_value)`
- `df` is the dataframe needed to be transformed, that is a long table
- `index` means what field we use to index the dataframe? It is equivalent to `group by ` of MySQL
- `columns` means which column we want to display the data by column. It is equivalent to the new fields we create by control flow in MySQL.
- `values` means which columns of the dataframe we want to aggregate?
- `aggfunc` specifies the aggregate function we want to use, such as `max`,`min`,`first`,`mean`.
- `fill_value` specifies whether we want to use a certain value to fill with NaN.
- `margins` specifies whether we want to add the subtotal.

Now let's pivot the products table to a wide format using this function
```
pd.pivot(Products,"product_id","store","price")
```
[Output]
|store|	store1|	store2|	store3|
|-----|-----|-----|-----|	
|product_id| | | |		
|0|	95.0|	100.0|	105.0|
|1	|70.0	|NaN|	80.0|


### Converting to Long Format
Now the long table has been transformed to the wide format, we can also recover it if we want. 

#### MySQL
Write an SQL query to rearrange the Products table so that each row has (product_id, store, price). If a product is not available in a store, do not include a row with that product_id and store combination in the result table.

In MySQL, the job is rather boring. We only need to use `union` to concatenate all possible store tables. I directly show how it works:

Based on the wide table above, first, we filter out the product price of products sold in store1 and name it as `price`, and we create `store` as a new field.
```

select
    product_id,
    "store1" as a store,
    store1 as price
from products
where store1 is not null
```

Then we write all possible situations and concatenate them
```

select
    product_id,
    "store1" as a store,
    store1 as price
from products
where store1 is not null
union all
select
    product_id,
    "store2" as a store,
    store2 as price
from products
where store2 is not null
union all
select
    product_id,
    "store3" as a store,
    store3 as price
from products
where store3 is not null;
```

#### Python
Python has a built-in function, `pd.melt()`, to convert a wide format table to a long table.

*Here's the syntax*
`pd.melt(df,id_vars,value_vars,var_name,value_name,ignore_index)`
- `df` is the wide table needed to be melted
- `id_vars` specifies the row of the new dataframe
- `value_vars` specifies which columns need to be unpivoted. If not specified, all other columns other than id_vars will be unpivoted. 
- `value_name` specifies the name of new value column
- `var_name` specifies the name of new variable column
- `ignore_index` specifies whether we ignore the original index. Default is `True`

Now Let's unpivot the dataframe
```
melted_df = pd.melt(Products,id_vars=["product_id"],
                    value_vars=["store1","store2","store3"],
                    var_name="store",value_name="price",
                   ignore_index=True)
```

Then we filter out any null values
```
melted_df = pd.melt(Products,id_vars=["product_id"],
                    value_vars=["store1","store2","store3"],
                    var_name="store",value_name="price",
                   ignore_index=True)
result = melted_df[melted_df["price"].notnull()].sort_values(by="product_id")
result
```
[OUTPUT]
|product_id|	store|	price|
|-----|-----|-----|
|0|	store1|	95.0|
|0|	store2|	100.0|
|0|	store3|	105.0|
|1|	store1|	70.0|
|1|	store3|	80.0|


We can do the same transformation for the `score` table using the same way, which is rather tedious. Now, let's try a much more complicated questions involving the advanced use of pandas and mysql.


## Advanced Use of Pandas and MySQL in Pivotting
Let's try some difficult questions in Leetcode, and I will use Python and MySQL to solve these questions to show how MySQL and Pandas work for pivotting and unpivotting.


### Question1:  [Students Report By Geography](https://leetcode-cn.com/problems/students-report-by-geography/)

#### MySQL

A U.S graduate school has students from Asia, Europe and America. The students' location information are stored in table student as below.
`student`

| name   | continent |
|--------|-----------|
| Jack   | America   |
| Pascal | Europe    |
| Xi     | Asia      |
| Jane   | America   |

We need to pivot the continent column in this table so that each name is sorted alphabetically and displayed underneath its corresponding continent. The output headers should be America, Asia and Europe respectively. It is guaranteed that the student number from America is no less than either Asia or Europe.

The result should be like this:
| America | Asia | Europe |
|---------|------|--------|
| Jack    | Xi   | Pascal |
| Jane    |      |        |

**Explanation**
We can easily find that this is a typical question of converting long to wide format. So the method of solving this question is absolutely  to  `group by` and `control flow`.
But the problem is that the original table only has two columns, one used for displaying data by columns and the other one for aggregating. So we do not have a column that we can use to group the dataset. In this case, we need to construct such a column ourselves. 

So we need to first rank the name by continent in ascending order, so we will have the result:
`Jack is ranked as 1, while Jane is ranked as 2;`
`Pascal is ranked as 1 because Europe has only one student displayed`;
`Xi is ranked as 1 because Asia has only one student displayed.`

Then, we can group the dataset by the rank column and use the exactly the same way we have practiced to solve this question.

**Code**

To get a rank field, we can either use a window function or variables.

*Window Function*
```
select
  *,
  row_number() over (partition by continent order by name asc) as "rank"
from student
```

*We can also define variables to add rank field*
```
select
  *,
  case when @cont=continent then @rk:=@rk+1
          else @rk:=1
  end as "rank",
  @cont:=continent
from student,(select @rk:=0,@cont:=null) t
order by continent, name asc
```

Either way, we can have the following table:
| name   | continent |Rank|
|--------|-----------|-----------|
| Jack   | America   |1|
| Jane   | America   |2|
| Pascal | Europe    |1|
| Xi     | Asia      |1|

Now the left job is just to reproduce the code I have written in the demonstration part.

`window function version`
```
# build one temp table to store the table containing rank
with temp as (
select
*,
  row_number() over (partition by continent order by name asc) as "rank"
from student)

select
  max(case when continent="America" then name else null end) as America,
  max(case when continent="Asia" then name else null end) as Asia,
  max(case when continent="Europe" then name else null end) as Europe
from temp
group by rank
```

`variable version`
```
with temp as (select
                *,
                case when @continent=continent then @rk:=@rk+1 else @rk:=1 end as rk,
                @continent:=continent
            from student,(select @continent:=null,@rk:=0) t
            order by continent asc,name asc)
select
    max(case when continent="America" then name else null end) as America,
    max(case when continent="Asia" then name else null end) as Asia,
    max(case when continent="Europe" then name else null end) as Europe
from temp
group by rk;
```

*Note: DO NOT FORGET TO ADD AGGREGATE FUNCTION*

#### Python
Now, let me use python to solve this question, which can be simpler than the solution of MySQL.

Same way. We need to use `pd.pivot_table` to pivot the dataframe.
```
# first we prepare the dataframe
Student = pd.DataFrame({"name":["Jack","Pascal","Xi","Jane"],
                        "continent":["America","Europe","Asia","America"]})
```

Now we copy the dataframe to avoid any manipulation affecting the original dataset
```
student = Student.copy()
```

Then, we also need to create one column used to group the data. Here's one powerful function `rank`
```
# set ascending=True to ensure that we order name alphabetically
student["rank"] = student["name"].groupby(student["continent"]).\
rank(method="first",ascending=True) 
```

Next, we call `pd.pivot_table` to perform pivoting.
```
result = student.pivot_table(
    index="rank",
    columns="continent",
    values="name",
    aggfunc="max",
    fill_value="")
```
[Ouput]
|continent|	America|	Asia|	Europe|
|-----|-----|-----|-----|
|rank||||			
|1.0|	Jack|	Xi|	Pascal|
|2.0|Jane|||

Tweak the result a little bit
```
res = result.reset_index().drop("rank",axis=1)
res.columns.names = ""
res
```
[Output]
||	America|	Asia|	Europe|
|-----|-----|-----|-----|
|0|	Jack|	Xi|	Pascal|
|1|	Jane|		||


### Question 2:[Sales by Day of the Week](https://leetcode-cn.com/problems/sales-by-day-of-the-week/)

`Orders`
| order_id   | customer_id  | order_date  | item_id      | quantity    |
|------------|--------------|-------------|--------------|-------------|
| 1          | 1            | 2020-06-01  | 1            | 10          |
| 2          | 1            | 2020-06-08  | 2            | 10          |
| 3          | 2            | 2020-06-02  | 1            | 5           |
| 4          | 3            | 2020-06-03  | 3            | 5           |
| 5          | 4            | 2020-06-04  | 4            | 1           |
| 6          | 4            | 2020-06-05  | 5            | 5           |
| 7          | 5            | 2020-06-05  | 1            | 10          |
| 8          | 5            | 2020-06-14  | 4            | 5           |
| 9          | 5            | 2020-06-21  | 3            | 5           |

`items`
| item_id    | item_name      | item_category |
|------------|----------------|---------------|
| 1          | LC Alg. Book   | Book          |
| 2          | LC DB. Book    | Book          |
| 3          | LC SmarthPhone | Phone         |
| 4          | LC Phone 2020  | Phone         |
| 5          | LC SmartGlass  | Glasses       |
| 6          | LC T-Shirt XL  | T-Shirt       |

You are the business owner and would like to obtain a sales report for category items and day of the week. Write an SQL query to report how many units in each category have been ordered on each day of the week. Return the result table ordered by category. The query result format is in the following example:

**Solution**
The result table requires us to display the sales indexed by each category of items per day of a week. So this is a typical question of pivoting a long table to a wide table. So
- First Step: Group by the data by category**
```
select
  xx
from orders o 
right join items i 
on o.item_id = i.item_id
group by i.item_category
order by category
```
- *Second Step: Basd on the joined table, write a control flow to automatically calculate the sum of quantity for each category*
```
select 
        i.item_category category,
        sum(if(weekday(o.order_date)=0,o.quantity,0)) 'Monday',
        sum(if(weekday(o.order_date)=1,o.quantity,0)) 'Tuesday',
        sum(if(weekday(o.order_date)=2,o.quantity,0)) 'Wednesday',
        sum(if(weekday(o.order_date)=3,o.quantity,0)) 'Thursday',
        sum(if(weekday(o.order_date)=4,o.quantity,0)) 'Friday',
        sum(if(weekday(o.order_date)=5,o.quantity,0)) 'Saturday',
        sum(if(weekday(o.order_date)=6,o.quantity,0)) 'Sunday'
from orders o 
right join items i 
on o.item_id = i.item_id
group by item_category
order by category
```
