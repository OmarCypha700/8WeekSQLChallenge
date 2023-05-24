### 1. What is the total amount each customer spent at the restaurant?
``` sql
SELECT customer_id, 
       SUM(menu.price) AS total_amt_spent
FROM dannys_diner.sales AS sales
JOIN dannys_diner.menu AS menu
USING(product_id)
GROUP BY customer_id
ORDER BY customer_id;
``` 
| customer_id | total_amt_spent |
|-------------|-------|
| A           | 76    |
| B           | 74    |
| C           | 36    |
<br> 

### 2. How many days has each customer visited the restaurant?
``` sql
SELECT customer_id, 
       COUNT(DISTINCT order_date) AS num_days_visited
FROM dannys_diner.sales
GROUP BY customer_id
ORDER BY customer_id;
```
| customer_id | num_days_visited |
|-------------|--------|
| A           | 4      |
| B           | 6      |
| C           | 2      |
<br>

### 3. What was the first item from the menu purchased by each customer?
```sql 
SELECT sub.customer_id, 
       menu.product_name AS first_item_purchased
FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS 'row'
  FROM dannys_diner.sales
) sub
JOIN dannys_diner.menu AS menu 
USING(product_id)
WHERE sub.row = 1
GROUP BY sub.customer_id, menu.product_name
ORDER BY sub.customer_id;
```
| customer_id | product_name |
|-------------|--------------|
| A           | sushi        |
| B           | curry        |
| C           | ramen        |
<br>

### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
``` sql
SELECT sub.product_name, total_buys
FROM (SELECT menu.product_name AS product_name, 
             COUNT(sales.product_id) total_buys,
	     ROW_NUMBER() OVER (ORDER BY sales.product_id) AS 'row'
FROM dannys_diner.sales AS sales
INNER JOIN dannys_diner.menu AS menu
USING(product_id)
GROUP BY menu.product_name, sales.product_id) sub
WHERE sub.row = 3;
```
| product_name | total_buys |
|--------------|------------|
| ramen        | 8          |
<br>

### 5. Which item was the most popular for each customer?
``` sql 
SELECT sub.customer_id, 
       sub.product_name, 
       sub.times_purchased
FROM (
  SELECT sales.customer_id AS customer_id, 
  	 menu.product_name AS product_name, 
	 COUNT(*) AS times_purchased,
         RANK() OVER (PARTITION BY sales.customer_id ORDER BY COUNT(*) DESC) AS 'rank'
  FROM dannys_diner.sales AS sales
  JOIN dannys_diner.menu AS menu
  USING (product_id)
  GROUP BY sales.customer_id,  menu.product_name
) sub
WHERE sub.rank < 2;
``` 
| customer_id | product_name | times_purchased |
|-------------|--------------|---------------|
| A           | ramen        | 3             |
| B           | curry        | 2             |
| B           | sushi        | 2             |
| B           | ramen        | 2             |
| C           | ramen        | 3             |
<br>

### 6. Which item was purchased first by the customer after they became a member?
``` sql 
SELECT sub.customer, 
       sub.product_name,
       sub.order_date,
       sub.join_date
FROM (SELECT sales.customer_id AS customer, 
	     menu.product_name AS product_name, 
             sales.order_date AS order_date, 
             member.join_date AS join_date,
        ROW_NUMBER() OVER (PARTITION BY sales.customer_id ORDER BY sales.order_date) AS 'row'
        FROM dannys_diner.members AS member
        INNER JOIN dannys_diner.sales AS sales
        ON member.customer_id = sales.customer_id
        INNER JOIN dannys_diner.menu AS menu
        ON sales.product_id = menu.product_id
        WHERE sales.order_date >= member.join_date) sub
WHERE sub.row = 1;
```
| customer | product_name | order_date | join_date  |
|----------|--------------|------------|------------|
| A        | curry        | 2021-01-07 | 2021-01-07 |
| B        | sushi        | 2021-01-11 | 2021-01-09 |
<br>

### 7. Which item was purchased just before the customer became a member?
``` sql 
SELECT sub.customer, 
       sub.product, 
       sub.order_date, 
       sub.join_date
FROM (SELECT sales.customer_id AS customer, 
	     menu.product_name AS product, 
             sales.order_date AS order_date, 
             member.join_date AS join_date,
        ROW_NUMBER() OVER (PARTITION BY sales.customer_id ORDER BY sales.order_date DESC) AS 'row'
        FROM dannys_diner.members member
        INNER JOIN dannys_diner.sales AS sales
        ON member.customer_id = sales.customer_id
        INNER JOIN dannys_diner.menu AS menu
        ON sales.product_id = menu.product_id
        WHERE sales.order_date < member.join_date) sub
WHERE sub.row = 1;
```
| customer | product | order_date | join_date  |
|----------|---------|------------|------------|
| A        | sushi   | 2021-01-01 | 2021-01-07 |
| B        | sushi   | 2021-01-04 | 2021-01-09 |
<br>

### 8. What is the total items and amount spent for each member before they became a member?
``` sql 
SELECT sales.customer_id AS customer, 
       COUNT(sales.product_id) AS total_items_purchased, 
       SUM(menu.price) AS amt_spent
FROM dannys_diner.sales AS sales
INNER JOIN dannys_diner.menu AS menu
USING(product_id)
INNER JOIN dannys_diner.members AS member
ON sales.customer_id = member.customer_id
WHERE sales.order_date < member.join_date
GROUP BY customer
ORDER BY customer;
```
| customer | total_items_bought | amt_spent |
|----------|--------------------|-----------|
| A        | 2                  | 25        |
| B        | 3                  | 40        |
<br>

### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
``` sql 
SELECT sales.customer_id AS customer_id,
       SUM(CASE WHEN menu.product_name = 'sushi' THEN menu.price * 20 
     	   ELSE menu.price * 10 END) total_points
FROM dannys_diner.sales AS sales
INNER JOIN dannys_diner.menu AS menu
ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
```
| customer_id | total_points |
|-------------|--------------|
| A           | 860          |
| B           | 940          |
| C           | 360          |

### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January? 
``` sql 
SELECT *
FROM (
    -- Points in the first week after customer joins the program
    SELECT sales.customer_id AS customer_id, 
           SUM(menu.price * 20) AS points
    FROM dannys_diner.sales AS sales
    INNER JOIN dannys_diner.members AS member USING (customer_id)
    INNER JOIN dannys_diner.menu AS menu USING (product_id)
    WHERE sales.order_date >= member.join_date 
          AND sales.order_date <= DATE_ADD(member.join_date, INTERVAL 1 WEEK)
    GROUP BY customer_id
    ORDER BY customer_id
) AS first_week

-- Stacking on top of points in other weeks
UNION ALL 

SELECT *
FROM (
    -- Points in any other week in January
    SELECT sales.customer_id AS customer_id,
           SUM(CASE WHEN menu.product_name = 'sushi' THEN menu.price * 20
                    ELSE menu.price * 10 END) AS points
    FROM dannys_diner.sales AS sales
    INNER JOIN dannys_diner.members AS member USING (customer_id)
    INNER JOIN dannys_diner.menu AS menu USING (product_id)
    WHERE sales.order_date < member.join_date 
          AND sales.order_date > DATE_ADD(member.join_date, INTERVAL 1 WEEK)
          AND sales.order_date BETWEEN '2021-01-01' AND '2021-01-31'
    GROUP BY customer_id
    ORDER BY customer_id
) AS other_weeks;
```
| customer_id | points |
|-------------|--------|
| A           | 1020   |
| B           | 440    |
<br>
