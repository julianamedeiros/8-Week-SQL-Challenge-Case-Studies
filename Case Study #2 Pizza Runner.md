![Ilustrative title of the case study.](https://8weeksqlchallenge.com/images/case-study-designs/2.png)

# 🍕 Case Study Questions

## A. Pizza Metrics
1. How many pizzas were ordered?
2. How many unique customer orders were made?
3. How many successful orders were delivered by each runner?
4. How many of each type of pizza was delivered?
5. How many Vegetarian and Meatlovers were ordered by each customer?
6. What was the maximum number of pizzas delivered in a single order?
7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
8. How many pizzas were delivered that had both exclusions and extras?
9. What was the total volume of pizzas ordered for each hour of the day?
10. What was the volume of orders for each day of the week?

## B. Runner and Customer Experience
1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)
2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
3. Is there any relationship between the number of pizzas and how long the order takes to prepare?
4. What was the average distance travelled for each customer?
5. What was the difference between the longest and shortest delivery times for all orders?
6. What was the average speed for each runner for each delivery and do you notice any trend for these values?
7. What is the successful delivery percentage for each runner?

## C. Ingredient Optimisation
1. What are the standard ingredients for each pizza?
2. What was the most commonly added extra?
3. What was the most common exclusion?
4. Generate an order item for each record in the customers_orders table in the format of one of the following:
Meat Lovers
Meat Lovers - Exclude Beef
Meat Lovers - Extra Bacon
Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers
5. Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients
For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"
6. What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?

## D. Pricing and Ratings
1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?
2. What if there was an additional $1 charge for any pizza extras?
Add cheese is $1 extra
3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.
4. Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?
customer_id
order_id
runner_id
rating
order_time
pickup_time
Time between order and pickup
Delivery duration
Average speed
Total number of pizzas
5. If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?

## E. Bonus Question
If Danny wants to expand his range of pizzas - how would this impact the existing data design? Write an INSERT statement to demonstrate what would happen if a new Supreme pizza with all the toppings was added to the Pizza Runner menu?

# 🍕 SQL queries

## Cleaning data as views:


for customer_orders table:
```sql
CREATE VIEW c_customer_orders as (
	WITH cleaned AS (
		SELECT 
			ROW_NUMBER() OVER(ORDER BY order_id) as pizza_ordered, order_id, pizza_id, customer_id,
			(CASE 
				WHEN extras IS NULL OR extras = '' OR extras LIKE '%null%' THEN '0' ELSE extras
			END) AS c_extras,
			(CASE 
				WHEN exclusions IS NULL OR exclusions = '' OR exclusions LIKE '%null%' THEN '0' ELSE exclusions
			END) AS c_exclusions, order_time
		FROM customer_orders
	),
	unnested_data AS (
		SELECT 
			pizza_ordered, order_id, pizza_id, customer_id,
			UNNEST(string_to_array(c_extras, ',')) AS ex_extras,
			UNNEST(string_to_array(c_exclusions, ',')) AS ex_exclusions, order_time
		FROM cleaned
	)
	SELECT 
		pizza_ordered, order_id, pizza_id, customer_id, 
		(CASE WHEN ex_exclusions IS NULL THEN '0' ELSE ex_exclusions END) as exclusions, ex_extras as extras, 
		order_time::date as order_date,
		order_time::time as order_time
		FROM unnested_data
)
```
- runner_orders table:
```sql
CREATE VIEW c_runner_orders as (
	WITH cleaned as(
		SELECT order_id, runner_id, 
			CASE WHEN pickup_time LIKE '%null%' THEN NULL ELSE pickup_time END,  
			CASE WHEN distance LIKE '%null%' THEN NULL ELSE distance END,
			CASE WHEN duration LIKE '%null%' THEN NULL ELSE duration END,
			CASE WHEN cancellation LIKE '%null%' or cancellation = '' or cancellation = 'NaN' THEN NULL ELSE cancellation END
		FROM runner_orders
	),
	data_type as (
		SELECT order_id, runner_id,
		CASE WHEN pickup_time IS NOT NULL THEN pickup_time::date END as pickup_date,
		CASE WHEN pickup_time IS NOT NULL THEN pickup_time::time END as pickup_time,
		CAST(REGEXP_REPLACE(distance, '[^0-9\.]+', '', 'g') as FLOAT) as distance, 
		CAST(REGEXP_REPLACE(duration, '[^0-9\.]+', '', 'g') as INTEGER) as duration, cancellation
		FROM cleaned
	)
	SELECT * 
	FROM data_type
)
```
- for delivered_orders:
```sql
CREATE VIEW delivered_orders as (
	WITH delivered as (
		SELECT c_customer_orders.pizza_ordered, c_customer_orders.order_id, c_customer_orders.customer_id, c_customer_orders.pizza_id, exclusions, extras, order_date, order_time
		FROM c_customer_orders
		LEFT JOIN runner_orders
		ON c_customer_orders.order_id = runner_orders.order_id
		WHERE runner_orders.pickup_time NOT LIKE '%null%'
	)
	SELECT *
	FROM delivered
)
```
- for pizza_recipes:
```sql
CREATE VIEW c_pizza_recipes as (
	WITH pizza as (
		SELECT pizza_id, UNNEST(string_to_array(toppings, ', '))::int as toppings
		FROM pizza_recipes
		order by pizza_id
	)
	SELECT pizza_id, ARRAY_AGG(toppings ORDER BY toppings ASC) as toppings
	FROM pizza
	GROUP BY pizza_id
)
```
## A. Pizza Metrics
 
### How many pizzas were ordered?
```sql
SELECT COUNT(pizza_id) as total_order_pizza
FROM customer_orders
```

### 2. How many unique customer orders were made?
```sql
SELECT COUNT(DISTINCT order_id) as total_orders
FROM customer_orders
```

### 3. How many successful orders were delivered by each runner?
```sql
SELECT runner_id, COUNT(order_id) as delivered_orders
FROM runner_orders
WHERE pickup_time NOT LIKE '%null%'
GROUP BY runner_id
order by runner_id
```

### 4. How many of each type of pizza was delivered?
```sql
WITH sucessfull_orders as(
	SELECT order_id
	FROM runner_orders
	WHERE pickup_time NOT LIKE '%null%'
),
pizza_type as(
	SELECT c.order_id, p.pizza_name
	FROM customer_orders as c
	LEFT JOIN pizza_names as p
	ON c.pizza_id = p.pizza_id
),
main as(
	SELECT s.order_id, p.pizza_name
	FROM sucessfull_orders as s
	LEFT JOIN pizza_type as p
	ON s.order_id = p.order_id
)
SELECT pizza_name, COUNT(pizza_name) as times_delivered
FROM main
GROUP BY pizza_name
```

### 5. How many Vegetarian and Meatlovers were ordered by each customer?
```sql
WITH orders as (
  SELECT c.order_id, c.customer_id, p.pizza_name
  FROM customer_orders as c
  LEFT JOIN pizza_names as p
  ON c.pizza_id = p.pizza_id
)
SELECT customer_id, 
  SUM(CASE WHEN pizza_name LIKE '%Vegetarian%' THEN 1 ELSE 0 END) as total_vege,
  SUM(CASE WHEN pizza_name LIKE '%Meatlovers%' THEN 1 ELSE 0 END) as total_meat
FROM orders
GROUP BY customer_id
ORDER BY customer_id
```
	
### 6. What was the maximum number of pizzas delivered in a single order?
```sql
WITH delivered_orders as (
  SELECT runner_orders.order_id, pizza_id 
  FROM runner_orders
  LEFT JOIN customer_orders
  ON runner_orders.order_id = customer_orders.order_id 
  WHERE pickup_time NOT LIKE '%null%'
),
orders as (
  SELECT order_id, COUNT(pizza_id) as total_pizzas
  FROM delivered_orders
  GROUP BY order_id
)
SELECT MAX(total_pizzas)
FROM orders
```
	
### 7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
```sql
WITH temp AS (
    SELECT customer_id, pizza_ordered,
           SUM(CASE WHEN exclusions = '0' AND extras = '0' THEN 0 ELSE 1 END) AS t_changes
    FROM delivered_orders
    GROUP BY pizza_ordered, customer_id
)
SELECT customer_id,
       SUM(CASE WHEN t_changes = 0 THEN 1 ELSE 0 END) AS t_unchanged,
       SUM(CASE WHEN t_changes != 0 THEN 1 ELSE 0 END) AS t_changed
FROM temp
GROUP BY customer_id
ORDER BY customer_id
```
	
### 8. How many pizzas were delivered that had both exclusions and extras?
```sql
SELECT COUNT(DISTINCT pizza_ordered) as total
FROM delivered_orders
WHERE exclusions NOT LIKE '%0%' AND extras NOT LIKE '%0%'
```

	
### 9. What was the total volume of pizzas ordered for each hour of the day?
```sql
SELECT order_date, order_time, COUNT(DISTINCT pizza_ordered) as total_pizza
FROM c_customer_orders
GROUP BY order_date, order_time
```
	
### 10. What was the volume of orders for each day of the week?
```sql
SELECT TO_CHAR(order_date, 'day') as day_of_week, COUNT(DISTINCT pizza_ordered) as total_pizza
FROM c_customer_orders
GROUP BY day_of_week
```

## B. Runner and Customer Experience

### 1. How many runners signed up for each 1 week period (week stats 2021-01-01)
```sql
SELECT TO_CHAR(registration_date, 'WW') AS week, COUNT(DISTINCT runner_id)
FROM runners
GROUP BY week
```

	
### 2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
```sql
WITH temp as(
  SELECT c.pickup_time, d.order_time, c.order_id, c.runner_id
  FROM delivered_orders as d
  LEFT JOIN c_runner_orders as c
  ON c.order_id = d.order_id
),
time as(
  SELECT runner_id, order_id,
  EXTRACT(MINUTE FROM pickup_time) AS pickup_minutes,
  EXTRACT(MINUTE FROM order_time) AS order_minutes
  FROM temp
),
difference as (
  select runner_id, order_id, (CASE WHEN pickup_minutes < order_minutes THEN (pickup_minutes + 60) - order_minutes ELSE pickup_minutes - order_minutes END) as diff
  from time
  ORDER BY order_id ASC
)
SELECT runner_id, ROUND(AVG(diff), 0) as time_difference
FROM difference
GROUP BY runner_id
ORDER BY runner_id
```
	
	
### 3. Is there any relationship between the number of pizzas and how long the order takes to prepare?
```sql	
WITH temp as(
  SELECT c.pickup_time, d.order_time, c.order_id, c.runner_id
  FROM delivered_orders as d
  LEFT JOIN c_runner_orders as c
  ON c.order_id = d.order_id
),
time as(
  SELECT runner_id, order_id,
  EXTRACT(MINUTE FROM pickup_time) AS pickup_minutes,
  EXTRACT(MINUTE FROM order_time) AS order_minutes
  FROM temp
),
difference as (
  select runner_id, order_id, (CASE WHEN pickup_minutes < order_minutes THEN (pickup_minutes + 60) - order_minutes ELSE pickup_minutes - order_minutes END) as diff
  from time
  ORDER BY order_id ASC
),
new_temp as (
  SELECT order_id, COUNT(DISTINCT pizza_ordered) as total_pizza
  from delivered_orders
  GROUP BY order_id
  ORDER BY order_id
)
SELECT new_temp.order_id, total_pizza, diff
FROM difference
LEFT JOIN new_temp
ON new_temp.order_id = difference.order_id
GROUP BY new_temp.order_id, diff, total_pizza
ORDER BY new_temp.order_id
```

	
### 4. What was the average distance travelled for each customer?
```sql
WITH temp as(
  SELECT c.order_id, distance, customer_id
  FROM c_runner_orders as c
  LEFT JOIN delivered_orders as d
  ON c.order_id = d.order_id
  WHERE distance IS NOT NULL
  GROUP BY customer_id, c.order_id, distance, customer_id
)
SELECT customer_id, AVG(distance) as avg_distance
FROM temp
GROUP BY customer_id
ORDER BY customer_id
```

	
### 5. What was the difference between the longest and shortest delivery times for all orders?
```sql	
WITH temp as(
  SELECT duration, order_id
  FROM c_runner_orders
  WHERE duration IS NOT NULL
),
times as (
  SELECT MAX(duration) as max_time,
    MIN(duration) as min_time
  FROM temp
)
SELECT SUM(max_time - min_time)	as time_diff
FROM times
```
	
### 6. What was the average speed for each runner for each delivery and do you notice any trend for these values?
```sql
WITH temp as (
  SELECT runner_id, order_id, distance,
       SUM(duration / 60.0) AS duration_hour 
  FROM c_runner_orders
  WHERE distance IS NOT NULL
  GROUP BY order_id, runner_id, distance
),
speed as(
  SELECT runner_id, order_id, (distance/duration_hour) as speed
  FROM temp
  ORDER BY  runner_id, order_id
)
SELECT runner_id, ROUND(AVG(speed::numeric),0) as avg_speed
FROM speed
GROUP BY runner_id
```
	
### 7. What is the successful delivery percentage for each runner?
```sql	
WITH temp as(
  SELECT runner_id,
    SUM(CASE WHEN cancellation IS NULL then 1 else 0 END) as s_deliveries, 
    SUM(CASE WHEN cancellation IS NOT NULL then 1 else 0 END) as f_deliveries,
    COUNT(DISTINCT order_id) as total_deliveries
  FROM c_runner_orders
  GROUP BY runner_id
  ORDER BY runner_id
)
SELECT runner_id, SUM(s_deliveries * 100/total_deliveries) as s_percentage
FROM temp
GROUP BY runner_id
order by runner_id
```

## C. Ingredient Optimisation

### 1. What are the standard ingredients for each pizza?
```sql
WITH temp as (
  SELECT r.pizza_id, 
    UNNEST(string_to_array(toppings, ',')) as toppings,
    pizza_name
  FROM pizza_recipes as r
  LEFT JOIN pizza_names as n
  ON n.pizza_id = r.pizza_id
),
c_top as (
  SELECT pizza_id, 
    toppings::INTEGER as topping_id,
    pizza_name
  FROM temp
),
names as (
  SELECT pizza_name, topping_name 
  FROM c_top as c
  LEFT JOIN pizza_toppings 
  ON pizza_toppings.topping_id = c.topping_id
  ORDER BY pizza_name
)
SELECT pizza_name, STRING_AGG(topping_name, ', ') AS concatenated_values
FROM names
GROUP BY pizza_name
```	

	
### 2. What was the most commonly added extra?
```sql	
WITH temp as (
  SELECT extras::integer as topping_id, count(extras) as count_add
  FROM c_customer_orders
  WHERE extras NOT LIKE '%0%'
  GROUP BY extras
),
name as (
  SELECT temp.topping_id, count_add, topping_name,
    RANK() OVER(ORDER BY count_add DESC) as rank
  FROM temp
  LEFT JOIN pizza_toppings as p
  ON temp.topping_id = p.topping_id
)
SELECT topping_name, count_add
FROM name
WHERE rank = 1
```
	
	
### 3. What was the most common exclusion?
```sql	
WITH temp as (
  SELECT exclusions::integer as topping_id, count(exclusions) as count_ex
  FROM c_customer_orders
  WHERE exclusions NOT LIKE '%0%'
  GROUP BY exclusions
),
name as (
  SELECT temp.topping_id, count_ex, topping_name,
    RANK() OVER(ORDER BY count_ex DESC) as rank
  FROM temp
  LEFT JOIN pizza_toppings as p
  ON temp.topping_id = p.topping_id
)
SELECT topping_name, count_ex
FROM name
WHERE rank = 1
```
	
	
### 4. Generate an order item for each record in the customers_orders table in the format of one of the following:
	Meat Lovers
	Meat Lovers - Exclude Beef
	Meat Lovers - Extra Bacon
	Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers
```sql	
WITH temp as (
  SELECT order_id, pizza_ordered, pizza_name, 
    exclusions::integer as exclusion_id,
    extras::integer as extra_id
  FROM c_customer_orders as c
  LEFT JOIN pizza_names as p
  ON c.pizza_id = p.pizza_id
  ORDER BY order_id, pizza_ordered
),
exclusions as (
  SELECT pizza_ordered, order_id, pizza_name,
  (CASE WHEN exclusion_id !=0 THEN topping_name ELSE NULL END) as ex, extra_id
  FROM temp
  LEFT JOIN pizza_toppings
  ON pizza_toppings.topping_id = temp.exclusion_id  
),
extras as (
  SELECT pizza_ordered, order_id, pizza_name, ex, 
    (CASE WHEN extra_id !=0 THEN topping_name ELSE NULL END) as add
  FROM exclusions
  LEFT JOIN pizza_toppings
  ON pizza_toppings.topping_id = exclusions.extra_id  
),
agg as (
  SELECT pizza_ordered, pizza_name,
  STRING_AGG(ex, ', ') as exclusions, 
  STRING_agg(add, ', ') as extras
  FROM extras
  GROUP BY pizza_ordered, pizza_name
  ORDER BY pizza_ordered
)
SELECT 
  CASE
    WHEN exclusions IS NOT NULL and extras IS NULL THEN CONCAT(pizza_name, ' - Exclude ', exclusions)
    WHEN exclusions IS NOT NULL and extras IS NOT NULL THEN CONCAT(pizza_name, ' - Exclude ', exclusions, ' - Extra ', extras)
    WHEN exclusions IS NULL and extras IS NOT NULL THEN CONCAT(pizza_name, ' - Extra ', extras)
  ELSE pizza_name END as item
FROM agg
```
	
	
### 5. Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients
	For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"
```sql
WITH temp as (
  SELECT pizza_ordered, p.pizza_id, p.pizza_name, (exclusions::int) as exclusions, (extras::int) as extras
  FROM c_customer_orders 
  LEFT JOIN pizza_names p
  ON p.pizza_id = c_customer_orders.pizza_id
  ORDER BY pizza_ordered, order_id
),
agg as (
  SELECT pizza_ordered, pizza_id, pizza_name, ARRAY_AGG(exclusions) as exclusions, ARRAY_AGG(extras) as extras
  FROM temp 
  GROUP BY pizza_ordered, pizza_id, pizza_name
),
pizza as (
  SELECT pizza_ordered, r.pizza_id, pizza_name, exclusions, array_Agg(toppings || extras) as toppings
  FROM agg
  LEFT JOIN c_pizza_recipes r
  ON agg.pizza_id = r.pizza_id
  group by pizza_ordered, r.pizza_id, pizza_name, exclusions
),
exclusions as(
  SELECT pizza_ordered, pizza_name, ARRAY_AGG(u_toppings) AS TOPPINGS
  from (
    SELECT pizza_ordered, pizza_name, unnest(toppings) as u_toppings, exclusions
    FROM pizza
  ) as subquery
  WHERE NOT (u_toppings = ANY(exclusions))
  GROUP BY pizza_ordered, pizza_name
),
cte as (
  SELECT pizza_ordered, pizza_name, STRING_AGG(topping_name, ', ') as ingredients
  FROM (
    SELECT pizza_ordered, pizza_name, unnest(toppings) as u_toppings
    FROM exclusions
  ) as subquery2
  LEFT JOIN pizza_toppings n
  ON n.topping_id = subquery2.u_toppings
  GROUP BY pizza_ordered, pizza_name
)
SELECT CONCAT(pizza_name, ': ', ingredients) as list
FROM cte
```


### 6. What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?
```sql
WITH temp as (
  SELECT pizza_ordered, p.pizza_id, p.pizza_name, (exclusions::int) as exclusions, (extras::int) as extras
  FROM delivered_orders
  LEFT JOIN pizza_names p
  ON p.pizza_id = delivered_orders.pizza_id
  ORDER BY pizza_ordered, order_id
),
agg as (
  SELECT pizza_ordered, pizza_id, pizza_name, ARRAY_AGG(exclusions) as exclusions, ARRAY_AGG(extras) as extras
  FROM temp 
  GROUP BY pizza_ordered, pizza_id, pizza_name
),
pizza as (
  SELECT pizza_ordered, r.pizza_id, pizza_name, exclusions, array_Agg(toppings || extras) as toppings
  FROM agg
  LEFT JOIN c_pizza_recipes r
  ON agg.pizza_id = r.pizza_id
  group by pizza_ordered, r.pizza_id, pizza_name, exclusions
),
exclusions as(
  SELECT pizza_ordered, pizza_name, ARRAY_AGG(u_toppings) AS TOPPINGS
  from (
    SELECT pizza_ordered, pizza_name, unnest(toppings) as u_toppings, exclusions
    FROM pizza
  ) as subquery
  WHERE NOT (u_toppings = ANY(exclusions))
  GROUP BY pizza_ordered, pizza_name
),
cte as (
  SELECT pizza_ordered, pizza_name, topping_name
  FROM (
    SELECT pizza_ordered, pizza_name, unnest(toppings) as u_toppings
    FROM exclusions
  ) as subquery2
  LEFT JOIN pizza_toppings n
  ON n.topping_id = subquery2.u_toppings
  GROUP BY pizza_ordered, pizza_name, topping_name
)
SELECT topping_name, count(topping_name) as i_count
from cte
where topping_name is not null
group by topping_name
order by i_count desc
```

## D. Pricing and Ratings

### 1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?
```sql	
with temp as (
  select row_number() OVER (order by order_id) as c_id, order_id, pizza_name
  from delivered_orders
  LEFT JOIN pizza_names
  ON delivered_orders.pizza_id = pizza_names.pizza_id
)
select SUM(CASE WHEN pizza_name LIKE '%Meatlovers%' THEN 12 ELSE 10 END) as total
from temp
```

	
### 2. What if there was an additional $1 charge for any pizza extras?
	Add cheese is $1 extra
```sql	
with temp as (
  select row_number() OVER (order by order_id) as c_id, order_id, pizza_name, extras::int
  from delivered_orders
  LEFT JOIN pizza_names
  ON delivered_orders.pizza_id = pizza_names.pizza_id
),
target as(
  select SUM(CASE WHEN pizza_name LIKE '%Meatlovers%' THEN 12 ELSE 10 END) as total, 
    SUM(CASE WHEN extras != 0 THEN 1 ELSE 0 END) as extra_total
  from temp
)
select sum(total+extra_total) as target
from target
```
	
### 3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.
```sql	
select d.order_id, customer_id, runner_id, floor(random() * 5 + 1) as rating
from delivered_orders d
left join runner_orders
on runner_orders.order_id = d.order_id
group by d.order_id, customer_id, runner_id
order by order_id, customer_id, runner_id
```
	
	
### 4. Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?
	customer_id
	order_id
	runner_id
	rating
	order_time
	pickup_time
	Time between order and pickup
	Delivery duration
	Average speed
	Total number of pizzas

```sql
with temp as(
  select d.order_id, customer_id, runner_id, floor(random() * 5 + 1) as rating, distance, pickup_time, duration
  from delivered_orders d
  left join c_runner_orders
  on c_runner_orders.order_id = d.order_id
  group by d.order_id, customer_id, runner_id, distance, pickup_time, duration
  order by order_id, customer_id, runner_id
),
cte as (
  select t.customer_id, t.order_id, t.runner_id, rating, c.order_time, pickup_time, 
    EXTRACT(MINUTE FROM pickup_time) as pickup_minutes,
    EXTRACT(MINUTE FROM order_time) as order_minutes,
    duration, 
    floor(SUM(distance / (duration/60.0))) as avg_speed, count(pizza_id) as pizza_per_order
  from temp t
  left join c_customer_orders c
  on t.order_id = c.order_id
  group by t.order_id, t.customer_id, t.runner_id, rating, c.order_time, pickup_time, duration
)
select customer_id, order_id, runner_id, rating, order_time, pickup_time,
  (CASE WHEN pickup_minutes < order_minutes THEN (pickup_minutes + 60) - order_minutes ELSE pickup_minutes - order_minutes END) as time_difference,
  duration, avg_speed, pizza_per_order
from cte
```
	
	
### 5. If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?
```sql
WITH temp as (
  select order_id, pizza_name
  from delivered_orders d 
  left join pizza_names p
  on d.pizza_id = p.pizza_id
),
sums as (
  SELECT temp.order_id, SUM(CASE WHEN pizza_name LIKE '%Meatlovers%' then 12 else 10 end) as pizza_cost,
    floor(sum(distance * 0.3)) as runner_cost
  from temp
  left join c_runner_orders c
  on temp.order_id = c.order_id
  group by temp.order_id
)
SELECT sum(pizza_cost - runner_cost) as target
from sums
```

## E. Bonus question

### If Danny wants to expand his range of pizzas - how would this impact the existing data design? Write an INSERT statement to demonstrate what would happen if a new Supreme pizza with all the toppings was added to the Pizza Runner menu?
```sql
INSERT INTO pizza_names(pizza_id, pizza_name)
VALUES (3, supreme)

INSERT INTO pizza_recipes(pizza_id, toppings)
VALUES (3, '1, 2, 3, 4, 5, 6, 7, 8, 9, 10')
```
