There are three questions that can be considered the hardest ones in Leetcode because they contain basically all-important skills we need to master as a data analyst proficient with SQL queries.

The main knowledge includes:
- Recursion
- Window Function
- The difference between `Outer Join` and `Inner Join`
- Date and time functions
- Control flow: if/case when
- subquery

## [Hopper Company Queries I](https://leetcode-cn.com/problems/hopper-company-queries-i/)

`Drivers`

| driver_id | join_date  |
|-----------|------------|
| 10        | 2019-12-10 |
| 8         | 2020-1-13  |
| 5         | 2020-2-16  |
| 7         | 2020-3-8   |
| 4         | 2020-5-17  |
| 1         | 2020-10-24 |
| 6         | 2021-1-5   |
driver_id is the primary key for this table.

`Rides`

| ride_id | user_id | requested_at |
|---------|---------|--------------|
| 6       | 75      | 2019-12-9    |
| 1       | 54      | 2020-2-9     |
| 10      | 63      | 2020-3-4     |
| 19      | 39      | 2020-4-6     |
| 3       | 41      | 2020-6-3     |
| 13      | 52      | 2020-6-22    |
| 7       | 69      | 2020-7-16    |
| 17      | 70      | 2020-8-25    |
| 20      | 81      | 2020-11-2    |
| 5       | 57      | 2020-11-9    |
| 2       | 42      | 2020-12-9    |
| 11      | 68      | 2021-1-11    |
| 15      | 32      | 2021-1-17    |
| 12      | 11      | 2021-1-19    |
| 14      | 18      | 2021-1-27    |
ride_id is the primary key for this table.

`AcceptedRides`

| ride_id | driver_id | ride_distance | ride_duration |
|---------|-----------|---------------|---------------|
| 10      | 10        | 63            | 38            |
| 13      | 10        | 73            | 96            |
| 7       | 8         | 100           | 28            |
| 17      | 7         | 119           | 68            |
| 20      | 1         | 121           | 92            |
| 5       | 7         | 42            | 101           |
| 2       | 4         | 6             | 38            |
| 11      | 8         | 37            | 43            |
| 15      | 8         | 108           | 82            |
| 12      | 8         | 38            | 34            |
| 14      | 1         | 90            | 74            |
ride_id is the primary key for this table.


Write an SQL query to report the following statistics for each month of 2020:

The number of drivers currently with the Hopper company by the end of the month (active_drivers).
The number of accepted rides in that month (accepted_rides).
Return the result table ordered by month in ascending order, where month is the month's number (January is 1, February is 2, etc.).


`result`

| month | active_drivers | accepted_rides |
|-------|----------------|----------------|
| 1     | 2              | 0              |
| 2     | 3              | 0              |
| 3     | 4              | 1              |
| 4     | 4              | 0              |
| 5     | 5              | 0              |
| 6     | 5              | 1              |
| 7     | 5              | 1              |
| 8     | 5              | 1              |
| 9     | 5              | 0              |
| 10    | 6              | 0              |
| 11    | 6              | 2              |
| 12    | 6              | 1              |


Let's analyze the question step by step. We need to compute two things:
- The number of active drivers by the end of **each month**(active drivers)
- The number of accepted rides in **that month**(accepted rides)

**We separate the question into two parts: active drivers and accepted rides**. 
- The number of active drivers can be computed by the `Drivers` table, but we need to notice that the table does not include full months of 2020; in other words, the `join_date` in the `Drivers`  table has only five months, 1,2,3,5, and 10, but it does not have any record with months 4,6,7,8,9, and 12. In this case, we have to **create one field** that includes full months of a year so that we can compute the number of drivers for each month. In addition, the required result is the number of drivers by the end of that month, so we have to draw upon `window function` to compute the cumulative sum of active drivers after getting the number of drivers for each month.
- The number of accepted rides needs to be computed using `Rides` and `AcceptedRides`. What we need to understand is that of rides requested by customers, not all of them are accepted by drivers and thus all ride records in `AcceptedRides` must be in the records in `Rides` table. Therefore, we can directly join the two tables using `inner join` to attain the accepted rides.




### Activer Drivers
*Step 1:Based on the analysis above, we first need to create one field containing full months of one year*. 
Here we can use recursive cte:
```Mysql
with RECURSIVE cte(mon) as (
    select 1
    union
    select mon+1
    from cte
    where mon<12
)
```
Through this code, we will have the following table:
|Month|
|---|
|1|
|2|
|3|
|...|
|12|


*Step 2: compute the number of active drivers for each month using Drivers table*
The detailed step is:
- filter out records of 2020 because we only want to compute the number of active dirvers in 2020
- `group` the dataset by month of `join_date` using `month()` function. Here, we need to be cautious about the record `2019-12-9`. Although the record is not in 2020, we still need to count it because this record indicates there is one activer driver in the beginning of 2020/1/1. 
- Combine all columns of `cte` table and `Drivers` table to compute the cumulative sum of acive drivers.

*Filter out records of 2021*
```Mysql
select
  *
from drivers 
where year(join_date)<="2020"
```

*group the dataset by month and compute the number of active drivers*
```Mysql
select
    if(year(join_date)<2020,1,month(join_date)) join_month,
    count(*) as driver_num
from drivers 
where year(join_date)<="2020"
group by if(year(join_date)<2020,1,month(join_date))
```
*Combine all columns of `cte` table and `Drivers` table*
```Mysql
with RECURSIVE cte(mon) as (
    select 1
    union
    select mon+1
    from cte
    where mon<12
)

select
    c.n month,
    sum(ifnull(t.driver_num,0)) over (order by c.n) as active_drivers,
from cte c
left join (select
            if(year(join_date)<'2020',1,month(join_date)) as join_month,
            count(*) as driver_num
        from drivers d
        where year(join_date)<=2020
        group by if(year(join_date)<2020,1,month(join_date))) t
```

### Accepted Rides
To compute the number of accepted rides, we can directly combine all columns of the `Rides` table and `AcceptedRides` table on the condition of the same ride id. In this case, as long as the ride is accepted, each row of accepted rides table will be matched with the row of `Rides` table. So we can compute the number of accepted rides for each month based on `requested_at` column.
```Mysql
select
    month(r.requested_at) as request_month,
    count(a.ride_id) as accepted_rides
from rides r 
join acceptedrides a
on r.ride_id = a.ride_id
where year(r.requested_at)=2020
group by month(r.requested_at)
```

### Combine active drivers and accepted rides
We have completed the two numbers the question asked. **But the `request_month` column in the accepted rides result does not include records of full months of a year.** So we need to join the two tables on the condition of the same month to query the `month` field in cte table. So we can compute the accepted rides for each month, and those unmatched records will be automatically considered as null by SQL. So we need to use `ifnull()` to convert null into 0.
```Mysql
with recursive cte(n) as (
    select 1
    union
    select n+1
    from cte where n<12
)

select
    c.n month,
    sum(ifnull(t.driver_num,0)) over (order by c.n) as active_drivers,
    ifnull(t1.accepted_rides,0) as accepted_rides
from cte c
left join (select
            if(year(join_date)<'2020',1,month(join_date)) as join_month,
            count(*) as driver_num
        from drivers d
        where year(join_date)<=2020
        group by if(year(join_date)<2020,1,month(join_date))) t
on c.n = t.join_month
left join (select
            month(r.requested_at) as request_month,
            count(a.ride_id) as accepted_rides
        from rides r 
        join acceptedrides a
        on r.ride_id = a.ride_id
        where year(r.requested_at)=2020
        group by month(r.requested_at)) t1
on c.n = t1.request_month
group by c.n
order by month asc;
```

Now, I also write a python code to solve this question. The solution is exactly the same as what I have done in MySQL.
```Python
# create a table storing the 12 months of a year
months = [mon for mon in range(1,13)]    
Months = pd.DataFrame({"Month":months})

# write a function to find out the month of join_date
def convert_month(date):
    """convert the date to a month"""
    if date.year<2020:
        return 1
    else:
        return date.month

# filter out records of 2020
year_condi  = Drivers["join_date"].dt.year<=2020
Drivers_2020 = Drivers[year_condi]
# merge drivers with month table to compute the driver number for each month
Drivers_2020["join_month"] = Drivers["join_date"].apply(lambda x:convert_month(x))
merged_df = pd.merge(Months,Drivers_2020,left_on="Month",right_on="join_month",how='left')
driver_numbers_df = merged_df.groupby("Month",as_index=False).agg({"driver_id":"count"}).rename(columns={"driver_id":"driver_num"})
# compute the cumulative sum of active driver using cumsum function
driver_numbers_df["driver_number"] = driver_numbers["driver_num"].cumsum(axis=0)

# Now compute the number of accepted rides
# filter out accepted rides from Rides table
ride_condi = Rides.ride_id.isin(AcceptedRides.ride_id)
accepted_rides_df = Rides[ride_condi]
# filter out records of 2020
date_condi = accepted_rides_df.requested_at.dt.year<2021
accepted_rides_df = accepted_rides_df[date_condi]
# compute the number of accepted rides for each month
accepted_rides_df_byMon = accepted_rides_df["ride_id"].groupby(accepted_rides_df["requested_at"].dt.month).count().to_frame().reset_index().\
rename(columns={"requested_at":"month","ride_id":"ride_count"})
res = pd.merge(driver_numbers_df,accepted_rides_df_byMon,left_on="Month",right_on="month",how='left')\
.drop("month",axis=1).fillna(0)[["Month","driver_number","ride_count"]]
# rename the result table to satisfy the requirement of question
result = res.rename(columns={"Month":"month","driver_number":"active_drivers","ride_count":"accepted_rides"})
```
`res`
![answer](https://upload-images.jianshu.io/upload_images/10429581-818592b43f28d278.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
