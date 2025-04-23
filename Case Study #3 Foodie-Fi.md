![Ilustrative title of the case study.](https://8weeksqlchallenge.com/images/case-study-designs/3.png)

# :pear: Data Analysis Questions
1. [**How many customers has Foodie-Fi ever had?**](#how-many-customers-has-foodie-fi-ever-had)
2. [**What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value**](#what-is-the-monthly-distribution-of-trial-plan-start_date-values-for-our-dataset---use-the-start-of-the-month-as-the-group-by-value)
3. [**What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name**](#what-plan-start_date-values-occur-after-the-year-2020-for-our-dataset-show-the-breakdown-by-count-of-events-for-each-plan_name)
4. [**What is the customer count and percentage of customers who have churned rounded to 1 decimal place?**](#what-is-the-customer-count-and-percentage-of-customers-who-have-churned-rounded-to-1-decimal-place)
5. [**How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?**](#how-many-customers-have-churned-straight-after-their-initial-free-trial---what-percentage-is-this-rounded-to-the-nearest-whole-number)
6. [**What is the number and percentage of customer plans after their initial free trial?**](#what-is-the-number-and-percentage-of-customer-plans-after-their-initial-free-trial)
7. [**What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?**](#what-is-the-customer-count-and-percentage-breakdown-of-all-5-plan_name-values-at-2020-12-31)
8. [**How many customers have upgraded to an annual plan in 2020?**](#how-many-customers-have-upgraded-to-an-annual-plan-in-2020)
9. [**How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?**](#how-many-days-on-average-does-it-take-for-a-customer-to-an-annual-plan-from-the-day-they-join-foodie-fi)
10. [**Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)**](#can-you-further-breakdown-this-average-value-into-30-day-periods-ie-0-30-days-31-60-days-etc)
11. [**How many customers downgraded from a pro monthly to a basic monthly plan in 2020?**](#how-many-customers-downgraded-from-a-pro-monthly-to-a-basic-monthly-plan-in-2020)




# :pear: SQL Queries

## **Data Analysis Questions**
**1. How many customers has Foodie-Fi ever had?**
```sql
select count(distinct customer_id)
from subscriptions
```

**2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value**
```sql
select (date_trunc('month', start_date))::DATE as month, count(plan_id) as total
from subscriptions
where plan_id = 0
group by month
order by month
```

**3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name**
```sql
select plan_id, count(plan_id)
from subscriptions
where date_part('year', start_date) >= 2020
group by plan_id
```

**4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?**
```sql
with cte as(
	select count(distinct customer_id) as churned
	from subscriptions
	where plan_id = 4
),
total as(
	select count(distinct customer_id) as total
	from subscriptions
)
select churned, round(churned*100.0/total, 1) as percentage
from cte
cross join total
```

**5. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?**
```sql
with cte as(
	select customer_id, plan_id, start_date,
		lag(plan_id) over(partition by customer_id order by start_date) as previous_plan_id
	from subscriptions
)
select count(distinct customer_id) as total, 
	round(count(distinct customer_id) * 100.0 / (select count(distinct customer_id)from cte)) as percentage
from cte
where plan_id = 4
and previous_plan_id = 0
```

**6. What is the number and percentage of customer plans after their initial free trial?**
```sql
with cte as(
	select customer_id, plan_id, start_date,
		lag(plan_id) over(partition by customer_id order by start_date) as previous_plan_id
	from subscriptions
)
select count(distinct customer_id) as total, 
	round(count(distinct customer_id) * 100.0 / (select count(distinct customer_id)from cte)) as percentage
from cte
where plan_id != 4
and previous_plan_id = 0
```

**7. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?**
```sql
select plan_name, count(distinct customer_id) as total_customers
from plans
left join subscriptions
on plans.plan_id = subscriptions.plan_id
and date_trunc('day', start_date) = '2020-12-31 '
group by plan_name
```

**8. How many customers have upgraded to an annual plan in 2020?**
```sql
select count(distinct customer_id)
from subscriptions
where plan_id = 3
and date_part('year', start_date) = 2020
```

**9. How many days on average does it take for a customer to buy an annual plan from the day they join Foodie-Fi?**
```sql
with cte as (
	select customer_id, start_date as annual_start_date
	from subscriptions
	where plan_id = 4
),
first as (
	select customer_id, plan_id,
		min(start_date) over(partition by customer_id) as first_date
	from subscriptions
),
joined as (
	select distinct customer_id, first_date, annual_start_date, (annual_start_date - first_date) as interval
	from cte
	left join first
	using(customer_id)
)
select round(avg(interval))
from joined
```

**10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)**
```sql
with cte as (
	select customer_id, start_date as annual_start_date
	from subscriptions
	where plan_id = 4
),
first as (
	select customer_id, plan_id,
		min(start_date) over(partition by customer_id) as first_date
	from subscriptions
),
joined as (
	select distinct customer_id, first_date, annual_start_date, (annual_start_date - first_date) as interval
	from cte
	left join first
	using(customer_id)
)
select count(distinct customer_id) as total_customers,
	(case
		when interval <= 30 then '0-30 days'
		when interval <= 60 then '31-60 days'
		when interval <= 90 then '61-90 days'
		when interval <= 120 then '91-120 days'
		when interval > 120 then '120+ days ' END
	) as days_interval
from joined
group by days_interval
order by days_interval
```

**11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?**
```sql
with cte as(
	select customer_id, plan_id, start_date,
		lag(plan_id) over(partition by customer_id order by start_date) as previous_plan
	from subscriptions
	where date_part('year', start_date) = 2020
)
select *
from cte
where plan_id = 1
and previous_plan = 2
```

