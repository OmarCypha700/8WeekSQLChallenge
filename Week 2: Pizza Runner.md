# Week 2: Pizza Runner 🍕
## Table of Contents
- [Data Cleaning](#data-cleaning)
- [Pizza Metrics](#a-pizza-metrics)
- [Runner and Customer Experience](#b-runner-and-customer-experience)
- [Ingredient Optimization](#c-ingredient-optimization)
- [Pricing and Rates](#d-pricing-and-rates)

## DATA CLEANING
```sql
-- ------------------------
--  CUSTOMER_ORDERS TABLE
-- ------------------------
-- Handling null values and empty strings.
UPDATE customer_orders
SET
  exclusions = NULLIF(NULLIF(exclusions, ''), 'null'),
  extras = NULLIF(NULLIF(extras, ''), 'null');
  
 -- -------------------------------------------------------------------------------------------------------------------------------
 -- Creating a row number to retain each row number since spliting the extras and exclusions into rows will result in duplicate rows
 -- -------------------------------------------------------------------------------------------------------------------------------
  
DROP TEMPORARY TABLE IF EXISTS new_customer_orders;

CREATE TEMPORARY TABLE new_customer_orders AS
SELECT *,
	ROW_NUMBER() OVER() AS 'row_number'
FROM customer_orders;

SELECT *
FROM new_customer_orders;

-- -------------------------------------------------
-- Create a TEMP TABLE for 'customer_orders_cleaned'
-- And spliting the comma seperated values in the extras and exclusions column into rows
-- ------------------------------------------------

CREATE TEMPORARY TABLE customer_orders_cleaned
SELECT 
  nco.row_number,
  order_id,
  customer_id,
  pizza_id,
  CAST(TRIM(SUBSTRING_INDEX(SUBSTRING_INDEX(nco.extras, ',', n1.n), ',', -1)) AS UNSIGNED) AS extras,
  CAST(TRIM(SUBSTRING_INDEX(SUBSTRING_INDEX(co.exclusions, ',', n2.n), ',', -1)) AS UNSIGNED) AS exclusions,
  order_time
FROM
  pizza_runner.new_customer_orders nco
LEFT JOIN
  (SELECT 1 AS n UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5) n1
ON
  n1.n <= LENGTH(nco.extras) - LENGTH(REPLACE(nco.extras, ',', '')) + 1
LEFT JOIN
  (SELECT 1 AS n UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 UNION ALL SELECT 5) n2
ON
  n2.n <= LENGTH(nco.exclusions) - LENGTH(REPLACE(nco.exclusions, ',', '')) + 1;

SELECT *
FROM customer_orders_cleaned;

-- ----------------------
-- RUNNER_ORDERS TABLE
-- ----------------------

-- ----------------------------------------
-- Handlling null values and empty strings
-- ----------------------------------------

UPDATE runner_orders
SET
  cancellation = NULLIF(NULLIF(cancellation, ''), 'null'),
  distance = NULLIF(NULLIF(distance, ''), 'null'),
  duration = NULLIF(NULLIF(duration, ''), 'null');

-- ---------------------------------------
-- Cleaning distance and duration columns
-- ---------------------------------------

UPDATE runner_orders
SET
  distance = REPLACE(distance, 'km', ''),
  duration = SUBSTR(duration, 1, 2);

-- --------------------------------------------
-- RENAMING DISTANCE AND DURATION COLUMN NAMES
-- --------------------------------------------

ALTER TABLE runner_orders
RENAME COLUMN distance TO distance_km;

ALTER TABLE runner_orders
RENAME COLUMN duration TO duration_min;

-- --------------------
-- PIZZA RECIPES TABLE
-- --------------------

-- --------------------------------------------------------------------------
-- Handlling commas in the toppings column into rows and store in a temporary column
-- -------------------------------------------------------------------------
DROP TEMPORARY TABLE IF EXISTS pizza_recipes_clean;

CREATE TEMPORARY TABLE pizza_recipes_clean as (
SELECT pr.pizza_id,
       REPLACE(SUBSTRING_INDEX(SUBSTRING_INDEX(pr.toppings, ',', numbers.n), ',', -1), ' ', '') AS toppings
FROM (select 1 AS n UNION ALL
		select 2 UNION ALL
        select 3 UNION ALL
        select 4 UNION ALL
        select 5 UNION ALL
        select 6 UNION ALL
        select 7 UNION ALL
        select 8 ) AS numbers
INNER JOIN pizza_recipes pr 
ON CHAR_LENGTH(pr.toppings) - CHAR_LENGTH(REPLACE(pr.toppings, ',', '')) >= numbers.n-1
ORDER BY pr.pizza_id, n);

SELECT *
FROM pizza_recipes_clean;
```

## A. Pizza Metrics
##### 1. How many pizzas were ordered?
``` sql
SELECT COUNT(order_id) as total_pizza
FROM pizza_runner.customer_orders;
``` 
| total_pizza |
|-------------|
| 14          |

##### 2. How many unique customer orders were made?
``` sql
SELECT COUNT(DISTINCT order_id) AS unique_orders
FROM pizza_runner.customer_orders;
``` 
| unique_orders|
|-------------|
| 14          |

##### 3. How many successful orders were delivered by each runner?
``` sql
SELECT runner_id, COUNT(order_id) AS orders_delivered
FROM pizza_runner.runner_orders
WHERE cancellation IS NULL
GROUP BY runner_id;
``` 
| runner_id | orders_delivered |
|-----------|------|
| 1         | 4    |
| 2         | 3    |
| 3         | 1    |

##### 4. How many of each type of pizza was delivered?

```sql
SELECT pizza_id, COUNT(pizza_id) AS num_delivered
FROM pizza_runner.customer_orders AS co
JOIN pizza_runner.runner_orders AS ro USING (order_id)
WHERE cancellation IS NULL
GROUP BY pizza_id;
```
| pizza_id | num_delivered |
|-----------|------|
| 1         | 9    |
| 2         | 3    |

##### 5. How many Vegetarian and Meatlovers were ordered by each customer?
```sql
SELECT co.customer_id, 
       COUNT(CASE WHEN pn.pizza_name = 'Vegetarian' THEN 1 ELSE NULL END) AS vegetarian,
       COUNT(CASE WHEN pn.pizza_name = 'Meatlovers' THEN 1 ELSE NULL END) AS meatlovers
FROM pizza_runner.customer_orders co
JOIN pizza_runner.pizza_names pn 
USING (pizza_id)
GROUP BY co.customer_id;
```
 customer_id     | vegetarian   |  meatlovers
:---------------:|:-------------|:------------:
 101             | 1            | 2 
 102             | 1            | 2
 103             | 1            | 3
 104             | 0            | 3
 105             | 1            | 0

##### 6. What was the maximum number of pizzas delivered in a single order?
```sql
SELECT co.order_id, COUNT(pizza_id) as pizza_delivered
FROM pizza_runner.customer_orders co
JOIN pizza_runner.runner_orders ro
USING (order_id)
WHERE ro.cancellation IS NULL
GROUP BY co.order_id
ORDER BY pizza_delivered DESC
LIMIT 1;
```
| order_id | pizza_delivered |
|----------|------|
| 4        | 3    |

##### 7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
```sql
SELECT co.customer_id, 
	  COUNT(CASE WHEN (co.exclusions IS NOT NULL OR co.extras IS NOT NULL) THEN pizza_id END) AS changed_orders,
	  COUNT(CASE WHEN co.exclusions IS NULL AND co.extras IS NULL THEN pizza_id END) AS unchanged_orders
FROM pizza_runner.customer_orders co
JOIN pizza_runner.runner_orders ro
USING (order_id)
WHERE ro.cancellation IS NULL
GROUP BY co.customer_id;
```
  customer_id   |  changed_orders   | unchanged_orders
 :-------------:|:-----------------:|:----------------:
  101           | 0                 | 2
  102           | 0                 | 3
  103           | 3                 | 0
  104           | 2                 | 1
  105           | 1                 | 0

##### 8. How many pizzas were delivered that had both exclusions and extras?
```sql
SELECT co.customer_id, COUNT(pizza_id) count
FROM pizza_runner.customer_orders co
JOIN pizza_runner.runner_orders ro
USING (order_id)
WHERE ro.cancellation IS NULL
AND co.exclusions IS NOT NULL 
AND co.extras IS NOT NULL
GROUP BY co.customer_id;
```
| customer_id | count |
|-------------|-------|
| 104         | 1     |

##### 9. What was the total volume of pizzas ordered for each hour of the day?
``` sql
SELECT EXTRACT(hour FROM order_time) AS hour, 
	   COUNT(pizza_id) AS pizza_ordered
FROM pizza_runner.customer_orders
GROUP BY hour
ORDER BY hour;
```
 hour     | pizza_ordred
:--------:|:-----------------:
 11       | 1
 13       | 3
 18       | 3
 19       | 1
 21       | 3
 23       | 3

##### 10. What was the volume of orders for each day of the week?
``` sql
SELECT DAYNAME(order_time) AS day_of_week, 
  COUNT(pizza_id) AS pizza_ordered
FROM pizza_runner.customer_orders
GROUP BY day_of_week;
```
 day_of_week    | dow_order_volume
:--------------:|:------------------:
 Wednesday      | 5
 Thursday       | 3
 Saturday       | 5
 Friday         | 1

## B. Runner and Customer Experience

##### 1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)
```sql
SELECT EXTRACT(week FROM registration_date + INTERVAL 3 day) AS week, 
	   MIN(registration_date) AS start_week,
	   COUNT(*) AS sign_ups
FROM pizza_runner.runners
GROUP BY week
ORDER BY week;
```
 week         | start_week     | sign_ups
:------------:|:--------------:|------------
 1            | 2021-01-01     | 2
 2            | 2021-01-08     | 1
 3            | 2021-01-15     | 1
 
##### 2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
```sql
SELECT ro.runner_id, 
	   CAST(AVG(TIMEDIFF(ro.pickup_time, co.order_time)) AS TIME) AS avg_time
FROM pizza_runner.runner_orders AS ro
JOIN pizza_runner.customer_orders AS co
USING (order_id)
GROUP BY ro.runner_id;
```
runner_id   |	avg_time
------------|------------
1	    |	00:15:54
2	    |	00:23:59
3	    |	00:10:28

##### 3. Is there any relationship between the number of pizzas and how long the order takes to prepare?
```sql
SELECT num_pizzas, CAST(AVG(prep_time) AS TIME)AS avg_prep_time
FROM (
	SELECT co.order_id, COUNT(co.pizza_id) as num_pizzas, 
		   CAST(TIMEDIFF(ro.pickup_time, co.order_time) AS TIME) AS prep_time
	FROM pizza_runner.runner_orders ro
	JOIN pizza_runner.customer_orders co
	USING (order_id)
	WHERE ro.pickup_time IS NOT NULL
	GROUP BY co.order_id, ro.pickup_time, co.order_time
) AS sub
GROUP BY num_pizzas;
```
runner_id   |	avg_time
------------|------------
1	    |	00:12:21
2	    |	00:18:23
3	    |	'00:29:17'

Looking at the average time taken and the number of pizzas prepared within that time, there seem to be a relationship between the number of pizzas and how long it takes to prepare.

##### 4. What was the average distance travelled for each customer?
```sql
SELECT co.customer_id, CEIL(AVG(distance_km)) AS avg_distance_km
FROM pizza_runner.runner_orders AS ro
JOIN pizza_runner.customer_orders AS co
USING (order_id)
GROUP BY co.customer_id;
```
 customer_id  |	avg_distance_km
------------- |-----------------
101	      | 20
102	      |	17
103	      |	24
104	      |	10
105	      | 25

##### 5. What was the difference between the longest and shortest delivery times for all orders?
```sql
SELECT (MAX(duration_min) - MIN(duration_min)) AS diff
FROM pizza_runner.runner_orders;
```
| diff |
|------|
| 30   |

##### 6. What was the average speed for each runner for each delivery and do you notice any trend for these values?
```sql
SELECT runner_id, 
	   CEIL(AVG(distance_km / (duration_min/60))) AS 'avg_speed_km/hr'
FROM pizza_runner.runner_orders
GROUP BY runner_id;
```
runner_id  |	avg_speed_km/hr
-----------|------------------
1	   |	46
2	   |	63
3	   |	40

##### 7. What is the successful delivery percentage for each runner?
```sql
SELECT runner_id,
       COUNT(*) AS total_deliveries,
       SUM(CASE WHEN cancellation IS NULL THEN 1 ELSE 0 END) AS successful_deliveries,
       (SUM(CASE WHEN cancellation IS NULL THEN 1 ELSE 0 END) / COUNT(*)) * 100 AS success_percentage
FROM pizza_runner.runner_orders
GROUP BY runner_id;
```
runner_id   |	total_deliveries     |	successful_deliveries | success_percentage
------------|------------------------|------------------------|--------------------
1	    |	     4		     |	      4		      |	   100.0000
2	    |	     4		     |	      3		      |    75.0000
3	    |	     2		     |	      1		      |    50.0000


## C. INGREDIENT OPTIMIZATION

##### 1. What are the standard ingredients for each pizza?
```sql
SELECT prc.pizza_id,
		GROUP_CONCAT(pt.topping_name SEPARATOR ', ') AS toppings
FROM pizza_recipes_clean AS prc
JOIN pizza_runner.pizza_toppings AS pt
ON prc.toppings = pt.topping_id
GROUP BY prc.pizza_id;
```
pizza_id    |	toppings
------------|------------
1	    | Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami
2	    | Cheese, Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce

##### 2. What was the most commonly added extra?
```sql
SELECT coc.extras,
       pt.topping_name,
       COUNT(DISTINCT coc.row_number) AS count
FROM customer_orders_cleaned AS coc
JOIN pizza_toppings AS pt
ON coc.extras = pt.topping_id
GROUP BY extras, topping_name
ORDER BY count DESC
LIMIT 1;
```
extras | topping_name | count
-------|--------------|------
1      | Bacon	      |	4

##### 3. What was the most common exclusion?
```sql
SELECT coc.exclusions,
       pt.topping_name,
       COUNT(DISTINCT coc.row_number) AS count
FROM customer_orders_cleaned AS coc
JOIN pizza_toppings AS pt
ON coc.exclusions = pt.topping_id
GROUP BY exclusions, topping_name
ORDER BY count DESC
LIMIT 1;
```
exclusions | topping_name | count
-----------|--------------|------
4          | Cheese	  | 4

##### 4. Generate an order item for each record in the customers_orders table in the format of one of the following:
- Meat Lovers
- Meat Lovers - Exclude Beef
- Meat Lovers - Extra Bacon
- Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers

```sql
-- --------------------------------------------------------------------------------------------
-- Create a temporary table to select all extras in a row with their row number as 'row_number'
-- --------------------------------------------------------------------------------------------

DROP TEMPORARY TABLE IF EXISTS extras_seperated;

CREATE TEMPORARY TABLE extras_seperated AS
SELECT 
  nco.row_number,
  CAST(TRIM(SUBSTRING_INDEX(SUBSTRING_INDEX(nco.extras, ',', n.n), ',', -1)) AS UNSIGNED) AS extra_id
FROM pizza_runner.new_customer_orders nco
CROSS JOIN
  (SELECT 1 AS n UNION ALL SELECT 2 UNION ALL SELECT 3 ) n
WHERE
  n.n <= LENGTH(nco.extras) - LENGTH(REPLACE(nco.extras, ',', '')) + 1;

-- --------------------------------------------------------------------------------------------
-- Create a temporary table to select all exclusions in a row with their row number as 'row_number'
-- --------------------------------------------------------------------------------------------

DROP TEMPORARY TABLE IF EXISTS exclusions_seperated;

CREATE TEMPORARY TABLE exclusions_seperated AS
SELECT 
  nco.row_number,
  CAST(TRIM(SUBSTRING_INDEX(SUBSTRING_INDEX(nco.exclusions, ',', n.n), ',', -1)) AS UNSIGNED) AS exclusions_id
FROM pizza_runner.new_customer_orders nco
CROSS JOIN
  (SELECT 1 AS n UNION ALL SELECT 2 UNION ALL SELECT 3 ) n
WHERE
  n.n <= LENGTH(nco.exclusions) - LENGTH(REPLACE(nco.exclusions, ',', '')) + 1;  

-- --------------------------------------------------------------------------------------
-- Create a Common Table Expression (CTE) to get the extras and the exclusions, 
-- combine them and join to the 'new_customer_orders' TEMP TABLE and 'pizza_names' table  
-- --------------------------------------------------------------------------------------

WITH extras_CTE AS (
-- Get all extras, use 'GROUP_CONCAT' to arrange them horizontaly seperated by ',' and 'CONCAT' them with 'Extra'
    SELECT
        es.row_number,
        CONCAT('Extra ', GROUP_CONCAT(pt.topping_name SEPARATOR ', ')) AS options
    FROM extras_seperated AS es
    JOIN pizza_runner.pizza_toppings AS pt ON es.extra_id = pt.topping_id
    GROUP BY es.row_number
),
exclusions_CTE AS (
-- Get all exclusions, use 'GROUP_CONCAT' to arrange them horizontaly seperated by ',' and 'CONCAT' them with 'Exclude'
    SELECT
        exs.row_number,
        CONCAT('Exclude ', GROUP_CONCAT(pt.topping_name SEPARATOR ', ')) AS options
    FROM exclusions_seperated AS exs
    JOIN pizza_runner.pizza_toppings AS pt ON exs.exclusions_id = pt.topping_id
    GROUP BY exs.row_number
),
-- Combine them with 'UNION' in the 'Union_CTE'
Union_CTE AS (
    SELECT * FROM extras_CTE
    UNION
    SELECT * FROM exclusions_CTE
)
-- Now select needed columns and concatinate the extras and exclusion of each order in one column 'pizza_information'
-- join with Union_CTE and pizza_names to get the pizza names
SELECT
  nco.row_number,
  nco.order_id,
  nco.customer_id,
  nco.pizza_id,
  nco.order_time,
  CONCAT_WS(' - ', pn.pizza_name, GROUP_CONCAT(u.options SEPARATOR ' - ')) AS pizza_information
FROM
  pizza_runner.new_customer_orders nco
LEFT JOIN Union_CTE u ON nco.row_number = u.row_number
JOIN
  pizza_runner.pizza_names pn USING (pizza_id)
GROUP BY
  nco.row_number,
  nco.order_id,
  nco.customer_id,
  nco.pizza_id,
  nco.order_time,
  pn.pizza_name
ORDER BY
  nco.order_id,
  nco.row_number;
```
#### Results
![SC_Q4](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/a56db7f4-4ff9-4ff8-9e7f-a2a5aabcbb91)

##### 6. What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?

```sql
-- ----------------------------------------------------------------------------------------------------------------------------------
-- Create a temporary table 'new_recipe' that shows pizza_id, topping_id and topping_name from pizza_recipe and pizza_toppings table
-- ----------------------------------------------------------------------------------------------------------------------------------

DROP TEMPORARY TABLE IF EXISTS new_recipe;
CREATE TEMPORARY TABLE new_recipe (
SELECT sub.*, topping_name
FROM (
SELECT 
    pr.pizza_id,
    CAST(TRIM(SUBSTRING_INDEX(SUBSTRING_INDEX(pr.toppings, ',', n.n), ',', -1)) AS UNSIGNED) AS topping_id
FROM 
    pizza_runner.pizza_recipes pr
CROSS JOIN 
    (
        SELECT 1 AS n UNION ALL
        SELECT 2 UNION ALL
        SELECT 3 UNION ALL
        SELECT 4 UNION ALL
        SELECT 5 UNION ALL
        SELECT 6 
    ) n
WHERE 
    n.n <= LENGTH(pr.toppings) - LENGTH(REPLACE(pr.toppings, ',', '')) + 1
ORDER BY pizza_id) sub
JOIN pizza_runner.pizza_toppings AS pt
ON sub.topping_id = pt.topping_id
);


WITH total_ingredients AS (
  SELECT 
    nco.row_number,
    nr.topping_name,
    CASE
      -- if extra ingredient, add 2
      WHEN nr.topping_id IN (
          SELECT extra_id 
          FROM extras_seperated es
          WHERE es.row_number = nco.row_number) 
      THEN 2
      -- if excluded ingredient, add 0
      WHEN nr.topping_id IN (
          SELECT exclusions_id 
          FROM exclusions_seperated exs 
          WHERE nco.row_number = exs.row_number)
      THEN 0
      -- no extras, no exclusions, add 1
      ELSE 1
    END AS quantity
  FROM pizza_runner.new_customer_orders nco
  JOIN new_recipe nr
    ON nr.pizza_id = nco.pizza_id
  JOIN pizza_runner.pizza_names pn
    ON pn.pizza_id = nco.pizza_id
)
SELECT 
  topping_name,
  SUM(quantity) AS quantity_used 
FROM total_ingredients
GROUP BY topping_name
ORDER BY quantity_used DESC;
```
 topping_name  | quantity_used
---------------|--------------
  Mushrooms    | 13
  Bacon        | 13
  Chicken      | 11
  Cheese       | 11
  Pepperoni    | 10
  Salami       | 10
  Beef         | 10
  BBQ Sauce    | 9 
  Tomatoes     | 4
  Onions       | 4
  Peppers      | 4
  Tomato Sauce | 4

## D. PRICING AND RATES

##### 1. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?
```sql
SELECT SUM(CASE WHEN pizza_id = 1 THEN 12
		WHEN pizza_id = 2 THEN 10
           END) AS money_made
FROM pizza_runner.customer_orders
JOIN pizza_runner.runner_orders USING (order_id)
WHERE cancellation IS NULL;
```
 money_made
:---------:
  138

##### 2. What if there was an additional $1 charge for any pizza extras?
- Add cheese is $1 extra
```sql
WITH price1_CTE AS (
	SELECT SUM(CASE WHEN pizza_id = 1 THEN 12 ELSE 10 END) AS 'money_made'
	FROM pizza_runner.customer_orders
	JOIN pizza_runner.runner_orders USING (order_id)
	WHERE cancellation IS NULL
),
price2_CTE AS (
	SELECT SUM(CASE WHEN extras = 4 THEN 2 ELSE 1 END) AS 'extras_charged'
	FROM pizza_runner.customer_orders_cleaned
	JOIN pizza_runner.runner_orders USING (order_id)
	WHERE cancellation IS NULL
    AND extras IS NOT NULL
)
SELECT (money_made + extras_charged) AS adjusted_money_made
FROM price1_CTE, price2_CTE;
```
 adjusted_money_made
:-------------------:
  146

##### 3. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.
```sql
DROP TABLE IF EXISTS runner_ratings;
CREATE TABLE pizza_runner.runner_ratings (
  order_id INT,
  rating INT
);
INSERT INTO pizza_runner.runner_ratings (order_id, rating)
VALUES 
  (1,5),
  (2,3),
  (3,2),
  (4,4),
  (5,2),
  (7,3),
  (8,4),
  (10,5);

SELECT * FROM pizza_runner.runner_ratings;
```
order_id    |	rating
------------|---------
1           |	5
2	    |   3
3	    |   2
4	    |   4
5	    |   2
7	    |   3
8	    |   4
10	    |   5

##### 4. Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?
- customer_id
- order_id
- runner_id
- rating
- order_time
- pickup_time
- Time between order and pickup
- Delivery duration
- Average speed
- Total number of pizzas

```sql
SELECT 
      co.customer_id,
      co.order_id,
      ro.runner_id,
      rr.rating,
      co.order_time,
      ro.pickup_time,
      CAST(TIMEDIFF(ro.pickup_time, co.order_time) AS TIME) AS 'time_btwn_order_&_pickup',
      ro.duration_min,
      CEIL(AVG(distance_km / (duration_min/60))) AS 'avg_speed_km/hr',
      COUNT(pizza_id) AS num_pizzas
FROM pizza_runner.customer_orders AS co
JOIN pizza_runner.runner_orders AS ro USING (order_id)
JOIN pizza_runner.runner_ratings AS rr USING (order_id)
GROUP BY 
  co.customer_id,
  co.order_id,
  ro.runner_id,
  rr.rating,
  co.order_time,
  ro.pickup_time, 
  ro.duration_min;
```
![SD_Q4](https://github.com/OmarCypha700/8WeekSQLChallenge/assets/98944012/ecc0cf78-d0c8-4ab2-bab4-e6a2c1c33605)

##### 5. If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled 
- how much money does Pizza Runner have left over after these deliveries?

```sql
WITH income_CTE AS (
	SELECT SUM(CASE WHEN c.pizza_id = 1 THEN 12 ELSE 10 END) AS money_made
	 FROM pizza_runner.customer_orders c
	 INNER JOIN pizza_runner.pizza_names pn
	 ON c.pizza_id = pn.pizza_id
	 INNER JOIN pizza_runner.runner_orders r
	 ON c.order_id = r.order_id
	 WHERE r.cancellation IS NULL
     ),
expense_CTE AS(
SELECT SUM((distance_km * 0.30)) AS total_driver_fee
FROM pizza_runner.runner_orders
WHERE distance_km IS NOT NULL
)
SELECT ROUND(money_made - total_driver_fee, 2) AS leftover_amt
FROM income_CTE, expense_CTE;
```
 leftover_amt
:--------------:
  94.44
