## A. Pizza Metrics
### 1. How many pizzas were ordered?
``` sql
SELECT COUNT(order_id) as total_pizza
FROM pizza_runner.customer_orders;
``` 
| total_pizza |
|-------------|
| 14          |

### 2. How many unique customer orders were made?
``` sql
SELECT COUNT(DISTINCT order_id) AS unique_orders
FROM pizza_runner.customer_orders;
``` 
| unique_orders|
|-------------|
| 14          |

### 3. How many successful orders were delivered by each runner?
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

### 4. How many of each type of pizza was delivered?

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

### 5. How many Vegetarian and Meatlovers were ordered by each customer?
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

### 6. What was the maximum number of pizzas delivered in a single order?
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
|-----------|------|
| 4         | 3    |

### 7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
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

### 8. How many pizzas were delivered that had both exclusions and extras?
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

### 9. What was the total volume of pizzas ordered for each hour of the day?
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

### 10. What was the volume of orders for each day of the week?
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
