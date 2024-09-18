![Ilustration of a bowl of ramen and title of the case study.](https://8weeksqlchallenge.com/images/case-study-designs/1.png)

# :ramen: Case Study Questions

1. What is the total amount each customer spent at the restaurant?
2. How many days has each customer visited the restaurant?
3. What was the first item from the menu purchased by each customer?
4. What is the most purchased item on the menu and how many times was it purchased by all customers?
5. Which item was the most popular for each customer?
6. Which item was purchased first by the customer after they became a member?
7. Which item was purchased just before the customer became a member?
8. What is the total items and amount spent for each member before they became a member?
9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
11. Bonus Join All The Things
12. Bonus Rank All The Things

# :ramen: SQL queries

I've used PostgreSQL for this project. 

```sql
#Creating database and tables

CREATE TABLE menu(
	"product_id" SERIAL UNIQUE,
	"product_name" VARCHAR (5),
	"price" INT
	);
	
INSERT INTO menu(product_name, price)
VALUES
	('sushi', 10),
	('curry', 15),
	('ramen', 12);
	
CREATE TABLE members(
	"customer_id" VARCHAR(1) UNIQUE,
	"join_date" DATE
	);

INSERT INTO members(customer_id, join_date)
VALUES
	('A', '2021-01-07'),
	('B', '2021-01-09'),
	('C', '2021-01-09');

CREATE TABLE sales(
	"customer_id" VARCHAR (1) REFERENCES members(customer_id),
	"order_date" DATE,
	"product_id" INT REFERENCES menu(product_id)
	);

INSERT INTO sales(customer_id, order_date, product_id)
VALUES
		('A', '2021-01-01', 1),
		('A', '2021-01-01', 2),
		('A', '2021-01-07', 2),
		('A', '2021-01-10', 3),
		('A', '2021-01-11', 3),
		('A', '2021-01-11', 3),
		('B', '2021-01-01', 2),
		('B', '2021-01-02', 2),
		('B', '2021-01-04', 1),
		('B', '2021-01-11', 1),
		('B', '2021-01-16', 3),
		('B', '2021-02-01', 3),
		('C', '2021-01-01', 3),
		('C', '2021-01-01', 3),
		('C', '2021-01-07', 3);



# Questions and solutions

**1. What is the total amount each customer spent at the restaurant?**

SELECT s.customer_id, SUM(m.price) as total_spent
FROM sales as s
LEFT JOIN menu as m
ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY total_spent DESC

**2. How many days has each customer visited the restaurant?**

SELECT customer_id, COUNT(DISTINCT order_date) as total_daily_visits
FROM sales
GROUP BY customer_id

**3. What was the first item from the menu purchased by each customer?**

SELECT customer_id, product_name, order_date
FROM sales as s
LEFT JOIN menu as m
ON s.product_id = m.product_id
ORDER BY order_date ASC

**4. What is the most purchased item on the menu and how many times was it purchased by all customers?**

SELECT product_name, COUNT(product_name) as total_purchases
FROM sales as s
LEFT JOIN menu as m
ON s.product_id = m.product_id
GROUP BY product_name
ORDER BY total_purchases DESC

**5. Which item was the most popular for each customer?**

WITH total AS (
	SELECT customer_id, product_name, COUNT(product_name) as total_purchases
	FROM sales as s
	LEFT JOIN menu as m
	ON s.product_id = m.product_id
	GROUP BY product_name, customer_id
),
rank_total as (
	SELECT customer_id, product_name, total_purchases,
	ROW_NUMBER() OVER (PARTITION BY customer_id) as rank
	FROM total
)
SELECT customer_id, product_name, total_purchases, rank
FROM rank_total
WHERE rank = 1

**6. Which item was purchased first by the customer after they became a member?**

WITH date as(
	SELECT s.customer_id, s.product_id, s.order_date
	FROM sales as s
	LEFT JOIN members as m
	ON s.customer_id = m.customer_id
	WHERE order_date < join_date
),
product as (
	SELECT d.customer_id, d.order_date, m.product_name
	FROM date as d
	LEFT JOIN menu as m
	ON d.product_id = m.product_id
),
ranked as (
	SELECT customer_id, order_date, product_name,
		ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) as rank
	FROM product
)
SELECT *
FROM ranked
WHERE rank = 1

**7. Which item was purchased just before the customer became a member?**

WITH date as(
	SELECT s.customer_id, s.product_id, s.order_date
	FROM sales as s
	LEFT JOIN members as m
	ON s.customer_id = m.customer_id
	WHERE order_date < join_date
),
product as (
	SELECT d.customer_id, d.order_date, m.product_name
	FROM date as d
	LEFT JOIN menu as m
	ON d.product_id = m.product_id
),
ranked as (
	SELECT customer_id, order_date, product_name,
		ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) as rank
	FROM product
)
SELECT *
FROM ranked
WHERE rank = 1

**8. What is the total items and amount spent for each member before they became a member?**

WITH before_join as(
	SELECT s.customer_id, s.product_id, s.order_date
	FROM sales as s
	LEFT JOIN members as m
	ON s.customer_id = m.customer_id
	WHERE order_date < join_date
),
product as (
	SELECT d.customer_id, d.order_date, m.product_name, m.price
	FROM before_join as d
	LEFT JOIN menu as m
	ON d.product_id = m.product_id
)
SELECT customer_id, 
	COUNT(product_name) as total_products,
	SUM(price)
FROM product
GROUP BY customer_id

**9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?**

WITH main as (
	SELECT s.customer_id, m.product_name, SUM(m.price) as total_spent
	FROM sales as s
	LEFT JOIN menu as m
	ON s.product_id = m.product_id 
	GROUP BY customer_id, product_name
	ORDER BY customer_id
)
SELECT customer_id, SUM(CASE WHEN product_name LIKE 'sushi' THEN total_spent * 20 ELSE total_spent * 10 END) as points
FROM main
GROUP BY customer_id

**10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?**

WITH main as (
	SELECT s.customer_id, m.product_name, s.order_date, SUM(m.price) as total_spent
	FROM sales as s
	LEFT JOIN menu as m
	ON s.product_id = m.product_id 
	GROUP BY s.customer_id, m.product_name, s.order_Date
	ORDER BY s.customer_id
),
dates as (
	SELECT sales.customer_id, members.join_Date
	FROM sales
	LEFT JOIN members
	ON sales.customer_id = members.customer_id
	GROUP BY sales.customer_id, members.join_Date
)
SELECT main.customer_id, 
	SUM(CASE WHEN product_name LIKE 'sushi' or (order_date >= join_date AND order_date <= join_date + INTERVAL '7 days') THEN total_spent * 20 ELSE total_spent * 10 END) as points
FROM main 
LEFT JOIN dates
ON main.customer_id = dates.customer_id
WHERE EXTRACT(MONTH FROM order_date) = 1 AND main.customer_id IN ('A','B')
GROUP BY main.customer_id

**11. Bonus Join All The Things**

WITH main as (
	SELECT s.customer_id, m.product_name, s.order_date, m.price
	FROM sales as s
	LEFT JOIN menu as m
	ON s.product_id = m.product_id 
)
SELECT main.customer_id, main.order_date, main.product_name, main.price, (CASE WHEN members.join_date > main.order_date THEN 'N' ELSE 'Y' END) as member
FROM main
FULL OUTER JOIN members
ON main.customer_id = members.customer_id
ORDER BY main.customer_id

**12. Bonus Rank All The Things**

WITH main as (
	SELECT s.customer_id, m.product_name, s.order_date, m.price,
		ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) as order_id
	FROM sales as s
	LEFT JOIN menu as m
	ON s.product_id = m.product_id 
),
joined as (
	SELECT main.customer_id, main.order_date, main.product_name, main.price, main.order_id,
		(CASE WHEN members.join_date > main.order_date THEN 'N' ELSE 'Y' END) as member
	FROM main
	FULL OUTER JOIN members
	ON main.customer_id = members.customer_id
	ORDER BY main.customer_id
),
ranked as (
	SELECT customer_id, order_date, order_id,
		RANK() OVER (PARTITION BY customer_id ORDER BY order_date) as temp_rank
	FROM joined
	WHERE member = 'Y'
)
SELECT joined.customer_id, joined.order_date, joined.product_name, joined.price, joined.member, 
	(CASE WHEN member = 'Y' THEN ranked.temp_rank ELSE NULL END) as rank
FROM joined
LEFT JOIN ranked
ON joined.customer_id = ranked.customer_id
AND joined.order_id = ranked.order_id

